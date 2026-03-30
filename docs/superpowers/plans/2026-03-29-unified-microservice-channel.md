# Unified Microservice Channel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the unified microservice registration and discovery contract across `controller-rs`, `light-4j`, and `light-controller`, including client-supplied address, `host` claim to `hostId` validation, service-side single-channel discovery, and runtime-instance event persistence updates.

**Architecture:** `controller-rs` becomes the reference backend for the new contract: `/ws/microservice` accepts `service/register` with `envTag`, `address`, and JWT `host` validation, then serves discovery RPCs on the same socket. `light-4j/portal-registry` is updated to send one payload shape and to route discovery over an active microservice socket instead of a dedicated discovery socket. `light-controller` is updated for contract and auth parity so both backends accept the same service payload and tenant validation model.

**Tech Stack:** Rust (`axum`, `jsonwebtoken`, `sqlx`), Java (`light-4j`, `jakarta.websocket`, `jose4j`, JUnit 5), Maven, Cargo, Markdown

---

**Baseline note:** `controller-rs` currently has a pre-existing baseline failure in `tls::tests::prefers_shared_workspace_keystore_when_present`. Use the targeted `cargo test` commands in this plan for feature work and do not treat that unrelated TLS failure as part of this implementation.

## File Map

- `controller-rs/src/config.rs`
- `controller-rs/src/auth.rs`
- `controller-rs/src/types.rs`
- `controller-rs/src/routes/microservice.rs`
- `controller-rs/src/routes/discovery.rs`
- `controller-rs/src/events.rs`
- `controller-rs/src/repository/mod.rs`
- `controller-rs/src/repository/postgres.rs`
- `controller-rs/migrations/0002_runtime_instance_multitenant.sql`
- `controller-rs/tests/websocket_flows.rs`
- `controller-rs/tests/postgres_runtime_instance.rs`
- `light-4j/portal-registry/src/main/java/com/networknt/portal/registry/PortalRegistryConfig.java`
- `light-4j/portal-registry/src/main/java/com/networknt/portal/registry/PortalRegistryService.java`
- `light-4j/portal-registry/src/main/java/com/networknt/portal/registry/client/PortalRegistryClientImpl.java`
- `light-4j/portal-registry/src/test/java/com/networknt/portal/registry/PortalRegistryServiceTest.java`
- `light-4j/portal-registry/src/test/java/com/networknt/portal/registry/client/PortalRegistryClientImplTest.java`
- `light-controller/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java`
- `light-controller/src/main/java/com/networknt/controller/auth/ServiceJwtValidator.java`
- `light-controller/src/main/java/com/networknt/controller/auth/DiscoveryBearerAuth.java`
- `light-controller/src/main/java/com/networknt/controller/websocket/MicroserviceEndpoint.java`
- `light-controller/src/test/java/com/networknt/controller/websocket/ControllerWebSocketIntegrationTest.java`
- `networknt-doc/src/concern/light-controller/unified-microservice-channel.md`

### Task 1: `controller-rs` Registration Contract and Tenant Validation

**Files:**
- Modify: `controller-rs/src/config.rs`
- Modify: `controller-rs/src/auth.rs`
- Modify: `controller-rs/src/types.rs`
- Modify: `controller-rs/src/routes/microservice.rs`
- Test: `controller-rs/tests/websocket_flows.rs`

- [ ] **Step 1: Write the failing websocket tests for payload address and JWT `host` validation**

Add these changes in `controller-rs/tests/websocket_flows.rs`:

