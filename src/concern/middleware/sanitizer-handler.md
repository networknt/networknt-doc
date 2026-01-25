# Sanitizer Handler

The `SanitizerHandler` is a middleware component in `light-4j` designed to address Cross-Site Scripting (XSS) concerns by sanitizing request headers and bodies. It leverages the OWASP Java Encoder to perform context-aware encoding of potentially malicious user input.

### Features

- **Context-Aware Encoding**: Uses the `owasp-java-encoder` to safely encode data.
- **Header Sanitization**: Encodes request headers to prevent header-based XSS or injection.
- **Body Sanitization**: Deep-scans JSON request bodies and encodes string values in Maps and Lists.
- **Selective Sanitization**: Fine-grained control with `AttributesToEncode` and `AttributesToIgnore` lists for both headers and bodies.
- **Flexible Encoders**: Supports multiple encoding formats like `javascript-source`, `javascript-attribute`, `javascript-block`, etc.
- **Auto-Registration**: Automatically registers with `ModuleRegistry` during configuration loading for runtime visibility.

### Dependency

To use the `SanitizerHandler`, include the following dependency in your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>sanitizer</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

### Configuration (sanitizer.yml)

The behavior of the handler is managed through `sanitizer.yml`.

```yaml
# Sanitizer Configuration
---
# Indicate if sanitizer is enabled or not. Default is false.
enabled: ${sanitizer.enabled:false}

# --- Body Sanitization ---
# If enabled, the request body will be sanitized. 
# Note: Requires BodyHandler to be in the chain before SanitizerHandler.
bodyEnabled: ${sanitizer.bodyEnabled:true}

# The encoder for the body. Default is javascript-source.
bodyEncoder: ${sanitizer.bodyEncoder:javascript-source}

# Optional list of body keys to encode. If specified, ONLY these keys are scanned.
bodyAttributesToEncode: ${sanitizer.bodyAttributesToEncode:}

# Optional list of body keys to ignore. All keys EXCEPT these will be scanned.
bodyAttributesToIgnore: ${sanitizer.bodyAttributesToIgnore:}

# --- Header Sanitization ---
# If enabled, the request headers will be sanitized.
headerEnabled: ${sanitizer.headerEnabled:true}

# The encoder for the headers. Default is javascript-source.
headerEncoder: ${sanitizer.headerEncoder:javascript-source}

# Optional list of header keys to encode. If specified, ONLY these headers are scanned.
headerAttributesToEncode: ${sanitizer.headerAttributesToEncode:}

# Optional list of header keys to ignore. All headers EXCEPT these will be scanned.
headerAttributesToIgnore: ${sanitizer.headerAttributesToIgnore:}
```

### Setup

#### 1. Order in Chain
If `bodyEnabled` is set to `true`, the `SanitizerHandler` **must** be placed in the handler chain after the [BodyHandler](/concern/middleware/body-handler.md). This is because the sanitizer expects the request body to be already parsed into the exchange attachment.

#### 2. Register Handler
In your `handler.yml`, register the handler and add it to your chain.

```yaml
handlers:
  - com.networknt.sanitizer.SanitizerHandler@sanitizer

chains:
  default:
    - ...
    - body
    - sanitizer
    - ...
```

### When to use Sanitizer

The `SanitizerHandler` is primarily intended for applications that collect user input via Web or Mobile UIs and later use that data to generate HTML pages (e.g., forums, blogs, profile pages). 

**Do not use** this handler for:
- Services where input is only processed internally and never rendered back to a browser.
- Encrypted or binary data.
- High-performance logging services where the overhead of string manipulation is unacceptable.

### Query Parameters

The `SanitizerHandler` does not process query parameters. Modern web servers like Undertow already perform robust sanitization and decoding of special characters in query parameters before they reach the handler chain.

### Operational Visibility

The sanitizer module utilizes the Singleton pattern for its configuration. During startup, it automatically registers itself with the `ModuleRegistry`. You can verify the current configuration, including active encoders and ignore/encode lists, at runtime via the [Server Info](/concern/admin/server-info.md) endpoint.

### Encode Library

The underlying library used for sanitization is the [OWASP Java Encoder](https://github.com/OWASP/owasp-java-encoder). The default encoding level is `javascript-source`, which provides a high degree of security without corrupting most types of textual data.
