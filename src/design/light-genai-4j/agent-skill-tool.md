# Agent Skill Tool Design

## Overview

This document outlines the design decisions for agent skills, tools, their relationship (Progressive Disclosure), and the execution architecture for the light-genai-4j agent system.

## 1. Defining Skills vs. Tools

In the light-genai-4j agent architecture, we adhere to the **Model Context Protocol (MCP)** separation of concerns:

- **Skills (The "Expertise")**: Sets of instructions, workflows, or domain knowledge that teach the agent *how* to think and behave. They are declarative and loaded into the LLM's prompt.
- **Tools (The "Hands")**: Deterministic, executable functions (like APIs, DB queries, or MCP server calls) that take action in the world. 

## 2. Progressive Disclosure

To optimize LLM context window usage and prevent token bloat, we use the **Progressive Disclosure** pattern:
1. **Agent -> Skills**: An agent is assigned a set of skills. When the agent starts, it only loads the high-level descriptions of its assigned skills into its system prompt.
2. **Skill -> Tools**: Each skill is mapped to one or more tools. When the LLM decides to use a specific skill based on its description, the system dynamically fetches and registers the associated tool definitions (JSON schemas) into the LLM's context.

## 3. Database Schema

### 3.1 Skills
Skills define the instructions and logic.

```sql
CREATE TABLE skill_t (
    host_id             UUID NOT NULL,
    skill_id            UUID NOT NULL,
    parent_skill_id     UUID,                  
    name                VARCHAR(126) NOT NULL,
    description         VARCHAR(500),          -- High-level description for the initial LLM prompt
    content_markdown    TEXT NOT NULL,         -- The detailed instructions/prompts
    description_embedding VECTOR(384),         -- For semantic lookup/discovery
    
    version             VARCHAR(20) DEFAULT '1.0.0',
    aggregate_version   BIGINT DEFAULT 1 NOT NULL,
    active              BOOLEAN DEFAULT true,
    update_ts           TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    update_user         VARCHAR(126) DEFAULT SESSION_USER,
    PRIMARY KEY(host_id, skill_id),
    FOREIGN KEY(host_id, parent_skill_id) REFERENCES skill_t(host_id, skill_id)
);

CREATE INDEX idx_skill_active ON skill_t(active);
CREATE INDEX idx_skill_name ON skill_t(name);
```

### 3.2 Tools
Tools define the technical execution metadata.

```sql
CREATE TABLE tool_t (
    host_id             UUID NOT NULL,
    tool_id             UUID NOT NULL,
    name                VARCHAR(126) NOT NULL,
    description         TEXT NOT NULL,         -- Instructions for LLM on when/how to use this tool
    
    -- Implementation specifics
    implementation_type VARCHAR(50),           -- 'java', 'mcp_server', 'rest', 'python', 'javascript'
    implementation_class VARCHAR(500),         -- FQCN if 'java'
    mcp_server_name      VARCHAR(126),         -- MCP server name if 'mcp_server'
    api_endpoint        VARCHAR(1024),         -- URL if 'rest'
    api_method          VARCHAR(10),           -- HTTP Method if 'rest'
    script_content      TEXT,                  -- Source code if 'python'/'javascript'
    description_embedding VECTOR(384),         -- For semantic lookup/discovery
    
    version             VARCHAR(20) DEFAULT '1.0.0',
    aggregate_version   BIGINT DEFAULT 1 NOT NULL,
    active              BOOLEAN DEFAULT true,
    update_ts           TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    update_user         VARCHAR(126) DEFAULT SESSION_USER,
    PRIMARY KEY(host_id, tool_id)
);

CREATE INDEX idx_tool_active ON tool_t(active);
CREATE INDEX idx_tool_name ON tool_t(name);
```

### 3.3 Tool Parameters
Defines the arguments for each tool, mapping directly to JSON Schema used by LangChain4j.

