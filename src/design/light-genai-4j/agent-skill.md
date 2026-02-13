# Agent Skill Design

## Overview

This document outlines the design decisions for agent skills (tools), storage, and execution architecture for the light-genai-4j agent system.

## 1. Skills Storage: SQL (Not Vector DB)

### 1.1 Why SQL for Skills?

Skills (often called Tools) are **executable code units** (Java classes, scripts, or API calls) with structured definitions. They are **not** typically stored as markdown files, although their descriptions are text-based for LLM consumption.

**Do NOT store skills in vector DB** - you typically call skills by precise name or ID based on intent classification, not by semantic similarity.

### 1.2 Skills Schema

This schema defines **what** the skill is, **how** to execute it, and **what parameters** it requires.

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
    name VARCHAR(255) UNIQUE NOT NULL, -- The function name called by LLM
    description TEXT,                  -- Natural language description for System Prompt
    category_id UUID REFERENCES skill_categories(id),
    
    -- Implementation Details
    implementation_type VARCHAR(50) NOT NULL, -- 'java', 'python', 'javascript', 'rest', 'mcp'
    implementation_class VARCHAR(500),        -- Fully qualified Java class name (for 'java')
    script_content TEXT,                      -- Actual source code (for 'python'/'javascript')
    api_endpoint VARCHAR(1024),               -- URL (for 'rest')
    api_method VARCHAR(10),                   -- HTTP Method (GET/POST)
    
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

-- Skill Parameters (Interface Definition)
-- Maps directly to arguments of the execute() method
CREATE TABLE skill_parameters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_id UUID REFERENCES skills(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,           -- Argument name (e.g., 'recipient_email')
    param_type VARCHAR(50) NOT NULL,      -- 'string', 'number', 'boolean', 'object', 'array'
    is_required BOOLEAN DEFAULT true,
    default_value JSONB,
    description TEXT,                     -- Instructions for LLM to extract value
    validation_schema JSONB,              -- JSON Schema for complex validation (regex, enum, etc.)
    order_index INTEGER DEFAULT 0,        -- Argument position index
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

-- Core Functionality of skill_dependencies:
-- 1. Availability Check: A skill is not loaded/listed unless all required dependencies are enabled and accessible.
-- 2. Execution Chaining: Allows composed skills (meta-skills) to automatically resolve and execute their prerequisites.
-- 3. Validation: Prevents circular dependencies in the capability graph.

-- Indexes
CREATE INDEX idx_skills_category ON skills(category_id);
CREATE INDEX idx_skills_enabled ON skills(is_enabled);
CREATE INDEX idx_skills_name ON skills(name);
CREATE INDEX idx_agent_skills_agent ON agent_skills(agent_id);
```

### 1.3 Skill Discovery (Semantic Search)

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

## 2. Relationship with MCP (Model Context Protocol)

With this design, the agent system **does not require** an MCP server architecture. The `skills` table allows direct invocation of any capability (Java code, REST APIs, GraphQL, scripts) without the overhead or complexity of the MCP protocol.

### 2.1 Comparison

| Feature | Light GenAI 4j Design | MCP Server Architecture |
|---------|-----------------------|-------------------------|
| **Execution** | In-process (Java) or Direct I/O | Networked JSON-RPC |
| **Latency** | Near-zero (function call) | HTTP/Network round-trip |
| **Control** | Full SQL schema & validation | Remote server definition |
| **Complexity** | Low (Unified DB table) | High (Separate servers/processes) |
| **Ecosystem** | Custom integrations | Community-maintained tools |

### 2.2 Recommendation: MCP as an Implementation Type

While the architecture replaces the *need* for MCP, it is recommended to support `mcp` as a valid `implementation_type` in the `skills` table. This allows the agent to consume existing third-party MCP servers (e.g., Google Drive, Slack, GitHub) without rewriting integration logic.

**Updated `skills` implementation types**:
- `java`: Local Java class execution (Fastest, Primary)
- `rest`: Direct HTTP/REST API calls
- `python`/`javascript`: Dynamic script execution
- `mcp`: Connect to an external MCP server (Optional, for 3rd party tools)

## 3. Implementation Guidelines

### 3.1 Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.1</version>
</dependency>
```

### 3.2 Design Principles

1.  **Separate Concerns**: Vector DB for semantic search, SQL for structured data.
2.  **Explicit Definitions**: Skills are defined by code/API, not just prompt text.
3.  **Dependency Management**: Use `skill_dependencies` to chain complex operations safely.
