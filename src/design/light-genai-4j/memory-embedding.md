# Memory and Embedding Design

## Overview

This document outlines the design decisions for memory management, embedding models, and storage architecture for the light-genai-4j agent system.

For agent skills and tool execution, see [Agent Skill Design](agent-skill.md).

## 1. Embedding Model Selection

### 1.1 Chosen Model: BGE-small-en-v1.5 (Quantized)

**Artifact**: `bge-small-en-v15-q` (migrated from langchain4j)

**Specifications**:
- **Dimensions**: 384
- **Pooling Mode**: CLS (uses [CLS] token representation)
- **Model Size**: ~17MB (quantized version)
- **Language**: English (primary), supports basic multilingual capability
- **Source**: BAAI (Beijing Academy of Artificial Intelligence)

### 1.2 Why BGE-small-en-v1.5?

| Use Case | Requirement | How BGE Fits |
|----------|-------------|--------------|
| **Short-term Memory** | Fast retrieval of recent context | Optimized for semantic similarity search |
| **Long-term Memory** | Finding relevant past conversations | State-of-the-art for asymmetric retrieval (query → document) |
| **Knowledge Base** | RAG (Retrieval-Augmented Generation) | Best-in-class for information retrieval |

### 1.3 Comparison with Alternatives

| Model | Dimensions | Best For | Why Not Chosen |
|-------|------------|----------|----------------|
| all-MiniLM-L6-v2 | 384 | General similarity | Not optimized for retrieval tasks |
| E5-small-v2 | 384 | Asymmetric search | Query/passage prefixes more complex |
| BGE-small-en | 384 | English retrieval | v1.5 has better performance |
| BGE-small-zh-v1.5 | 512 | Chinese | Different dimensions (see Section 4) |

### 1.4 Usage Pattern

```java
// For queries (retrieving memories/knowledge)
String queryPrefix = "Represent this sentence for searching relevant passages:";
String query = queryPrefix + " " + userMessage;

// For storing (memories, knowledge documents)
String document = conversationText; // No prefix needed
```

## 2. Vector Database: PostgreSQL with pgvector

### 2.1 Schema Design

The schema adopts a **scope-based memory model** (similar to [mem0](https://github.com/mem0ai/mem0)), organizing memory by lifetime and visibility rather than abstract types.

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Session Memory (Short-term)
-- Stores in-flight conversation context. Auto-expires via TTL.
CREATE TABLE session_memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL,
    agent_id UUID NOT NULL,
    user_id UUID, -- NULL allowed for anonymous sessions or background system tasks
    content TEXT NOT NULL,
    embedding VECTOR(384),
    importance_score FLOAT DEFAULT 1.0,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '1 hour', -- TTL
    metadata JSONB -- Rich filtering (e.g., {"topic": "debug", "turn": 5})
);

-- User Memory (Long-term)
-- Stores persistent facts/preferences about a user. Manual or inferred.
CREATE TABLE user_memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL,
    user_id UUID NOT NULL, -- Must be tied to a specific user
    content TEXT NOT NULL, -- e.g., "User prefers Java over Python"
    embedding VECTOR(384),
    memory_type VARCHAR(50), -- 'fact', 'preference', 'summary'
    importance_score FLOAT DEFAULT 1.0,
    access_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed TIMESTAMP DEFAULT NOW(),
    metadata JSONB -- e.g., {"confidence": 0.9, "source": "conversation_123"}
);

-- Agent Memory (Private/Operational)
-- Stores agent-specific learning, state, or persistent persona data.
-- Scope: Private to the agent, typically across multiple users or sessions.
CREATE TABLE agent_memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL,
    -- NO user_id: This is agent-centric knowledge
    content TEXT NOT NULL,
    embedding VECTOR(384),
    memory_type VARCHAR(50), -- 'learning', 'state', 'persona', 'scratchpad'
    created_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB
);

-- Organizational Memory (Knowledge Base)
-- Stores global, shared knowledge available to all agents/users.
CREATE TABLE organizational_memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL,
    source VARCHAR(255), -- Document source/name
    content TEXT NOT NULL,
    embedding VECTOR(384),
    chunk_index INTEGER, -- For large documents split into chunks
    document_id UUID, -- Reference to parent document
    created_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB -- e.g., {"department": "HR", "version": "1.0"}
);

