# Memory and Embedding Design

## Overview

This document outlines the design decisions for memory management, embedding models, and storage architecture for the light-genai-4j agent system.

## 1. Embedding Model Selection

### 1.1 Chosen Model: BGE-small-en-v1.5 (Quantized)

**Artifact**: `langchain4j-embeddings-bge-small-en-v15-q`

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

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Short-term memory (session context)
CREATE TABLE short_term_memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL,
    agent_id UUID NOT NULL,
    user_id UUID,
    content TEXT NOT NULL,
    embedding VECTOR(384),
    importance_score FLOAT DEFAULT 1.0,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '1 hour',
    metadata JSONB
);

-- Long-term memory (persistent across sessions)
CREATE TABLE long_term_memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL,
    user_id UUID,
    content TEXT NOT NULL,
    embedding VECTOR(384),
    memory_type VARCHAR(50), -- 'conversation', 'preference', 'fact'
    importance_score FLOAT DEFAULT 1.0,
    access_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed TIMESTAMP DEFAULT NOW(),
    metadata JSONB
);

-- Knowledge base documents
CREATE TABLE knowledge_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL,
    source VARCHAR(255), -- Document source/name
    content TEXT NOT NULL,
    embedding VECTOR(384),
    chunk_index INTEGER, -- For large documents split into chunks
    document_id UUID, -- Reference to parent document
    created_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB
);

-- Indexes for similarity search
CREATE INDEX idx_short_term_embedding ON short_term_memories 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_long_term_embedding ON long_term_memories 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_knowledge_embedding ON knowledge_documents 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

### 2.2 Memory Lifecycle

```
User Input
    │
    ▼
┌─────────────────────────────────────┐
│  Query Embedding (with prefix)      │
└─────────────────────────────────────┘
    │
    ├──► Search Short-term Memory (high priority)
    │     └── Recent context (last N turns)
    │
    ├──► Search Long-term Memory (if needed)
    │     └── Similar past interactions
    │
    └──► Search Knowledge Base (if needed)
          └── Relevant documents
    │
    ▼
Consolidated Context → LLM
    │
    ▼
Store Response ──────► Short-term Memory
    │                         │
    │                         ▼
    │              Scheduled for Summarization
    │                         │
    │                         ▼
    │              Important? ──► Long-term Memory
    │
    ▼
    (Session ends)
    Short-term Expires
```

### 2.3 Memory Retrieval Strategy

```java
public class MemoryService {
    
    public List<Memory> retrieveRelevantMemories(String query, UUID sessionId, UUID agentId) {
        // 1. Embed the query with prefix
        String prefixedQuery = "Represent this sentence for searching relevant passages: " + query;
        Embedding queryEmbedding = embeddingModel.embed(prefixedQuery);
        
        // 2. Search short-term memory (recent context)
        List<Memory> shortTerm = searchShortTerm(queryEmbedding, sessionId, limit = 10);
        
        // 3. Search long-term memory (if short-term insufficient)
        List<Memory> longTerm = new ArrayList<>();
        if (shortTerm.size() < 5) {
            longTerm = searchLongTerm(queryEmbedding, agentId, limit = 5);
        }
        
        // 4. Combine and rank
        return combineAndRank(shortTerm, longTerm);
    }
    
    private List<Memory> searchShortTerm(Embedding query, UUID sessionId, int limit) {
        String sql = """
            SELECT id, content, embedding <=> ? AS distance
            FROM short_term_memories
            WHERE session_id = ? AND expires_at > NOW()
            ORDER BY embedding <=> ?
            LIMIT ?
            """;
        // Execute query...
    }
}
```

## 3. Skills Storage: SQL (Not Vector DB)

### 3.1 Why SQL for Skills?

Skills are **structured data** requiring:
- Exact lookups (by name/ID)
- Hierarchical relationships
- Parameter validation
- ACID transactions

**Do NOT store skills in vector DB** - you don't search for skills by semantic similarity.

### 3.2 Skills Schema