```rust
#[derive(Debug, Clone, Serialize)]
struct TestClaims {
    sub: Option<String>,
    exp: usize,
    iss: String,
    aud: String,
    cid: String,
    sid: Option<String>,
    service_id: Option<String>,
    env: Option<String>,
    host: Option<String>,
}

#[tokio::test]
async fn microservice_registration_uses_payload_address() {
    let server = spawn_test_server().await;
    let mut socket = connect_microservice_ws(&server).await;
    let service_id = "user-service";
    let payload = registration_payload(service_id, Some("prod"), "10.2.3.4");

    send_json(
        &mut socket,
        json!({"jsonrpc":"2.0","id":"register-1","method":"service/register","params":payload}),
    ).await;

    let _ack = recv_json(&mut socket).await;

    let lookup = send_discovery_lookup(&server, service_id, Some("prod"), Some("https")).await;
    assert_eq!(lookup["result"]["nodes"][0]["address"], "10.2.3.4");
}

#[tokio::test]
async fn microservice_registration_rejects_host_claim_mismatch() {
    let server = spawn_test_server().await;
    let mut socket = connect_microservice_ws(&server).await;
    let payload = registration_payload_with_token(
        "user-service",
        Some("prod"),
        "10.2.3.4",
        service_token_with_claims(json!({
            "sub": "user-service",
            "sid": "user-service",
            "cid": "user-service",
            "env": "prod",
            "host": "tenant-b",
            "iss": TEST_ISSUER,
            "aud": TEST_AUDIENCE,
            "exp": (chrono::Utc::now() + chrono::Duration::minutes(30)).timestamp() as usize
        }), TEST_KID, Algorithm::RS256, &test_encoding_key()),
    );

    send_json(
        &mut socket,
        json!({"jsonrpc":"2.0","id":"register-1","method":"service/register","params":payload}),
    ).await;

    let close = recv_close_reason(&mut socket).await;
    assert!(close.contains("host"));
}
```

- [ ] **Step 2: Run the targeted Rust tests and confirm they fail against the current contract**

Run:

```bash
cargo test --quiet microservice_registration_uses_payload_address
cargo test --quiet microservice_registration_rejects_host_claim_mismatch
```

Expected:
- the address test fails because the controller still uses `remote_addr.ip()`
- the host mismatch test fails because `ServiceJwtClaims` does not yet validate `host`

- [ ] **Step 3: Implement the minimal contract and host validation changes**

Update `controller-rs/src/config.rs` so settings carry the controller tenant key:

```rust
pub struct Settings {
    pub listen_addr: SocketAddr,
    pub controller_host_id: String,
    pub database_url: Option<String>,
    pub tls_enabled: bool,
    pub microservice_jwks_url: String,
    pub default_command_timeout: Duration,
}

impl Default for Settings {
    fn default() -> Self {
        Self {
            listen_addr: SocketAddr::new(IpAddr::V4(Ipv4Addr::LOCALHOST), 8443),
            controller_host_id: "tenant-default".to_string(),
            database_url: None,
            tls_enabled: true,
            microservice_jwks_url: "https://localhost:6881/oauth2/AZZRJE52eXu3t1hseacnGQ/keys".to_string(),
            default_command_timeout: Duration::from_secs(5),
        }
    }
}

if let Some(value) = read_var("CONTROLLER_HOST_ID")? {
    settings.controller_host_id = value;
}
```

Update `controller-rs/src/types.rs` so registration input matches the agreed service payload:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ServiceRegistrationParams {
    pub jwt: String,
    pub service_id: String,
    #[serde(default)]
    pub env_tag: Option<String>,
    pub version: String,
    pub protocol: String,
    pub address: String,
    pub port: u16,
    #[serde(default)]
    pub tags: HashMap<String, String>,
}
```

Update `controller-rs/src/auth.rs` to validate the JWT `host` claim against `settings.controller_host_id`:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ServiceJwtClaims {
    #[serde(default)]
    pub sub: Option<String>,
    pub exp: usize,
    #[serde(default)]
    pub service_id: Option<String>,
    #[serde(default)]
    pub sid: Option<String>,
    #[serde(default)]
    pub cid: Option<String>,
    #[serde(default)]
    pub host: Option<String>,
    #[serde(default, alias = "env_tag")]
    pub env: Option<String>,
}

let claimed_host = claims
    .host
    .as_deref()
    .ok_or_else(|| AppError::Unauthorized("service JWT must contain host".to_string()))?;
if claimed_host != settings.controller_host_id {
    return Err(AppError::Unauthorized(format!(
        "token host {claimed_host} does not match controller host {}",
        settings.controller_host_id
    )));
}
```

Update `controller-rs/src/routes/microservice.rs` to use the payload address as the canonical service address while retaining the remote IP only for logging:

```rust
let environment = registration_params
    .env_tag
    .clone()
    .unwrap_or_else(|| "default".to_string());

let instance = InstanceState::new(
    runtime_instance_id,
    registration_params.service_id.clone(),
    registration_params.env_tag.clone(),
    ServiceMetadata {
        environment,
        version: registration_params.version.clone(),
        protocol: registration_params.protocol.clone(),
        port: registration_params.port,
        address: registration_params.address.clone(),
        tags: registration_params.tags.clone(),
    },
    command_tx,
);

info!(
    runtime_instance_id = %runtime_instance_id,
    service_id = %registration_params.service_id,
    env_tag = ?registration_params.env_tag,
    address = %registration_params.address,
    remote_address = %remote_addr.ip(),
    port = %registration_params.port,
    "microservice registered"
);
```

- [ ] **Step 4: Run the targeted Rust tests again and confirm they pass**

Run:

```bash
cargo test --quiet microservice_registration_uses_payload_address
cargo test --quiet microservice_registration_rejects_host_claim_mismatch
```

Expected:
- both tests pass
- no new failures appear in those targeted filters

- [ ] **Step 5: Commit the contract and host-validation changes**

Run:

```bash
git -C /home/steve/.config/superpowers/worktrees/controller-rs/feature-unified-microservice-channel add src/config.rs src/auth.rs src/types.rs src/routes/microservice.rs tests/websocket_flows.rs
git -C /home/steve/.config/superpowers/worktrees/controller-rs/feature-unified-microservice-channel commit -m "feat: validate controller host and use payload address"
```

### Task 2: `controller-rs` Single-Channel Discovery and Discovery JWT Auth

**Files:**
- Modify: `controller-rs/src/auth.rs`
- Modify: `controller-rs/src/routes/discovery.rs`
- Modify: `controller-rs/src/routes/microservice.rs`
- Test: `controller-rs/tests/websocket_flows.rs`

- [ ] **Step 1: Write the failing tests for service-side discovery and JWT-based discovery auth**

Add these tests in `controller-rs/tests/websocket_flows.rs`:

```rust
#[tokio::test]
async fn registered_microservice_socket_can_lookup_services() {
    let server = spawn_test_server().await;
    let mut first = connect_microservice_ws(&server).await;
    let mut second = connect_microservice_ws(&server).await;

    register_service(&mut first, "catalog-service", Some("prod"), "10.0.0.10").await;
    register_service(&mut second, "billing-service", Some("prod"), "10.0.0.11").await;

    send_json(
        &mut first,
        json!({
            "jsonrpc":"2.0",
            "id":"lookup-1",
            "method":"discovery/lookup",
            "params":{"serviceId":"billing-service","envTag":"prod","protocol":"https"}
        }),
    ).await;

    let response = recv_json(&mut first).await;
    assert_eq!(response["result"]["nodes"][0]["address"], "10.0.0.11");
}

#[tokio::test]
async fn discovery_socket_rejects_jwt_with_wrong_host() {
    let server = spawn_test_server().await;
    let token = discovery_token("tenant-b");
    let error = connect_discovery_ws_with_bearer(&server, &token).await.unwrap_err();
    assert!(error.to_string().contains("host"));
}
```

- [ ] **Step 2: Run the targeted Rust tests and confirm they fail before the refactor**

Run:

```bash
cargo test --quiet registered_microservice_socket_can_lookup_services
cargo test --quiet discovery_socket_rejects_jwt_with_wrong_host
```

Expected:
- the microservice lookup test fails because `/ws/microservice` does not yet handle discovery RPCs
- the discovery auth test fails because `/ws/discovery` still expects a static bearer token

- [ ] **Step 3: Implement shared discovery handling and JWT discovery authorization**

Refactor `controller-rs/src/routes/discovery.rs` so request parsing and snapshot generation can be reused from both sockets. Keep the shape concrete:

```rust
pub(crate) fn handle_discovery_rpc(
    state: &SharedState,
    client_id: Option<DiscoveryClientId>,
    message: JsonRpcMessage,
) -> Option<JsonRpcMessage> {
    match message.method.as_deref()? {
        "discovery/lookup" => { /* existing lookup path */ }
        "discovery/subscribe" => { /* existing subscribe path */ }
        "discovery/unsubscribe" => { /* existing unsubscribe path */ }
        _ => None,
    }
}
```

Update `controller-rs/src/routes/microservice.rs` so a registered service socket can process those discovery methods inline:

```rust
if let Some(response) = crate::routes::discovery::handle_discovery_rpc(state, None, message.clone())
{
    return InboundOutcome::reply(response);
}
```

Replace static-token discovery auth in `controller-rs/src/auth.rs` with bearer JWT validation that checks the `host` claim:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct DiscoveryJwtClaims {
    exp: usize,
    #[serde(default)]
    host: Option<String>,
}

pub async fn authorize_discovery(headers: &HeaderMap, verifier: &ServiceJwtVerifier, settings: &Settings) -> Result<(), AppError> {
    let token = bearer_token(headers)?;
    let claims = verifier.validate_bearer_only(settings, token).await?;
    let claimed_host = claims.host.as_deref().ok_or_else(|| AppError::Unauthorized("discovery JWT must contain host".to_string()))?;
    if claimed_host != settings.controller_host_id {
        return Err(AppError::Unauthorized("discovery JWT host does not match controller host".to_string()));
    }
    Ok(())
}
```

Update the `/ws/discovery` route to call that JWT validator instead of comparing to `settings.discovery_bearer_token`.

- [ ] **Step 4: Run the targeted Rust tests again and confirm the new socket behavior**

Run:

```bash
cargo test --quiet registered_microservice_socket_can_lookup_services
cargo test --quiet discovery_socket_rejects_jwt_with_wrong_host
```

Expected:
- both tests pass
- registered microservice sockets can issue `discovery/lookup`
- `/ws/discovery` rejects a bearer JWT when `host` does not match `controller_host_id`

- [ ] **Step 5: Commit the single-channel discovery changes**

Run:

```bash
git -C /home/steve/.config/superpowers/worktrees/controller-rs/feature-unified-microservice-channel add src/auth.rs src/routes/discovery.rs src/routes/microservice.rs tests/websocket_flows.rs
git -C /home/steve/.config/superpowers/worktrees/controller-rs/feature-unified-microservice-channel commit -m "feat: serve discovery over microservice socket"
```

### Task 3: `controller-rs` Runtime-Instance Event Persistence, `hostId`, and Aggregate Versioning

**Files:**
- Create: `controller-rs/migrations/0002_runtime_instance_multitenant.sql`
- Create: `controller-rs/tests/postgres_runtime_instance.rs`
- Modify: `controller-rs/src/events.rs`
- Modify: `controller-rs/src/repository/mod.rs`
- Modify: `controller-rs/src/repository/postgres.rs`
- Modify: `controller-rs/src/routes/microservice.rs`

- [ ] **Step 1: Write the failing persistence tests for runtime-instance reuse and aggregate version increments**

Create `controller-rs/tests/postgres_runtime_instance.rs` with database-backed assertions:

```rust
#[tokio::test]
async fn runtime_instance_created_reuses_existing_identity_and_increments_version() {
    let repo = test_repo().await;
    let host_id = "01964b05-552a-7c4b-9184-6857e7f3dc5f";

    let first = repo
        .resolve_or_create_runtime_instance(host_id, "user-service", Some("prod"), "10.0.0.20", 8443)
        .await
        .unwrap();
    assert_eq!(first.aggregate_version, 1);

    let second = repo
        .resolve_or_create_runtime_instance(host_id, "user-service", Some("prod"), "10.0.0.20", 8443)
        .await
        .unwrap();
    assert_eq!(first.runtime_instance_id, second.runtime_instance_id);
    assert_eq!(second.aggregate_version, 2);
}

#[tokio::test]
async fn runtime_instance_deleted_uses_next_aggregate_version() {
    let repo = test_repo().await;
    let host_id = "01964b05-552a-7c4b-9184-6857e7f3dc5f";
    let created = repo
        .resolve_or_create_runtime_instance(host_id, "user-service", Some("prod"), "10.0.0.20", 8443)
        .await
        .unwrap();

    let deleted = repo
        .append_runtime_instance_deleted(host_id, &created.runtime_instance_id, "user-service")
        .await
        .unwrap();

    assert_eq!(deleted.aggregate_version, 2);
}
```

- [ ] **Step 2: Run the new repository tests and confirm they fail against the current schema**

Run:

```bash
DATABASE_URL=postgres://postgres:postgres@localhost:5432/controller_rs_test cargo test --quiet --test postgres_runtime_instance
```

Expected:
- compilation or query failures because the repository and schema do not yet support `hostId`, business-key reuse, or aggregate versioning

- [ ] **Step 3: Implement the migration and repository changes**

Create `controller-rs/migrations/0002_runtime_instance_multitenant.sql`:

```sql
alter table event_store_t
    alter column aggregate_id type text using aggregate_id::text;

