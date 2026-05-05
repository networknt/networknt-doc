# Kubernetes Sidecar Template

Status: Draft

Last updated: 2026-05-05

## Introduction

This document defines the Kubernetes sidecar deployment pattern for Light APIs
that run an `http-sidecar` container next to the business API container in the
same pod.

The first concrete target is `openapi-petstore` in the `openapi-petstore`
namespace. The same pattern should be reusable for other APIs after the first
deployment is proven in MicroK8s.

The related registry migration is documented separately in
[Portal Registry Transition](./portal-registry-transition.md). This document
focuses only on the Kubernetes template, pod shape, config bootstrap, and
operator workflow.

## Goals

- Define a reusable sidecar deployment pattern for Light APIs on Kubernetes.
- Deploy `http-sidecar` and the backend API in the same pod.
- Expose the sidecar as the only Kubernetes Service target for client traffic.
- Keep backend API traffic private to the pod through `127.0.0.1`.
- Keep the concrete API deployment template in the API repository.
- Keep reusable sidecar contract details in the `http-sidecar` repository.
- Support `light-deployer` template rendering with `${key:default}`
  placeholders.
- Keep the first rollout simple enough for local MicroK8s validation.

## Non-Goals

- Do not introduce Helm, Kustomize, or a Kubernetes operator in the first
  iteration.
- Do not make `http-sidecar` own a deployable manifest for every API.
- Do not expose the backend API container through a separate Kubernetes
  Service.
- Do not solve registry migration in this document.
- Do not require every backend API to use the same runtime language or Light
  framework internals.

## Design Summary

Each API repository owns its deployable Kubernetes template. The template
defines one Deployment with two containers:

- `http-sidecar`: the ingress and optional egress boundary for the pod
- backend API container: business handlers and local backend HTTP server

The Kubernetes Service selects the pod and targets the sidecar port:

```text
client or gateway
  -> Service/openapi-petstore
  -> http-sidecar container
  -> 127.0.0.1 backend API container
```

The backend container should not be reachable directly by other pods. This
keeps security, validation, audit, rate limiting, request/response injection,
and routing behavior centralized at the sidecar.

## Repository Ownership

### `http-sidecar`

The `http-sidecar` repository owns the reusable sidecar contract:

- supported sidecar image names and tags
- sidecar default ports
- sidecar health endpoints
- required environment variables
- config-server bootstrap expectations
- standard handler-chain examples
- `proxy.hosts` examples for pod-local backends
- troubleshooting notes for sidecar startup and proxy behavior

It may contain examples, but it should not become the source of truth for a
specific API deployment.

### API Repositories

Each API repository owns the concrete Kubernetes template that deploys that
API with the sidecar. For `openapi-petstore`, this means
`openapi-petstore/k8s` owns:

- Deployment
- Service
- API-specific ConfigMap or Secret templates
- deployer render request examples
- image defaults for both containers
- resource requests and limits
- probe timings

This keeps the deployment close to the API version, API image, and backend
runtime settings.

## Namespace Model

Create the namespace outside the application template:

```bash
microk8s kubectl create namespace openapi-petstore --dry-run=client -o yaml \
  | microk8s kubectl apply -f -
```

The deployer template should render namespaced application resources only.
Keeping Namespace creation separate avoids cluster-policy problems and makes
the application template portable across local and managed clusters.

For the first deployment:

```text
namespace: openapi-petstore
deployment: openapi-petstore
service: openapi-petstore
```

## Pod Contract

Both containers run in the same pod, so they share the pod network namespace.
The sidecar can proxy to the backend through `127.0.0.1`.

Recommended first target:

| Container | Port | Purpose |
| --- | --- | --- |
| `http-sidecar` | `9445` | HTTPS ingress exposed through Service |
| `http-sidecar` | `9080` | HTTP sidecar port exposed through Service |
| `openapi-petstore` | `8080` | pod-local backend HTTP |

The first `openapi-petstore` rollout should use backend HTTP only. The backend
listens on `8080`, disables HTTPS, and is reachable only through the sidecar.
The sidecar proxies to the backend through the pod-local loopback address:

```yaml
proxy.hosts: http://127.0.0.1:8080
```

## Service Contract

The Kubernetes Service exposes the sidecar only:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${service.name:openapi-petstore}
  namespace: ${namespace:openapi-petstore}
spec:
  type: ${service.type:ClusterIP}
  selector:
    app.kubernetes.io/name: ${name:openapi-petstore}
  ports:
    - name: https
      protocol: TCP
      port: ${service.httpsPort:9445}
      targetPort: sidecar-https
    - name: http
      protocol: TCP
      port: ${service.httpPort:9080}
      targetPort: sidecar-http
