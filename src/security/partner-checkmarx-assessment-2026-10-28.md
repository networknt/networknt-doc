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
| Unresolved | 18 | Technical applicability remains unverified; 17 findings concern repositories now archived (see lifecycle follow-up below). |

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

The original assessment verdicts, counts and source baseline below are retained as historical evidence. Merged source remediation does not establish that the partner has deployed the repaired artifacts or rerun its scan; those remain deployment/partner closure checks. The separate MRAS logging and base-decryptor concerns are not resolved by these four PRs; the light-4j follow-up below now records the MRAS fix and the retained base-password waiver.

## Portal-view remediation follow-up — 2026-10-28

[portal-view #939](https://github.com/lightapi/portal-view/pull/939), tracking
[issue #938](https://github.com/lightapi/portal-view/issues/938), is merged into
`master` at `48a7e027aa4fa778a8a05926301a1056ec92e088`. It addresses all six
portal-view rows that were not classified as false positives:

| Assessment | Excel rows | Merged source remediation |
| --- | --- | --- |
| [A03](#a03) | 4–5 | Null-prototype dictionaries protect regular/detail comparison construction. Tests verify dangerous keys remain own data properties through worker transfer; null prototypes themselves do not survive structured clone. |
| [A09](#a09) | 24 | Chat message IDs require Web Crypto, using native UUIDs or getRandomValues. Missing crypto preserves the draft and displays an error, including during expired-session renewal. |
| [A09](#a09) | 26 | JSON-RPC correlation IDs use a per-page sequence without Math.random or a Web Crypto dependency. These are not credentials or idempotency keys; explicit caller IDs are preserved. |
| [A09](#a09) | 27 | Knowledge command idempotency keys require the shared cryptographic helper; missing crypto rejects before dispatch. Ordinary updates do not request an idempotency key. |
| [A10](#a10) | 25 | Repository history establishes that the old ProfileMenu generator protected OAuth state. Sign-in now uses the shared cryptographic helper, and the menu shows an accessible, retryable failure. Integration tests exercise the real state storage/redirect flow. |

Local validation on the reviewed head `7a8a8cd`: **1,153 tests in 166 files
passed**, as did the production build, lint (zero errors, 215 baseline warnings)
and controller typecheck. Full-repository TypeScript checking still reports 172
baseline errors, with no new diagnostics after accounting for shifted line
numbers. The GitHub `portal-control-plane` check passed on that head before merge.
The [repository follow-up](https://github.com/lightapi/portal-view/blob/48a7e027aa4fa778a8a05926301a1056ec92e088/docs/security/partner-checkmarx-938.md)
records scope, historical evidence and deployment acceptance criteria.

These are merged frontend changes and regression evidence. They do not establish
partner deployment, live OAuth/server state validation or backend cross-owner
idempotency isolation. The exact partner scan revision and rescan remain
outstanding. Other UUID call sites are outside this migration. The initial
assessment verdicts/counts below remain historical; row 25's original uncertainty
about purpose is now resolved by repository history, without reclassifying the
historical weak OAuth state generator as a false positive.

## Gateway A13/A14 remediation follow-up — 2026-10-28

**Superseded lifecycle follow-up:** the examples described below were subsequently
retired by [master #265](https://github.com/networknt/light-gateway/pull/265) and
[2.x #266](https://github.com/networknt/light-gateway/pull/266). See the
[retirement record](#gateway-retirement-2026-10-29). The browser fixes and residual
analysis below remain historical evidence for surviving deployed copies.

Both remaining non-false-positive light-gateway findings after A12 have merged
browser-source fixes:

| Branch | Merged PR | Merge commit |
| --- | --- | --- |
| master | [#262](https://github.com/networknt/light-gateway/pull/262) | `dc8347a4a6f635d7fdd387d5c10bfcbfbdf68cc0` |
| 2.x | [#263](https://github.com/networknt/light-gateway/pull/263) | `b63ec10570c0bf47311813102e00dcaed053182d` |

**A13 / Excel row 36:** the canonical AI page and its embedded Kubernetes ConfigMap
copy keep reconnect IDs in page memory, reuse them only for the same selected
user/model, and no longer display them. Both storage types receive best-effort
cleanup of the legacy sessionId key when accessible; blocked storage cannot be
erased until a later load with access. Reconnect resets the assistant bubble,
disables sending until OPEN and ignores replaced-socket events. Reload starts a
new conversation. These client-side selection checks are not owner authentication.

**A14 / Excel row 37:** rendezvous channels use native crypto.randomUUID or
UUID-v4 bytes from getRandomValues; missing/failing crypto reports an error before
socket/callback creation. The same encoded channel is used for both requests,
and direct channel/URL logging is removed. Replacing a connection closes the old
socket, aborts its callback request and ignores late events or callback results.
Callback abort cannot guarantee that the backend stopped processing a request.

Local validation passed on both reviewed heads (`3d2bcec` / `6b7c869`): **7 tests,
including 111 browser checks across six runs and a canonical/ConfigMap parity
check**. Tests cover both AI delivery paths with and without storage, asynchronous
close ordering, stream separation, send-state guards, late callbacks, cryptographic
channel generation and existing XSS protections. Negative controls reject the
previous affected PR versions. Browser regressions and pre-commit passed on both
GitHub PR heads. At this update, master's Java source-build check is still running;
no completed master build/distribution result is claimed. Java code was unchanged
and local Maven suites were not run for these browser-only changes.

**Residual protocol/deployment work:** the reconnect session ID still travels in
the existing URL query and can enter gateway/proxy access logs. Memory-only storage
does not fix that exposure. Redact sensitive values; moving resume credentials off
the URL requires a coordinated backend protocol change. Same-origin script
compromise can still access page memory. Verify authenticated session ownership,
peer admission, expiry/replay enforcement and HTTPS/WSS against the deployed
backend, then rerun the partner scan. A13/A14 remain conditional for deployment
closure; neither browser tests nor random channel IDs establish those controls.
See the [merged security guide](https://github.com/networknt/light-gateway/blob/dc8347a4a6f635d7fdd387d5c10bfcbfbdf68cc0/docs/security/chat-demo-session-channel.md).

<a id="gateway-retirement-2026-10-29"></a>

## Gateway AI demo retirement — 2026-10-29

The owner chose retirement of the examples backed by archived light-genai-4j,
rather than a new backend resume protocol. Tracking:
[light-gateway #264](https://github.com/networknt/light-gateway/issues/264).
GitHub confirms both PRs merged on 2026-10-29:

| Branch | Merged PR | Final reviewed head | Merge commit |
| --- | --- | --- | --- |
| master | [#265](https://github.com/networknt/light-gateway/pull/265) | `72de3eeaf513eb63b8f45ccd0184afb4aa3d56b9` | `e759980ef240ef307df0127545fe1554bd5b4461` |
| 2.x | [#266](https://github.com/networknt/light-gateway/pull/266) | `13ca9075aa146104a11b0447cfcad161cbc27df5` | `35f743bd0f267f4ac8ff533caa92cebe013462ab` |

The source removal covers config/ai-gateway, config/llmchat-gateway and
config/llmchat-rendezvous, their demo configuration/fixture stores, the embedded
Kubernetes AI page and its public ConfigMap/volume mount, and the browser-only
regression suite/workflow whose subjects were removed. These examples contain the
A12/A13/A14 entry points (Excel rows 34–37). General gateway/WebSocket Java
implementations are unchanged; no fix or protocol migration was published to the
archived backend. This is retirement evidence, not a new spreadsheet finding.

The Kubernetes gateway template remains available without bundled public UI.
Historical startup.serviceId and sample name/instanceId defaults are deliberately
preserved. They can still select old config-server values, so deployments must
review their explicit identity and configuration. The retired local credential
values.yml path remains ignored. The new
[retirement guide](https://github.com/networknt/light-gateway/blob/e759980ef240ef307df0127545fe1554bd5b4461/docs/security/ai-demo-retirement.md)
provides cleanup/migration instructions; the old security-guide URL remains a
historical compatibility link.

Local validation rendered and parsed all four remaining Kubernetes resources on
each branch with disposable inputs. Volume and ConfigMap/Secret references
resolved, identity defaults matched the pre-removal template, the legacy local
values path remained ignored, and retired assets/workflow references were absent.
Pre-commit and diff checks passed. Java code was unchanged; no new Java or live
cluster qualification is claimed for this removal.

**Deployment closure:** merging or pulling does not remove copied/cached pages,
old public ConfigMaps, config-server routes/values, ignored workstation files or
existing backend deployments. Retire those copies and confirm the affected public
chat/callback endpoints are no longer exposed. For any deliberately retained
custom deployment, the original ownership/admission, expiry/replay, URL logging
and HTTPS/WSS checks still apply. Attach retirement/artifact evidence or a partner
rescan before closing A12/A13/A14; original worksheet verdicts/counts remain
historical.

## JDBC session A15 remediation follow-up — 2026-10-28

The re-evaluation of **A15 / Excel row 38** confirmed that the default allowlist
blocks unlisted deserialization hooks, but found a resource-limit gap: a serialized
2 MiB string (2,097,165 bytes) and roughly 4 MiB string array bypassed the intended
1 MiB filter streamBytes check. The same string was accepted through the repository
conversion path. This demonstrated a missing byte bound, not deployed RCE or DoS.

Both source-fix PRs are merged:

| Branch | Merged PR | Merge commit |
| --- | --- | --- |
| master | [#68](https://github.com/networknt/light-session-4j/pull/68) | `6be0e0c92da1b11e0e6bcc381e0e88e0b75f431a` |
| 2.x | [#69](https://github.com/networknt/light-session-4j/pull/69) | `852eaf3f7de51100121e2f85bb195323c5f46709` |

The reader now pre-validates at most limit + 1 serialized bytes before constructing
ObjectInputStream; the converter also bounds writes. The shared default is 1 MiB
per attribute, configurable at instance construction using
`light.session.jdbc.serialization.maxAttributeBytes`. Invalid settings fail
construction; raising the limit requires a reviewed migration and heap budget.
The class allowlist and depth/reference/array filters remain intact.

JDBC loading uses getBinaryStream and hydrates without writes or reserialization.
A rejected attribute excludes its entire session, while healthy sessions remain
available in getSessions(); findById returns null for that rejected session.
Close-only stream errors do not discard successfully decoded data. Saves preflight
pending attributes before SQL, propagate failure, attempt explicit rollback and
retain pending edits until correction/retry. Flags clear after confirmed commit;
a failed commit can still have an uncertain database outcome requiring reconciliation.

Each branch passed the local full suite: **37 tests, zero failures/errors, two
existing skips**, including public H2 save/load/rollback regressions. All five new
public repository tests fail on the prior implementation. Pre-commit passed on
both reviewed PR heads (`39f2e68` / `b2ad19b`); no remote Maven test job was listed.
See the [merged JDBC guide](https://github.com/networknt/light-session-4j/blob/6be0e0c92da1b11e0e6bcc381e0e88e0b75f431a/jdbc-manager/README.md)
for compatibility, failure semantics and migration instructions.

**Remaining boundaries:** this is a per-attribute limit, not an attribute-count,
per-session aggregate/decoded-heap or listing-wide budget. getSessions() materializes
valid results and write preflight buffers pending attributes. JDBC drivers may
buffer columns internally; PostgreSQL/Oracle production-driver behavior was not
qualified by H2 tests. Review database constraints, allowlist/filter-factory
settings, deployed request-error handling and actual artifacts, then rerun the
partner scan. Merged source safeguards do not establish deployment closure or
complete resource-exhaustion protection. Initial assessment counts remain historical.

## Light-4j A17/A18/A20 remediation follow-up — 2026-10-28

Both PRs tracking [light-4j #2828](https://github.com/networknt/light-4j/issues/2828)
are merged. GitHub merge metadata and containment of the final reviewed heads
in the fetched target branches were verified after merge:

| Branch | Merged PR | Final reviewed head | Merge commit |
| --- | --- | --- | --- |
| master | [#2829](https://github.com/networknt/light-4j/pull/2829) | `174cbe5cfb943e99b7a3f2edfbbaed0908598847` | `62b06f43d7ae505e68b8cae5ebf60b70946443f8` |
| 2.x | [#2830](https://github.com/networknt/light-4j/pull/2830) | `2b77619bc6831dc0a4ca039ec4995b68bff1f588` | `05a86a5d9510f704168d58f3817d9a2514ae178c` |

**A17 / Excel row 43:** the decryptor reads the salt from ciphertext; the master
password is not the salt. Source inspection of the current light-encryptor producer
found fresh random 16-byte salts and 12-byte GCM nonces for each encryption. The
reader now validates these lengths, hex fields and a minimum authentication-tag
length before requesting a password or running PBKDF2. This qualifies the original
scanner interpretation; it does not establish how historical/deployed ciphertext
was generated. Inventory the actual producing version and salt reuse before closure.

**A18 / Excel row 45:** both branches reject three-part zero-IV CBC values before
password acquisition. There is no legacy opt-in or warn-first transition on 2.x.
Users must re-encrypt those secrets to standard four-part AES-GCM before upgrading;
nonstandard four-part lengths also require regeneration with the supported producer.
YAML failures report source line/column and the config loader supplies the filename,
without echoing the value. Only a three-part rejection advises legacy re-encryption;
wrong-password/tampered GCM failures retain general password/format guidance.
Standard GCM values retain the PBKDF2-HMAC-SHA256 parameters (65,536 iterations,
256-bit key) and 128-bit authentication tag. The legacy reader is removed in source;
partner deployment and migration remain unverified.

**A20 / light-4j Excel rows 47–68:** scoped hardening removes the base/Auto derived-key
cache and Auto master-password array cache, clears owned temporary password/PBE/key/
plaintext buffers, and clears temporary TLS password copies across the reviewed
server, client, MRAS, Salesforce, bootstrap, fingerprint and JwtIssuer paths.
JwtIssuer also closes the keystore stream. Arrays returned by downstream password
providers remain borrowed unless explicitly registered as newly owned; shared
configuration arrays are preserved. TokenKeyStoreManager already accepts borrowed
char[] inputs and makes no toCharArray copies; an ownership test protects that
contract. These changes do not cover A20 rows in other repositories or eliminate
immutable configuration/request Strings, provider-internal keys or retained
TlsConfig/authentication credentials.

The interactive Manual provider retains at most 32 successfully authenticated
derived keys per instance with a five-minute reuse window, avoiding a prompt for
every repeat decrypt. Failed authentication does not populate the cache. Expiration
is lazy on the next key acquisition; idle keys can remain beyond five minutes.
Eviction and explicit close clear cache-owned key bytes. Applications must arrange
close at the end of the manual configuration session; use a new instance after a
master-password change. Base/Auto deliberately repeat PBKDF2 instead of retaining
keys. A local Java 25.0.3 repeated-value probe measured about 10.69 ms/decrypt versus
0.073 ms with the old warm cache; this is a measured cost, not a capacity qualification.

The additional **MRAS password-prefix logging** concern is fixed in these PRs.
TRACE no longer logs fragments, and tests exercise actual key/trust-store loading,
success/failure paths, short passwords and canary checks over log messages,
arguments, throwable proxies and MDC.

**Validation:** the expanded focused suite passed on both review heads
(`ff11492` / `135594f`): 35 tests, zero failures/errors, one existing disabled test.
Clean affected-module/dependency reactors passed 969 tests on master and 921 on 2.x,
with zero failures/errors and 53 skips each. Four client classes were explicitly
excluded after the original unrestricted run stalled; these are selected reactor
results, not full-suite qualification. Later diagnostic tests passed (two per branch),
and all 18 tests affected by the final Sonar cleanup passed on each final head.
The broader reactor was not rerun for those diagnostic/test-structure/comment changes.
Final-head SonarCloud analyses reported zero issues and passed quality gates on both
PRs; pre-commit also passed. This follow-up does not claim completed final-head
source-build/native-binding CI or deployed/partner qualification.

**Accepted residual concern:** the public base-decryptor password default remains
by explicit compatibility waiver in #2828. It is unsuitable as a production master
secret. Password-provider selection and the Auto JUnit fallback remain unchanged;
production should use an approved runtime-secret provider and re-encrypt any values
protected with the public default. The measured PBKDF2 cost is also retained and
documented, rather than hidden by a persistent base/Auto key cache.

See the [merged migration guide](https://github.com/networknt/light-4j/blob/62b06f43d7ae505e68b8cae5ebf60b70946443f8/decryptor/MIGRATION.md)
for both branches' format cutoff, credential ownership and manual-cache lifecycle.
Verify the configured decryptor, actual ciphertext inventory, deployed artifacts,
remaining memory-retention boundaries and partner rescan. A17 and deployment closure
remain conditional; A20 remains scoped hardening. Original verdicts/counts below
are historical and are not reclassified wholesale by these merges.

<a id="a20-follow-up-2026-10-29"></a>

## Portal, Lambda and HTTP-client A20 remediation follow-up — 2026-10-29

The following five PRs are merged. GitHub merge metadata, merge-commit containment
in the fetched target branches, and matching full trees between the final reviewed
heads and merge commits were verified on 2026-10-29. The tree comparison also
covers merges whose rewritten history does not contain the original review head.
The light-4j A17/A18/A20 PRs #2829/#2830 remain recorded in the preceding follow-up.

| Repository / tracking issue | Branch | Merged PR | Final reviewed head | Merge commit |
| --- | --- | --- | --- | --- |
| [light-portal #878](https://github.com/lightapi/light-portal/issues/878) | master | [#879](https://github.com/lightapi/light-portal/pull/879) | `5259bd1e905d10f744b75b9bdce1f6e64a6fd411` | `350a38e0790855a28975b64cd6baed0fa7cf9885` |
| [light-lambda-native #191](https://github.com/networknt/light-lambda-native/issues/191) | master | [#192](https://github.com/networknt/light-lambda-native/pull/192) | `6d3b1fff3f9105e70b2331223330843b8f4bb64d` | `68fc8492ef54f7cc2a8e1d1782316064e75234e4` |
| [light-lambda-native #191](https://github.com/networknt/light-lambda-native/issues/191) | 2.x | [#193](https://github.com/networknt/light-lambda-native/pull/193) | `75c712f90571e1851797b949fd8256d38ba740d2` | `1196b014492cf03b2e97c1e55c220ec0af4b0fec` |
| [http-client #64](https://github.com/networknt/http-client/issues/64) | master | [#65](https://github.com/networknt/http-client/pull/65) | `cfe38fae68e04c8b85fe3fbe3c605909428e1a46` | `3d07a4c92d828ab1166c5da24a8078787e1acbff` |
| [http-client #64](https://github.com/networknt/http-client/issues/64) | 2.x | [#66](https://github.com/networknt/http-client/pull/66) | `aa22a0e767d8fdab0529a5ca2da498115e5e7401` | `a6504d13c76fb019ceac70defc423505b221ed07` |

### Light-portal — A20 / Excel rows 73 and 76

MembershipProjectionRepairMain and EntityIdentityBackfillMain now acquire a direct,
non-pooled JDBC connection instead of constructing Hikari pools that retain
additional password references. Optional protected password files take precedence;
nonblank invalid file settings fail without falling back, while absent/blank file
settings retain the environment-password fallback. Reads are bounded, use strict
UTF-8 with leading-BOM/trailing-newline handling, and support secret-file symlink
rotation. Owned file byte/character buffers are cleared. POSIX permission warnings
name only the input variable, and sanitized connection errors retain class names
and validated SQL metadata without raw driver messages or duplicate logging.

Connection acquisition has a 30-second caller deadline, adjustable with
`MEMBERSHIP_REPAIR_DB_LOGIN_TIMEOUT_SECONDS` or
`ENTITY_CREATION_DB_LOGIN_TIMEOUT_SECONDS`. Credentials remain owned by the worker until
the driver finishes; Properties are then cleared and late connections are closed.
An uncooperative driver can outlive the caller deadline and retain credentials in
the meantime. JDBC password Strings, environment/configuration values and driver
copies remain outside erasure guarantees. SQL, transaction/fencing rules, report
formats and report/apply defaults are unchanged.

Recorded final-head local validation: **28 focused tests passed**, zero failures,
errors or skips, covering both CLI entry points with H2, file precedence/encoding/
rotation, owned-buffer/Properties cleanup, timeout/late-connection handling and
sanitized diagnostics. Pre-commit passed. These are not PostgreSQL production-driver
or live repair/backfill results. Packaged adoption requires rebuilding the hybrid
command/query images that include db-provider, rather than only replacing thin
service jars. See the [merged operator guide](https://github.com/lightapi/light-portal/blob/350a38e0790855a28975b64cd6baed0fa7cf9885/doc/security/operator-jdbc-credentials.md).

### Light-lambda-native — A20 / row 98; A25 / rows 90–97

The A25 re-evaluation confirmed that the reported credential/password logging paths
were already mitigated on both branch baselines; the new TRACE privacy test also
passed against the original implementations. The source fix addresses A20 and adds
regression protection for A25. Configured username and sanitized path diagnostics
remain; this is scoped protection of submitted passwords/headers, not removal of
all identity data from logs.

Local Basic authentication avoids immutable decoded credential-pair/password
Strings. Request-owned decoded/password bytes, comparison copies and temporary
conversion buffers are cleared on returns and exceptions. Explicit encoder/decoder
output arrays avoid discarded buffers during charset resizing. Unicode scheme
matching, raw/encoded credentials, password colons, historical malformed-UTF-8
replacement behavior, statuses and LDAP dispatch remain compatible. The existing
LDAP API still requires an immutable password String; borrowed request headers,
configuration credentials, provider copies and JVM state may retain secrets.

Recorded final-head local validation: **34 focused tests passed on each branch**,
zero failures, errors or skips; pre-commit passed. Tests observe cleanup through
real middleware return/exception paths, use mocked LDAP, and check TRACE log
messages, arguments, MDC, throwable chains and returned statuses for full encoded
credentials and distinct password fragments. Negative controls caught parsing,
normalization, cleanup and logging regressions. No live LDAP or native/AWS deployment
qualification is established. Rebuild and distribute the native Lambda artifact
for adoption. See the [merged Basic Auth guide](https://github.com/networknt/light-lambda-native/blob/68fc8492ef54f7cc2a8e1d1782316064e75234e4/doc/security/basic-auth-credentials.md).

### HTTP-client — A20 / Excel rows 108–111

The client-keystore, private-key, configured-truststore and default/JDK-truststore
handoffs now clear owned char arrays in finally blocks, including provider errors.
The cleanup uses JDK APIs directly and adds no new minimum provided-config version
or PasswordChars dependency. Configuration/system-property precedence, required/null
password behavior, TLS policy, trust-manager composition and SSLContext caching
remain unchanged for the qualified JDK providers. Borrowed Strings and
provider-internal copies can remain in memory.

Early clearing assumes KeyStore.load and KeyManagerFactory.init consume or copy
the caller's array before returning. Providers that retain that reference for
later use would see zeros. PKCS11, FIPS, builder-based and other custom providers
need provider-specific ownership/lifecycle qualification; no universal JCA ownership
guarantee is claimed.

Recorded final-head local validation: **34 focused tests passed on each branch**,
zero failures, errors or skips: 13 TLS ownership/isolation tests and 21 existing
OAuth logging regressions. Pre-commit passed. Checked-in disposable JKS fixtures
remove the keytool subprocess requirement. Full-context tests observe all four
copies with default plus configured trust, check both issuer contributions to the
composite trust manager, and cover default-store failure. Isolated/same-thread
execution, a system-properties lock and an AutoClose state guard protect global
state, including an injected cleanup failure. A compilation probe with PasswordChars
removed from a copied config jar passes for both repaired loaders and fails for
the previous PR heads. See the [merged TLS ownership guide](https://github.com/networknt/http-client/blob/3d07a4c92d828ab1166c5da24a8078787e1acbff/doc/security/tls-password-buffers.md).

These are verified source merges and recorded focused JVM results. They do not
establish new remote-CI/full-suite/native/distribution qualification, partner
artifact adoption or a rescan. Confirm deployed artifacts, rebuild requirements,
secret sources and logging/memory controls, and qualify actual JDBC/LDAP/TLS
providers. A20 remains scoped hardening and A25 remains mitigated in reviewed
source; the original worksheet verdicts/counts and pinned source mappings below
are retained as historical evidence.

<a id="a28-follow-up-2026-10-29"></a>

## Light-spa-4j A28 remediation follow-up — 2026-10-29

For **A28 / Excel rows 117–119**, [light-spa-4j #188](https://github.com/networknt/light-spa-4j/issues/188)
tracks the branch re-evaluation and regression protection. GitHub confirms both
PRs merged on 2026-10-29:

| Branch | Merged PR | Scope | Final reviewed head | Merge commit |
| --- | --- | --- | --- | --- |
| master | [#189](https://github.com/networknt/light-spa-4j/pull/189) | Regression tests and documentation; production source already mitigated | `d00260281168210d9ca2202fc9598de179448bd9` | `d4b1c0d3cd984272de85c91039d931ccb1d71b0a` |
| 2.x | [#190](https://github.com/networknt/light-spa-4j/pull/190) | Remaining session logging sinks and rejection/callback handling | `694f9860956af0100dbf96f3cf8e0d35376f5dd0` | `277feab66218932d2e8518114a1957cc8454a03c` |

The original mitigated verdict applies to the reviewed master baseline, not every
branch. Re-evaluation found remaining JWT/CSRF diagnostics on 2.x. Its source fix
removes those logs, redacts CSRF mismatch arguments, and removes the callback
result/code/state and Facebook access-token logs identified during review.
Configured status.yml entries are retained: CSRF mismatch ERR10039 defaults to
HTTP 400, missing CSRF retains ERR10036, malformed tokens use ERR10000, and expired
tokens use ERR10001. Key-resolution failures emit a fixed diagnostic with the
exception class, without provider messages or causes. Microsoft-token key failures
now return a logged 500 instead of the previous 401, deliberately matching the
cookie path. Callback and renewal callers stop after failed cookie issuance.
This does not backport master's broader trust/CSRF redesign.

Recorded final-head local validation: **13 logging probes on master and 23 on
2.x passed**. The selected affected reactors reported **151 tests on master and
77 on 2.x**, zero failures/errors and one existing skip each. Two legacy Azure
tests were explicitly excluded on 2.x because their fixtures name Microsoft
endpoints. Tests pin error bodies and inspect messages, arguments, MDC, throwable
chains, responses and escaped exceptions at TRACE. The 2.x probes exercise the
real verifier's cache-hit path, and local TLS integration tests exercise actual
authorization/exchange callbacks with rejected issued tokens. Logging-only
negative controls preserve HTTP behavior and fail both handlers' successful-session
probes on credential canaries. The earlier whole-handler restore experiment was
confounded by status changes and is not evidence of 15 independent logging
detections.

The duplicated module-local harness remains intentionally documented. See the
[master regression guide](https://github.com/networknt/light-spa-4j/blob/d4b1c0d3cd984272de85c91039d931ccb1d71b0a/docs/security/partner-checkmarx-a28.md)
and [2.x remediation guide](https://github.com/networknt/light-spa-4j/blob/277feab66218932d2e8518114a1957cc8454a03c/docs/security/partner-checkmarx-a28.md).
These source merges and recorded local results do not establish live identity-provider,
production logging, full-suite/remote-CI or partner deployment qualification.
The adjacent callback/provider diagnostics were subsequently reviewed separately
in [issue #191 and PRs #192/#193](#spa-callback-follow-up-2026-10-29). Raw transport
logging and deployment checks remain separate. Verify adopted artifacts and
logging configuration before partner closure. Original
worksheet counts and pinned appendix mappings remain historical evidence.

<a id="spa-callback-follow-up-2026-10-29"></a>

## SPA callback/provider privacy follow-up — 2026-10-29

The separate [light-spa-4j #191](https://github.com/networknt/light-spa-4j/issues/191)
review addresses callback/provider diagnostics explicitly deferred from A28.
It supplements, rather than replaces, issue #188 and PRs #189/#190. GitHub confirms
both follow-up PRs merged on 2026-10-29:

| Branch | Merged PR | Final reviewed head | Merge commit |
| --- | --- | --- | --- |
| master | [#192](https://github.com/networknt/light-spa-4j/pull/192) | `a7af7697a3195fa2ae3151c876381fbcaa6fb925` | `9227ce42b772311cb3901269db83e58e84ebf2eb` |
| 2.x | [#193](https://github.com/networknt/light-spa-4j/pull/193) | `fac6024c1741f916f9025a0acc68ce85462fd528` | `68d2aa293d5c17fda91c63395fbc01176658d61d` |

The scoped callers no longer log authorization code/state, complete token results,
GitHub codes or Portal social-user command payloads. Stateless/Facebook token
failures preserve the provider HTTP status but return configured ERR10014 with
fixed text instead of the raw provider Status. MSAL retains configured ERR11648
with a fixed safe argument. Success responses, trust policy and cookie handling
are unchanged; the redacted error bodies are an intentional behavior change.

Portal diagnostics retain sanitized copies of cause types, bounded stack/cause/
suppressed chains and validated SQLState/vendor metadata, without provider
exception messages. Fully qualified names also cover anonymous types. MSAL now
uses PortalClient from its existing stateless-auth dependency instead of a shadow
copy; deploy matching module artifacts. Both module contexts have privacy tests.

Recorded clean affected-reactor validation passed **159 tests on master and 85 on
2.x**, zero failures/errors and one existing skip each. Two legacy Azure methods
were explicitly excluded on 2.x because their fixtures contact Microsoft endpoints.
The corrected refresh-token canary is asserted in the returned cookie before log
checks. Local probes cover callback/Facebook error responses, GitHub's existing
code-processing stub, MSAL exchange failure, constructor failures and anonymous/
nested/cyclic Portal exceptions. Logging-only negative controls catch reintroduced
token-result logging, raw MSAL Status logging and raw provider responses without
changing HTTP outcomes. Final focused privacy checks and pre-commit also passed.

Root appenders inspect messages, arguments, MDC and throwable chains. Portal
failure probes run all loggers/root at TRACE; callback probes run auth at TRACE
with root capture at WARN. Raw transport TRACE can still print legacy callback
URLs, and Portal's encoded command/query URL transport remains a separate
boundary. These are scoped source/JVM results, not full-repository, live-provider,
production-log or partner-rescan qualification. See the
[merged provider-diagnostics guide](https://github.com/networknt/light-spa-4j/blob/9227ce42b772311cb3901269db83e58e84ebf2eb/docs/security/oauth-provider-diagnostics.md).
This follow-up adds no rows or category-wide reclassification to the workbook.

## HSTS configuration follow-up — 2026-10-28

For **A02 / Excel row 3**, the original companion PRs, both `2.x` backports and the `master` fixture follow-up are merged:

| Repository | Branch | Merged PR | Remediation | Merge commit |
| --- | --- | --- | --- | --- |
| light-rest-4j | master | [#452](https://github.com/networknt/light-rest-4j/pull/452) | Consuming-application HSTS guide and isolated configuration example; no policy hard-coded in FaviconHandler. | `4f865e54c11fa2b150ebdad7db8db050ab1e03f8` |
| http-sidecar | master | [#141](https://github.com/networknt/http-sidecar/pull/141) | Header first in default/admin chains and seven direct routes; opt-in late response policy prevents backend headers overriding HSTS. | `f54c5c25554bef38df80a353d4f9be34289ccca2` |
| http-sidecar | 2.x | [#142](https://github.com/networknt/http-sidecar/pull/142) | Backport of #141, including atomic fixture ready-file publication and listener lookup by TLS state. | `f683a2f212db4e69ff1cd11f7ea7bb710a271148` |
| light-rest-4j | 2.x | [#453](https://github.com/networknt/light-rest-4j/pull/453) | Backport of #452 using the 2.x YAML API; companion documentation links pinned to an existing reviewed commit. | `c16ff27f7418b498d9e6888013b0573ece78703c` |
| http-sidecar | master | [#143](https://github.com/networknt/http-sidecar/pull/143) | Matching fixture corrections: atomic ready-file publication and no dependence on Undertow listener order; production policy unchanged. | `38c9e940cf7bb4b70c337760a264322328b2abed` |

GitHub checks passed on the reviewed PR heads: source tests and pre-commit for both, plus distribution-readiness for http-sidecar. Local suites reported 244 tests (7 skips) for light-rest-4j and 34 tests (9 skips) for http-sidecar, with no failures or errors. Sidecar loopback tests exercise actual handler pipelines, direct/admin routes and proxied success, redirect, 404 and 500 responses with conflicting backend headers; the existing test keystores are reused.

Backport validation: local clean suites passed with 31 tests (9 skips) for http-sidecar `2.x` and 238 tests (7 skips) for light-rest-4j `2.x`, with no failures or errors. The matching master fixture correction passed 34 tests (9 skips), also with no failures or errors. The sidecar production configuration matches across branches; only the YAML parsing API differs in the tests. These are local validation results; the GitHub-check statement above refers to the original #141/#452 PR heads.

**Deployment restriction:** the opt-in overlay is for a dedicated HTTPS ingress instance. It disables plaintext HTTP, including the normal co-located sidecar egress port 9080; do not apply it to that topology. Use a separate ingress instance or enforce HSTS at a trusted TLS edge. Generic HeaderHandler does not distinguish HTTP from HTTPS listeners. Packaged defaults retain plaintext egress and do not enable HSTS. The example uses `max-age=31536000` without `includeSubDomains` or `preload`.

Source/configuration remediation is merged; **A02 remains conditional pending deployment verification**. Verify the effective public HTTPS policy on favicon, normal, redirect and error responses, public HTTP handling and direct-origin restrictions, then rerun the partner scan. The original assessment counts below are unchanged.

## Archived repository follow-up — 2026-10-28

The repository owner has archived `oauth-kafka` and `light-controller`. GitHub archive status was verified on 2026-10-28 at the canonical repository locations below. This lifecycle update applies to **17 worksheet findings**; it is separate from the merged fixes for the four confirmed findings above.

| Repository | Archived repository | Affected Excel rows | Findings | Technical disposition |
| --- | --- | --- | ---: | --- |
| oauth-kafka | [lightapi/oauth-kafka](https://github.com/lightapi/oauth-kafka) | 77–88 | 12 | Unresolved; archived repository |
| light-controller | [lightapi/light-controller](https://github.com/lightapi/light-controller) | 112–116 | 5 | Unresolved; archived repository |

The original scan findings and row mappings are retained for audit. Archiving is not a security patch or evidence that a finding is a false positive, and it does not establish that existing deployments have stopped using these components. For partner disposition, record the archived status and confirm whether the affected artifacts remain in use. Retire or migrate any remaining use to a maintained component; if legacy use continues, obtain the scanned source/artifact and assess the findings for that deployment. No replacement component, deployment retirement, vulnerability fix or technical closure has been verified by this archive update.

These entries refer to the spreadsheet findings, not to newly opened or closed GitHub security issues. The original 18 unresolved assessments comprised these 17 archived-repository findings plus the historical ProfileMenu finding in row 25; the portal-view follow-up above now establishes the historical generator's OAuth-state purpose and records current source remediation.

## Evidence baseline

All eleven available code repositories were clean when inspected. The documentation repository was also clean before this work. Source references in the appendix are pinned to these commits; they point to the *reported* file and line, which can have drifted. Detailed assessments name current symbols/lines.

| Repository | Reviewed HEAD |
| --- | --- |
| event-importer | `1f5279420ae2800fb6539f4bbda6d57d6283b67b` |
| http-client | `fdc96cc5d9df2475ee2129f7a70f300b5b6c0a06` |
| light-4j | `bc3e4b5e2719d705d58a2edeabeb9bb5009c7401` |
| light-controller | `Unavailable` at assessment; [repository now archived](https://github.com/lightapi/light-controller) |
| light-gateway | `06ca1abc5a7ce84611219685fb2589fefc4f6ae3` |
| light-lambda-native | `b6d27a7d42ecdc94efe387acdd1cb4b3f47a7ce7` |
| light-portal | `a8da0c88ee3fb8178dd43c2a02495d6b99084c69` |
| light-rest-4j | `0b65bfb78a15d71060d81158d29a0276ac31fae4` |
| light-session-4j | `7a20816baabfcaacae29fcf6e9111b0761ec7fbe` |
| light-spa-4j | `af06c19738a6f898201ee9f71998bfa75d9293e8` |
| oauth-kafka | `Unavailable` at assessment; [repository now archived](https://github.com/lightapi/oauth-kafka) |
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

This handler does not set HSTS itself, but that does not establish the externally visible response lacks it: middleware or a TLS edge can add it. HTTPS-only origin service does not by itself prevent browser HTTP downgrade attempts. Inspect the public HTTPS response, redirect and error paths, and direct-origin access. If HSTS is absent, this is a real browser-facing configuration gap. If it is present at the edge, the missing application-local header is a defense-in-depth policy gap rather than proof of a missing browser control. First-visit protection additionally requires an appropriate preload policy; adding HSTS is not retroactive protection for an initial insecure visit. See [RFC 6797](https://www.rfc-editor.org/rfc/rfc6797). The existing `light-4j/header` HeaderHandler supports response policies; it must be enabled and registered on the applicable routes. The merged [light-rest-4j #452](https://github.com/networknt/light-rest-4j/pull/452) and [http-sidecar #141](https://github.com/networknt/http-sidecar/pull/141) provide the consuming configuration and correct sidecar route coverage. Their HSTS example uses `header.pathPrefixHeader` to apply the policy at response-send time, after proxy backend headers are copied. See the HSTS follow-up above for validation and the dedicated HTTPS ingress restriction. Enable includeSubDomains/preload only after domain-wide compatibility review. Deployment/partner closure remains unverified.

<a id="a03"></a>

### A03: Snapshot dictionary prototype mutation

**Conditional — Excel rows 4, 5.**

**Merged follow-up:** [portal-view #939](https://github.com/lightapi/portal-view/pull/939) removes the unsafe construction primitive with null-prototype cell dictionaries and worker-transfer regressions. The paragraph below describes the initial source baseline; deployment/partner verification remains outstanding.

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

**Merged follow-up:** [portal-view #939](https://github.com/lightapi/portal-view/pull/939) removes weak fallbacks: chat/idempotency IDs use Web Crypto, while JSON-RPC correlation uses a per-page sequence. The paragraph below describes the initial source baseline. Backend owner/operation scoping still requires deployment verification.

`codingRequest.ts:11–16` prefers crypto.randomUUID, then getRandomValues, with a Math.random fallback for client message IDs. `fetchClient.ts:46` uses the fallback for JSON-RPC request IDs. `knowledgeApi.ts:41–45` uses it for idempotency keys. These values correlate requests or deduplicate commands; the reviewed clients do not use them as credentials. Predictability alone does not prove authentication bypass. Use Web Crypto consistently and remove weak fallbacks where practical. Verify server idempotency keys are scoped to authenticated owner/operation and cannot expose another user's response; collision or cross-owner reuse is a separate risk that these source lines alone do not prove.

<a id="a10"></a>

### A10: Historical ProfileMenu randomness

**Unresolved — Excel rows 25.**

At the initial assessment, ProfileMenu delegated to `signIn` and had no Math.random call; the scanned revision was unavailable, leaving the historical purpose unresolved. Follow-up history inspection establishes that commit `c14f56b` introduced Math.random for OAuth state, `d239afd` moved it to the shared sign-in helper, and `7f200a3` replaced it with crypto.randomUUID. The merged [portal-view #939](https://github.com/lightapi/portal-view/pull/939) now uses the shared secureRandomUUID helper, including its cryptographic byte fallback, and adds real menu/sign-in regressions and a visible retryable failure. The historical purpose is resolved and current source is remediated; this is not a historical false positive. The partner's exact scan revision, deployed artifact and live OAuth/server validation remain unverified.

<a id="a11"></a>

### A11: Gateway WebSocket destination injection

**False positive — Excel rows 28, 29, 30, 31, 32, 33.**

All three pages construct the destination from window.location.protocol and window.location.host with a fixed `/chat` path. userId/model are interpolated after `?`; characters in those values can change query parameters but cannot replace the URL's authority with an attacker host. No attacker-selected WebSocket origin is established. Use URL/URLSearchParams for query encoding and require server-side authorization of user/model selection. The use of a UI-supplied userId is not proof of an authenticated identity and must never substitute for backend authentication.

<a id="a12"></a>

### A12: DOM XSS in Gateway demo pages

**Confirmed at assessment; source fixes merged — Excel rows 34, 35.**

**Subsequent retirement:** [master #265](https://github.com/networknt/light-gateway/pull/265)
and [2.x #266](https://github.com/networknt/light-gateway/pull/266) remove these demo
entry points. See the [retirement record](#gateway-retirement-2026-10-29); the
baseline and earlier browser remediation below remain historical evidence.

Resolved through [issue #258](https://github.com/networknt/light-gateway/issues/258) and [issue #259](https://github.com/networknt/light-gateway/issues/259) by [master PR #260](https://github.com/networknt/light-gateway/pull/260) and [2.x PR #261](https://github.com/networknt/light-gateway/pull/261); see the row-specific issue links in the remediation table above.

Both `config/llmchat-rendezvous/public/index.html:155` and `config/llmchat-gateway/public/index.html:91` interpolate untrusted text directly into div.innerHTML in log(). WebSocket event.data and, in the rendezvous page, backend callback response text reach that sink; user-entered text reaches it too. A message containing an image element with an event-handler attribute is parsed as HTML, rather than displayed as text. Exploitation against other users requires influence over a message or callback they receive; deployment exposure and CSP affect impact. This is a real source-level defect even though these are demo assets. Replace text interpolation with textContent/createTextNode, keeping a separately created strong element for the fixed role. The ai-gateway demo already shows the createTextNode pattern. Test malicious-looking messages as literal text before serving these demos to users.

<a id="a13"></a>

### A13: Browser storage of chat session ID

**Conditional — Excel rows 36.**

**Subsequent retirement:** [master #265](https://github.com/networknt/light-gateway/pull/265)
and [2.x #266](https://github.com/networknt/light-gateway/pull/266) remove these demo
entry points. See the [retirement record](#gateway-retirement-2026-10-29); the
baseline and earlier browser remediation below remain historical evidence.

**Merged follow-up:** [master #262](https://github.com/networknt/light-gateway/pull/262) and [2.x #263](https://github.com/networknt/light-gateway/pull/263) remove browser persistence from both the canonical and Kubernetes AI pages. The initial-baseline analysis below is retained; the reconnect URL exposure and backend/deployment checks remain outstanding.

`ai-gateway/public/index.html:61–66` stores json.sessionId in sessionStorage and later sends it as a reconnect parameter. A session ID used only as an owner-authorized conversation identifier is not automatically a bearer credential; sessionStorage is tab-scoped but readable by same-origin JavaScript. If possession authorizes reconnection, XSS or script compromise can disclose the capability and the finding is real. Confirm backend ownership checks, expiry and replay behavior before closing; do not claim browser storage is inherently secure or move an actual authentication secret to localStorage.

<a id="a14"></a>

### A14: Rendezvous channel randomness

**Conditional — Excel rows 37.**

**Subsequent retirement:** [master #265](https://github.com/networknt/light-gateway/pull/265)
and [2.x #266](https://github.com/networknt/light-gateway/pull/266) remove these demo
entry points. See the [retirement record](#gateway-retirement-2026-10-29); the
baseline and earlier browser remediation below remain historical evidence.

**Merged follow-up:** [master #262](https://github.com/networknt/light-gateway/pull/262) and [2.x #263](https://github.com/networknt/light-gateway/pull/263) replace Math.random with Web Crypto and guard connection/callback lifecycle. The initial-baseline analysis below is retained; authenticated peer admission, ownership and expiry still need deployment verification.

The demo's generateUUID uses Math.random for channelId and puts it in `/chat` and `/connect`. This has a stronger security implication than a JSON-RPC ID: the adjacent `light-websocket-4j` WebSocketRendezvousHandler.onConnect pairs peers through a shared CHANNELS map keyed by channelId, with backend role inferred from the configured path. That method does not itself prove ownership binding. If deployed admission relies on channel secrecy, weak generation is a real security defect. Generate with Web Crypto and authenticate/authorize both peers with owner binding and expiry. Obtain the actual Gateway chain and backend `/connect` authorization before asserting hijacking is reachable; no live exploit was attempted.

<a id="a15"></a>

### A15: JDBC session deserialization

**Initial assessment: Mitigated in current source — Excel row 38. Reassessment qualified that verdict; source byte-limit fixes are now merged.**

At the original pinned revision, database bytes reached ObjectInputStream.readObject
through a default class allowlist and graph limits (depth 20, references 10,000,
array length 100,000). Tests confirmed unlisted readObject classes were rejected
before their hooks ran. However, the filter's streamBytes check was not a hard
1 MiB bound: concretely encoded strings do not invoke the filter, and larger string
payloads were accepted. See [Oracle ObjectInputStream filtering](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/io/ObjectInputStream.html#setObjectInputFilter(java.io.ObjectInputFilter)).

Merged [master #68](https://github.com/networknt/light-session-4j/pull/68) and
[2.x #69](https://github.com/networknt/light-session-4j/pull/69) add an independent
serialized-byte bound, bounded JDBC consumption, observable transactional save
failures and isolation of invalid sessions during loading. See the A15 follow-up
above for exact commits, validation and migration impact. The merged default
class/byte controls are substantive mitigation, not proof that arbitrary Java
deserialization is safe. Aggregate limits, driver buffering and deployed ownership/
database-write boundaries still require review. Configured allowedClasses and
allowedPackages expand the trust boundary. Verify the deployed artifact/settings
and partner rescan before closure; the older unfiltered revision can be vulnerable
when an attacker can alter serialized attributes and a usable gadget is present.

<a id="a16"></a>

### A16: Labels and test password classified as keys

**False positive — Excel rows 39, 40, 41, 42, 44.**

SOAP PASSWORD_TYPE and DEFAULT_PASSWORD_TYPE are a configuration-field name and WS-Security type URI; actual credentials are read from configuration. AutoAESSaltDecryptor's `light_4j_config_password` is an environment/property lookup name. ManualAESSaltDescryptor's console string is a prompt. AutoAESSaltDecryptor's `light` fallback is gated by a current-thread org.junit stack check; otherwise an absent runtime secret throws. These exact flows do not establish a hardcoded production key. Separately, base AESSaltDecryptor.getPassword itself returns a fixed default; see the additional real concern below and do not confuse it with the environment-variable label.

<a id="a17"></a>

### A17: Predictable PBKDF2 salt

**Conditional — Excel rows 43.**

AutoAESSaltDecryptor supplies the password, not the salt. AESSaltDecryptor.getSecret takes salt bytes from the CRYPT value's salt field, and derives a key with PBKDF2WithHmacSHA256. A salt is public and need not be secret or unpredictable after generation; it should be unique and generated appropriately. The reported password-to-generateSecret flow does not establish a fixed salt. The merged light-4j follow-up above records current producer inspection and reader validation. Audit the actual producing revision and ciphertext inventory for reused/static salts before closure. Do not publish decrypted values or production secrets as evidence.

<a id="a18"></a>

### A18: Legacy AES-CBC support

**Conditional — Excel rows 45.**

At the initial assessment, AESSaltDecryptor supported AES-GCM for four-part values, but decryptLegacyCbc:68–72 retained AES/CBC/PKCS5Padding with an all-zero IV for three-part values. This is a real legacy cryptographic weakness when that format is still used: deterministic CBC leaks equality under the same key and does not authenticate ciphertext. It is not proof that AES itself is broken or that a remotely usable padding oracle exists. Inventory formats without disclosing plaintext, regenerate legacy secrets using the current GCM encryptor, and plan an explicit compatibility cutoff. Keeping a legacy reader does not prove deployed values have migrated. The merged light-4j follow-up above removes that reader on both branches and requires re-encryption before upgrade; deployment migration is still unverified.

<a id="a19"></a>

### A19: Default public truststore password

**False positive — Excel rows 46.**

Http2Client.loadDefaultTrustStore:1405–1444 uses the conventional cacerts default and permits configuration/system-property overrides. A truststore of public CA certificates is not a private-key credential or remote login password, and the value is not immutable. Its integrity still depends on file permissions and controlled trust-anchor updates. If that store is repurposed to contain private keys, reassess its contents and access policy.

<a id="a20"></a>

### A20: Heap retention of credentials and key material

**Hardening — Excel rows 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 73, 76, 98, 108, 109, 110, 111.**

These rows predominantly identify real in-process secret retention, not a demonstrated remote exploit. At the initial assessment, server/client TLS loaders, MRAS, Salesforce, Basic authentication, config bootstrap and repair utilities held configured or request credentials as Strings, and AutoAESSaltDecryptor cached password/key material for reuse. The merged light-4j follow-up above removes base/Auto caches and clears owned temporary copies; the manual provider has an explicit bounded cache, and retained configuration/API credentials remain outside complete erasure guarantees. An attacker able to read heap dumps, debugger state or process memory may recover them. Scope heap dumps/diagnostics, restrict process and filesystem access, minimize credential lifetime and rotate credentials after suspected disclosure. Where APIs allow it, use owned char[] buffers and clear temporary copies/PBEKeySpec after use; converting an existing String to char[] does not erase the original and clearing shared config arrays can break runtime behavior. The [2026-10-29 follow-up](#a20-follow-up-2026-10-29) records the merged Portal (rows 73/76), Lambda (row 98) and HTTP-client (rows 108–111) hardening in addition to light-4j.

At the initial assessment, rows 56–59 pointed to char[] fields in TlsConfig, so the literal claim that those fields were immutable Strings was stale; those configuration arrays remain retained for reuse. Several scan lines had moved in that baseline: MRAS credentials were at 556–575, Salesforce certPassword at 226–250, DefaultConfigLoader truststorePassword at 574–588, User BasicAuth password at 176, and Lambda password at 159. The merged follow-ups above supersede these historical local-copy lifecycles, while borrowed configuration credentials remain retained. Treat this as low-priority memory exposure hardening subject to the threat model, not blanket false-positive suppression or proof that all secret retention can be eliminated.

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

### A24: Archived repositories; technical findings unverified

**Unresolved technical assessment; repositories archived — Excel rows 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 112, 113, 114, 115, 116.**

Lifecycle update: [oauth-kafka](https://github.com/lightapi/oauth-kafka) and [light-controller](https://github.com/lightapi/light-controller) are archived, verified on 2026-10-28. See the archived repository follow-up above for treatment of these 17 findings and remaining deployment checks.

At the initial assessment, neither oauth-kafka nor light-controller nor the named source files was found under /home/steve/workspace or /home/steve/networknt. No substitute repository was assumed. For oauth-kafka, literal authentication field names and error constants may be false positives, but DefaultAuthenticator's expectedPassword may be a genuine default credential; heap findings need the actual lifecycle. For light-controller, inspect whether bundled keystore material includes a shared private key and whether production can select it; distinguish a public truststore default from an actual database password. Obtain the scanned commit/artifact, source and redacted effective configuration before assigning a real/false disposition to any of these 17 rows. Do not close them from variable names or scanner comments alone.

<a id="a25"></a>

### A25: Lambda Basic authentication logging

**Mitigated in current source — Excel rows 90, 91, 92, 93, 94, 95, 96, 97.**

At the initial assessment, BasicAuthMiddleware.handleBasicAuth:146–200 no longer logged decoded credentials at the reported sinks. Failure messages and progress traces are fixed text; the path denial message removes CR/LF from requestPath. The scanner's duplicated Base64/substr source flows do not demonstrate password disclosure in this revision. Historical comments cannot establish what the partner runs. Verify its deployed revision and use synthetic canary credentials to assert they never appear in ERROR/DEBUG/TRACE logs before closing the original eight results. Username/path logging is a separate data-minimization consideration. The [merged Lambda follow-up](#a20-follow-up-2026-10-29), tracking issue #191 and PRs #192/#193, confirms this mitigation on both branch baselines and adds TRACE canary coverage alongside the A20 buffer cleanup.

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

**Mitigated in reviewed master source; merged 2.x remediation — Excel rows 117, 118, 119.**

**Merged follow-up:** [master #189](https://github.com/networknt/light-spa-4j/pull/189)
adds regression protection for the existing master mitigation;
[2.x #190](https://github.com/networknt/light-spa-4j/pull/190) fixes the remaining
legacy logging paths and preserves configured rejection codes. See the
[A28 follow-up](#a28-follow-up-2026-10-29) for tracking issue #188, merge commits,
branch-specific behavior and validation limits. The paragraph below describes the
initial master baseline; it did not establish that 2.x was already mitigated.

The current MsalTokenExchangeHandler and StatelessAuthHandler verify tokens through their configured trust providers and no longer contain the reported JWT/claim debug/trace messages around the old sinks. These scan line numbers now refer to different statements. The reported cookie-to-log path is not reproduced by current source inspection. This is a current-source mitigation, not evidence that the historical finding was false or that all logging in these classes is safe: other request-path, authorization-code/state or token-result logging deserves a separate review. Confirm the deployed artifact and exercise CR/LF-bearing rejected input with canary-only logs before closing.

**Adjacent diagnostics follow-up:** [master #192](https://github.com/networknt/light-spa-4j/pull/192)
and [2.x #193](https://github.com/networknt/light-spa-4j/pull/193), tracking issue
#191, address the separate callback/provider logging review. See the
[follow-up record](#spa-callback-follow-up-2026-10-29) for error-body behavior,
regression evidence and remaining transport/deployment boundaries.

## Additional concern found during review

**Fixed default in the base decryptor:** `light-4j/decryptor/.../AESSaltDecryptor.java:getPassword` returns a literal default password. This remains in the merged light-4j source by the explicit compatibility waiver in #2828; it is not fixed by the AES-GCM format cutoff. Unlike the AutoAESSaltDecryptor test fallback in row 44, direct production use of this base implementation provides no deployment-specific master secret. Anyone who obtains ciphertext and knows the public default can derive its key. Confirm the configured decryptor class before closure; use an approved runtime-secret implementation such as AutoAESSaltDecryptor and re-encrypt affected values. This is an additional root cause, not an extra row in the 118-row counts.

**MRAS partial secret logging — merged source fix:** at the initial assessment, MrasHandler traced the first four characters of keystore/key/truststore passwords. This was a real disclosure when TRACE was enabled, beyond the heap-retention category in rows 51–53; truncation was not masking and short passwords could throw. PRs #2829/#2830 remove those statements and validate actual TLS paths with synthetic canary logs, as recorded above. Verify the deployed artifact and logging behavior before partner closure. No production credentials were accessed to establish or test this.

## Recommended closure sequence

1. Adopt the merged Gateway demo retirement and remove surviving deployed/cached copies. Deploy and verify the OAuth logging, portal-view and SPA callback/provider fixes linked above, then obtain partner scan or documented retirement closure. Deploy the merged light-4j MRAS logging fix as well and verify credential-free TRACE logs; its source regression tests use synthetic canary secrets.
2. Inventory the configured decryptor and ciphertext formats before adopting light-4j #2829/#2830. Replace fixed-default master secrets and regenerate legacy CBC/nonstandard values as authenticated standard GCM values; both branches now enforce the cutoff without a legacy opt-in.
3. Adopt and deploy the merged HSTS configuration linked in the A02 follow-up where applicable, respecting the dedicated HTTPS ingress restriction (normal sidecar egress needs port 9080). Verify public HSTS using HTTPS response captures and effective application/edge configuration; cover normal, redirect, error and favicon responses and direct-origin access before partner closure.
4. Complete A13/A14 retirement in actual deployments, including old config-server values and public ConfigMaps. For any deliberately retained custom deployment, verify session ownership, rendezvous peer admission and expiry/replay controls, and address reconnect-ID URL logging; repository removal does not repair that deployment. Confirm snapshot UUID validation; adopt the merged A15 safeguards and verify deployed filters, byte policy and remaining aggregate/driver resource boundaries.
5. Obtain the partner scan SHA/artifacts for applicability and closure. For archived oauth-kafka and light-controller, first confirm whether any deployment still uses them and plan retirement or migration; legacy use still requires source/artifact assessment. Re-run the scan and focused regressions on repaired artifacts for maintained components; maintain result-specific evidence rather than category-wide suppressions.
6. Adopt the merged A20 hardening for light-4j, light-portal, light-lambda-native and http-client linked above, with the applicable hybrid/native rebuilds and provider-lifecycle checks. Review remaining heap retention under a documented threat model, prioritizing long-lived master/private-key/DB secrets over public truststore defaults, hashes and ciphertext.

For partner closure, attach the source commit/artifact digest, exact result identifier, redacted effective configuration, focused test output and deployment verification. No real secrets or access tokens should be included. Infrastructure HSTS can satisfy a browser control even when a separate partner policy still requires application defense in depth. An inbound sanitizer cannot justify unrelated output, logging, deserialization or cryptographic findings.

## Complete worksheet cross-reference

Every worksheet finding appears once below. Duplicate scanner rows remain separate for reconciliation. The Checkmarx result link/identifier remains in column P of the original workbook; it is not needed to access private partner systems for this review. Source links use the repository name as supplied by the scan; missing repositories have no claimed verified source link.

| Excel row | Repository | Query / original severity | Reported source → sink | Assessment |
| ---: | --- | --- | --- | --- |
| 2 | light-rest-4j | Stored_XSS / CRITICAL | [`/specification/src/main/java/com/networknt/specification/FaviconHandler.java:42`](https://github.com/networknt/light-rest-4j/blob/0b65bfb78a15d71060d81158d29a0276ac31fae4/specification/src/main/java/com/networknt/specification/FaviconHandler.java#L42) → `/specification/src/main/java/com/networknt/specification/FaviconHandler.java:43` | [A01](#a01) — False positive |
| 3 | light-rest-4j | Missing_HSTS_Header / MEDIUM | [`/specification/src/main/java/com/networknt/specification/FaviconHandler.java:43`](https://github.com/networknt/light-rest-4j/blob/0b65bfb78a15d71060d81158d29a0276ac31fae4/specification/src/main/java/com/networknt/specification/FaviconHandler.java#L43) → `/specification/src/main/java/com/networknt/specification/FaviconHandler.java:43` | [A02](#a02) — Conditional; [#452](https://github.com/networknt/light-rest-4j/pull/452) / [#141](https://github.com/networknt/http-sidecar/pull/141) merged; deployment verification pending |
| 4 | portal-view | Prototype_Pollution / HIGH | [`/src/pages/snapshot/snapshotComparison.worker.ts:19`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/snapshot/snapshotComparison.worker.ts#L19) → `/src/pages/snapshot/snapshotComparison.ts:56` | [A03](#a03) — Conditional; [#939](https://github.com/lightapi/portal-view/pull/939) merged; deployment verification pending |
| 5 | portal-view | Prototype_Pollution / HIGH | [`/src/pages/snapshot/snapshotComparison.worker.ts:19`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/snapshot/snapshotComparison.worker.ts#L19) → `/src/pages/snapshot/snapshotComparison.ts:43` | [A03](#a03) — Conditional; [#939](https://github.com/lightapi/portal-view/pull/939) merged; deployment verification pending |
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
| 24 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/pages/genai/codingRequest.ts:16`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/codingRequest.ts#L16) → `/src/pages/genai/codingRequest.ts:16` | [A09](#a09) — Hardening; [#939](https://github.com/lightapi/portal-view/pull/939) merged; deployment verification pending |
| 25 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/components/Header/ProfileMenu.tsx:65`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/components/Header/ProfileMenu.tsx#L65) → `/src/components/Header/ProfileMenu.tsx:65` | [A10](#a10) — Unresolved; [#939](https://github.com/lightapi/portal-view/pull/939) merged; deployment verification pending |
| 26 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/utils/fetchClient.ts:54`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/utils/fetchClient.ts#L54) → `/src/utils/fetchClient.ts:54` | [A09](#a09) — Hardening; [#939](https://github.com/lightapi/portal-view/pull/939) merged; deployment verification pending |
| 27 | portal-view | Use_of_Insufficiently_Random_Values / LOW | [`/src/pages/genai/knowledgeApi.ts:45`](https://github.com/networknt/portal-view/blob/c2e1d5f5f0ca48e69462a4021abe6a4a0c5176f6/src/pages/genai/knowledgeApi.ts#L45) → `/src/pages/genai/knowledgeApi.ts:45` | [A09](#a09) — Hardening; [#939](https://github.com/lightapi/portal-view/pull/939) merged; deployment verification pending |
| 28 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-gateway/public/index.html:41`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-gateway/public/index.html#L41) → `/config/llmchat-gateway/public/index.html:45` | [A11](#a11) — False positive |
| 29 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-gateway/public/index.html:40`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-gateway/public/index.html#L40) → `/config/llmchat-gateway/public/index.html:45` | [A11](#a11) — False positive |
| 30 | light-gateway | Client_Resource_Injection / HIGH | [`/config/ai-gateway/public/index.html:41`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/ai-gateway/public/index.html#L41) → `/config/ai-gateway/public/index.html:51` | [A11](#a11) — False positive |
| 31 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-rendezvous/public/index.html:79`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L79) → `/config/llmchat-rendezvous/public/index.html:86` | [A11](#a11) — False positive |
| 32 | light-gateway | Client_Resource_Injection / HIGH | [`/config/ai-gateway/public/index.html:42`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/ai-gateway/public/index.html#L42) → `/config/ai-gateway/public/index.html:51` | [A11](#a11) — False positive |
| 33 | light-gateway | Client_Resource_Injection / HIGH | [`/config/llmchat-rendezvous/public/index.html:78`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L78) → `/config/llmchat-rendezvous/public/index.html:86` | [A11](#a11) — False positive |
| 34 | light-gateway | Client_DOM_XSS / HIGH | [`/config/llmchat-rendezvous/public/index.html:124`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L124) → `/config/llmchat-rendezvous/public/index.html:155` | [A12](#a12) — Confirmed |
| 35 | light-gateway | Client_DOM_XSS / HIGH | [`/config/llmchat-gateway/public/index.html:60`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-gateway/public/index.html#L60) → `/config/llmchat-gateway/public/index.html:91` | [A12](#a12) — Confirmed |
| 36 | light-gateway | Client_HTML5_Insecure_Storage / MEDIUM | [`/config/ai-gateway/public/index.html:61`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/ai-gateway/public/index.html#L61) → `/config/ai-gateway/public/index.html:66` | [A13](#a13) — Conditional; [master #262](https://github.com/networknt/light-gateway/pull/262) / [2.x #263](https://github.com/networknt/light-gateway/pull/263) merged; protocol/deployment verification pending |
| 37 | light-gateway | Use_of_Insufficiently_Random_Values / LOW | [`/config/llmchat-rendezvous/public/index.html:72`](https://github.com/networknt/light-gateway/blob/06ca1abc5a7ce84611219685fb2589fefc4f6ae3/config/llmchat-rendezvous/public/index.html#L72) → `/config/llmchat-rendezvous/public/index.html:72` | [A14](#a14) — Conditional; [master #262](https://github.com/networknt/light-gateway/pull/262) / [2.x #263](https://github.com/networknt/light-gateway/pull/263) merged; protocol/deployment verification pending |
| 38 | light-session-4j | Deserialization_of_Untrusted_Data / HIGH | [`/jdbc-manager/src/main/java/com/networknt/session/jdbc/JdbcSessionRepository.java:375`](https://github.com/networknt/light-session-4j/blob/7a20816baabfcaacae29fcf6e9111b0761ec7fbe/jdbc-manager/src/main/java/com/networknt/session/jdbc/JdbcSessionRepository.java#L375) → `/jdbc-manager/src/main/java/com/networknt/session/jdbc/serializer/DefaultDeserializer.java:103` | [A15](#a15) — Initial mitigation qualified; [master #68](https://github.com/networknt/light-session-4j/pull/68) / [2.x #69](https://github.com/networknt/light-session-4j/pull/69) merged; deployment/resource checks pending |
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
| 77 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:34` → `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:34` | [A24](#a24) — Unresolved; archived repository |
| 78 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:75` → `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:75` | [A24](#a24) — Unresolved; archived repository |
| 79 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:76` → `/src/main/java/com/networknt/oauth/handler/ProviderIdTokenPostHandler.java:76` | [A24](#a24) — Unresolved; archived repository |
| 80 | oauth-kafka | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:34` → `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:34` | [A24](#a24) — Unresolved; archived repository |
| 81 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/LightPortalAuthenticator.java:78` → `/src/main/java/com/networknt/oauth/auth/LightPortalAuthenticator.java:78` | [A24](#a24) — Unresolved; archived repository |
| 82 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/security/LightBasicAuthenticationMechanism.java:196` → `/src/main/java/com/networknt/oauth/security/LightBasicAuthenticationMechanism.java:196` | [A24](#a24) — Unresolved; archived repository |
| 83 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:179` → `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:179` | [A24](#a24) — Unresolved; archived repository |
| 84 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:49` → `/src/main/java/com/networknt/oauth/auth/LightDbAuthenticator.java:49` | [A24](#a24) — Unresolved; archived repository |
| 85 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:187` → `/src/main/java/com/networknt/oauth/security/LightFormAuthenticationMechanism.java:187` | [A24](#a24) — Unresolved; archived repository |
| 86 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/handler/BaseWrapper.java:41` → `/src/main/java/com/networknt/oauth/handler/BaseWrapper.java:41` | [A24](#a24) — Unresolved; archived repository |
| 87 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:50` → `/src/main/java/com/networknt/oauth/auth/DefaultAuthenticator.java:50` | [A24](#a24) — Unresolved; archived repository |
| 88 | oauth-kafka | Heap_Inspection / LOW | `/src/main/java/com/networknt/oauth/auth/MarketPlaceAuthenticator.java:34` → `/src/main/java/com/networknt/oauth/auth/MarketPlaceAuthenticator.java:34` | [A24](#a24) — Unresolved; archived repository |
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
| 112 | light-controller | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/controller/tls/TlsMaterialResolver.java:17` → `/src/main/java/com/networknt/controller/tls/TlsMaterialResolver.java:17` | [A24](#a24) — Unresolved; archived repository |
| 113 | light-controller | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` | [A24](#a24) — Unresolved; archived repository |
| 114 | light-controller | Use_Of_Hardcoded_Password / MEDIUM | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` | [A24](#a24) — Unresolved; archived repository |
| 115 | light-controller | Heap_Inspection / LOW | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:40` | [A24](#a24) — Unresolved; archived repository |
| 116 | light-controller | Heap_Inspection / LOW | `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` → `/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java:30` | [A24](#a24) — Unresolved; archived repository |
| 117 | light-spa-4j | Log_Forging / LOW | [`/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:202`](https://github.com/networknt/light-spa-4j/blob/af06c19738a6f898201ee9f71998bfa75d9293e8/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java#L202) → `/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:259` | [A28](#a28) — Mitigated in current source |
| 118 | light-spa-4j | Log_Forging / LOW | [`/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:202`](https://github.com/networknt/light-spa-4j/blob/af06c19738a6f898201ee9f71998bfa75d9293e8/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java#L202) → `/msal-exchange/src/main/java/com/networknt/auth/MsalTokenExchangeHandler.java:244` | [A28](#a28) — Mitigated in current source |
| 119 | light-spa-4j | Log_Forging / LOW | [`/stateless-auth/src/main/java/com/networknt/auth/StatelessAuthHandler.java:186`](https://github.com/networknt/light-spa-4j/blob/af06c19738a6f898201ee9f71998bfa75d9293e8/stateless-auth/src/main/java/com/networknt/auth/StatelessAuthHandler.java#L186) → `/stateless-auth/src/main/java/com/networknt/auth/StatelessAuthHandler.java:228` | [A28](#a28) — Mitigated in current source |