```sql
CREATE TABLE tool_param_t (
    host_id             UUID NOT NULL,
    param_id            UUID NOT NULL,
    tool_id             UUID NOT NULL,
    name                VARCHAR(255) NOT NULL,     
    param_type          VARCHAR(50) NOT NULL,      -- 'string', 'number', 'boolean', 'object', 'array'
    required            BOOLEAN DEFAULT true,
    default_value       JSONB,
    description         TEXT,                      -- Helps LLM understand what value to extract
    validation_schema   JSONB,                     -- JSON Schema for complex validation
    order_index         INTEGER DEFAULT 0,         
    aggregate_version   BIGINT DEFAULT 1 NOT NULL,
    active              BOOLEAN DEFAULT true,
    update_ts           TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    update_user         VARCHAR(126) DEFAULT SESSION_USER,
    PRIMARY KEY(host_id, param_id),
    FOREIGN KEY(host_id, tool_id) REFERENCES tool_t(host_id, tool_id) ON DELETE CASCADE
);
```

### 3.4 Mappings and Dependencies

**Agent to Skill Mapping (`agent_skill_t`)**
Defines an agent's capabilities (which skills it possesses).

```sql
CREATE TABLE agent_skill_t (
    host_id             UUID NOT NULL,
    agent_def_id        UUID NOT NULL,
    skill_id            UUID NOT NULL,
    
    config              JSONB DEFAULT '{}',
    priority            INTEGER DEFAULT 0,
    sequence_id         INTEGER DEFAULT 0,
    
    aggregate_version   BIGINT DEFAULT 1 NOT NULL,
    active              BOOLEAN DEFAULT true,
    update_ts           TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    update_user         VARCHAR(126) DEFAULT SESSION_USER,
    PRIMARY KEY(host_id, agent_def_id, skill_id),
    FOREIGN KEY(host_id, agent_def_id) REFERENCES agent_definition_t(host_id, agent_def_id) ON DELETE CASCADE,
    FOREIGN KEY(host_id, skill_id) REFERENCES skill_t(host_id, skill_id) ON DELETE CASCADE
);
CREATE INDEX idx_agent_skill_agent ON agent_skill_t(agent_def_id);
```

**Skill to Tool Mapping (`skill_tool_t`)**
Implements the Progressive Disclosure pattern linking tools needed by specific skills.

```sql
CREATE TABLE skill_tool_t (
    host_id             UUID NOT NULL,
    skill_id            UUID NOT NULL,
    tool_id             UUID NOT NULL,
    
    config              JSONB DEFAULT '{}',
    access_level        VARCHAR(20) DEFAULT 'read', -- e.g., 'read', 'write', 'execute'
    
    aggregate_version   BIGINT DEFAULT 1 NOT NULL,
    active              BOOLEAN DEFAULT true,
    update_ts           TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    update_user         VARCHAR(126) DEFAULT SESSION_USER,
    PRIMARY KEY(host_id, skill_id, tool_id),
    FOREIGN KEY(host_id, skill_id) REFERENCES skill_t(host_id, skill_id) ON DELETE CASCADE,
    FOREIGN KEY(host_id, tool_id) REFERENCES tool_t(host_id, tool_id) ON DELETE CASCADE
);
CREATE INDEX idx_skill_tool_skill ON skill_tool_t(skill_id);
```

**Skill Dependencies (`skill_dependency_t`)**
Manages hierarchies where one skill requires another.

```sql
CREATE TABLE skill_dependency_t (
    host_id             UUID NOT NULL,
    skill_id            UUID NOT NULL,
    depends_on_skill_id UUID NOT NULL,
    required            BOOLEAN DEFAULT true,
    
    aggregate_version   BIGINT DEFAULT 1 NOT NULL,
    active              BOOLEAN DEFAULT true,
    update_ts           TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    update_user         VARCHAR(126) DEFAULT SESSION_USER,
    PRIMARY KEY (host_id, skill_id, depends_on_skill_id),
    FOREIGN KEY(host_id, skill_id) REFERENCES skill_t(host_id, skill_id),
    FOREIGN KEY(host_id, depends_on_skill_id) REFERENCES skill_t(host_id, skill_id)
);
```

## 4. Implementation Types

The `tool_t` table supports various `implementation_type` values to determine how a tool is physically executed:

-   **`java`**: Local Java class execution mapping to `@Tool` methods (Fastest, Primary).
-   **`mcp_server`**: Connect to an external Model Context Protocol server (Standard for 3rd party tools).
-   **`rest`**: Direct HTTP/REST API calls.
-   **`python`/`javascript`**: Dynamic script execution.
