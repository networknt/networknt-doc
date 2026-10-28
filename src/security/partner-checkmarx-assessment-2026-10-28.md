# Partner Checkmarx scan assessment — 2026-10-28

## Scope and conclusion

Reviewed `ghs_checkmarx_sanitized.xlsx`, sheet `ghs_checkmarx_sanitized`: **118 findings, worksheet rows 2–119, across 13 repository names**. Row numbers below are physical Excel rows (row 1 is the header), not scanner result ordinals. Workbook SHA-256: `7aa8ea5a4f4ad630f156db53d3071a1f3ed019c054c54364aa64bbb34228084d`.

This is a source-based assessment, not penetration testing or certification of the partner deployment. During the initial assessment, no application code, deployment, database or workbook was changed. Subsequent source fixes are linked in the remediation follow-up below. The workbook supplies source/sink locations and comments, but no scanned commit, dependency lock/artifact, effective middleware chain, public response captures or full taint path. Verdicts apply to the reviewed local revisions below; historical and deployed closure needs the named evidence. Scanner severity and existing Result State are inputs, not independent proof.

| Disposition | Findings | Meaning |
| --- | ---: | --- |
| Confirmed | 4 | A real source-level defect; deployment determines exposure. |
| False positive | 45 | The alleged flow or security role is unsupported for this exact result in reviewed source. |
| Conditional | 7 | A relevant primitive/control gap exists, but exploitability or deployed applicability needs evidence. |
| Hardening | 32 | Memory retention or weak nonsecurity ID generation warrants improvement; no remote exploit established. |
| Mitigated in current source | 12 | Current source has a substantive control or removed sink; scanned/deployed version still needs checking. |
| Unresolved | 18 | Required source/revision is unavailable; keep open. |

The four confirmed rows represent two root causes: DOM XSS in two demo pages (rows 34–35), and OAuth response logging at two sinks (106–107). There are also important conditional issues: legacy CBC, rendezvous channel secrecy, missing public HSTS and session identifier semantics. Do not dismiss conditional findings as false alarms.

## Merged remediation follow-up — 2026-10-28

All four confirmed findings now have merged source fixes on both `master` and `2.x`. GitHub merge and issue states were verified on 2026-10-28: all four PRs are merged and all four issues are closed. Each repository's pair of PRs resolves both of its reported findings.