-- Indexes for similarity search and metadata filtering
CREATE INDEX idx_session_memory_embedding ON session_memories 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_session_metadata ON session_memories USING GIN (metadata);

CREATE INDEX idx_user_memory_embedding ON user_memories 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_user_metadata ON user_memories USING GIN (metadata);

CREATE INDEX idx_agent_memory_embedding ON agent_memories 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_agent_metadata ON agent_memories USING GIN (metadata);

CREATE INDEX idx_org_memory_embedding ON organizational_memories 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_org_metadata ON organizational_memories USING GIN (metadata);
```

### 2.2 Memory Lifecycle

The system follows a promotion strategy:

```
User Input
    │
    ▼
┌─────────────────────────────────────┐
│  Query Embedding (with prefix)      │
└─────────────────────────────────────┘
    │
    ├──► Search Session Memory (Context)
    │     └── Recent turns, immediate task context
    │
    ├──► Search User Memory (Personalization)
    │     └── User preferences, past decisions, facts
    │
    ├──► Search Agent Memory (Self-Knowledge)
    │     └── Agent persona, learned behaviors, operational state
    │
    └──► Search Organizational Memory (Knowledge)
          └── Policies, docs, FAQs
    │
    ▼
Consolidated Context → LLM
    │
    ▼
Store Response ──────► Session Memory
    │                         │
    │                         ▼
    │              (Optional: Extraction/Inference)
    │              Run LLM to extract facts from conversation
    │                         │
    │          ┌──────────────┴──────────────┐
    │          ▼                             ▼
    │   User Memory                   Agent Memory
    │   (User Facts)                  (Agent Learnings)
    │
    ▼
    (Session ends)
    Session Memory Expires
```

### 2.3 Memory Retrieval Strategy

```java
public class MemoryService {
    
    public List<Memory> retrieveRelevantMemories(String query, UUID sessionId, UUID userId, UUID agentId) {
        // 1. Embed the query with prefix
        String prefixedQuery = "Represent this sentence for searching relevant passages: " + query;
        Embedding queryEmbedding = embeddingModel.embed(prefixedQuery);
        
        // 2. Search Session Memory (Short-term context)
        List<Memory> sessionContext = searchSession(queryEmbedding, sessionId, limit = 10);
        
        // 3. Search User Memory (Personalization)
        List<Memory> userFacts = searchUser(queryEmbedding, userId, limit = 5);
        
        // 4. Search Agent Memory (Agent Self-Knowledge)
        List<Memory> agentFacts = searchAgent(queryEmbedding, agentId, limit = 3);
        
        // 5. Search Organizational Memory (Knowledge Base)
        List<Memory> orgKnowledge = searchOrg(queryEmbedding, agentId, limit = 5);
        
        // 6. Combine and Rerank
        return combineAndRank(sessionContext, userFacts, agentFacts, orgKnowledge);
    }
    
    private List<Memory> searchSession(Embedding query, UUID sessionId, int limit) {
        String sql = """
            SELECT id, content, embedding <=> ? AS distance
            FROM session_memories
            WHERE session_id = ? AND expires_at > NOW()
            ORDER BY embedding <=> ?
            LIMIT ?
            """;
        // Execute query...
    }
}
```

## 3. Scaling & Advanced Patterns

### 3.1 HNSW Indexing (Hierarchical Navigable Small World)

As the organizational memory grows (millions of records), standard `ivfflat` indexes may become too slow or inaccurate. We recommend using **HNSW** indexes for scalable vector search.

**Implementation in pgvector:**
```sql
-- Replace standard index with HNSW
CREATE INDEX idx_org_memory_hnsw ON organizational_memories 
    USING hnsw (embedding vector_cosine_ops) 
    WITH (m = 16, ef_construction = 64);
```
- `m`: Max connections per layer (higher = better recall, larger index).
- `ef_construction`: Size of dynamic list during build (higher = better quality, slower build).

### 3.2 GraphRAG (Future Roadmap)

To support multi-hop reasoning (e.g., "How does Project X relate to Policy Y?"), standard vector search is insufficient. **GraphRAG** combines vector search with knowledge graph traversal.

**Relational Graph Schema (PostgreSQL):**
Instead of a separate Graph DB, we can model entities and relationships directly in SQL.

```sql
-- Extracted Entities (Nodes)
CREATE TABLE entities (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    type VARCHAR(50), -- 'Person', 'Project', 'Technology'
    description TEXT,
    embedding VECTOR(384) -- For hybrid search
);

