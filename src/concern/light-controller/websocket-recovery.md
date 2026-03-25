# WebSocket Connection Recovery Strategy

## 1. Overview
In the `light-controller` WebSocket-based architecture, a continuous and healthy WebSocket connection serves as the definitive proof of a microservice's "liveliness." If the connection drops, the controller instantly assumes the microservice is dead or unreachable. 

Therefore, a robust, autonomous connection recovery mechanism is critical. This document outlines the standard operating procedure for detecting connection drops and executing successful reconnections without overwhelming the controller.

## 2. Client-Side Responsibility
In this Control Plane / Data Plane model, **the microservice (Client) is 100% responsible for maintaining and recovering the connection.** 

The `light-controller` (Server) simply listens for connections and updates its database (or in-memory state) when a socket opens or closes. The controller will never attempt to initiate an outbound connection to a microservice.

---

## 3. Detecting Disconnections
A connection can be severed in two ways. The microservice must be equipped to handle both:

### 3.1 Clean Disconnects
When a connection is explicitly closed by the OS, a proxy, or the `light-controller` gracefully shutting down, the underlying TCP stack sends a `FIN` or `RST` packet. 
* **Detection:** The WebSocket client framework in the microservice will fire an `onClose` or `onError` event.
* **Action:** Immediately transition the internal state to "Disconnected" and initiate the Reconnection Strategy (Section 4).

### 3.2 Zombie Connections (Half-Open Sockets)
Often, a router, NAT gateway, or firewall will silently drop a connection due to a timeout or hardware failure. Neither the microservice's OS nor the controller's OS receives a termination packet. The socket remains "open" in memory, but no data can flow.
* **Detection:** The microservice must rely on **Ping/Pong frames**.
  * The microservice sends a native WebSocket `Ping` frame at a regular interval (e.g., every 10 seconds).
  * It starts a timer expecting a `Pong` response from the controller.
  * If the `Pong` is not received within the timeout threshold (e.g., 5 seconds), the connection is considered a "Zombie".
* **Action:** The microservice must **forcefully close** its own local socket to trigger an `onClose` event and initiate the Reconnection Strategy.

---

## 4. Reconnection Strategy: Exponential Backoff with Jitter

When a disconnect is detected, the microservice must **not** attempt to reconnect immediately or in a tight loop. 

### 4.1 The "Thundering Herd" Problem
If the `light-controller` experiences a brief outage or a network blip occurs, thousands of managed microservices will lose their connections simultaneously. If they all attempt to reconnect at the exact same millisecond, the resulting surge of TCP handshakes and TLS negotiations will overwhelm the controller's CPU, effectively causing a self-inflicted Distributed Denial of Service (DDoS) attack.

### 4.2 The Algorithm
To safely stagger reconnections, microservices must implement **Exponential Backoff with Jitter**.

1. **Base Delay:** Start with a baseline delay (e.g., 1 second).
2. **Exponential Multiplier:** Double the wait time after each failed attempt.
3. **Jitter:** Add a randomized number of milliseconds to the delay. This ensures that even if 1,000 services drop at the exact same time, their retry timers will naturally spread out.
4. **Maximum Cap:** Set a ceiling on the maximum wait time so services don't wait hours to reconnect.

**Example Implementation Flow:**
* **Attempt 1:** Wait 1s + random(0 to 1000ms)
* **Attempt 2:** Wait 2s + random(0 to 1000ms)
* **Attempt 3:** Wait 4s + random(0 to 1000ms)
* **Attempt 4:** Wait 8s + random(0 to 1000ms)
* **Attempt N:** Wait 30s + random(0 to 1000ms) *(Capped at 30 seconds)*

Once the connection is successfully established, the backoff multiplier resets to zero.

---

## 5. Post-Recovery: The Re-Registration Payload

Establishing the TCP/WebSocket connection is only step one of recovery. 

When a microservice disconnects, the `light-controller` treats that service as "Dead", deregisters it from active routing/monitoring, and flushes any associated state in the database. 

Therefore, **a re-established connection must be treated exactly like a brand-new application startup.**

### 5.1 The Re-Registration Step
Immediately upon the `onOpen` event firing for the new WebSocket, the microservice must push a **Registration Message** to the controller.

**Example Payload:**
```json
{
  "id": "req-startup-001",
  "action": "register",
  "payload": {
    "serviceId": "user-management-service",
    "instanceId": "pod-a2b4-xyz",
    "ip": "10.0.5.12",
    "port": 8443,
    "environment": "production",
    "version": "1.2.4"
  }
}
