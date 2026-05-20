# SSE Passthrough

SSE passthrough is used when a client sends a normal HTTP request to Light
Gateway and the downstream service, such as a chatbot agent, returns a
`text/event-stream` response. The gateway keeps the response channel open and
streams downstream events back to the client.

For the design background, see [SSE Support](../../design/sse-support.md).

This is different from the `SseHandler`. The `SseHandler` owns a local SSE
endpoint in the light-4j service. SSE passthrough is proxy behavior: the gateway
does not generate the events, it forwards the downstream agent response.

```text
chatbot client -> light-gateway -> agent service
                              <- text/event-stream response
```

## When to Use It

Use SSE passthrough for APIs where:

* The client sends a normal HTTP request, often `POST /chat` or `POST /agent`.
* The downstream service responds with `Content-Type: text/event-stream`.
* The response is long-lived and must not be buffered by the gateway.
* The same connection carries incremental tokens, status events, or completion
  events back to the client.

Do not use this feature for a gateway-owned SSE endpoint. For that case, use
`SseHandler`.

## Configuration Fields

The same streaming fields are available in both `proxy.yml` and `router.yml`.
Use the `proxy.` property prefix for `LightProxyHandler` and the `router.`
property prefix for `RouterHandler`.

| Field | Default | Description |
| :--- | :--- | :--- |
| `streamResponseContentTypes` | `["text/event-stream"]` | Downstream response content types treated as streaming responses. |
| `streamRequestAcceptTypes` | `["text/event-stream"]` | Client `Accept` media types that mark the request as expecting a stream before downstream headers arrive. |
| `streamPathPrefixes` | empty | Request path prefixes that should use streaming timeout behavior before downstream headers arrive. |
| `streamMaxRequestTime` | `0` | Whole-exchange timeout for requests classified as streaming before response headers arrive. `0` disables the normal request timeout for streams. |
| `streamIdleTimeout` | `0` | Maximum silent period between downstream streaming bytes in milliseconds. `0` disables idle timeout enforcement. |
| `streamResponseHeaderOverwrite` | `["Content-Type","Cache-Control","Connection","Transfer-Encoding","Content-Encoding","Content-Length"]` | Headers removed before copying downstream streaming response headers. |

The global `maxRequestTime` still applies to normal, non-streaming requests.

## Variant 1: Direct Proxy

Use this variant when Light Gateway or a sidecar forwards all matched requests
to configured downstream hosts through `LightProxyHandler`.

`proxy.yml`:

```yaml
enabled: true
hosts: http://agent-service:8080
maxRequestTime: 1000

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
  - Content-Encoding
  - Content-Length

metricsInjection: false
```

This configuration disables the whole-exchange request timeout for `/chat` and
`/agent` streams while keeping `maxRequestTime` at `1000` ms for normal proxy
requests.

## Variant 2: Router

Use this variant when the gateway uses `RouterHandler` for service discovery,
URL rewriting, method rewriting, or multiple downstream services.

`router.yml`:

```yaml
maxRequestTime: 1000
pathPrefixMaxRequestTime:
  /v1/report: 30000

streamResponseContentTypes:
  - text/event-stream
streamRequestAcceptTypes:
  - text/event-stream
streamPathPrefixes:
  - /v1/chat
  - /v1/agent
streamMaxRequestTime: 0
streamIdleTimeout: 0
streamResponseHeaderOverwrite:
  - Content-Type
  - Cache-Control
  - Connection
  - Transfer-Encoding
  - Content-Encoding
  - Content-Length

metricsInjection: false
```

`pathPrefixMaxRequestTime` is still useful for non-streaming long-running APIs.
For SSE endpoints, prefer `streamPathPrefixes` so the request uses streaming
timeout behavior from the beginning of the exchange.

## Variant 3: Response Header Detection Only

If the downstream agent returns response headers quickly, the default response
content-type detection may be enough.

```yaml
streamResponseContentTypes:
  - text/event-stream
streamPathPrefixes:
streamMaxRequestTime: 0
```

With this configuration, the request starts with the normal `maxRequestTime`.
When the downstream response arrives with `Content-Type: text/event-stream`, the
gateway switches to streaming response handling and preserves the downstream SSE
headers.

Use this only when the agent sends SSE response headers before the normal
request timeout expires.

## Variant 4: Client Accept Header Detection