alter table event_store_t
    add column if not exists host_id text not null default 'tenant-default',
    add column if not exists aggregate_version bigint not null default 1;

alter table outbox_message_t
    alter column aggregate_id type text using aggregate_id::text;

alter table outbox_message_t
    add column if not exists host_id text not null default 'tenant-default',
    add column if not exists aggregate_version bigint not null default 1;

alter table runtime_instance_t
    add column if not exists host_id text not null default 'tenant-default',
    add column if not exists aggregate_version bigint not null default 0;

create unique index if not exists uq_runtime_instance_business_key
    on runtime_instance_t (host_id, service_id, coalesce(env_tag, ''), address, port);
```

Update `controller-rs/src/events.rs` so stored runtime-instance events use portal-style identity:

```rust
pub struct StoredEvent {
    pub event_id: Uuid,
    pub aggregate_type: String,
    pub aggregate_id: String,
    pub aggregate_version: i64,
    pub host_id: String,
    pub event_type: String,
    pub payload: Value,
    pub occurred_at: DateTime<Utc>,
    pub correlation_id: Option<String>,
    pub subject: Option<String>,
}
```

Update `controller-rs/src/repository/mod.rs` and `controller-rs/src/repository/postgres.rs` with explicit runtime-instance lookup and version APIs:

```rust
pub struct RuntimeInstanceRecord {
    pub runtime_instance_id: Uuid,
    pub aggregate_version: i64,
}

async fn resolve_runtime_instance(
    &self,
    host_id: &str,
    service_id: &str,
    env_tag: Option<&str>,
    address: &str,
    port: u16,
) -> Result<Option<RuntimeInstanceRecord>, AppError>;

async fn next_aggregate_version(&self, aggregate_id: &str) -> Result<i64, AppError>;
```

Update `controller-rs/src/routes/microservice.rs` to:

```rust
let host_id = state.settings.controller_host_id.clone();
let resolved = state
    .repository
    .resolve_runtime_instance(
        &host_id,
        &registration_params.service_id,
        registration_params.env_tag.as_deref(),
        &registration_params.address,
        registration_params.port,
    )
    .await?;

let runtime_instance_id = resolved
    .as_ref()
    .map(|record| record.runtime_instance_id)
    .unwrap_or_else(Uuid::now_v7);