| Excel row | Finding | Resolved issue | Merged master PR | Merged 2.x PR |
| ---: | --- | --- | --- | --- |
| 34 | Rendezvous demo DOM XSS (A12) | [light-gateway #258](https://github.com/networknt/light-gateway/issues/258) | [#260](https://github.com/networknt/light-gateway/pull/260) | [#261](https://github.com/networknt/light-gateway/pull/261) |
| 35 | Gateway demo DOM XSS (A12) | [light-gateway #259](https://github.com/networknt/light-gateway/issues/259) | [#260](https://github.com/networknt/light-gateway/pull/260) | [#261](https://github.com/networknt/light-gateway/pull/261) |
| 106 | OAuth token response logging (A27) | [http-client #60](https://github.com/networknt/http-client/issues/60) | [#62](https://github.com/networknt/http-client/pull/62) | [#63](https://github.com/networknt/http-client/pull/63) |
| 107 | OAuth introspection response logging (A27) | [http-client #61](https://github.com/networknt/http-client/issues/61) | [#62](https://github.com/networknt/http-client/pull/62) | [#63](https://github.com/networknt/http-client/pull/63) |

The `light-gateway` fixes replace unsafe browser HTML rendering with text nodes; the reviewed follow-up also hardens the third demo and query-value encoding. The `http-client` fixes remove credential/provider payloads from request, response, status and exception logging, including the introspection sink. Regression coverage and validation results are recorded in the linked PRs.

Merge commits: `light-gateway` master `122330692c8ff991b6cb9886d9221cd932320895`, 2.x `3f299c82d3eca1d8657bc97e5281e99e5a9278cc`; `http-client` master `fb95cb472b6ce382679efa344037e2ad2cae9a46`, 2.x `35d1364ea01fb9ac0779d0bc3b68b56d312ac356`.

The original assessment verdicts, counts and source baseline below are retained as historical evidence. Merged source remediation does not establish that the partner has deployed the repaired artifacts or rerun its scan; those remain deployment/partner closure checks. The separate MRAS logging and base-decryptor concerns are not resolved by these four PRs.

## Evidence baseline

All eleven available code repositories were clean when inspected. The documentation repository was also clean before this work. Source references in the appendix are pinned to these commits; they point to the *reported* file and line, which can have drifted. Detailed assessments name current symbols/lines.

| Repository | Reviewed HEAD |
| --- | --- |
| event-importer | `1f5279420ae2800fb6539f4bbda6d57d6283b67b` |
| http-client | `fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06` |
| light-4j | `bc3e4b5e2719d705d58a2edeabeb9bb5009c7401` |
| light-controller | `Unavailable` |
| light-gateway | `06ca1abc5a7ce84611219685fb2589fefc4f6ae3` |
| light-lambda-native | `b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7` |
| light-portal | `a8da0c88ee3fb8178dd43c2a02495d6b99084c69` |
| light-rest-4j | `0b65bfb78a15d71060d81158d29a0276ac31fae4` |
| light-session-4j | `7a20816baabfcaacae29fcf6e9111b0761ec7fbe` |
| light-spa-4j | `af06c19738a6f898201ee9f71998bfa75d9293e8` |
| oauth-kafka | `Unavailable` |
| portal-view | `c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6` |
| yaml-rule-plugin | `cb67b18deb98cfb00384c9ae7995fc24c41cf645` |

Method: parsed the XLSX ZIP/XML directly without executing macros or external links; inspected source and relevant producers/consumers; checked UUID validation, Blob construction, cookie ownership, serialization filters, crypto format branching and log sinks. GitNexus query/context assisted the favicon trace. GitNexus explain reported **no taint layer**; its empty result is not evidence of safety. No code suites, browser exploit, live endpoint tests or partner Checkmarx rerun were performed for this documentation-only assessment. Documentation validation passed: all 118 worksheet rows are mapped exactly once, all assessment anchors resolve, `git diff --check` is clean, and `mdbook build networknt-doc --dest-dir /tmp/networknt-checkmarx-book` succeeds.

## Detailed dispositions

<a id="a01"></a>

### A01: Favicon stored XSS

**False positive — Excel rows 2.**

`FaviconHandler.handleRequest` reads the fixed deployment resource `favicon.ico`, not a database or request body. Current lines 37–39 set `image/x-icon` and `X-Content-Type-Options: nosniff` before streaming bytes. No attacker-controlled resource name or HTML template is present. The claimed database-to-HTML flow is unsupported. A compromised deployment resource is a separate trust-boundary question. The workbook's claim that every InputStream passes through SanitizerHandler is not a valid explanation: inbound body sanitization does not sanitize a file response or replace output-context controls. Do not apply an HTML encoder to icon bytes. Recheck headers on the partner's scanned revision because its SHA is absent.

<a id="a02"></a>

### A02: HSTS on the favicon response

**Conditional — Excel rows 3.**

This handler does not set HSTS itself, but that does not establish the externally visible response lacks it: middleware or a TLS edge can add it. HTTPS-only origin service does not by itself prevent browser HTTP downgrade attempts. Inspect the public HTTPS response, redirect and error paths, and direct-origin access. If HSTS is absent, this is a real browser-facing configuration gap. If it is present at the edge, the missing application-local header is a defense-in-depth policy gap rather than proof of a missing browser control. First-visit protection additionally requires an appropriate preload policy; adding HSTS is not retroactive protection for an initial insecure visit. See [RFC 6797](https://www.rfc-editor.org/rfc/rfc6797). The existing `light-4j/header` HeaderHandler can add response headers via `header.response.update`; defaults leave it disabled. Prefer configuring and registering that middleware before proposing a change in each resource handler. Enable includeSubDomains/preload only after domain-wide compatibility review.

<a id="a03"></a>

### A03: Snapshot dictionary prototype mutation

**Conditional — Excel rows 4, 5.**

`snapshotComparison.ts:40–56` really assigns into ordinary `{}` dictionaries with `cells[snapshot.snapshotId]`. A synthetic ID `__proto__` with an object entry changes that dictionary's prototype; this does not automatically mutate global Object.prototype. The production API has a significant guard: `ConfigSnapshotValuesHandlerSupport:53–54` parses all requested IDs as UUIDs, and `ConfigServerQueryPersistenceImpl.loadSnapshotMetadata:812–814` reads snapshot IDs as UUIDs from the database. Those IDs cannot be `__proto__`. Thus the assignment is a real unsafe primitive, but exploitability via the supported API is not established. Worker messages originate from the application, not an arbitrary cross-origin postMessage endpoint. Close as not exploitable only after confirming the deployed producer has these UUID guarantees and no alternate unvalidated producer. A null-prototype dictionary or Map is a small future hardening option; validate IDs at any alternate entry point.

<a id="a04"></a>

### A04: Test fixtures flagged as production secrets or cookies

**False positive — Excel rows 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18.**

All twelve cookie results point to `Chat.test.tsx`, which sets simulated CSRF/owner cookies while mocking authentication and WebSockets. HttpOnly cannot be set through document.cookie; a CSRF token used by JavaScript must be readable. These results do not establish production authentication cookies lack HttpOnly. Check actual server Set-Cookie headers separately. `operationalStoreRegistrationContract.test.ts:25` supplies a dummy password specifically to assert that plaintext secret fields are rejected; it is not a production credential. Scope test files separately without suppressing equivalent findings in production code.

<a id="a05"></a>

### A05: Promotion export open redirect

**False positive — Excel rows 19, 20.**

At `PromotionExport.tsx:405–413`, `href` receives `URL.createObjectURL` for a freshly created JSON Blob and the anchor has a download filename. Query parameters affect export selection but do not become an arbitrary navigation URL. No external-origin redirect is established. Keep the Blob URL construction and revocation; revisiting this verdict is necessary if a future change assigns a query-supplied URL directly.

<a id="a06"></a>

### A06: Controller context cookie flag

**False positive — Excel rows 21.**

`ControllerContext.tsx:135–145` constructs a WebSocket MCP client and reads a CSRF cookie via universal-cookie. It does not issue an authentication cookie or a Set-Cookie response; current line 153 is a client-lifetime guard. JavaScript-readable CSRF is intentional. Verify HttpOnly/Secure/SameSite on server authentication cookies independently.

<a id="a07"></a>

### A07: Cart passcode hardcoding

**False positive — Excel rows 22.**

`CartMenu.tsx:130` is the literal API path `/portal/command`; line 152 renders `data.passCode` received from the order response. Neither is a hardcoded credential. This verdict does not certify the separate authorization or lifecycle of order passcodes.

<a id="a08"></a>

### A08: HSTS in browser automation script

**False positive — Excel rows 23.**

`scripts/mcp-schema-browser-workflow.mjs:21–26` is a Node client fetching the loopback Chrome debugging API and parsing response.json(). It is not an application HTTP server generating browser-facing responses. HSTS is not fixed by adding a header to this fetch call. Keep the debugging interface loopback-only; public site HSTS remains covered by A02.

<a id="a09"></a>

### A09: Noncryptographic correlation and idempotency IDs

**Hardening — Excel rows 24, 26, 27.**

`codingRequest.ts:11–16` prefers crypto.randomUUID, then getRandomValues, with a Math.random fallback for client message IDs. `fetchClient.ts:46` uses the fallback for JSON-RPC request IDs. `knowledgeApi.ts:41–45` uses it for idempotency keys. These values correlate requests or deduplicate commands; the reviewed clients do not use them as credentials. Predictability alone does not prove authentication bypass. Use Web Crypto consistently and remove weak fallbacks where practical. Verify server idempotency keys are scoped to authenticated owner/operation and cannot expose another user's response; collision or cross-owner reuse is a separate risk that these source lines alone do not prove.

<a id="a10"></a>

### A10: Historical ProfileMenu randomness

**Unresolved — Excel rows 25.**

The current ProfileMenu delegates login to `signIn` and contains no Math.random call. The spreadsheet does not include the scanned source or revision, so the original purpose cannot be established. Do not label a historical OAuth state/nonce generator a false positive based on its removal. Obtain the scanned function and verify whether randomness protected OAuth state, PKCE, or merely a correlation ID; security tokens require a cryptographic generator and server validation.

<a id="a11"></a>

### A11: Gateway WebSocket destination injection

**False positive — Excel rows 28, 29, 30, 31, 32, 33.**

All three pages construct the destination from window.location.protocol and window.location.host with a fixed `/chat` path. userId/model are interpolated after `?`; characters in those values can change query parameters but cannot replace the URL's authority with an attacker host. No attacker-selected WebSocket origin is established. Use URL/URLSearchParams for query encoding and require server-side authorization of user/model selection. The use of a UI-supplied userId is not proof of an authenticated identity and must never substitute for backend authentication.

<a id="a12"></a>

### A12: DOM XSS in Gateway demo pages

**Confirmed at assessment; source fixes merged — Excel rows 34, 35.**

Resolved through [issue #258](https://github.com/networknt/light-gateway/issues/258) and [issue #259](https://github.com/networknt/light-gateway/issues/259) by [master PR #260](https://github.com/networknt/light-gateway/pull/260) and [2.x PR #261](https://github.com/networknt/light-gateway/pull/261); see the row-specific issue links in the remediation table above.

Both `config/llmchat-rendezvous/public/index.html:155` and `config/llmchat-gateway/public/index.html:91` interpolate untrusted text directly into div.innerHTML in log(). WebSocket event.data and, in the rendezvous page, backend callback response text reach that sink; user-entered text reaches it too. A message containing an image element with an event-handler attribute is parsed as HTML, rather than displayed as text. Exploitation against other users requires influence over a message or callback they receive; deployment exposure and CSP affect impact. This is a real source-level defect even though these are demo assets. Replace text interpolation with textContent/createTextNode, keeping a separately created strong element for the fixed role. The ai-gateway demo already shows the createTextNode pattern. Test malicious-looking messages as literal text before serving these demos to users.

<a id="a13"></a>

### A13: Browser storage of chat session ID

**Conditional — Excel rows 36.**

`ai-gateway/public/index.html:61–66` stores json.sessionId in sessionStorage and later sends it as a reconnect parameter. A session ID used only as an owner-authorized conversation identifier is not automatically a bearer credential; sessionStorage is tab-scoped but readable by same-origin JavaScript. If possession authorizes reconnection, XSS or script compromise can disclose the capability and the finding is real. Confirm backend ownership checks, expiry and replay behavior before closing; do not claim browser storage is inherently secure or move an actual authentication secret to localStorage.

<a id="a14"></a>

### A14: Rendezvous channel randomness

**Conditional — Excel rows 37.**

The demo's generateUUID uses Math.random for channelId and puts it in `/chat` and `/connect`. This has a stronger security implication than a JSON-RPC ID: the adjacent `light-websocket-4j` WebSocketRendezvousHandler.onConnect pairs peers through a shared CHANNELS map keyed by channelId, with backend role inferred from the configured path. That method does not itself prove ownership binding. If deployed admission relies on channel secrecy, weak generation is a real security defect. Generate with Web Crypto and authenticate/authorize both peers with owner binding and expiry. Obtain the actual Gateway chain and backend `/connect` authorization before asserting hijacking is reachable; no live exploit was attempted.

<a id="a15"></a>

### A15: JDBC session deserialization

**Mitigated in current source — Excel rows 38.**

The database bytes still reach ObjectInputStream.readObject, but current `DefaultDeserializer.deserialize:101–104` first installs a class allowlist and graph limits: depth 20, references 10,000, array length 100,000 and stream bytes 1,048,576. Unknown classes are rejected. This is a substantive control the scanner's sink match misses, not evidence that arbitrary deserialization is safe. Custom allowedClasses/allowedPackages properties expand the trust boundary, and custom converters can bypass this default; inspect deployed settings and preserve least-privilege DB writes. See [Oracle ObjectInputFilter](https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/io/ObjectInputFilter.html). For an older unfiltered revision, an attacker able to alter serialized attributes plus an available gadget can make this a real vulnerability. Require a deployed-version check and rejection tests before closing the historical result.

<a id="a16"></a>

### A16: Labels and test password classified as keys

**False positive — Excel rows 39, 40, 41, 42, 44.**

SOAP PASSWORD_TYPE and DEFAULT_PASSWORD_TYPE are a configuration-field name and WS-Security type URI; actual credentials are read from configuration. AutoAESSaltDecryptor's `light_4j_config_password` is an environment/property lookup name. ManualAESSaltDescryptor's console string is a prompt. AutoAESSaltDecryptor's `light` fallback is gated by a current-thread org.junit stack check; otherwise an absent runtime secret throws. These exact flows do not establish a hardcoded production key. Separately, base AESSaltDecryptor.getPassword itself returns a fixed default; see the additional real concern below and do not confuse it with the environment-variable label.

<a id="a17"></a>

### A17: Predictable PBKDF2 salt

**Conditional — Excel rows 43.**

AutoAESSaltDecryptor supplies the password, not the salt. AESSaltDecryptor.getSecret takes salt bytes from the CRYPT value's salt field, and derives a key with PBKDF2WithHmacSHA256. A salt is public and need not be secret or unpredictable after generation; it should be unique and generated appropriately. The reported password-to-generateSecret flow does not establish a fixed salt. Audit the producing encryptor and actual ciphertext inventory for reused/static salts before closure. Do not publish decrypted values or production secrets as evidence.

<a id="a18"></a>

### A18: Legacy AES-CBC support

**Conditional — Excel rows 45.**

Current AESSaltDecryptor supports AES-GCM for four-part values, but decryptLegacyCbc:68–72 retains AES/CBC/PKCS5Padding with an all-zero IV for three-part values. This is a real legacy cryptographic weakness when that format is still used: deterministic CBC leaks equality under the same key and does not authenticate ciphertext. It is not proof that AES itself is broken or that a remotely usable padding oracle exists. Inventory formats without disclosing plaintext, regenerate legacy secrets using the current GCM encryptor, and plan an explicit compatibility cutoff. Keeping a legacy reader does not prove deployed values have migrated.

<a id="a19"></a>

### A19: Default public truststore password

**False positive — Excel rows 46.**

Http2Client.loadDefaultTrustStore:1405–1444 uses the conventional cacerts default and permits configuration/system-property overrides. A truststore of public CA certificates is not a private-key credential or remote login password, and the value is not immutable. Its integrity still depends on file permissions and controlled trust-anchor updates. If that store is repurposed to contain private keys, reassess its contents and access policy.

<a id="a20"></a>

### A20: Heap retention of credentials and key material

**Hardening — Excel rows 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 73, 76, 98, 108, 109, 110, 111.**

These rows predominantly identify real in-process secret retention, not a demonstrated remote exploit. Server/client TLS loaders, MRAS, Salesforce, Basic authentication, config bootstrap and repair utilities hold configured or request credentials as Strings; AutoAESSaltDecryptor caches a char[] and derived keys for reuse. An attacker able to read heap dumps, debugger state or process memory may recover them. Scope heap dumps/diagnostics, restrict process and filesystem access, minimize credential lifetime and rotate credentials after suspected disclosure. Where APIs allow it, use owned char[] buffers and clear temporary copies/PBEKeySpec after use; converting an existing String to char[] does not erase the original and clearing shared config arrays can break runtime behavior.

Rows 56–59 now point to char[] fields in TlsConfig, so the literal claim that those fields are immutable Strings is stale; the arrays remain retained and unzeroed for configuration reuse. Several other scan lines have moved; MRAS credentials are now at 556–575, Salesforce certPassword at 226–250, DefaultConfigLoader truststorePassword at 574–588, User BasicAuth password at 176, Lambda password at 159. Treat this as low-priority memory exposure hardening subject to the threat model, not blanket false-positive suppression or proof that all secret retention can be eliminated.

<a id="a21"></a>

### A21: SHA-1 for deterministic identifiers

**False positive — Excel rows 69, 105.**

InstanceCloneIds.uuidV5 and Cli.deterministicUuid hash namespace/name data to construct stable identifiers; they do not encrypt sensitive data, hash passwords, sign bundles or create an authentication proof. UUIDv5 intentionally uses SHA-1. SHA-1 collision resistance must not be relied on for authorization, content authenticity or adversarial uniqueness, but these results do not establish the claimed broken protection of sensitive data. Do not silently change the algorithm: stable IDs are a persistence/replay compatibility contract. Bundle signatures in EnvironmentBundleArchive use Ed25519 separately.

<a id="a22"></a>

### A22: Event names and error identifiers

**False positive — Excel rows 70, 71, 72, 89.**

PortalConstants PASSWORD_FORGOT_EVENT, PASSWORD_RESET_EVENT and PASSWORD_CHANGED_EVENT contain event type names, not credentials. Lambda INVALID_USERNAME_OR_PASSWORD is an ERR status identifier. Variable names containing password are insufficient evidence for hardcoded passwords.

<a id="a23"></a>

### A23: Heap findings on a password hash and ciphertext

**False positive — Excel rows 74, 75.**

UserPersistenceImpl:155 reads the password field from the persistence map. The standard CreateUser:116–122 and OnboardUser:98–106 command paths hash it before persistence; the field name alone does not prove plaintext retention at this sink. This does not certify every import/replay writer: verify any alternate producer supplies hashes, and protect hashes from disclosure and offline guessing. ReplayPayloadCodec.StoredPayload.ciphertext is encrypted bytes, not a plaintext password. Its separate plain field may contain sensitive event data and needs its own lifecycle review; the reported ciphertext variable is not proof of plaintext credential leakage.

<a id="a24"></a>

### A24: Repositories absent from the review workspace

**Unresolved — Excel rows 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 112, 113, 114, 115, 116.**

Neither oauth-kafka nor light-controller nor the named source files was found under /home/steve/workspace or /home/steve/networknt. No substitute repository was assumed. For oauth-kafka, literal authentication field names and error constants may be false positives, but DefaultAuthenticator's expectedPassword may be a genuine default credential; heap findings need the actual lifecycle. For light-controller, inspect whether bundled keystore material includes a shared private key and whether production can select it; distinguish a public truststore default from an actual database password. Obtain the scanned commit/artifact, source and redacted effective configuration before assigning a real/false disposition to any of these 17 rows. Do not close them from variable names or scanner comments alone.

<a id="a25"></a>

### A25: Lambda Basic authentication logging

**Mitigated in current source — Excel rows 90, 91, 92, 93, 94, 95, 96, 97.**

Current BasicAuthMiddleware.handleBasicAuth:146–200 no longer logs decoded credentials at the reported sinks. Failure messages and progress traces are fixed text; the path denial message removes CR/LF from requestPath. The scanner's duplicated Base64/substr source flows do not demonstrate password disclosure in this revision. Historical comments cannot establish what the partner runs. Verify its deployed revision and use synthetic canary credentials to assert they never appear in ERROR/DEBUG/TRACE logs before closing the original eight results. Username/path logging is a separate data-minimization consideration.

<a id="a26"></a>

### A26: PEM labels classified as key material

**False positive — Excel rows 99, 100, 101, 102, 103, 104.**

EnvironmentBundleArchive:625–640 loads key bytes from a supplied Path with Files.readString, strips PEM BEGIN/END delimiters and decodes Base64. PUBLIC KEY, PRIVATE KEY and dash delimiters describe the PEM format; they are not cryptographic key bytes. Preserve external key files and permissions; no hardcoded signing key is established by these six results.

<a id="a27"></a>

### A27: OAuth response logging

**Confirmed at assessment; source fixes merged — Excel rows 106, 107.**

Resolved through [issue #60](https://github.com/networknt/http-client/issues/60) and [issue #61](https://github.com/networknt/http-client/issues/61) by [master PR #62](https://github.com/networknt/http-client/pull/62) and [2.x PR #63](https://github.com/networknt/http-client/pull/63).

OauthHelper:670 traces the complete introspection body, which may include identity/claim data. At 836 it traces the full token responseBody before mapping it; successful responses can contain access and refresh tokens. This is a real disclosure when TRACE logging is enabled and accessible to a less-trusted reader. The scanner's specific keystore-password-to-body flow is unsupported: using a password to initialize TLS does not imply the remote response contains it. Confirm the logging defect on the actual body content, not that taint inference. Remove full-body logging and allowlist safe metadata; similarly review token request form logging at 827–828 because it serializes params. A low log level reduces exposure but is not a code fix. Test synthetic tokens/client secrets across success and error paths before closure.

<a id="a28"></a>

### A28: SPA JWT log forging

**Mitigated in current source — Excel rows 117, 118, 119.**

The current MsalTokenExchangeHandler and StatelessAuthHandler verify tokens through their configured trust providers and no longer contain the reported JWT/claim debug/trace messages around the old sinks. These scan line numbers now refer to different statements. The reported cookie-to-log path is not reproduced by current source inspection. This is a current-source mitigation, not evidence that the historical finding was false or that all logging in these classes is safe: other request-path, authorization-code/state or token-result logging deserves a separate review. Confirm the deployed artifact and exercise CR/LF-bearing rejected input with canary-only logs before closing.

## Additional concern found during review

**Fixed default in the base decryptor:** `light-4j/decryptor/.../AESSaltDecryptor.java:getPassword` returns a literal default password (current lines 140–141). Unlike the AutoAESSaltDecryptor test fallback in row 44, direct production use of this base implementation provides no deployment-specific master secret. Anyone who obtains ciphertext and knows the public default can derive its key. Confirm the configured decryptor class before closure; use an approved runtime-secret implementation such as AutoAESSaltDecryptor and re-encrypt affected values. This is an additional root cause, not an extra row in the 118-row counts.

**MRAS partial secret logging:** MrasHandler current lines 558 and 576 trace the first four characters of keystore/key/truststore passwords. This is a real disclosure when TRACE is enabled, beyond the heap-retention category reported in rows 51–53. Remove secret fragments from logs; substring truncation is not masking. Also avoid using this trace path with short passwords because substring can throw. No production credentials were accessed to establish this.

## Recommended closure sequence

1. Deploy and verify the merged Gateway text-rendering and OAuth logging fixes linked above, then obtain partner scan closure. Separately remove MRAS password-prefix logs and validate with synthetic canary secrets; that concern remains outside these PRs.
2. Inventory the configured decryptor and ciphertext formats. Replace fixed-default master secrets and regenerate legacy CBC values as authenticated GCM values through a reviewed compatibility migration.
3. Verify public HSTS using HTTPS response captures and effective application/edge configuration. Apply existing HeaderHandler configuration where application-level policy requires it; cover normal, error and favicon responses.
4. Establish whether stored session IDs and rendezvous channel IDs are capabilities; verify owner binding on reconnect and backend admission. Use cryptographic channel generation regardless. Confirm snapshot UUID validation and deployed deserialization filters/customizations.
5. Obtain the partner scan SHA/artifacts for every repository, especially missing oauth-kafka and light-controller. Re-run scan and focused regression tests on the actual repaired artifact; maintain result-specific evidence rather than category-wide suppressions.
6. Review heap-retention items under a documented threat model, prioritizing long-lived master/private-key/DB secrets over public truststore defaults, hashes and ciphertext.

For partner closure, attach the source commit/artifact digest, exact result identifier, redacted effective configuration, focused test output and deployment verification. No real secrets or access tokens should be included. Infrastructure HSTS can satisfy a browser control even when a separate partner policy still requires application defense in depth. An inbound sanitizer cannot justify unrelated output, logging, deserialization or cryptographic findings.

## Complete worksheet cross-reference

Every worksheet finding appears once below. Duplicate scanner rows remain separate for reconciliation. The Checkmarx result link/identifier remains in column P of the original workbook; it is not needed to access private partner systems for this review. Source links use the repository name as supplied by the scan; missing repositories have no claimed verified source link.

| Excel row | Repository | Query / original severity | Reported source → sink | Assessment |
| ---: | --- | --- | --- | --- |
| 2 | light-rest-4j | Stored_XSS / CRITICAL | [`/specification/src/main/java/com/networknt/specification/FaviconHandler.java:42`](https://github.com/networknt/light-rest-4j/blob/0b65bfb78a15d71060d81158d29a0276ac31fae4/specification/src/main/java/com/networknt/specification/FaviconHandler.java#L42) → `/specification/src/main/java/com/networknt/specification/FaviconHandler.java:43` | [A01](#a01) — False positive |
| 3 | light-rest-4j | Missing_HSTS_Header / MEDIUM | [`/specification/src/main/java/com/networknt/specification/FaviconHandler.java:43`](https://github.com/networknt/light-rest-4j/blob/0b65bfb78a15d71060d81158d29a0276ac31fae4/specification/src/main/java/com/networknt/specification/FaviconHandler.java#L43) → `/specification/src/main/java/com/networknt/specification/FaviconHandler.java:43` | [A02](#a02) — Conditional |
| 4 | portal-view | Prototype_Pollution / HIGH | [`/src/pages/snapshot/snapshotComparison.worker.ts:19`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/snapshot/snapshotComparison.worker.ts#L19) → `/src/pages/snapshot/snapshotComparison.ts:56` | [A03](#a03) — Conditional |
| 5 | portal-view | Prototype_Pollution / HIGH | [`/src/pages/snapshot/snapshotComparison.worker.ts:19`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/snapshot/snapshotComparison.worker.ts#L19) → `/src/pages/snapshot/snapshotComparison.ts:43` | [A03](#a03) — Conditional |
| 6 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:306`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L306) → `/src/pages/genai/Chat.test.tsx:306` | [A04](#a04) — False positive |
| 7 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:314`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L314) → `/src/pages/genai/Chat.test.tsx:314` | [A04](#a04) — False positive |
| 8 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:290`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L290) → `/src/pages/genai/Chat.test.tsx:290` | [A04](#a04) — False positive |
| 9 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:307`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L307) → `/src/pages/genai/Chat.test.tsx:307` | [A04](#a04) — False positive |
| 10 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:306`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L306) → `/src/pages/genai/Chat.test.tsx:306` | [A04](#a04) — False positive |
| 11 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:314`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L314) → `/src/pages/genai/Chat.test.tsx:314` | [A04](#a04) — False positive |
| 12 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:272`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L272) → `/src/pages/genai/Chat.test.tsx:272` | [A04](#a04) — False positive |
| 13 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:290`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L290) → `/src/pages/genai/Chat.test.tsx:290` | [A04](#a04) — False positive |
| 14 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:272`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L272) → `/src/pages/genai/Chat.test.tsx:272` | [A04](#a04) — False positive |
| 15 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:119`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L119) → `/src/pages/genai/Chat.test.tsx:119` | [A04](#a04) — False positive |
| 16 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:115`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L115) → `/src/pages/genai/Chat.test.tsx:115` | [A04](#a04) — False positive |
| 17 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/pages/genai/Chat.test.tsx:32`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/Chat.test.tsx#L32) → `/src/pages/genai/Chat.test.tsx:32` | [A04](#a04) — False positive |
| 18 | portal-view | Use_Of_Hardcoded_Password / MEDIUM | [`/src/pages/host/operationalStoreRegistrationContract.test.ts:25`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/host/operationalStoreRegistrationContract.test.ts#L25) → `/src/pages/host/operationalStoreRegistrationContract.test.ts:25` | [A04](#a04) — False positive |
| 19 | portal-view | Client_DOM_Open_Redirect / MEDIUM | [`/src/pages/promotion/PromotionExport.tsx:172`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/promotion/PromotionExport.tsx#L172) → `/src/pages/promotion/PromotionExport.tsx:408` | [A05](#a05) — False positive |
| 20 | portal-view | Client_DOM_Open_Redirect / MEDIUM | [`/src/pages/promotion/PromotionExport.tsx:172`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/promotion/PromotionExport.tsx#L172) → `/src/pages/promotion/PromotionExport.tsx:408` | [A05](#a05) — False positive |
| 21 | portal-view | HttpOnly_Cookie_Flag_Not_Set / MEDIUM | [`/src/contexts/ControllerContext.tsx:153`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/contexts/ControllerContext.tsx#L153) → `/src/contexts/ControllerContext.tsx:153` | [A06](#a06) — False positive |
| 22 | portal-view | Use_Of_Hardcoded_Password / MEDIUM | [`/src/components/Header/CartMenu.tsx:130`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/components/Header/CartMenu.tsx#L130) → `/src/components/Header/CartMenu.tsx:152` | [A07](#a07) — False positive |
| 23 | portal-view | Missing_HSTS_Header / MEDIUM | [`/scripts/mcp-schema-browser-workflow.mjs:26`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/scripts/mcp-schema-browser-workflow.mjs#L26) → `/scripts/mcp-schema-browser-workflow.mjs:26` | [A08](#a08) — False positive |
| 24 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/pages/genai/codingRequest.ts:16`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/codingRequest.ts#L16) → `/src/pages/genai/codingRequest.ts:16` | [A09](#a09) — Hardening |
| 25 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/components/Header/ProfileMenu.tsx:65`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/components/Header/ProfileMenu.tsx#L65) → `/src/components/Header/ProfileMenu.tsx:65` | [A10](#a10) — Unresolved |
| 26 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/utils/fetchClient.ts:54`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/utils/fetchClient.ts#L54) → `/src/utils/fetchClient.ts:54` | [A09](#a09) — Hardening |
| 27 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/pages/genai/knowledgeApi.ts:45`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/knowledgeApi.ts#L45) → `/src/pages/genai/knowledgeApi.ts:45` | [A09](#a09) — Hardening |
| 28 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-gateway/public/index.html:41`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-gateway/public/index.html#L41) → `/config/llmchat-gateway/public/index.html:45` | [A11](#a11) — False positive |
| 29 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-gateway/public/index.html:40`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-gateway/public/index.html#L40) → `/config/llmchat-gateway/public/index.html:45` | [A11](#a11) — False positive |
| 30 | light-gateway | Client_Resource_Injection / HIGH | [`/config/ai-gateway/public/index.html:41`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/ai-gateway/public/index.html#L41) → `/config/ai-gateway/public/index.html:51` | [A11](#a11) — False positive |
| 31 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-rendezvous/public/index.html:79`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L79) → `/config/llmchat-rendezvous/public/index.html:86` | [A11](#a11) — False positive |
| 32 | light-gateway | Client_Resource_Injection / HIGH | [`/config/ai-gateway/public/index.html:42`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/ai-gateway/public/index.html#L42) → `/config/ai-gateway/public/index.html:51` | [A11](#a11) — False positive |
| 33 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-rendezvous/public/index.html:78`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L78) → `/config/llmchat-rendezvous/public/index.html:86` | [A11](#a11) — False positive |
| 34 | light-gateway | Client_DOM_XSS / HIGH | [`/config/llmchat-rendezvous/public/index.html:124`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L124) → `/config/llmchat-rendezvous/public/index.html:155` | [A12](#a12) — Confirmed |
| 35 | light-gateway | Client_DOM_XSS / HIGH | [`/config/llmchat-gateway/public/index.html:60`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-gateway/public/index.html#L60) → `/config/llmchat-gateway/public/index.html:91` | [A12](#a12) — Confirmed |
| 36 | light-gateway | Client_HTML5_Insecure_Storage / MEDIUM | [`/config/ai-gateway/public/index.html:61`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/ai-gateway/public/index.html#L61) → `/config/ai-gateway/public/index.html:66` | [A13](#a13) — Conditional |
| 37 | light-gateway | Use_of_Insufficiently_Random_Values / LOW | [`/config/llmchat-rendezvous/public/index.html:72`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L72) → `/config/llmchat-rendezvous/public/index.html:72` | [A14](#a14) — Conditional |
| 38 | light-session-4j | Deserialization_of_Untrusted_Data / HIGH | [`/jdbc-manager/src/main/java/com/networknt/session/jdbc/JdbcSessionRepository.java:375`](https://github.com/networknt/light-session-4j/blob/7a20816baabfcaacae29fcf6e9111b0761ec7fbe/jdbc-manager/src/main/java/com/networknt/session/jdbc/JdbcSessionRepository.java#L375) → `/jdbc-manager/src/main/java/com/networknt/session/jdbc/serializer/DefaultDeserializer.java:103` | [A15](#a15) — Mitigated in current source |
| 39 | yaml-rule-plugin | Use_Of_Hardcoded_Password / MEDIUM | [`/soap-security/src/main/java/com/networknt/soap/SoapSecurityTransformAction.java:35`](https://github.com/networknt/yaml-rule-plugin/blob/cb67b18deb98cfb00384c9ae7995fc24c41cf645/soap-security/src/main/java/com/networknt/soap/SoapSecurityTransformAction.java#L35) → `/soap-security/src/main/java/com/networknt/soap/SoapSecurityTransformAction.java:35` | [A16](#a16) — False positive |
| 40 | yaml-rule-plugin | Use_Of_Hardcoded_Password / MEDIUM | [`/soap-security/src/main/java/com/networknt/soap/SoapSecurityTransformAction.java:37`](https://github.com/networknt/yaml-rule-plugin/blob/cb67b18deb98cfb00384c9ae7995fc24c41cf645/soap-security/src/main/java/com/networknt/soap/SoapSecurityTransformAction.java#L37) → `/soap-security/src/main/java/com/networknt/soap/SoapSecurityTransformAction.java:37` | [A16](#a16) — False positive |
| 41 | light-4j | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java:23`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java#L23) → `/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:96` | [A16](#a16) — False positive |
| 42 | light-4j | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/decryptor/src/main/java/com/networknt/decrypt/ManualAESSaltDescryptor.java:30`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/decryptor/src/main/java/com/networknt/decrypt/ManualAESSaltDescryptor.java#L30) → `/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:96` | [A16](#a16) — False positive |
| 43 | light-4j | Use_of_a_One_Way_Hash_with_a_Predictable_Salt / MEDIUM | [`/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java:59`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java#L59) → `/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:98` | [A17](#a17) — Conditional |
| 44 | light-4j | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java:28`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java#L28) → `/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:96` | [A16](#a16) — False positive |
| 45 | light-4j | Use_of_Broken_or_Risky_Cryptographic_Algorithm / MEDIUM | [`/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:71`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java#L71) → `/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:71` | [A18](#a18) — Conditional |
| 46 | light-4j | Use_Of_Hardcoded_Password / MEDIUM | [`/client/src/main/java/com/networknt/client/Http2Client.java:1407`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client/src/main/java/com/networknt/client/Http2Client.java#L1407) → `/client/src/main/java/com/networknt/client/Http2Client.java:1407` | [A19](#a19) — False positive |
| 47 | light-4j | Heap_Inspection / LOW | [`/server/src/main/java/com/networknt/server/Server.java:502`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/server/src/main/java/com/networknt/server/Server.java#L502) → `/server/src/main/java/com/networknt/server/Server.java:502` | [A20](#a20) — Hardening |
| 48 | light-4j | Heap_Inspection / LOW | [`/server/src/main/java/com/networknt/server/Server.java:444`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/server/src/main/java/com/networknt/server/Server.java#L444) → `/server/src/main/java/com/networknt/server/Server.java:444` | [A20](#a20) — Hardening |
| 49 | light-4j | Heap_Inspection / LOW | [`/server/src/main/java/com/networknt/server/Server.java:458`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/server/src/main/java/com/networknt/server/Server.java#L458) → `/server/src/main/java/com/networknt/server/Server.java:458` | [A20](#a20) — Hardening |
| 50 | light-4j | Heap_Inspection / LOW | [`/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:94`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java#L94) → `/decryptor/src/main/java/com/networknt/decrypt/AESSaltDecryptor.java:94` | [A20](#a20) — Hardening |
| 51 | light-4j | Heap_Inspection / LOW | [`/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java:544`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java#L544) → `/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java:544` | [A20](#a20) — Hardening |
| 52 | light-4j | Heap_Inspection / LOW | [`/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java:562`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java#L562) → `/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java:562` | [A20](#a20) — Hardening |
| 53 | light-4j | Heap_Inspection / LOW | [`/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java:543`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java#L543) → `/ingress-proxy/src/main/java/com/networknt/proxy/mras/MrasHandler.java:543` | [A20](#a20) — Hardening |
| 54 | light-4j | Heap_Inspection / LOW | [`/info-config/src/main/java/com/networknt/info/ServerInfoUtil.java:99`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/info-config/src/main/java/com/networknt/info/ServerInfoUtil.java#L99) → `/info-config/src/main/java/com/networknt/info/ServerInfoUtil.java:99` | [A20](#a20) — Hardening |
| 55 | light-4j | Heap_Inspection / LOW | [`/ingress-proxy/src/main/java/com/networknt/proxy/salesforce/SalesforceHandler.java:211`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/ingress-proxy/src/main/java/com/networknt/proxy/salesforce/SalesforceHandler.java#L211) → `/ingress-proxy/src/main/java/com/networknt/proxy/salesforce/SalesforceHandler.java:211` | [A20](#a20) — Hardening |
| 56 | light-4j | Heap_Inspection / LOW | [`/client-config/src/main/java/com/networknt/client/TlsConfig.java:97`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client-config/src/main/java/com/networknt/client/TlsConfig.java#L97) → `/client-config/src/main/java/com/networknt/client/TlsConfig.java:97` | [A20](#a20) — Hardening |
| 57 | light-4j | Heap_Inspection / LOW | [`/client-config/src/main/java/com/networknt/client/TlsConfig.java:61`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client-config/src/main/java/com/networknt/client/TlsConfig.java#L61) → `/client-config/src/main/java/com/networknt/client/TlsConfig.java:61` | [A20](#a20) — Hardening |
| 58 | light-4j | Heap_Inspection / LOW | [`/client-config/src/main/java/com/networknt/client/TlsConfig.java:88`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client-config/src/main/java/com/networknt/client/TlsConfig.java#L88) → `/client-config/src/main/java/com/networknt/client/TlsConfig.java:88` | [A20](#a20) — Hardening |
| 59 | light-4j | Heap_Inspection / LOW | [`/client-config/src/main/java/com/networknt/client/TlsConfig.java:106`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client-config/src/main/java/com/networknt/client/TlsConfig.java#L106) → `/client-config/src/main/java/com/networknt/client/TlsConfig.java:106` | [A20](#a20) — Hardening |
| 60 | light-4j | Heap_Inspection / LOW | [`/client/src/main/java/com/networknt/client/Http2Client.java:1431`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client/src/main/java/com/networknt/client/Http2Client.java#L1431) → `/client/src/main/java/com/networknt/client/Http2Client.java:1431` | [A20](#a20) — Hardening |
| 61 | light-4j | Heap_Inspection / LOW | [`/client/src/main/java/com/networknt/client/Http2Client.java:1335`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client/src/main/java/com/networknt/client/Http2Client.java#L1335) → `/client/src/main/java/com/networknt/client/Http2Client.java:1335` | [A20](#a20) — Hardening |
| 62 | light-4j | Heap_Inspection / LOW | [`/client/src/main/java/com/networknt/client/Http2Client.java:1310`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client/src/main/java/com/networknt/client/Http2Client.java#L1310) → `/client/src/main/java/com/networknt/client/Http2Client.java:1310` | [A20](#a20) — Hardening |
| 63 | light-4j | Heap_Inspection / LOW | [`/client/src/main/java/com/networknt/client/Http2Client.java:1298`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client/src/main/java/com/networknt/client/Http2Client.java#L1298) → `/client/src/main/java/com/networknt/client/Http2Client.java:1298` | [A20](#a20) — Hardening |
| 64 | light-4j | Heap_Inspection / LOW | [`/basic-auth/src/main/java/com/networknt/basicauth/BasicAuthHandler.java:175`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/basic-auth/src/main/java/com/networknt/basicauth/BasicAuthHandler.java#L175) → `/basic-auth/src/main/java/com/networknt/basicauth/BasicAuthHandler.java:175` | [A20](#a20) — Hardening |
| 65 | light-4j | Heap_Inspection / LOW | [`/server/src/main/java/com/networknt/server/UrlConfigLoader.java:363`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/server/src/main/java/com/networknt/server/UrlConfigLoader.java#L363) → `/server/src/main/java/com/networknt/server/UrlConfigLoader.java:363` | [A20](#a20) — Hardening |
| 66 | light-4j | Heap_Inspection / LOW | [`/server/src/main/java/com/networknt/server/DefaultConfigLoader.java:545`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/server/src/main/java/com/networknt/server/DefaultConfigLoader.java#L545) → `/server/src/main/java/com/networknt/server/DefaultConfigLoader.java:545` | [A20](#a20) — Hardening |
| 67 | light-4j | Heap_Inspection / LOW | [`/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java:39`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java#L39) → `/decryptor/src/main/java/com/networknt/decrypt/AutoAESSaltDecryptor.java:39` | [A20](#a20) — Hardening |
| 68 | light-4j | Heap_Inspection / LOW | [`/client/src/main/java/com/networknt/client/Http2Client.java:1407`](https://github.com/networknt/light-4j/blob/bc3e4b5e2719d705d58a2edeabeb9bb5009c7401/client/src/main/java/com/networknt/client/Http2Client.java#L1407) → `/client/src/main/java/com/networknt/client/Http2Client.java:1407` | [A20](#a20) — Hardening |
| 69 | light-portal | Use_of_Broken_or_Risky_Cryptographic_Algorithm / MEDIUM | [`/db-provider/src/main/java/net/lightapi/portal/db/clone/InstanceCloneIds.java:41`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/db-provider/src/main/java/net/lightapi/portal/db/clone/InstanceCloneIds.java#L41) → `/db-provider/src/main/java/net/lightapi/portal/db/clone/InstanceCloneIds.java:41` | [A21](#a21) — False positive |
| 70 | light-portal | Use_Of_Hardcoded_Password / MEDIUM | [`/common-util/src/main/java/net/lightapi/portal/PortalConstants.java:71`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/common-util/src/main/java/net/lightapi/portal/PortalConstants.java#L71) → `/common-util/src/main/java/net/lightapi/portal/PortalConstants.java:71` | [A22](#a22) — False positive |
| 71 | light-portal | Use_Of_Hardcoded_Password / MEDIUM | [`/common-util/src/main/java/net/lightapi/portal/PortalConstants.java:69`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/common-util/src/main/java/net/lightapi/portal/PortalConstants.java#L69) → `/common-util/src/main/java/net/lightapi/portal/PortalConstants.java:69` | [A22](#a22) — False positive |
| 72 | light-portal | Use_Of_Hardcoded_Password / MEDIUM | [`/common-util/src/main/java/net/lightapi/portal/PortalConstants.java:73`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/common-util/src/main/java/net/lightapi/portal/PortalConstants.java#L73) → `/common-util/src/main/java/net/lightapi/portal/PortalConstants.java:73` | [A22](#a22) — False positive |
| 73 | light-portal | Heap_Inspection / LOW | [`/db-provider/src/main/java/net/lightapi/portal/db/repair/MembershipProjectionRepairMain.java:23`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/db-provider/src/main/java/net/lightapi/portal/db/repair/MembershipProjectionRepairMain.java#L23) → `/db-provider/src/main/java/net/lightapi/portal/db/repair/MembershipProjectionRepairMain.java:23` | [A20](#a20) — Hardening |
| 74 | light-portal | Heap_Inspection / LOW | [`/db-provider/src/main/java/net/lightapi/portal/db/persistence/UserPersistenceImpl.java:155`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/db-provider/src/main/java/net/lightapi/portal/db/persistence/UserPersistenceImpl.java#L155) → `/db-provider/src/main/java/net/lightapi/portal/db/persistence/UserPersistenceImpl.java:155` | [A23](#a23) — False positive |
| 75 | light-portal | Heap_Inspection / LOW | [`/db-provider/src/main/java/net/lightapi/portal/db/replay/ReplayPayloadCodec.java:18`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/db-provider/src/main/java/net/lightapi/portal/db/replay/ReplayPayloadCodec.java#L18) → `/db-provider/src/main/java/net/lightapi/portal/db/replay/ReplayPayloadCodec.java:18` | [A23](#a23) — False positive |
| 76 | light-portal | Heap_Inspection / LOW | [`/db-provider/src/main/java/net/lightapi/portal/db/clone/EntityIdentityBackfillMain.java:32`](https://github.com/networknt/light-portal/blob/a8da0c88ee3fb8178dd43c2a02495d6b99084c69/db-provider/src/main/java/net/lightapi/portal/db/clone/EntityIdentityBackfillMain.java#L32) → `/db-provider/src/main/java/net/lightapi/portal/db/clone/EntityIdentityBackfillMain.java:32` | [A20](#a20) — Hardening |
| 77 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:34` → `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:34` | [A24](#a24) — Unresolved |
| 78 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:75` → `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:75` | [A24](#a24) — Unresolved |
| 79 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:76` → `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:76` | [A24](#a24) — Unresolved |
| 80 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:34` → `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:34` | [A24](#a24) — Unresolved |
| 81 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/LightPortalAuthenticator.java:78` → `/src/main/java/com/networknt/oauth/auth/LightPortalAuthenticator.java:78` | [A24](#a24) — Unresolved |
| 82 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/security/LightBasicAuthenticationMechanism.java:196` → `/src/main/java/com/networknt/oauth/security/LightBasicAuthenticationMechanism.java:196` | [A24](#a24) — Unresolved |
| 83 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:179` → `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:179` | [A24](#a24) — Unresolved |
| 84 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:49` → `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:49` | [A24](#a24) — Unresolved |
| 85 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:187` → `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:187` | [A24](#a24) — Unresolved |
| 86 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/handler/BaseWrapper.java:41` → `/src/main/java/com/networknt/oauth/handler/BaseWrapper.java:41` | [A24](#a24) — Unresolved |
| 87 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:50` → `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:50` | [A24](#a24) — Unresolved |
| 88 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/MarketPlaceAuthenticator.java:34` → `/src/main/java/com/networknt/oauth/auth/MarketPlaceAuthenticator.java:34` | [A24](#a24) — Unresolved |
| 89 | light-lambda-native | Use_Of_Hardcoded_Password / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:31`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L31) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:31` | [A22](#a22) — False positive |
| 90 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:153`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L153) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:159` | [A25](#a25) — Mitigated in current source |
| 91 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:153`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L153) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:188` | [A25](#a25) — Mitigated in current source |
| 92 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:153`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L153) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:172` | [A25](#a25) — Mitigated in current source |
| 93 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:153`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L153) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:163` | [A25](#a25) — Mitigated in current source |
| 94 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:151`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L151) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:159` | [A25](#a25) — Mitigated in current source |
| 95 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:151`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L151) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:188` | [A25](#a25) — Mitigated in current source |
| 96 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:151`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L151) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:172` | [A25](#a25) — Mitigated in current source |
| 97 | light-lambda-native | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:151`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L151) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:163` | [A25](#a25) — Mitigated in current source |
| 98 | light-lambda-native | Heap_Inspection / LOW | [`/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:158`](https://github.com/networknt/light-lambda-native/blob/b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java#L158) → `/src/main/java/com/networknt/aws/lambda/handler/middleware/security/BasicAuthMiddleware.java:158` | [A20](#a20) — Hardening |
| 99 | event-importer | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:638`](https://github.com/networknt/event-importer/blob/1f5279420ae2800fb6539f4bbda6d57d6283b67b/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java#L638) → `/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:632` | [A26](#a26) — False positive |
| 100 | event-importer | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:632`](https://github.com/networknt/event-importer/blob/1f5279420ae2800fb6539f4bbda6d57d6283b67b/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java#L632) → `/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:632` | [A26](#a26) — False positive |
| 101 | event-importer | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:638`](https://github.com/networknt/event-importer/blob/1f5279420ae2800fb6539f4bbda6d57d6283b67b/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java#L638) → `/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:632` | [A26](#a26) — False positive |
| 102 | event-importer | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:627`](https://github.com/networknt/event-importer/blob/1f5279420ae2800fb6539f4bbda6d57d6283b67b/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java#L627) → `/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:627` | [A26](#a26) — False positive |
| 103 | event-importer | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:638`](https://github.com/networknt/event-importer/blob/1f5279420ae2800fb6539f4bbda6d57d6283b67b/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java#L638) → `/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:627` | [A26](#a26) — False positive |
| 104 | event-importer | Use_of_Hard_coded_Cryptographic_Key / MEDIUM | [`/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:638`](https://github.com/networknt/event-importer/blob/1f5279420ae2800fb6539f4bbda6d57d6283b67b/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java#L638) → `/src/main/java/net/lightapi/importer/EnvironmentBundleArchive.java:627` | [A26](#a26) — False positive |
| 105 | event-importer | Use_of_Broken_or_Risky_Cryptographic_Algorithm / MEDIUM | [`/src/main/java/net/lightapi/importer/Cli.java:1556`](https://github.com/networknt/event-importer/blob/1f5279420ae2800fb6539f4bbda6d57d6283b67b/src/main/java/net/lightapi/importer/Cli.java#L1556) → `/src/main/java/net/lightapi/importer/Cli.java:1556` | [A21](#a21) — False positive |
| 106 | http-client | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/http/client/HttpClientRequest.java:384`](https://github.com/networknt/http-client/blob/fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06/src/main/java/com/networknt/http/client/HttpClientRequest.java#L384) → `/src/main/java/com/networknt/client/oauth/OauthHelper.java:836` | [A27](#a27) — Confirmed |
| 107 | http-client | Privacy_Violation / MEDIUM | [`/src/main/java/com/networknt/http/client/HttpClientRequest.java:384`](https://github.com/networknt/http-client/blob/fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06/src/main/java/com/networknt/http/client/HttpClientRequest.java#L384) → `/src/main/java/com/networknt/client/oauth/OauthHelper.java:670` | [A27](#a27) — Confirmed |
| 108 | http-client | Heap_Inspection / LOW | [`/src/main/java/com/networknt/http/client/HttpClientRequest.java:499`](https://github.com/networknt/http-client/blob/fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06/src/main/java/com/networknt/http/client/HttpClientRequest.java#L499) → `/src/main/java/com/networknt/http/client/HttpClientRequest.java:499` | [A20](#a20) — Hardening |
| 109 | http-client | Heap_Inspection / LOW | [`/src/main/java/com/networknt/http/client/HttpClientRequest.java:408`](https://github.com/networknt/http-client/blob/fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06/src/main/java/com/networknt/http/client/HttpClientRequest.java#L408) → `/src/main/java/com/networknt/http/client/HttpClientRequest.java:408` | [A20](#a20) — Hardening |
| 110 | http-client | Heap_Inspection / LOW | [`/src/main/java/com/networknt/http/client/HttpClientRequest.java:380`](https://github.com/networknt/http-client/blob/fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06/src/main/java/com/networknt/http/client/HttpClientRequest.java#L380) → `/src/main/java/com/networknt/http/client/HttpClientRequest.java:380` | [A20](#a20) — Hardening |
| 111 | http-client | Heap_Inspection / LOW | [`/src/main/java/com/networknt/http/client/HttpClientRequest.java:365`](https://github.com/networknt/http-client/blob/fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06/src/main/java/com/networknt/http/client/HttpClientRequest.java#L365) → `/src/main/java/com/networknt/http/client/HttpClientRequest.java:365` | [A20](#a20) — Hardening |
| 112 | light-controller | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/controller/tls/TlsMaterialResolver.java:17` → `/src/main/java/com/networknt/controller/tls/TlsMaterialResolver.java:17` | [A24](#a24) — Unresolved |
| 113 | light-controller | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` | [A24](#a24) — Unresolved |
| 114 | light-controller | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` | [A24](#a24) — Unresolved |
| 115 | light-controller | Heap_Inspection / LOW | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` | [A24](#a24) — Unresolved |
| 116 | light-controller | Heap_Inspection / LOW | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` | [A24](#a24) — Unresolved |
| 117 | light-spa-4j | Log_Forging / LOW | [`/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:202`](https://github.com/networknt/light-spa-4j/blob/af06c19738a6f898201ee9f71998bfa75d9293e8/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java#L202) → `/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:259` | [A28](#a28) — Mitigated in current source |
| 118 | light-spa-4j | Log_Forging / LOW | [`/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:202`](https://github.com/networknt/light-spa-4j/blob/af06c19738a6f898201ee9f71998bfa75d9293e8/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java#L202) → `/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:244` | [A28](#a28) — Mitigated in current source |
| 119 | light-spa-4j | Log_Forging / LOW | [`/stateless-auth/src/main/java/com/networknt/auth/StatelessAuthHandler.java:186`](https://github.com/networknt/light-spa-4j/blob/af06c19738a6f898201ee9f71998bfa75d9293e8/stateless-auth/src/main/java/com/networknt/auth/StatelessAuthHandler.java#L186) → `/stateless-auth/src/main/java/com/networknt/auth/StatelessAuthHandler.java:228` | [A28](#a28) — Mitigated in current source |