Most browser `EventSource` clients and many chatbot clients can send:

```http
Accept: text/event-stream
```

The default `streamRequestAcceptTypes` value detects this and applies
`streamMaxRequestTime` before the downstream response headers arrive.

```yaml
streamRequestAcceptTypes:
  - text/event-stream
streamMaxRequestTime: 0
```

This is useful when the same path can return different response types and the
client explicitly asks for SSE.

## Variant 5: Path Prefix Detection

Some clients do not send an `Accept: text/event-stream` header. For chatbot
`POST` APIs, path-prefix detection is usually the safest configuration.

```yaml
streamPathPrefixes:
  - /chat
  - /v1/chat
  - /v1/agents/
streamMaxRequestTime: 0
```

Any request whose path starts with one of these prefixes is treated as a stream
candidate before downstream response headers arrive. If the downstream service
returns a JSON error response instead of SSE, the response body is still proxied
normally; the prefix only changes the timeout behavior before response headers
exist.

## Variant 6: Bounded Stream Lifetime

Use `streamMaxRequestTime` when streams must have a hard maximum lifetime.

```yaml
streamPathPrefixes:
  - /chat
streamMaxRequestTime: 300000
streamIdleTimeout: 0
```

This allows up to five minutes for requests classified as streaming before
response headers arrive. After the timeout expires, the gateway closes the
exchange. This is useful for short chatbot completions where indefinite streams
are not allowed.

## Variant 7: Idle Timeout

Use `streamIdleTimeout` when a stream may stay open for a long time but should
be closed if the downstream service stops sending bytes.

```yaml
streamPathPrefixes:
  - /chat
streamMaxRequestTime: 0
streamIdleTimeout: 60000
```

This allows an unlimited stream lifetime but closes the stream after 60 seconds
with no downstream bytes. Set this value higher than the downstream agent's
expected keep-alive or heartbeat interval.

## Header Handling

For streaming responses, the gateway removes the configured
`streamResponseHeaderOverwrite` headers before copying headers from the
downstream response. This allows the downstream `Content-Type:
text/event-stream` header to replace any earlier default such as
`application/json`.

Keep the default overwrite list unless there is a specific reason to preserve a
header.

`Content-Length` should not be used for SSE because the response length is not
known in advance. Compression is also not recommended for the SSE chain unless
it has been tested with the target clients.

## Handler Chain

SSE routes should use a narrow handler chain. Do not place body-consuming,
response-buffering, response-transforming, or response-compressing handlers in
the SSE chain.

Recommended handlers:

* `CorrelationHandler`
* `TraceabilityHandler`, if used by the deployment
* `AuditHandler`, if it does not read or buffer the response body
* `SecurityHandler` or `UnifiedSecurityHandler`
* `CorsHttpHandler`, if browser clients call the endpoint directly
* `HeaderHandler`, for request/response headers that do not conflict with SSE
* `RouterHandler` or `LightProxyHandler` as the terminal handler

Avoid these handlers for SSE routes:

* `BodyHandler`
* `DumpHandler` with response dumping
* `ResponseInterceptorInjectionHandler`
* Response transformers
* Response cache handlers
* `ResponseEncodeHandler`
* Metrics injection for the SSE chain

Example `handler.yml` chain:

```yaml
chains:
  sse:
    - correlation
    - traceability
    - security
    - cors
    - router
```

For direct proxy deployments, replace `router` with `proxy`.

## Environment Property Examples

For container deployments, the list values can be supplied as JSON-style
strings.

Direct proxy:

```yaml
proxy.streamPathPrefixes: '["/chat","/agent"]'
proxy.streamMaxRequestTime: 0
proxy.streamIdleTimeout: 60000
proxy.metricsInjection: false
```

Router:

```yaml
router.streamPathPrefixes: '["/v1/chat","/v1/agent"]'
router.streamMaxRequestTime: 0
router.streamIdleTimeout: 60000
router.metricsInjection: false
```

## Verification

A quick manual verification can be done with `curl`:

```bash
curl -N \
  -H 'Accept: text/event-stream' \
  -H 'Content-Type: application/json' \
  -d '{"message":"hello"}' \
  http://localhost:8080/chat
```

The response should include `Content-Type: text/event-stream` and events should
arrive incrementally without waiting for the downstream agent to finish the
entire response.