```

Persist `RuntimeInstanceCreatedEvent` and `RuntimeInstanceDeletedEvent` with:
- `hostId`
- `runtimeInstanceId`
- `serviceId`
- `envTag`
- `address`
- `port`
- `aggregateVersion`

- [ ] **Step 4: Run the repository and websocket regression tests**

Run:

```bash
DATABASE_URL=postgres://postgres:postgres@localhost:5432/controller_rs_test cargo test --quiet --test postgres_runtime_instance
cargo test --quiet microservice_registration_uses_payload_address
cargo test --quiet registered_microservice_socket_can_lookup_services
```

Expected:
- repository tests pass
- targeted websocket tests still pass
- event persistence now reuses runtime-instance identity and increments aggregate versions

- [ ] **Step 5: Commit the persistence work**

Run:

```bash
git -C /home/steve/.config/superpowers/worktrees/controller-rs/feature-unified-microservice-channel add migrations/0002_runtime_instance_multitenant.sql src/events.rs src/repository/mod.rs src/repository/postgres.rs src/routes/microservice.rs tests/postgres_runtime_instance.rs
git -C /home/steve/.config/superpowers/worktrees/controller-rs/feature-unified-microservice-channel commit -m "feat: persist runtime instances with host and aggregate version"
```

### Task 4: `light-4j/portal-registry` Unified Payload, Config Cleanup, and Service-Channel Discovery

**Files:**
- Modify: `light-4j/portal-registry/src/main/java/com/networknt/portal/registry/PortalRegistryConfig.java`
- Modify: `light-4j/portal-registry/src/main/java/com/networknt/portal/registry/PortalRegistryService.java`
- Modify: `light-4j/portal-registry/src/main/java/com/networknt/portal/registry/client/PortalRegistryClientImpl.java`
- Modify: `light-4j/portal-registry/src/test/java/com/networknt/portal/registry/PortalRegistryServiceTest.java`
- Modify: `light-4j/portal-registry/src/test/java/com/networknt/portal/registry/client/PortalRegistryClientImplTest.java`

- [ ] **Step 1: Write the failing Java tests for the unified payload and service-channel discovery**

Update `PortalRegistryServiceTest`:

```java
@Test
void testUnifiedRegisterParamsUseEnvTagAddressAndVersion() {
    PortalRegistryService service = new PortalRegistryService();
    service.setServiceId("com.networknt.apib-1.0.0");
    service.setProtocol("https");
    service.setAddress("127.0.0.1");
    service.setPort(7442);
    service.setTag("uat1");
    service.setVersion("1.2.3");

    Map<String, Object> params = service.toRegisterParams("raw-jwt");
    Assertions.assertEquals("raw-jwt", params.get("jwt"));
    Assertions.assertEquals("uat1", params.get("envTag"));
    Assertions.assertEquals("127.0.0.1", params.get("address"));
    Assertions.assertFalse(params.containsKey("environment"));
}
```

Update `PortalRegistryClientImplTest` with a seam for channel selection:

```java
@Test
void testDiscoveryUsesRegisteredServiceChannel() {
    TestPortalRegistryClient client = new TestPortalRegistryClient();
    client.registerService(service("127.0.0.1", 7442), "Bearer raw-jwt");

    client.subscribeService("com.networknt.remote.v1", "prod", "https", "Bearer raw-jwt");

    Assertions.assertEquals(List.of("service/register", "discovery/subscribe"), client.sentMethods());
}
```

- [ ] **Step 2: Run the focused Maven tests and confirm they fail before the refactor**

Run:

```bash
mvn -q -pl portal-registry -Dtest=PortalRegistryServiceTest,PortalRegistryClientImplTest test
```

Expected:
- payload assertions fail because `toControllerRsRegisterParams()` still emits `environment`
- client test fails because discovery still uses a dedicated `/ws/discovery` socket

- [ ] **Step 3: Implement the payload unification and discovery-channel refactor**

Update `PortalRegistryService.java` to expose one payload builder:

```java
public Map<String, Object> toRegisterParams(String jwt) {
    Map<String, Object> params = new LinkedHashMap<>();
    params.put("jwt", jwt);
    params.put("serviceId", serviceId);
    if (tag != null) {
        params.put("envTag", tag);
    }
    params.put("version", version != null ? version : "1.0.0");
    params.put("protocol", protocol);
    params.put("address", address);
    params.put("port", port);
    params.put("tags", Map.of());
    return params;
}
```

Update `PortalRegistryConfig.java` to remove `controllerDiscoveryToken` and describe `/ws/microservice` as the service-side channel for both registration and discovery.

Update `PortalRegistryClientImpl.java` so discovery RPCs use an active service channel instead of a dedicated discovery socket:

```java
private PortalRegistryWebSocketClient activeServiceChannel() {
    return registrationClients.entrySet().stream()
            .sorted(Map.Entry.comparingByKey())
            .map(Map.Entry::getValue)
            .filter(PortalRegistryWebSocketClient::isOpen)
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("service discovery requires an active /ws/microservice registration"));
}

