# WebSocket Network Topology: Direct Connection vs. Gateway Proxy

## 1. Executive Summary
As part of the `light-portal` architecture, microservices must establish a persistent WebSocket connection to the `light-controller` for real-time health monitoring and administrative commands. 

A critical networking decision is whether to route these internal WebSocket connections through the `light-gateway` (BFF) or allow microservices to connect directly to the `light-controller`. This document outlines the architectural reasoning for choosing a **Direct Connection** model, keeping the gateway out of internal management traffic.

## 2. Control Plane vs. Data Plane Separation
In robust microservice architectures, network traffic is strictly divided into two categories:
* **Data Plane (Business Traffic):** The actual user requests and business logic payload. The `light-gateway` is explicitly designed to handle this, providing routing, rate-limiting, OAuth token validation, and payload transformations.
* **Control Plane (Management Traffic):** The internal infrastructure communication required to maintain system health, configuration, and routing rules. The `light-controller` is a core component of the Control Plane.

**Architectural Principle:** Control Plane traffic should never be routed through Data Plane infrastructure if it can be avoided. If a malicious user executes a DDoS attack on the `light-gateway`, or if the gateway experiences a resource exhaustion event, the Data Plane will degrade. If health checks are routed through that same gateway, the Control Plane will also fail, blinding the `light-controller` to the actual state of the network.

## 3. The Drawbacks of Gateway Proxying

Routing WebSocket connections through the `light-gateway` introduces several severe anti-patterns:

### 3.1 The Chained Connection Problem (Resource Doubling)
Putting the gateway in the middle forces it to act as a Stateful WebSocket Proxy:
`[Microservice] <—— WebSocket A ——> [Gateway] <—— WebSocket B ——> [Controller]`

If there are 10,000 microservices, the controller holds 10,000 connections. However, the gateway must hold **20,000 connections** (10,000 inbound from services, 10,000 outbound to the controller). This wastes massive amounts of memory, CPU, and file descriptors on the gateway, which should be reserved for high-throughput business APIs.

### 3.2 Timeout Masking and Health Inaccuracy
The primary purpose of this WebSocket connection is to verify microservice liveliness via `Ping/Pong` frames. If the gateway sits in the middle, a microservice might crash silently (leaving a zombie connection). The controller remains unaware because the `Gateway-to-Controller` socket (WebSocket B) is still perfectly healthy. The gateway would need complex, custom logic to intercept, translate, and bridge Ping/Pong timeouts across two separate TCP sockets.

### 3.3 The Blast Radius of Gateway Redeployments
Gateways are frequently scaled, redeployed, or restarted as API routes change or traffic spikes occur. 
If all internal health-check WebSockets are routed through the `light-gateway`, a simple redeployment of the gateway will forcefully sever the connections for **every single microservice simultaneously**. 
This triggers a massive "Thundering Herd" reconnection event and causes the `light-controller` to falsely alert that the entire microservice ecosystem has crashed.

---

## 4. Recommended Architecture: Direct Internal Routing

Because the microservices and the `light-controller` reside within the same internal network (e.g., the same Kubernetes cluster, VPC, or private subnet), they must communicate directly.

**Implementation Details:**
1. The `light-controller` is exposed via an internal DNS record or Kubernetes Service (e.g., `light-controller.internal.svc.cluster.local`).
2. Upon startup (Stage 2), microservices resolve this internal address.
3. The microservice opens a direct TCP/WebSocket connection to the `light-controller`.
4. The `light-gateway` is entirely bypassed for this internal infrastructure communication.

## 5. When SHOULD the Gateway be Used?
The `light-gateway` (BFF) should still be utilized for the `light-controller`, but **only for external clients**. 

If human administrators are using a web-based Admin UI (SPA) running in a browser over the public internet to view the health of the network, that browser must connect to the `light-gateway`. The gateway will authenticate the external user, validate their OAuth tokens, and then proxy the REST or WebSocket request to the `light-controller`.

## 6. Conclusion
By mandating direct internal connections between microservices and the `light-controller`, the architecture drastically reduces resource consumption, eliminates complex proxying logic, prevents false-positive health alerts during gateway deployments, and strictly enforces the separation of Control Plane and Data Plane traffic.
