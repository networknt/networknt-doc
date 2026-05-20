# SSE Support

This document describes how `light-4j` should support Server-Sent Events (SSE)
for gateway and sidecar deployments where a client sends an HTTP request to an
agent or downstream API and the downstream response is streamed back as
`text/event-stream`.

## Background

SSE is a response-side streaming protocol. The client sends a normal HTTP
request and keeps the response connection open while the server writes events in
the `text/event-stream` format.

For chatbot and agent use cases, the common flow is:

```text
chatbot client -> light-gateway or sidecar -> agent service
                                      <- text/event-stream response
```

This is different from the existing `sse-handler` module. `SseHandler` opens and
owns a local SSE endpoint. When a request path matches `sse.path` or
`sse.pathPrefixes`, it creates an Undertow `ServerSentEventHandler`, registers
the connection in `SseConnectionRegistry`, and does not forward that exchange to
the next handler. It is useful when the light-4j service itself owns the SSE
connection, but it is not a downstream SSE proxy.

For downstream agent responses, the proxy and router handlers are the right
place to support SSE passthrough.

## Current Proxy Behavior

`ProxyHandler` already has the core behavior needed for SSE passthrough:

* It forwards the request to the downstream service with Undertow client APIs.
* If the incoming request body is still available, it transfers the request body
  from the client request channel to the downstream request channel.
* When the downstream response arrives, it copies the downstream status and
  headers to the client exchange.
* It transfers the downstream response channel directly to the client response
  channel with `Transfer.initiateTransfer(...)`.

Because the response body is transferred channel-to-channel, the proxy does not
need to know the full body before it starts sending data to the client. This is
the correct foundation for SSE.

## Current Gaps

### Request Timeout

`router.maxRequestTime` and `proxy.maxRequestTime` default to `1000`
milliseconds. `ProxyHandler` schedules this timeout against the entire exchange
and removes it only when the exchange completes.

An SSE response is intentionally long-lived, so a normal one-second request
timeout will close the stream. A gateway that proxies SSE must either disable
the whole-exchange timeout for that route or use a timeout long enough for the
expected stream lifetime.

There is also a current implementation risk in `pathPrefixMaxRequestTime`.
`ProxyHandler` mutates the handler field `maxRequestTime` when a prefix matches
and schedules the timeout with that value. This is unsafe in a singleton handler
because concurrent requests can affect each other. It also means a per-prefix
value of `0` is not a reliable way to disable the timeout for one path.

### Response Header Overwrite

`ProxyHandler.copyHeaders(...)` intentionally does not overwrite a response
header that is already present on the exchange.

That creates a problem when `ContentHandler` or another upstream middleware has
already set `Content-Type` to `application/json`. If the downstream agent returns
`Content-Type: text/event-stream`, the proxy may not replace the existing header.
The client then receives the wrong content type and may not treat the response
as SSE.

### Body-Consuming Middleware

`BodyHandler` reads the request body into an attachment and does not restore it
to the request channel for the proxy. For POST-based agent calls this can leave
the downstream service without the original request body.

For an SSE proxy route, middleware that consumes the request or response body
must be excluded unless it restores the stream for downstream transfer.

### Response Buffering and Transformation

`ResponseInterceptorInjectionHandler` can install a `ModifiableContentSinkConduit`
when response interceptors require content. That conduit buffers the response
until termination so interceptors can inspect or modify the body. This is
incompatible with long-lived SSE responses because the stream may not terminate
for a long time.

`DumpHandler` can also store the whole response when response dumping is enabled.
`ResponseEncodeHandler` can add compression wrappers. Both are poor defaults for
SSE passthrough.

## Design Goals

* Allow `RouterHandler` and `LightProxyHandler` to proxy downstream
  `text/event-stream` responses without buffering.
* Preserve normal proxy timeout behavior for non-streaming APIs.
* Allow per-path streaming timeout behavior without mutating shared handler
  state.
* Preserve downstream SSE headers even if upstream middleware has already set a
  default response header.
* Make safe handler-chain recommendations explicit for gateway operators.
* Keep the current `SseHandler` semantics unchanged.

## ProxyHandler Enhancements

### 1. Use Request-Local Timeout Values

Replace the mutable timeout logic with request-local variables:

```java
int effectiveMaxRequestTime = this.maxRequestTime;
long timeout = effectiveMaxRequestTime > 0
        ? System.currentTimeMillis() + effectiveMaxRequestTime
        : 0;

if (pathPrefixMaxRequestTime != null) {
    for (Map.Entry<String, Integer> entry : pathPrefixMaxRequestTime.entrySet()) {
        if (reqPath.startsWith(entry.getKey())) {
            effectiveMaxRequestTime = entry.getValue();
            timeout = effectiveMaxRequestTime > 0
                    ? System.currentTimeMillis() + effectiveMaxRequestTime
                    : 0;
            break;
        }
    }
}
```

