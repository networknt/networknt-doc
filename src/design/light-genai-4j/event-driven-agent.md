# Event-Driven Agent Architecture

## Overview

This document details the **Event-Driven Architecture (EDA)** for the `light-genai-4j` agent system. This architecture decouples agent invocation from execution, enabling high scalability, resilience, and asynchronous processing suitable for enterprise workloads.

## 1. Core Architecture

While SQL defines *what* a skill is (see [Agent Skill Design](agent-skill.md)), the implementation for enterprise usage leverages an **Event-Driven Architecture (EDA)**. This decouples the agent requesting the skill (the "invoker") from the agent or service executing the skill (the "worker").

### 1.1 Core Components

1.  **Invoker Agent**: The agent that decides to call a tool/skill.
2.  **A2A Service (`genai-agentic-kafka`)**: The bridge between the synchronous `AgentExecutor` and the asynchronous message bus.
3.  **Topic Topology**:
    *   `agent-commands`: Topics where agents publish intent/skill execution requests.
    *   `agent-events`: Topics where agents/workers publish completion events.

### 1.2 Execution Flow

1.  **LLM Decision**: The LLM outputs a `ToolExecutionRequest`.
2.  **Interception**: The `AgentExecutor`'s `A2AService` implementation intercepts this request.
3.  **Command Emission**:
    *   Instead of executing a Java method directly, it constructs a `SkillInvocationEvent` containing:
        *   `correlationId`: Unique ID for this interaction.
        *   `skillName`: Name of the skill to execute.
        *   `arguments`: JSON payload of arguments.
        *   `replyTo`: Topic to send the result to.
    *   This event is published to the `agent-commands` topic.
4.  **Async Wait**: The `AgentExecutor` returns an `AsyncResponse` (a `CompletableFuture`) and suspends the agent's execution thread (virtually, if using Virtual Threads).
5.  **Worker Execution**:
    *   A subscribed "Skill Worker" (which could be another Generic Agent or a dedicated microservice) picks up the event.
    *   It executes the logic (DB query, API call, calculation).
6.  **Response Emission**:
    *   The worker publishes a `SkillCompletionEvent` to the `replyTo` topic.
    *   Payload includes `correlationId` and the result/error.
7.  **Resumption**:
    *   The original `A2AService` consumes the completion event.
    *   It matches the `correlationId` and completes the pending `CompletableFuture`.
    *   The Agent resumes generation with the tool output.

### 1.3 Benefits

*   **Scalability**: Heavy skills (e.g., "Generate Report") don't block the lightweight Agent Commander.
*   **Resilience**: If the Worker is down, the command persists in Kafka.
*   **Decoupling**: Agents don't need to know the network location of skills.

## 2. Relationship with MCP (Model Context Protocol)

With this design, the agent system **does not require** an MCP server architecture. The `skills` table allows direct invocation of any capability (Java code, REST APIs, GraphQL, scripts) without the overhead or complexity of the MCP protocol.

### 2.1 Comparison

| Feature | Light GenAI 4j Design | MCP Server Architecture |
|---------|-----------------------|-------------------------|
| **Execution** | In-process (Java) or **Event-Driven** | Networked JSON-RPC |
| **Latency** | Near-zero (local) / Async (EDA) | HTTP/Network round-trip |
| **Control** | Full SQL schema & validation | Remote server definition |
| **Complexity** | Low (Unified DB/Kafka) | High (Separate servers/processes) |
| **Ecosystem** | Custom integrations | Community-maintained tools |

## 3. Implementation Guidelines

### 3.1 Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>genai-agentic-kafka</artifactId> <!-- Future Module -->
    <version>${project.version}</version>
</dependency>
```

### 3.2 Design Principles

1.  **Asynchrony**: Default to async/event-driven for any I/O heavy skill.
2.  **Correlation**: Use `correlationId` rigorously to track request/response pairs across the bus.
3.  **Idempotency**: Workers should handle duplicate events gracefully.
