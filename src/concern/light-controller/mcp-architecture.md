# Model Context Protocol (MCP) Architecture for light-controller

## 1. Executive Summary
As the `light-controller` transitions to a fully event-driven WebSocket architecture for internal microservice management, its external-facing administrative APIs are also being overhauled. 

Instead of exposing traditional REST endpoints for the `light-portal` UI to poll, the controller will expose a single, real-time WebSocket connection to the Portal (routed through the `light-gateway` BFF). Furthermore, all administrative capabilities (e.g., `reload_config`, `set_log_level`, `get_server_info`) will be implemented as **Model Context Protocol (MCP) Tools**.

This document outlines how leveraging MCP over JSON-RPC 2.0 creates a unified, AI-native Control Plane that serves both human administrators and autonomous AI agents.

## 2. The Shift: REST to Real-Time WebSockets
Historically, the Portal UI relied on REST APIs to fetch the status of registered microservices. 
* **The Problem:** The UI had to constantly poll the controller to detect if a service went offline or if an administrative command (like reloading a configuration) completed successfully.
* **The Solution:** By moving the UI-to-Controller communication to WebSockets, the `light-controller` can actively *push* state changes. If a microservice's internal WebSocket drops, the controller instantly pushes an event to the UI's WebSocket, updating the dashboard in milliseconds without polling.

## 3. Why MCP (Model Context Protocol)?
While standard JSON-RPC 2.0 is highly effective for bidirectional WebSocket communication, **MCP is an open standard built strictly on top of JSON-RPC 2.0**. It defines a specific vocabulary designed to allow AI models to securely discover and execute tools on external systems.

By building the controller's APIs as MCP Tools, we achieve **AIOps (AI for IT Operations) natively**.

### 3.1 Unified Interface for Humans and AI
In a traditional setup, the UI uses a JSON API, and a separate integration layer is built to teach an AI chatbot how to trigger those same APIs. 
With MCP, the `light-controller` acts as an **MCP Server**. Both the human-driven Portal UI and the AI Chatbot act as **MCP Clients** using the exact same WebSocket connection and protocol.

* **The Portal UI:** When an administrator clicks the "Reload Config" button, the UI constructs a standard MCP `tools/call` message.
* **The AI Chatbot:** When an administrator types, *"The user service is throwing debug errors, please reset its log level to INFO"*, the AI agent autonomously constructs the exact same MCP `tools/call` message.

## 4. Dynamic UI and Tool Discovery
One of the most powerful features of MCP is **Tool Discovery**. 
Instead of hardcoding every available administrative action into the Portal UI's frontend code, the UI (and the Chatbot) can query the `light-controller` upon connection to ask what commands are currently supported.

### 4.1 Step 1: Tool Discovery (`tools/list`)
When the Portal UI connects to the controller via the BFF, it sends a discovery request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}

Controller Response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools":[
      {
        "name": "reload_config",
        "description": "Instructs a specific microservice to fetch the latest configuration from the config server.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "serviceId": { "type": "string" },
            "module": { "type": "string" }
          },
          "required": ["serviceId"]
        }
      },
      {
        "name": "set_log_level",
        "description": "Dynamically changes the logging level of a registered microservice.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "serviceId": { "type": "string" },
            "level": { "type": "string", "enum": ["DEBUG", "INFO", "WARN", "ERROR"] }
          },
          "required": ["serviceId", "level"]
        }
      }
    ]
  }
}
```
Result: The UI can dynamically render buttons and forms based on the inputSchema, and the AI agent instantly knows exactly how to format its requests.


### 4.1 Step 2: Tool Execution (`tools/call`)

When an action is triggered (by a human click or an AI decision), the client sends:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "set_log_level",
    "arguments": {
      "serviceId": "user-management-service",
      "level": "INFO"
    }
  }
}

```

## 5. Network Topology for Admin APIs

To maintain the strict separation of Data Plane and Control Plane:

* **Internal Traffic (Microservices):** Microservices connect directly to the light-controller via raw multiplexed WebSockets (bypassing the gateway).

* **External Traffic (Portal UI & Chatbot):** The React/Vue web application and AI Chatbot run in the user's browser. They connect to the light-gateway (BFF) over WSS (WebSocket Secure). The gateway validates the user's OAuth2 session token and proxies the MCP WebSocket connection to the light-controller.

## 6. Conclusion

Implementing the light-controller's administrative APIs as MCP Tools over WebSockets represents a generational leap in system design. It entirely removes polling latency for human operators while instantly transforming the light-portal ecosystem into an AI-manageable, autonomous infrastructure. New administrative features added to the controller are immediately discoverable and usable by both the UI and the built-in Chatbot without requiring frontend code changes.