The timeout task should be scheduled only when `timeout > 0`, and the scheduled
delay should use `effectiveMaxRequestTime`, not the mutable field.

This change makes per-path `0` mean "no whole-exchange timeout" and avoids
cross-request contamination.

### 2. Add Explicit Streaming Route Configuration

Add streaming configuration to both direct proxy and router deployments. The
fields should be exposed through `ProxyConfig`/`LightProxyHandler` and through
`RouterConfig`/`RouterHandler`, with `RouterHandler` passing the resolved values
into the shared `ProxyHandler` builder.

Suggested fields:

```yaml
streamResponseContentTypes:
  - text/event-stream

streamRequestAcceptTypes:
  - text/event-stream

streamPathPrefixes:
  - /chat
  - /agent

streamMaxRequestTime: 0
streamIdleTimeout: 0

streamResponseHeaderOverwrite:
  - Content-Type
  - Cache-Control
  - Connection
  - Transfer-Encoding
```

The path-prefix list should mark requests as streaming before the downstream
response arrives. The response content type should also be checked after
downstream headers arrive so a service can opt into streaming by returning
`text/event-stream`. Both detection modes are required: path-based detection
lets the gateway apply streaming timeout behavior before the downstream response
exists, and response-header detection handles services that declare streaming
only through `Content-Type`.

### 3. Cancel Whole-Exchange Timeout for Streaming Responses

If a timeout task has already been scheduled and the downstream response is
classified as streaming, cancel the whole-exchange timeout when the response
headers arrive.

The existing timeout attachment can be reused. A small helper can make the
behavior explicit:

```java
private static void cancelRequestTimeout(HttpServerExchange exchange) {
    XnioExecutor.Key key = exchange.getAttachment(TIMEOUT_KEY);
    if (key != null) {
        key.remove();
        exchange.removeAttachment(TIMEOUT_KEY);
    }
}
```

For streaming routes, the gateway should rely on TCP close, client disconnect,
downstream close, or a dedicated optional streaming idle timeout rather than the
normal whole-exchange request timeout. `streamIdleTimeout` should represent the
maximum silent period between downstream bytes or events. A value of `0` should
disable idle timeout enforcement.

### 4. Preserve SSE Response Headers

When the downstream response content type starts with `text/event-stream`, the
proxy should overwrite conflicting response headers that were set before proxy
response handling.

At minimum, overwrite:

* `Content-Type`
* `Cache-Control`
* `Connection`
* `Transfer-Encoding`
* `Content-Encoding`
* `Content-Length`

`Content-Length` should usually be removed for SSE because the response length is
not known up front. `Content-Encoding` should be removed unless compression is
explicitly supported and tested for streaming clients.

This can be implemented as a streaming-aware response header copy path rather
than changing the existing `copyHeaders(...)` behavior globally.

### 5. Keep Streaming Routes in a Dedicated Chain

SSE routes should avoid response buffering and transformation through handler
chain configuration. The design does not require `DumpHandler`,
`ResponseInterceptorInjectionHandler`, response transformers, response cache, or
response body interceptors to inspect a shared streaming attachment. Operators
should configure SSE paths with a narrow chain that excludes body-consuming and
response-buffering handlers.

### 6. Add Streaming Tests

Add focused tests in `proxy-handler` and `egress-router`:

* downstream returns `text/event-stream`; client receives the first event before
  the downstream stream completes.
* default non-streaming timeout behavior still works.
* per-prefix `0` disables timeout for the matched streaming route and does not
  mutate `maxRequestTime` for other requests.
* downstream `Content-Type: text/event-stream` overwrites an earlier
  `Content-Type: application/json`.
* POST request body reaches the downstream agent when body-consuming middleware
  is not in the chain.

## RouterHandler Enhancements

`RouterHandler` wraps `ProxyHandler` and builds it from `RouterConfig`.
Enhancing router support means exposing the streaming proxy settings through
`router.yml` and passing them into the proxy builder.

Suggested router configuration:

```yaml
streamResponseContentTypes: ${router.streamResponseContentTypes:["text/event-stream"]}
streamRequestAcceptTypes: ${router.streamRequestAcceptTypes:["text/event-stream"]}
streamPathPrefixes: ${router.streamPathPrefixes:}
streamMaxRequestTime: ${router.streamMaxRequestTime:0}
streamIdleTimeout: ${router.streamIdleTimeout:0}
streamResponseHeaderOverwrite: ${router.streamResponseHeaderOverwrite:["Content-Type","Cache-Control","Connection","Transfer-Encoding","Content-Encoding","Content-Length"]}
```

`RouterConfig` should parse these fields and `RouterHandler.buildProxy()` should
set them on `ProxyHandler.builder()`.