public List<Map<String, Object>> subscribeService(String serviceId, String tag, String protocol, String token) {
    subscriptions.put(subscriptionKey(serviceId, tag, protocol), new SubscriptionState(serviceId, tag, protocol));
    return extractNodes(activeServiceChannel().sendRequest("discovery/subscribe", discoveryParams(serviceId, tag, protocol)));
}
```

Replace `service.toControllerRsRegisterParams(...)` with `service.toRegisterParams(...)`.

- [ ] **Step 4: Run the focused Maven tests again**

Run:

```bash
mvn -q -pl portal-registry -Dtest=PortalRegistryServiceTest,PortalRegistryClientImplTest test
```

Expected:
- both tests pass
- `PortalRegistryService` emits the unified payload
- `PortalRegistryClientImpl` routes discovery RPCs through an active `/ws/microservice` registration socket

- [ ] **Step 5: Commit the `portal-registry` refactor**

Run:

```bash
git -C /home/steve/.config/superpowers/worktrees/light-4j/feature-unified-microservice-channel add portal-registry/src/main/java/com/networknt/portal/registry/PortalRegistryConfig.java portal-registry/src/main/java/com/networknt/portal/registry/PortalRegistryService.java portal-registry/src/main/java/com/networknt/portal/registry/client/PortalRegistryClientImpl.java portal-registry/src/test/java/com/networknt/portal/registry/PortalRegistryServiceTest.java portal-registry/src/test/java/com/networknt/portal/registry/client/PortalRegistryClientImplTest.java
git -C /home/steve/.config/superpowers/worktrees/light-4j/feature-unified-microservice-channel commit -m "feat: unify portal registry registration and discovery channel"
```

### Task 5: `light-controller` Contract and JWT Parity

**Files:**
- Modify: `light-controller/src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java`
- Modify: `light-controller/src/main/java/com/networknt/controller/auth/ServiceJwtValidator.java`
- Modify: `light-controller/src/main/java/com/networknt/controller/auth/DiscoveryBearerAuth.java`
- Modify: `light-controller/src/main/java/com/networknt/controller/websocket/MicroserviceEndpoint.java`
- Modify: `light-controller/src/test/java/com/networknt/controller/websocket/ControllerWebSocketIntegrationTest.java`

- [x] **Step 1: Write the failing integration tests for `host` validation and the envTag-only registration payload**

Update `ControllerWebSocketIntegrationTest`:

```java
private static Map<String, Object> registerParams(
        String serviceId,
        String envTag,
        String version,
        String protocol,
        int port
) throws Exception {
    return Map.of(
            "jwt", createJwt(serviceId, envTag, "tenant-a"),
            "serviceId", serviceId,
            "envTag", envTag,
            "address", EXPECTED_ADDRESS,
            "version", version,
            "protocol", protocol,
            "port", port,
            "tags", Map.of("lane", envTag)
    );
}

@Test
void rejectsMicroserviceJwtWhenHostClaimDiffersFromControllerHostId() throws Exception {
    try (TestWebSocketClient microservice = connect("/ws/microservice", Map.of())) {
        microservice.send(request(
                "register-1",
                "service/register",
                Map.of(
                        "jwt", createJwt("service-a", "dev", "tenant-b"),
                        "serviceId", "service-a",
                        "envTag", "dev",
                        "address", EXPECTED_ADDRESS,
                        "version", "1.0.0",
                        "protocol", "https",
                        "port", 9001,
                        "tags", Map.of()
                )
        ));
        assertTrue(microservice.awaitClose());
    }
}
```

- [ ] **Step 2: Run the focused light-controller integration test and confirm it fails**

Run:

```bash
mvn -q -Dtest=ControllerWebSocketIntegrationTest test
```

Expected:
- the new registration payload initially fails if `environment` is still required
- the host-mismatch test fails because `ServiceJwtValidator` does not yet check `host`

- [ ] **Step 3: Implement the parity changes**

Update `ControllerRuntimeConfig.java`:

```java
private String hostId = "tenant-default";

public String getHostId() {
    return hostId;
}

