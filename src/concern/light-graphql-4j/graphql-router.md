# GraphQL Router

The `graphql-router` is the core dispatching mechanism in `light-graphql-4j`. Unlike traditional REST routers that dispatch based on URL paths, the GraphQL router sends requests to specific handlers based on the operation type (query, mutation, or subscription) and the configured endpoint.

## Core Components

### GraphqlRouter
The `GraphqlRouter` implements `HandlerProvider` and configures the `HttpHandler` instance for the server.
*   **Path Mapping**: Registers handlers for the GraphQL endpoint (default `/graphql`) and the Subscription endpoint (default `/subscriptions`) as defined in `GraphqlConfig`.
*   **WebSockets**: Configures the WebSocket protocol handshake handler for subscriptions using subprotocols like `graphql-ws`.

### Handlers
*   **GraphqlPathHandler**: Dispatches requests based on the HTTP method:
    *   **GET**: `GraphqlGetHandler` - Handles query parameters.
    *   **POST**: `GraphqlPostHandler` - Handles JSON body payloads.
    *   **OPTIONS**: `GraphqlOptionsHandler` - Handles CORS and introspection requests.

### GraphqlSubscriptionHandler
Manages WebSocket connections for GraphQL subscriptions, handling the handshake and connection callbacks.

## Configuration

This module relies on the `graphql.yml` configuration managed by `graphql-common`.

## Usage

In your project's `service.yml` (for SPI service discovery), register the router as the main handler provider if you are building a standalone GraphQL server.

**service.yml**
```yaml
singletons:
  - com.networknt.handler.HandlerProvider:
    - com.networknt.graphql.router.GraphqlRouter
```
