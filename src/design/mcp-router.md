# MCP Router

Implementing an **MCP (Model Context Protocol) Router** inside **light-4j** gateway is a **visionary and highly strategic idea**. 

The industry is currently struggling with "Tool Sprawl"—where every microservice needs to be manually taught to an LLM. By placing an MCP Router at the gateway level, we effectively turn our entire microservice ecosystem into a **single, searchable library of capabilities** for AI agents.

Here is a breakdown of why this is a good idea and how to design it for the light-4j ecosystem.

---

### 1. Why it is a Strategic Win
*   **Discovery at Scale:** Instead of configuring 50 tools for an agent, the agent connects to our gateway via MCP. The gateway then "advertises" the available tools based on the services it already knows.
*   **Protocol Translation:** Our backend services don't need to know what MCP is. The gateway handles the conversion from **MCP (JSON-RPC over SSE/HTTP)** to **REST (OpenAPI)** or **GraphQL**.
*   **Security & Governance:** We can apply light-4j’s existing JWT validation, rate limiting, and audit logging to AI interactions. We control which agents have access to which "tools" (APIs).
*   **Schema Re-use:** We already have OpenAPI specs or GraphQL schemas in light-4j. We can dynamically generate the MCP "Tool Definitions" from these existing schemas.

---

### 2. Design Considerations

#### A. The Transport Layer
MCP supports two primary transports: **Stdio** (for local scripts) and **HTTP with SSE (Server-Sent Events)** (for remote services).
*   **Decision:** For a gateway, we **must use the HTTP + SSE transport**. 
*   **Implementation:** Light-4j is built on Undertow, which has excellent support for SSE. We will need to implement an MCP endpoint (e.g., `/mcp/message`) and an SSE endpoint (e.g., `/mcp/sse`).

#### B. Tool Discovery & Dynamic Mapping
How does the Gateway decide which APIs to expose as MCP Tools?
*   **Metadata Driven:** Use light-4j configuration or annotations in the OpenAPI files to mark specific endpoints as "AI-enabled."
*   **The Mapper:** Create a component that converts an **OpenAPI Operation** into an **MCP Tool Definition**.
    *   `description` in OpenAPI becomes the tool's `description` (crucial for LLM reasoning).
    *   `requestBody` schema becomes the tool's `inputSchema`.

#### C. Authentication & Context Pass-through
This is the hardest part.
*   **The Problem:** The LLM agent connects to the Gateway, but the Backend Microservice needs a user-specific JWT.
*   **The Solution:** The MCP Router must be able to take the identity from the MCP connection (initial handshake) and either pass it through or exchange it for a backend token (OAuth2 Token Exchange).

#### D. Statefulness vs. Statelessness
MCP is often stateful (sessions). 
*   **Implementation:** Since light-4j is designed for high performance and statelessness, we may need light-session-4j a small **Session Manager** (potentially backed by Redis or PostgresQL) to keep track of which MCP Client is mapped to which internal context during the SSE connection.

---

### 3. Implementation Plan for light-4j

#### Step 1: Create the `McpHandler`
Create a new middleware handler in light-4j that intercepts calls to `/mcp`. 
*   This handler must implement the MCP lifecycle: `initialize` -> `list_tools` -> `call_tool`.

#### Step 2: Tool Registry
Implement a registry that scans our gateway's internal routing table.
*   **REST:** For every path (e.g., `GET /customers/{id}`), generate an MCP tool named `get_customers_by_id`.
*   **GraphQL:** For every Query/Mutation, generate a corresponding MCP tool.

#### Step 3: JSON-RPC over SSE
MCP uses JSON-RPC 2.0. We will need a simple parser that:
1. Receives an MCP `call_tool` request.
2. Identifies the internal REST/GraphQL route.
3. Executes a **local dispatch** (internal call) to the existing light-4j handler for that service.
4. Wraps the response in an MCP `content` object and sends it back via SSE.

#### Step 4: Governance (The "Agentic" Layer)
Add a "Critique" or "Guardrail" check. Since this is at the gateway, we can inspect the tool output. If the LLM requested sensitive data, the Gateway can mask it before the agent sees it.

#### Step 5: MCP Proxy Support
The MCP Router supports acting as a proxy for backend MCP servers.
*   **Configuration:** Tools can be configured with a `protocol` field (`http` or `mcp`).
*   **Behavior:**
    *   `http` (default): Transforming MCP tool calls to REST requests (JSON translation).
    *   `mcp`: Forwarding JSON-RPC requests directly to a backend MCP server via HTTP POST.
*   **Use Case:** This allows integrating existing MCP servers (e.g., Node.js, Python) into the light-4j gateway without rewriting them implementation.

---

### 4. Potential Challenges to Watch

1.  **Context Window Overload:** If our gateway has 500 APIs, sending 500 tool definitions to the LLM will crash its context window. 
    *   *Solution:* Implement **Categorization**. When an agent connects, it should specify a "Scope" (e.g., `mcp?scope=accounting`), and the gateway only returns tools relevant to that scope.
2.  **Latency:** Adding a protocol translation layer at the gateway adds milliseconds.
    *   *Solution:* Light-4j’s native performance is our advantage here. Minimize JSON serialization/deserialization by using direct buffer access where possible.
3.  **Complex Schemas:** Some API payloads are too complex for an LLM to understand.
    *   *Solution:* Provide a "Summary" view. Allow our MCP router to transform a complex 100-field JSON response into a 5-field summary that the LLM can actually use.

### Conclusion
Building an MCP Router for light-4j transforms our API Gateway from a "Traffic Cop" into an **"AI Brain Center."** It allows our Java-based enterprise services to be "Agent-Ready" without touching the underlying microservice code.