The same fields should be added to `ProxyConfig` and passed by
`LightProxyHandler` so both router-based and direct proxy deployments have the
same behavior.

## Handler Chain Recommendation

For an SSE proxy route, keep the chain narrow and avoid body or response
buffering handlers.

Recommended chain:

```yaml
handlers:
  - com.networknt.correlation.CorrelationHandler@correlation
  - com.networknt.cors.CorsHttpHandler@cors
  - com.networknt.whitelist.WhitelistHandler@whitelist
  - com.networknt.security.UnifiedSecurityHandler@security
  - com.networknt.header.HeaderHandler@header
  - com.networknt.router.RouterHandler@router

paths:
  - path: '/agent/chat'
    method: 'post'
    exec:
      - correlation
      - cors
      - whitelist
      - security
      - header
      - router
```

Use `LightProxyHandler` instead of `RouterHandler` for a simple sidecar or
single-target reverse proxy.

Safe or conditionally safe handlers:

* `CorrelationHandler`: safe.
* `CorsHttpHandler`: safe and normally required for browser clients.
* `WhitelistHandler`: safe.
* `LimitHandler`: usable, but size limits and concurrency semantics must account
  for long-lived requests.
* `UnifiedSecurityHandler`, `ApiKeyHandler`, `BasicAuthHandler`,
  `DerefMiddlewareHandler`: safe if configured for the route.
* `HeaderHandler`: safe only if it does not overwrite SSE response headers with
  non-streaming values.

Avoid these handlers on SSE proxy routes:

* `SseHandler`: owns a local SSE endpoint and does not proxy matched requests.
* `BodyHandler`: consumes POST/PUT/PATCH bodies before the proxy can forward
  them.
* `ContentHandler`: can pre-set `Content-Type` and prevent downstream
  `text/event-stream` from being copied.
* `SanitizerHandler`: depends on parsed request bodies.
* `DumpHandler`: can buffer/store the response body.
* `ResponseEncodeHandler`: can compress a stream that clients expect to receive
  as plain event-stream frames.
* `ResponseInterceptorInjectionHandler` with content-required response
  interceptors: buffers the response until termination.
* Response transformer, response cache, response filter, and response body
  interceptors when they require response content.
* Request transformer/body interceptors that read or replace the body unless
  they restore it correctly for the proxy.
* `MetricsHandler`: avoid in the dedicated SSE chain. Long-lived streams produce
  long request durations and do not provide useful per-event visibility through
  normal request metrics.

## Browser and Client Notes

The browser `EventSource` API only supports GET. If the chatbot client must send
a POST body to the agent and receive an SSE response, it should use `fetch()` or
a client library that can read the response stream incrementally.

The gateway should pass through:

* `Accept: text/event-stream`
* authorization headers
* correlation and traceability headers
* request body for POST-based agent calls

The downstream agent should return:

```http
Content-Type: text/event-stream
Cache-Control: no-cache
```

It should periodically write comments or events as keep-alives if long pauses
are possible.

## Implementation Plan

1. Fix `ProxyHandler` timeout calculation to use request-local values and allow
   per-prefix `0` to disable the whole-exchange timeout.
2. Add streaming classification configuration to both `ProxyConfig` and
   `RouterConfig`.
3. Extend `ProxyHandler.Builder` to accept streaming content types, accept
   headers, path prefixes, timeout behavior, idle timeout behavior, and header
   overwrite names.
4. Detect streaming requests before scheduling the normal timeout when possible.
5. Detect downstream `text/event-stream` responses in `ResponseCallback`.
6. Cancel the normal timeout and overwrite conflicting response headers for
   streaming responses.
7. Keep SSE paths on dedicated chains that avoid body-consuming,
   response-buffering, and metrics handlers.
8. Add targeted proxy and router tests for streaming, timeout, idle timeout,
   header overwrite, and POST body passthrough.
9. Update the `proxy-handler`, `router-handler`, and `sse-handler`
   documentation pages after implementation.

## Resolved Questions

* Streaming route configuration must be available in both `ProxyHandler` and
  `RouterHandler` paths.
* Streaming detection must use both request/path configuration and downstream
  response headers.
* The gateway should provide an optional streaming idle timeout separate from
  whole-exchange `maxRequestTime`.
* SSE routes should avoid response-buffering handlers through dedicated chain
  configuration instead of requiring those handlers to honor a shared streaming
  attachment.
* Metrics should be avoided in the dedicated SSE chain.

## Decision

Use `ProxyHandler` and its wrappers for downstream SSE passthrough. Keep
`SseHandler` as a local server-owned SSE endpoint. Enhance proxy and router
configuration to make streaming routes explicit, disable whole-exchange timeout
for those routes, add an optional streaming idle timeout, preserve downstream
SSE headers, and keep SSE routes on dedicated chains that avoid body-consuming,
response-buffering, and metrics handlers.
