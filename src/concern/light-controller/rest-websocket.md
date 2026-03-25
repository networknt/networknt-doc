# Rest vs WebSocket

## 1. Executive Summary
This document outlines the architectural decision to transition the `light-controller` from a scheduled REST-polling model to a persistent, event-driven WebSocket model. This shift fundamentally changes how the controller monitors microservice health and issues administrative commands, moving towards a highly efficient **Control Plane / Data Plane** architecture.

## 2. Background
Historically, the `light-controller` utilized Kafka Streams to schedule and execute health checks against registered microservices. An alternative proposed architecture involved replacing Kafka with a PostgreSQL-backed scheduler that would poll microservice REST endpoints every 3 seconds. 

While a PostgreSQL scheduler with REST is viable, polling at such an aggressive interval introduces significant CPU, network, and database I/O overhead. To achieve real-time monitoring and better resource utilization, the architecture is moving entirely to **WebSockets**.

## 3. Core Architecture: The Control Plane Model
Instead of the `light-controller` acting as a client that schedules outbound requests to microservices, the relationship is inverted:
* **Microservices (Clients)** initiate a persistent WebSocket connection to the `light-controller` (Server) upon startup.
* The WebSocket connection acts as a bidirectional, full-duplex tunnel.
* This single connection handles **both** continuous health monitoring and asynchronous administrative commands.

---

## 4. Health Checks: REST Polling vs. WebSocket Keep-Alive

### 4.1 The Overhead of REST Polling (Every 3 Seconds)
* **Network/CPU Waste:** Every 3 seconds, a REST poll requires HTTP header generation, potential TCP/TLS handshakes (if connections drop), and payload parsing. 
* **Database IOPS:** Relying on a PostgreSQL scheduler to track and trigger intervals for thousands of microservices every 3 seconds generates immense and unnecessary database load.
* **Delayed Detection:** If a service crashes immediately after a health check, the controller remains unaware for up to 3 seconds.

### 4.2 The WebSocket Advantage
* **Zero-Overhead Monitoring:** The TCP/TLS handshake occurs exactly once during startup. Health is verified using native WebSocket binary `Ping/Pong` frames, which consume near-zero CPU and network bandwidth.
* **Instantaneous Failure Detection:** Because the connection is kept alive at the TCP layer, any microservice crash or network drop results in an immediate `EOF` or `Connection Reset` exception. The controller detects failures in milliseconds.
* **Event-Driven Database Updates:** The database is only queried/updated on two events: when a service connects (Registration/Healthy) or disconnects (Deregistration/Unhealthy). The scheduler is entirely eliminated.

**Capacity:** A single `light-controller` instance (e.g., allocated 2GB of RAM utilizing non-blocking I/O like Undertow/Netty) can easily handle 100,000+ concurrent idle WebSocket connections, provided OS file descriptors are tuned appropriately.

---

## 5. Centralizing Admin Commands over WebSocket

Administrative endpoints (e.g., `reload-config`, `get-server-info`, `set-debug-level`) will be migrated from inbound REST endpoints on the microservices to the established WebSocket tunnel.

### 5.1 Architectural Benefits
* **Bypassing NAT & Firewalls (Reverse Control):** Many microservices reside behind strict firewalls or in private Kubernetes networks that cannot be accessed directly by the external controller. Because the microservice initiates the outbound WebSocket connection to the controller, the controller can push admin commands down this established tunnel, completely bypassing ingress restrictions.
* **Reduced Attack Surface:** Moving admin features to WebSocket means these sensitive endpoints are completely removed from the microservice's REST HTTP server. Malicious actors scanning the microservice directly cannot access `/admin` or `/health` routes.

### 5.2 Implementation Protocol (Multiplexed Messaging)
Because WebSocket is a raw message pipe, a multiplexed protocol (such as JSON-RPC) will be implemented to route commands and map responses. Every payload requires a **Correlation ID**.

**Example Controller Request:**
```json
{
  "id": "req-98765", 
  "action": "reload_config",
  "payload": { "module": "security" }
}