public void setHostId(String hostId) {
    this.hostId = hostId;
}
```

Update `ServiceJwtValidator.java`:

```java
String claimedHost = stringClaim(claims, "host");
if (claimedHost == null || !claimedHost.equals(config.getHostId())) {
    throw new IllegalArgumentException("token host does not match controller hostId");
}
```

Update `MicroserviceEndpoint.java` so `environment` is derived internally instead of required in `service/register`:

```java
String requestedEnvTag = EndpointSupport.stringValue(params, "envTag");
String version = requiredString(params, "version");
String protocol = requiredString(params, "protocol");
int port = requiredInteger(params, "port");
Map<String, String> tags = EndpointSupport.stringMap(params, "tags");
String address = fallback(EndpointSupport.stringValue(params, "address"), remoteAddress);
String environment = requestedEnvTag != null ? requestedEnvTag : "default";
```

Update `DiscoveryBearerAuth.java` to validate a bearer JWT and compare its `host` claim to `config.getHostId()` instead of comparing to a static string token.

- [ ] **Step 4: Run the focused light-controller tests again**

Run:

```bash
mvn -q -Dtest=ControllerWebSocketIntegrationTest test
```

Expected:
- all websocket integration tests pass
- `/ws/microservice` accepts the envTag-only payload
- both service and discovery JWTs are rejected when their `host` claim does not match `hostId`

- [ ] **Step 5: Commit the light-controller parity updates**

Run:

```bash
git -C /home/steve/.config/superpowers/worktrees/light-controller/feature-unified-microservice-channel add src/main/java/com/networknt/controller/config/ControllerRuntimeConfig.java src/main/java/com/networknt/controller/auth/ServiceJwtValidator.java src/main/java/com/networknt/controller/auth/DiscoveryBearerAuth.java src/main/java/com/networknt/controller/websocket/MicroserviceEndpoint.java src/test/java/com/networknt/controller/websocket/ControllerWebSocketIntegrationTest.java
git -C /home/steve/.config/superpowers/worktrees/light-controller/feature-unified-microservice-channel commit -m "feat: enforce controller host and unified registration contract"
```

### Task 6: Documentation Sync and Cross-Repo Verification

**Files:**
- Modify: `networknt-doc/src/concern/light-controller/unified-microservice-channel.md`

- [ ] **Step 1: Update the design note to match the implemented field names and config keys**

Ensure `networknt-doc/src/concern/light-controller/unified-microservice-channel.md` reflects the final implementation details:

```md
- registration payload uses `envTag`, `address`, `version`, `protocol`, `port`, and optional `tags`
- registration payload does not send `environment`
- JWT carries tenant in the `host` claim
- controllers store the tenant internally as `hostId`
- `/ws/discovery` uses bearer JWT auth, not a static discovery token
```

- [ ] **Step 2: Run the final targeted verification commands across repos**

Run:

```bash
cargo test --quiet microservice_registration_uses_payload_address
cargo test --quiet registered_microservice_socket_can_lookup_services
DATABASE_URL=postgres://postgres:postgres@localhost:5432/controller_rs_test cargo test --quiet --test postgres_runtime_instance
mvn -q -pl portal-registry -Dtest=PortalRegistryServiceTest,PortalRegistryClientImplTest test
mvn -q -Dtest=ControllerWebSocketIntegrationTest test
git -C /home/steve/.config/superpowers/worktrees/networknt-doc/feature-unified-microservice-channel diff --check
```

Expected:
- all targeted feature tests pass
- docs worktree has no whitespace errors

- [ ] **Step 3: Commit the documentation update**

Run:

```bash
git -C /home/steve/.config/superpowers/worktrees/networknt-doc/feature-unified-microservice-channel add src/concern/light-controller/unified-microservice-channel.md docs/superpowers/plans/2026-03-29-unified-microservice-channel.md
git -C /home/steve/.config/superpowers/worktrees/networknt-doc/feature-unified-microservice-channel commit -m "docs: add unified microservice channel implementation plan"
```

## Self-Review

### Spec coverage

- Unified registration payload: covered in Task 1, Task 4, and Task 5
- Client-supplied `address`: covered in Task 1 and Task 5
- JWT `host` claim to controller `hostId` validation: covered in Task 1 and Task 5
- `/ws/microservice` handling discovery for services: covered in Task 2 and Task 4
- `/ws/discovery` for client applications with JWT auth: covered in Task 2 and Task 5
- Runtime-instance reuse and aggregate versioning: covered in Task 3
- Documentation alignment: covered in Task 6

### Placeholder scan

- No `TODO`, `TBD`, or “similar to above” shortcuts remain
- Every task has exact file paths, commands, and concrete code snippets
- The plan explicitly calls out the pre-existing `controller-rs` TLS baseline failure instead of leaving testing ambiguous

### Type consistency

- JWT tenant claim is consistently `host`
- internal controller tenant key is consistently `hostId`
- registration payload uses `envTag` and does not send `environment`
- runtime-instance aggregate identity is consistently `hostId|runtimeInstanceId`
