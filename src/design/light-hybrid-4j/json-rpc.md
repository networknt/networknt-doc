# JSON-RPC 2.0 Migration for light-hybrid-4j

## 1. Introduction
The `light-hybrid-4j` framework is the backbone of the `light-portal`, acting as a highly efficient, modularized monolithic architecture. Currently, it relies on a bespoke JSON structure containing a 4-tuple (`host`, `service`, `action`, `version`) alongside the domain `data`.

With the introduction of the **Model Context Protocol (MCP)** as a first-class citizen in the `light-gateway` ecosystem, the need for a standardized RPC format has become critical. MCP relies entirely on **JSON-RPC 2.0**.

This document outlines the architectural design and migration strategy for adapting `light-hybrid-4j` to natively support the JSON-RPC 2.0 standard.

## 2. Benefits and Motivations

### 2.1 Native MCP Integration
By standardizing `light-hybrid-4j` to JSON-RPC 2.0, every single command and query service instantly becomes a native MCP tool. The `light-gateway` can route traffic without requiring any expensive payload translation layer. AI Agents can directly call our backend microservices out-of-the-box.

### 2.2 Immediate Ecosystem Compatibility
The custom payload format forces third-party developers (and internal UI components) to format their requests manually. JSON-RPC 2.0 is an industry staple with massive ecosystem support (SDKs, Postman, test runners).

### 2.3 Batch Processing
JSON-RPC 2.0 native batch capabilities allow the `portal-view` UI to aggregate multiple independent queries into a single HTTP request packet, vastly reducing network overhead and HTTP/2 connection pooling complexity.

### 2.4 Separation of Concerns
The current payload mixes routing topology (`host`, `service`, `action`, `version`) and UI metadata (`success`, `failure`, `title`) with actual domain data (`data`). JSON-RPC forces a clean boundary: routing is handled entirely by the `method`, while domain data is strictly isolated to the `params` object.

## 3. Architecture Design

### 3.1 Mapping to JSON-RPC 2.0

The legacy `light-hybrid-4j` request looks like this:
```json
{
  "host": "lightapi.net",
  "service": "service",
  "action": "createApiVersion",
  "version": "0.1.0",
  "title": "Create Api Version",
  "success": "/app/success",
  "failure": "/app/failure",
  "data": {
    "apiId": "MCP0001",
    "hostId": "01964b05-552a-7c4b-9184-6857e7f3dc5f"
  }
}
```

This will be mapped to the standard JSON-RPC 2.0 format:
```json
{
  "jsonrpc": "2.0",
  "method": "lightapi.net/service/createApiVersion/0.1.0",
  "params": {
    "apiId": "MCP0001",
    "hostId": "01964b05-552a-7c4b-9184-6857e7f3dc5f"
  },
  "id": 1
}
```

*   **`jsonrpc`**: Must be exactly `"2.0"`.
*   **`method`**: A string constructed by joining the legacy 4-tuple with forward slashes: `{host}/{service}/{action}/{version}`. This perfectly maps to the internal handler ID used by `RpcStartupHookProvider`.
*   **`params`**: The exact contents of the legacy `data` block.
*   **`id`**: A unique identifier for the request, enabling accurate matching of asynchronous responses and batch processing.

### 3.2 Server-Side Changes (`light-hybrid-4j`)

1.  **Dual-Protocol Router (`SchemaHandler` & `JsonHandler`)**: 
    The router must gracefully support both legacy and JSON-RPC payloads during the transition.
    *   **Detection**: Inspect the root keys of the incoming JSON. If `"jsonrpc": "2.0"` is present, invoke the new JSON-RPC 2.0 parser route. Otherwise, fall back to the legacy parser.
    *   **Extraction**:
        *   Legacy: Extract `host`, `service`, `action`, `version` to resolve the Handler.
        *   JSON-RPC: Split the string in the `method` parameter `host/service/action/version` to resolve the Handler.

2.  **Response Construction**:
    If the request included an `"id"` parameter, the response *must* also be formatted as a JSON-RPC 2.0 response:

    *   **Success**:
        ```json
        {
          "jsonrpc": "2.0",
          "result": { ... handler payload ... },
          "id": 1
        }
        ```
    *   **Error**:
        ```json
        {
          "jsonrpc": "2.0",
          "error": {
            "code": -32600,
            "message": "Invalid Request",
            "data": { ... custom light-4j status object ... }
          },
          "id": 1
        }
        ```

3.  **Batch Request Handling**:
    If the incoming request body is a JSON Array instead of a JSON Object, the gateway must process it as a batch request, iterating over each JSON-RPC object, executing them, and returning a JSON Array of responses.

## 4. Client-Side Changes (`portal-view` SPA)

The `portal-view` currently utilizes a `fetchClient` utility built around the legacy structure.

1.  **Update `fetchClient`**: Modify the core fetch utility to optionally format outbound requests as JSON-RPC 2.0 when an opt-in toggle is set, or permanently switch the underlying transport format.
2.  **Schema Forms**: Update `react-schema-form` components (or the server-side definitions in `Forms.json`) to stop injecting UI routing metadata (`success`, `failure`, `title`) into the network payload. These should remain strictly client-side configuration properties.

## 5. Migration Strategy

1.  **Phase 1: Backend Dual-Support**
    *   Update `light-hybrid-4j` router components (`SchemaHandler`, `JsonHandler`) to detect and accept BOTH legacy and JSON-RPC 2.0 formats simultaneously.
    *   Deploy the updated framework across all `light-portal` microservices (Command and Query nodes).
    *   *Result: Backend is ready, no client changes required yet.*

2.  **Phase 2: Gateway MCP Routing**
    *   Implement MCP HTTP transport logic on the `light-gateway` to expose the backend tools list.
    *   When an MCP client lists tools, the gateway reads the hybrid service schemas and publishes them.
    *   When an MCP client invokes a tool, the gateway proxies the JSON-RPC request transparently to the backend.

3.  **Phase 3: Frontend Migration**
    *   Update the `portal-view` UI to use the new JSON-RPC 2.0 wrapper in `fetchClient`.
    *   Refactor legacy components sending hardcoded `host/service/action` structures.

4.  **Phase 4: Deprecation (Long Term)**
    *   Add warnings to the logs when legacy payloads are received.
    *   Eventually phase out the legacy parser code in `light-hybrid-4j` and rely solely on the JSON-RPC 2.0 standard.