```

Both Service ports target the sidecar. The backend port must not be the public
Service target. If a temporary debug Service is needed, it should be created
outside the application template and removed after troubleshooting.

## Deployment Shape

The Deployment template should use stable labels shared by the Deployment,
pod template, and Service:

```yaml
app.kubernetes.io/name: ${name:openapi-petstore}
app.kubernetes.io/component: api
app.kubernetes.io/part-of: lightapi
```

The pod contains both containers:

```yaml
containers:
  - name: http-sidecar
    image: ${sidecar.image.repository:networknt/http-sidecar}:${sidecar.image.tag:2.2.1}
    ports:
      - name: sidecar-https
        containerPort: ${sidecar.ports.https:9445}
      - name: sidecar-http
        containerPort: ${sidecar.ports.http:9080}

  - name: openapi-petstore
    image: ${api.image.repository:networknt/openapi-petstore}:${api.image.tag:2.3.4}
    ports:
      - name: backend-http
        containerPort: ${api.ports.http:8080}
```

Container order is not a dependency mechanism. Both containers start together.
Readiness probes must prevent the pod from receiving traffic until the sidecar
is ready to accept requests and the backend is healthy.

## Probes

Use both sidecar and backend readiness checks. The sidecar probe is the public
readiness signal for the pod:

```yaml
readinessProbe:
  httpGet:
    scheme: HTTPS
    path: /health
    port: sidecar-https
  initialDelaySeconds: ${sidecar.probes.readinessInitialDelaySeconds:10}
  periodSeconds: ${sidecar.probes.readinessPeriodSeconds:10}
```

If the sidecar composite health endpoint is enabled, use the endpoint that also
checks backend reachability. If TLS probe handling is inconvenient in a local
cluster, a TCP probe is acceptable for the first iteration:

```yaml
readinessProbe:
  tcpSocket:
    port: sidecar-https
```

The backend should also have a local probe so a broken backend container is
visible directly in pod status:

```yaml
readinessProbe:
  tcpSocket:
    port: backend-http
