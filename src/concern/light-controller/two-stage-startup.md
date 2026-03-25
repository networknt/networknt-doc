# light-4j Two-Stage Startup Architecture

## 1. Executive Summary
The `light-4j` framework utilizes a two-stage startup process to ensure services are fully configured before initializing their main application contexts and network listeners. With the transition of the `light-controller` to a WebSocket-based Control Plane, a critical architectural decision is how to handle initial configuration loading.

This document outlines the **Hybrid Approach**: maintaining **REST** for the synchronous Bootstrap Stage (Stage 1) and utilizing **WebSocket** exclusively for the asynchronous Runtime Stage (Stage 2). 

## 2. Why Not WebSocket for Bootstrapping?

While it is tempting to unify all external communication under a single WebSocket connection, using WebSockets for the initial configuration pull introduces significant anti-patterns during server boot:

### 2.1 The "Chicken and Egg" Problem
To establish a secure and robust WebSocket connection to the `light-controller`, the `light-4j` instance requires configuration data, including:
* TLS/SSL Certificates (Keystores and Truststores)
* OAuth2 Client Credentials (to authenticate the connection)
* Network configurations (connection timeouts, proxies, retry limits)

If a WebSocket is used to fetch the configuration, the system lacks the configuration required to configure the WebSocket client itself. A simple REST call using a basic bootstrap token (defined in a local `startup.yml` or environment variables) safely breaks this circular dependency.

### 2.2 Synchronous vs. Asynchronous Paradigms
Bootstrapping is fundamentally a **synchronous** blocking operation. The server cannot bind to its ports, initialize database connection pools, or start business logic until the configuration is fully downloaded and parsed. 
* **REST (HTTP GET)** is naturally synchronous, perfectly fitting the blocking boot sequence.
* **WebSocket** is inherently asynchronous and event-driven. Using it for bootstrapping requires artificially pausing the main startup thread (e.g., using `CountDownLatch` or `CompletableFuture.get()`) while waiting for the `onMessage` event to fire, adding unnecessary complexity and fragility to the startup sequence.

### 2.3 Separation of Concerns
In the `light-portal` ecosystem, the **Config Server** and the **Controller** serve distinct architectural purposes:
* **Config Server:** A stateless data plane that serves configuration files on demand.
* **Controller:** A stateful control plane that tracks live instances, monitors health, and routes administrative commands.

Forcing the initial config pull through the Controller's WebSocket tightly couples these two components, forcing the Controller to act as an unnecessary proxy for the Config Server.

---

## 3. The Recommended Startup Lifecycle

By adopting a hybrid model, `light-4j` leverages the strengths of both protocols. The lifecycle operates as follows:

### Stage 1: Bootstrap (REST)
1. The `light-4j` instance starts and reads its local `startup.yml` (or environment variables) to locate the `light-portal` Config Server URL and initial authentication tokens.
2. The framework makes a synchronous **REST GET** request to the Config Server.
3. The main thread blocks until the configuration payload (e.g., `values.yml`) is completely downloaded, parsed, and merged into the application's configuration cache.

### Stage 2: Main Server Initialization (WebSocket)
1. Now fully configured with proper TLS settings, database credentials, and OAuth2 tokens, the main `light-4j` server modules initialize.
2. The service's internal WebSocket Client establishes a persistent, secure connection to the `light-controller`.
3. Upon connection (`onOpen`), the service immediately sends a **Registration** payload over the WebSocket to announce it is alive, healthy, and ready for routing.
4. The WebSocket remains open, handling native `Ping/Pong` keep-alives and listening for incoming administrative commands.

---

## 4. Runtime: Dynamic Configuration Reloading

What happens if configuration properties change in the `light-portal` *after* the service has started? This is where the hybrid approach excels, combining WebSocket's push capabilities with REST's data-fetching simplicity.

1. **The Trigger (WebSocket):** An administrator updates a configuration property in the `light-portal`. The `light-controller` pushes an asynchronous JSON command down the established WebSocket tunnel to the specific `light-4j` microservice.
```json
   {
     "id": "cmd-reload-001",
     "action": "config_updated"
   }
```
