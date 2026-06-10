# Known Vulnerabilities

This page tracks known vulnerability reports that can appear in dependency scans and require NetworkNT-specific applicability analysis.

## Undertow Request Smuggling CVEs

The following CVEs are all Undertow HTTP request smuggling issues in the HTTP/1.1 request parsing layer. They are relevant to `light-4j` because `light-4j` uses `io.undertow:undertow-core` as its embedded HTTP server. As of June 10, 2026, `light-4j` manages `undertow-core` at `2.4.1.Final`, and Maven Central still lists `2.4.1.Final` as the latest release.

| CVE | Vulnerability pattern | NetworkNT assessment |
| --- | --- | --- |
| [CVE-2026-28367](./known-vulnerabilities/CVE-2026-28367.md) | Non-standard header block terminator forwarded by certain proxies | Conditionally applicable; mitigated by strict edge proxy validation or by not placing Undertow behind an affected HTTP/1.1 proxy path. |
| [CVE-2026-28368](./known-vulnerabilities/CVE-2026-28368.md) | Header names interpreted differently by an upstream proxy and Undertow | Conditionally applicable; requires a proxy/backend parser mismatch. |
| [CVE-2026-28369](./known-vulnerabilities/CVE-2026-28369.md) | First header line with leading spaces interpreted differently | Conditionally applicable; requires malformed HTTP/1.1 traffic to reach Undertow through a permissive upstream proxy. |

## General Applicability to light-4j

These issues should not be treated as directly exploitable in every `light-4j` deployment only because `undertow-core` appears in the dependency tree. Request smuggling requires at least two HTTP processors that disagree on where a request starts, ends, or which headers are present. A normal `light-4j` service that terminates client traffic itself does not have a front-end/back-end parser disagreement in the same request path.

The risk becomes relevant when a `light-4j` service is deployed as an Undertow origin behind an upstream HTTP/1.1 proxy, load balancer, gateway, or WAF that accepts malformed requests and forwards them to Undertow in a form Undertow interprets differently. The same inbound condition applies when a `light-4j` router or proxy service is placed behind another upstream proxy. The `light-4j` proxy modules operate on Undertow's parsed `HeaderMap` and create a new outbound Undertow client request; they do not intentionally forward the original raw HTTP header block.

## Standard Mitigations

Use the following controls for customer environments where scanner findings identify these CVEs:

- Prefer the default `light-4j` posture: `enableHttps: true`, `enableHttp: false`, `enableHttp2: true`, and `allowUnescapedCharactersInUrl: false`.
- Do not expose a `light-4j` HTTP/1.1 listener directly to untrusted networks unless there is a strict edge control in front of it.
- Configure upstream proxies, load balancers, gateways, and WAFs to reject malformed HTTP/1.1 requests instead of forwarding or normalizing ambiguous header syntax.
- Avoid older proxy products or legacy load balancer modes known to forward malformed HTTP/1.1 traffic to Undertow. For CVE-2026-28367, Red Hat specifically calls out older Apache Traffic Server and Google Cloud Classic Application Load Balancer configurations as examples.
- Prefer TLS and HTTP/2 between the client-facing edge and `light-4j` where possible. If the edge terminates HTTP/2 and forwards HTTP/1.1 to `light-4j`, the edge-to-origin HTTP/1.1 hop must still enforce strict request validation.
- Keep `light-4j` and deployment images current. At the time of this review, Maven Central did not provide a newer `undertow-core` release than `2.4.1.Final`, so deployment controls are still required.

## Red Hat Status Context

Red Hat rates all three CVEs as Important with CNA CVSS 3.1 score 8.7 and high attack complexity. NVD lists a separate 9.1 Critical score, but the Red Hat CNA scoring is more specific to the proxy-dependent request smuggling condition.

Red Hat's product status is not uniform across all products. Red Hat Security Data marks some `undertow-core` product streams, including Red Hat Data Grid 8, Red Hat Fuse 7, and Red Hat JBoss Enterprise Application Platform 7, as `Will not fix`, while other product streams are marked `Affected` or `Not affected`. That status supports treating these as deployment-specific findings rather than assuming every scanner match requires an immediate library replacement.

## References

- [Maven Central undertow-core metadata](https://repo1.maven.org/maven2/io/undertow/undertow-core/maven-metadata.xml)
- [Red Hat Security Data JSON: CVE-2026-28367](https://access.redhat.com/hydra/rest/securitydata/cve/CVE-2026-28367.json)
- [Red Hat Security Data JSON: CVE-2026-28368](https://access.redhat.com/hydra/rest/securitydata/cve/CVE-2026-28368.json)
- [Red Hat Security Data JSON: CVE-2026-28369](https://access.redhat.com/hydra/rest/securitydata/cve/CVE-2026-28369.json)