```

## Config Bootstrap

The template should support config-server bootstrap for the sidecar and,
optionally, the backend.

Use separate authorization Secret keys when both containers load from the config
server. The `openapi-petstore` token is for backend service id
`com.networknt.petstore-3.1.0` in `dev`. The sidecar should use a token for
`com.networknt.petstore-lpsc-3.1.0` in `dev`.

Recommended sidecar environment variables:

```yaml
env:
  - name: LIGHT_4J_CONFIG_PASSWORD
    valueFrom:
      secretKeyRef:
        name: ${name:openapi-petstore}-bootstrap
        key: light-4j-config-password
  - name: LIGHT_CONFIG_SERVER_URI
    value: ${configServer.uri:https://light-config-server:8435}
  - name: CONFIG_SERVER_CLIENT_TRUSTSTORE_LOCATION
    value: /config/bootstrap.truststore
  - name: CONFIG_SERVER_CLIENT_TRUSTSTORE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: ${name:openapi-petstore}-bootstrap
        key: config-server-client-truststore-password
  - name: CONFIG_SERVER_CLIENT_VERIFY_HOST_NAME
    value: "${configServer.verifyHostName:false}"
  - name: LIGHT_PORTAL_AUTHORIZATION
    valueFrom:
      secretKeyRef:
        name: ${name:openapi-petstore}-bootstrap
        key: sidecar-light-portal-authorization
  - name: LIGHT_ENV
    value: ${lightEnv:dev}
```

The startup file should identify the sidecar runtime config:

```yaml
startup.yml: |
  configLoaderClass: com.networknt.server.DefaultConfigLoader
  host: ${sidecar.startup.host:dev.lightapi.net}
  serviceId: ${sidecar.startup.serviceId:com.networknt.petstore-lpsc-3.1.0}
  envTag: ${sidecar.startup.envTag:dev}
  acceptHeader: application/yaml
  timeout: ${sidecar.startup.timeout:3000}
```

If the backend loads its own config from the config server, give it separate
startup values. The backend should not use the same external service identity
as the sidecar. The backend container can use the same bootstrap pattern with
`api-light-portal-authorization` as the Secret key.

## Runtime Configuration

The sidecar owns the external API behavior:

```yaml
server.serviceId: com.networknt.petstore-lpsc-3.1.0
server.environment: dev
server.httpPort: 9080
server.httpsPort: 9445
proxy.hosts: http://127.0.0.1:8080
```

The backend should focus on business logic:

```yaml
server.serviceId: com.networknt.petstore-3.1.0
server.environment: dev
server.httpPort: 8080
server.enableHttp: true
server.enableHttps: false
server.enableRegistry: false
security.enableVerifyJwt: false
```

The exact backend settings can vary by API, but the important rule is that the
sidecar owns externally visible cross-cutting concerns.

## Service Identity

Only one container should register the externally routable API identity with
the controller.

Recommended rule:

- sidecar registers as `com.networknt.petstore-lpsc-3.1.0` in `dev`
- backend uses `com.networknt.petstore-3.1.0` in `dev` but disables controller
  registration inside the sidecar pod

If the backend must register for diagnostics, it should use a private internal
service id that consumers do not discover or route to directly.

## Light Deployer Values

The template should use nested values so sidecar and backend settings do not
collide:

```yaml
namespace: openapi-petstore
name: openapi-petstore
replicas: 1

sidecar:
  image:
    repository: networknt/http-sidecar
    tag: 2.2.1
    pullPolicy: IfNotPresent
  ports:
    https: 9445
    http: 9080
  startup:
    host: dev.lightapi.net
    serviceId: com.networknt.petstore-lpsc-3.1.0
    envTag: dev
  lightPortalAuthorization: Bearer <http-sidecar-token>

api:
  image:
    repository: networknt/openapi-petstore
    tag: 2.3.4
    pullPolicy: IfNotPresent
  ports:
    http: 8080
  startup:
    serviceId: com.networknt.petstore-3.1.0
    envTag: dev
  lightPortalAuthorization: Bearer <openapi-petstore-token>

service:
  name: openapi-petstore
  type: ClusterIP
  httpsPort: 9445
  httpPort: 9080

configServer:
  uri: https://192.168.5.85:8435
  verifyHostName: false

lightEnv: dev
light4jConfigPassword: DEV
configServerTruststorePassword: password
bootstrapTruststoreBase64: <base64-bootstrap-truststore>
```

The deploy request should point to the API repository template:

```json
{
  "namespace": "openapi-petstore",
  "action": "deploy",
  "template": {
    "repoUrl": "local",
    "ref": "main",
    "path": "k8s"
  }
}
```

For an in-cluster deployer without a mounted local checkout, use a Git URL and
a committed branch instead of `repoUrl: "local"`.

## Resource Defaults

Start with conservative resource defaults and tune after the first MicroK8s
run:

```yaml
sidecar:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 768Mi

api:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 768Mi
```

Small examples can use lower defaults, but the template should make sidecar and
backend resources independently configurable.

## Rollout Flow

1. Build or publish the `http-sidecar` image.
2. Build or publish the API image.
3. Create the namespace.
4. Ensure config-server values exist for the sidecar startup identity.
5. Ensure backend config matches the selected port model.
6. Render the template through `light-deployer`.
7. Deploy the rendered resources to MicroK8s.
8. Verify pod readiness and sidecar logs.
9. Port-forward the Service and call the API through the sidecar.
10. Verify controller registration for the sidecar-facing service identity.

## Validation

### Template Validation

Render before deploy and check:

- Deployment namespace is `openapi-petstore`.
- Deployment has two containers.
- Service selector matches pod labels.
- Service ports target `sidecar-https` and `sidecar-http`.
- Backend port is not exposed as the Service target.
- Secret and ConfigMap names match container references.

### MicroK8s Validation

```bash
microk8s kubectl get namespace openapi-petstore
microk8s kubectl -n openapi-petstore get deploy,svc,pod
microk8s kubectl -n openapi-petstore rollout status deploy/openapi-petstore
microk8s kubectl -n openapi-petstore describe pod -l app.kubernetes.io/name=openapi-petstore
```

Check logs separately:

```bash
microk8s kubectl -n openapi-petstore logs deploy/openapi-petstore -c http-sidecar
microk8s kubectl -n openapi-petstore logs deploy/openapi-petstore -c openapi-petstore
```

Port-forward and test through the sidecar:

```bash
microk8s kubectl -n openapi-petstore port-forward svc/openapi-petstore 9445:9445 9080:9080
curl -k https://127.0.0.1:9445/health
curl -k https://127.0.0.1:9445/v1/pets
curl http://127.0.0.1:9080/health
```

### Controller Validation

Verify that the controller sees one external petstore service identity. The
backend should not appear as a duplicate externally routable petstore instance.

## Risks and Mitigations

### Backend Bypass

If the Service targets the backend container, callers bypass sidecar security
and validation.

Mitigation: the API Service must target only the sidecar ports. Backend debug
access should be temporary and outside the application template.

### Duplicate Controller Registration

Both containers could register the same service id.

Mitigation: the sidecar owns the external service id. Disable backend
registration in the sidecar pod.

### Startup Race

The sidecar can start before the backend is ready.

Mitigation: use readiness probes. If needed, configure sidecar health to include
backend reachability before marking the pod ready.

### Config Drift

The Kubernetes template and config-server values can disagree on ports or
service identity.

Mitigation: keep the deploy request values and config-server values together in
the deployment notes. Validate rendered ports before deploy.

### Template Duplication

Each API repository will initially copy the same sidecar block.

Mitigation: accept this for the first few APIs. Promote a shared generator or
template only after the repeated fields are stable.

## Decision

Use API-owned Kubernetes templates for concrete sidecar deployments.
`openapi-petstore/k8s` owns the first deployable two-container pod template.
`http-sidecar/k8s` owns reusable sidecar contract material and examples.

The Kubernetes Service exposes both sidecar ports, `9445` and `9080`. The
backend API listens on HTTP port `8080` only and remains pod-local. Readiness is
checked in both places: the sidecar reports public/composite readiness and the
backend container has its own readiness probe. The sidecar owns the externally
routable service identity and the cross-cutting concerns at the pod boundary.
The first deployment uses `networknt/openapi-petstore:2.3.4` and
`networknt/http-sidecar:2.2.1`.