```sql
-- Skill categories (hierarchical)
CREATE TABLE skill_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    parent_id UUID REFERENCES skill_categories(id),
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Skills definition
CREATE TABLE skills (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    category_id UUID REFERENCES skill_categories(id),
    
    -- Implementation
    implementation_class VARCHAR(500) NOT NULL,
    implementation_type VARCHAR(50), -- 'java', 'python', 'javascript', 'rest'
    
    -- Metadata
    version INTEGER DEFAULT 1,
    is_enabled BOOLEAN DEFAULT true,
    author VARCHAR(255),
    tags TEXT[],
    
    -- For semantic discovery (optional)
    description_embedding VECTOR(384),
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Skill parameters
CREATE TABLE skill_parameters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_id UUID REFERENCES skills(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    param_type VARCHAR(50), -- 'string', 'number', 'boolean', 'object', 'array'
    is_required BOOLEAN DEFAULT true,
    default_value JSONB,
    description TEXT,
    validation_schema JSONB, -- JSON Schema
    order_index INTEGER DEFAULT 0,
    UNIQUE(skill_id, name)
);

-- Agent-skill assignments
CREATE TABLE agent_skills (
    agent_id UUID REFERENCES agents(id),
    skill_id UUID REFERENCES skills(id),
    
    -- Skill-specific configuration
    config JSONB DEFAULT '{}',
    
    -- Execution settings
    priority INTEGER DEFAULT 0,
    timeout_seconds INTEGER DEFAULT 30,
    max_retries INTEGER DEFAULT 3,
    
    -- Permissions
    allowed_users UUID[], -- NULL = all users
    
    is_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    
    PRIMARY KEY (agent_id, skill_id)
);

-- Skill dependencies
CREATE TABLE skill_dependencies (
    skill_id UUID REFERENCES skills(id) ON DELETE CASCADE,
    depends_on_skill_id UUID REFERENCES skills(id) ON DELETE CASCADE,
    is_required BOOLEAN DEFAULT true,
    PRIMARY KEY (skill_id, depends_on_skill_id)
);

-- Indexes
CREATE INDEX idx_skills_category ON skills(category_id);
CREATE INDEX idx_skills_enabled ON skills(is_enabled);
CREATE INDEX idx_skills_name ON skills(name);
CREATE INDEX idx_agent_skills_agent ON agent_skills(agent_id);
```

### 3.3 Skill Discovery (Semantic Search)

If you want natural language skill discovery (e.g., "how do I send email?" → find email skill):

```sql
-- Search skills by semantic similarity
SELECT s.id, s.name, s.description, 
       s.description_embedding <=> ? AS similarity
FROM skills s
WHERE s.is_enabled = true
ORDER BY s.description_embedding <=> ?
LIMIT 5;
```

## 4. Internationalization: Chinese Support

### 4.1 The Dimension Problem

- **BGE-small-en-v1.5**: 384 dimensions
- **BGE-small-zh-v1.5**: 512 dimensions

**Cannot mix in same vector column!**

### 4.2 Solutions

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

### 4.3 Recommendation

**Phase 1 (English only)**:
- Use `BGE-small-en-v1.5-q` (384d)
- Single table structure

**Phase 2 (Add Chinese)**:
- Option: Use `BGE-small-en-v1.5` for both (decent Chinese support)
- Or create separate `_zh` tables with `BGE-small-zh-v1.5` (512d)
- Query service merges results from both tables

## 5. Implementation Guidelines

### 5.1 Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-embeddings-bge-small-en-v15-q</artifactId>
    <version>1.0.0</version>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.1</version>
</dependency>
```

### 5.2 Configuration

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

### 5.3 Key Design Principles

1. **Separate Concerns**: Vector DB for semantic search, SQL for structured data
2. **Query Prefixing**: Always use BGE's recommended prefix for queries
3. **Lifecycle Management**: Short-term expires, long-term persists, both consolidate
4. **Dimension Consistency**: Plan for Chinese support from day one
5. **Skill Discovery**: Optional semantic search on skill descriptions only

## 6. References

- [BGE Paper](https://arxiv.org/abs/2309.07597)
- [LangChain4j Embeddings](https://docs.langchain4j.dev/category/embedding-models)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
- [Hugging Face BGE Models](https://huggingface.co/BAAI)