-- Relationships (Edges)
CREATE TABLE relationships (
    source_id UUID REFERENCES entities(id),
    target_id UUID REFERENCES entities(id),
    relation_type VARCHAR(50), -- 'CREATED_BY', 'DEPENDS_ON'
    description TEXT,
    weight FLOAT DEFAULT 1.0,
    PRIMARY KEY (source_id, target_id, relation_type)
);

-- Linking Text Chunks to Knowledge Graph
CREATE TABLE memory_entities (
    memory_id UUID REFERENCES organizational_memories(id),
    entity_id UUID REFERENCES entities(id),
    PRIMARY KEY (memory_id, entity_id)
);
```

### 3.3 Evaluation of Recursive Language Models (RLM)

We evaluated [Recursive Language Models (RLM)](https://alexzhang13.github.io/blog/2025/rlm/) as a potential solution for large-scale memory.

**Conclusion: NOT ADOPTED as Storage Architecture.**

RLM is an *inference strategy* (processing huge inputs by letting an LLM recursively chunk and summarize text via code execution), not a *storage solution*.
- **Pros**: Can "reason" over 10M+ tokens without context rot.
- **Cons**: Extremely slow (minutes), high cost, and requires complex code execution sandboxing.

**Recommendation**: Use PostgreSQL/pgvector (with HNSW) as the primary storage. Implement RLM-like logic only as a specific **"Deep Research" Skill** (e.g., `analyze_large_document`) if the agent needs to process massive files on-demand, but do not use it for general memory retrieval.

## 4. Internationalization: Chinese Support

### 3.1 The Dimension Problem

- **BGE-small-en-v1.5**: 384 dimensions
- **BGE-small-zh-v1.5**: 512 dimensions

**Cannot mix in same vector column!**

### 3.2 Solutions

#### Option A: Separate Tables (Recommended)

```sql
-- English memories
CREATE TABLE short_term_memories_en (
    -- ... same schema ...
    embedding VECTOR(384)
);

-- Chinese memories  
CREATE TABLE short_term_memories_zh (
    -- ... same schema ...
    embedding VECTOR(512)
);

-- Query both and merge results in application layer
```

#### Option B: Unified Multilingual Model

When adding Chinese support, switch to a multilingual model:

| Model | Dimensions | Languages | Trade-off |
|-------|------------|-----------|-----------|
| BGE-small-en-v1.5 | 384 | En + basic multilingual | Use for both initially |
| BGE-base-en-v1.5 | 768 | Better multilingual | Higher dimensions |
| Multilingual-E5 | 384 | 100+ languages | May require re-embedding |

#### Option C: Re-embed Everything

When ready for Chinese:
1. Choose new multilingual model
2. Re-embed all existing memories
3. Single table with new dimensions

### 3.3 Recommendation

**Phase 1 (English only)**:
- Use `BGE-small-en-v1.5-q` (384d)
- Single table structure

**Phase 2 (Add Chinese)**:
- Option: Use `BGE-small-en-v1.5` for both (decent Chinese support)
- Or create separate `_zh` tables with `BGE-small-zh-v1.5` (512d)
- Query service merges results from both tables

## 4. Implementation Guidelines

### 4.1 Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>bge-small-en-v15-q</artifactId>
    <version>${project.version}</version>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.1</version>
</dependency>
```

### 4.2 Configuration

```yaml
# application.yml
memory:
  embedding:
    model: bge-small-en-v1.5-q
    dimensions: 384
    query-prefix: "Represent this sentence for searching relevant passages:"
  
  storage:
    type: postgresql
    url: ${PGVECTOR_URL}
    username: ${PGVECTOR_USER}
    password: ${PGVECTOR_PASSWORD}
    
  short-term:
    ttl-minutes: 60
    max-memories: 100
    
  long-term:
    min-importance: 0.5
    max-results: 10
    
  knowledge:
    chunk-size: 512
    overlap: 50
```

### 4.3 Key Design Principles

1. **Separate Concerns**: Vector DB for semantic search, SQL for structured data
2. **Query Prefixing**: Always use BGE's recommended prefix for queries
3. **Lifecycle Management**: Short-term expires, long-term persists, both consolidate
4. **Dimension Consistency**: Plan for Chinese support from day one

## 5. References

- [BGE Paper](https://arxiv.org/abs/2309.07597)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
- [Hugging Face BGE Models](https://huggingface.co/BAAI)
