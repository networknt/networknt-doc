# Intelligent Routing





Implementing **Intelligent Routing & Load Balancing** in an MCP (Model Context Protocol) Gateway is a fascinating architectural challenge. Because MCP requests are typically sent to a single endpoint (e.g., via JSON-RPC or a single SSE connection) rather than standard RESTful URLs, traditional URL-based API gateway routing will not work. 

To implement this feature, you need to break it down into three distinct architectural pillars. Since you are working in the **NetworkNT (Light-4j)** ecosystem—which is built for high-throughput Java API gateways—here is exactly how you should approach this.

---

### Pillar 1: Routing by Tool Name (Content-Based Routing)
In standard HTTP routing, the gateway looks at the URL path (e.g., `/api/weather`). In MCP, the gateway must look *inside* the JSON payload.

An MCP tool call looks like this:
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_customer_data",
    "arguments": { "customerId": "123" }
  }
}
```

**Implementation Strategy:**
1.  **Payload Interception:** Create a middleware handler that parses the incoming request body (using Jackson `JsonNode`).
2.  **Tool Registry Lookup:** Extract the `params.name` ("get_customer_data"). You need an in-memory map or a distributed registry (like Consul, which NetworkNT uses heavily) that maps tool names to backend service IDs.
    *   `get_customer_data` -> `service-id: customer-service`
    *   `execute_sql` -> `service-id: db-agent`
3.  **Dynamic Upstream Routing:** Once the service ID is identified, mutate the request context so the gateway's HTTP client forwards the request to the correct downstream server.

---

### Pillar 2: Routing by Semantic Intent (AI-Driven Routing)
This is the "Intelligent" part. Sometimes the AI model (or user) doesn't specify an exact tool name, but sends a raw prompt, and the gateway must decide which backend tool server is best equipped to handle it.

**Implementation Strategy (Two Approaches):**

*   **Approach A: Fast LLM Classifier (Recommended for Accuracy)**
    Intercept the request and send it to a very fast, cheap LLM (like Claude 3 Haiku or GPT-4o-mini). Provide the LLM with a list of available downstream services and ask it to output ONLY the service name based on the user's intent. Then, route the request.

*   **Approach B: Embeddings & Vector Search (Recommended for Latency/Cost)**
    1.  **Pre-computation:** Create a text description for every backend server/tool you have, generate vector embeddings for those descriptions, and store them in memory.
    2.  **Runtime:** When a semantic request comes in, generate an embedding for the user's intent.
    3.  **Cosine Similarity Search:** Calculate the distance between the user's request vector and your tool vectors. Route the request to the tool server with the highest similarity score.

*Note: Semantic routing adds latency. You should only trigger this flow if the request does NOT contain a strict `tool name`.*

---

### Pillar 3: Reliability (Load Balancing, Retries, Circuit Breaking)
Because AI tool calls often hit legacy backends, databases, or third-party APIs, failure rates are higher than standard web traffic. You need robust resilience patterns.

**Implementation Strategy:**

1.  **Client-Side Load Balancing:** 
    Instead of hardcoding IPs, the Gateway should resolve the `service-id` (from Pillar 1) to a list of available nodes via a discovery service (Consul/Zookeeper). Use algorithms like **Round Robin** or **Consistent Hashing** (useful if you want the same user/context to hit the same tool server to utilize caching). *NetworkNT provides built-in client-side load balancing via the `cluster` module.*

2.  **Retries:** 
    AI tool calls can fail due to rate limits (HTTP 429) or transient timeouts. 
    *   Implement an exponential backoff retry mechanism.
    *   **Caution:** Only retry if the MCP tool is *idempotent* (e.g., `get_weather`). Do not blindly retry tools that mutate state (e.g., `process_refund`) unless you have idempotency keys in place.

3.  **Circuit Breaking:** 
    If your `database-tool-server` goes down, requests will queue up and exhaust gateway threads.
    *   Implement a circuit breaker (e.g., using **Resilience4j** in Java, or NetworkNT's native circuit breaker).
    *   If a specific tool server fails 50% of the time over a 10-second window, **open the circuit**.
    *   When the circuit is open, immediately return a fast-failure to the AI model: *"System error: The database tool is currently unavailable."* The AI model can then decide to apologize to the user or try an alternative tool.

4.  **Failover (Fallback Routing):** 
    If the primary tool server is down, the Gateway can attempt to route to a secondary cluster in a different region, or fall back to an "echo/mock" service that returns graceful degradation messages.

---

### Summary: The Gateway Request Flow

If you build this module, the lifecycle of a request passing through the gateway would look like this:

1.  **Ingress:** Request enters the MCP Gateway.
2.  **Intent Evaluation:** 
    *   Is `params.name` present? -> Proceed to Step 3.
    *   If semantic intent only -> Run Vector Search to guess the tool -> Set `params.name`.
3.  **Service Discovery:** Lookup the Tool Name -> Get `Service-ID`.
4.  **Load Balancing:** Get healthy IPs for `Service-ID` from Consul. Pick one node.
5.  **Execution:** HTTP Client calls the target node.
    *   *If Timeout/429:* Trigger **Retry** logic.
    *   *If Node Down:* Mark node unhealthy, trigger **Failover** to another node.
    *   *If Systemic Failure:* **Circuit Breaker** trips, returns graceful error to the LLM.
6.  **Egress:** Result is returned back to the LLM/User.

