# Client Credentials to Authorization Code Token Exchange

## Introduction

This document outlines the design for exchanging an external Client Credentials (CC) token (e.g., from Okta) for an internal Authorization Code (AC) style token (containing user/function identity) within the `light-4j` ecosystem.

## Use Case

External partners or applications integrate with `light-portal` using their own Identity Provider (e.g., Okta) via the Client Credentials flow. 
- **External Token**: A JWT issued by Okta representing the *External App*.
- **Internal Requirement**: `light-portal` services require a JWT containing internal `userId`, `roles`, and `custom_claims` to perform business logic (Event Sourcing/CQRS).
- **Goal**: Bridge the external identity to the internal identity transparently at the ingress/handler layer.

## Architecture

The solution uses **RFC 8693 OAuth 2.0 Token Exchange**.

### High-Level Flow

1.  **External Request**: The external app calls `light-portal` with `Authorization: Bearer <Okta-CC-Token>`.
2.  **Interception**: A `TokenExchangeHandler` in `light-portal` intercepts the request.
3.  **Validation**: The handler validates the Okta token (signature, expiration) using `JwtVerifier` and Okta's JWK.
4.  **Exchange**:
    *   The handler requests a token exchange from the internal `light-oauth2` service.
    *   **Grant Type**: `urn:ietf:params:oauth:grant-type:token-exchange`
    *   **Subject Token**: The verified Okta token.
    *   **Subject Token Type**: `urn:ietf:params:oauth:token-type:jwt`
5.  **Minting**:
    *   `light-oauth2` validates the request.
    *   It identifies the "Shadow Project" mapped to the external client ID.
    *   It issues a new internal JWT containing the mapped internal `userId` and roles.
6.  **Injection**: The handler replaces the `Authorization` header with the new internal token.
7.  **Processing**: Downstream services processing the request see a standard internal user token.

## Configuration

The `TokenExchangeHandler` configures the exchange parameters via `client.yml` or `values.yml` using `OAuthTokenExchangeConfig`.

```yaml
client:
  tokenExUri: "https://light-oauth2/oauth2/token"
  tokenExClientId: "${portal.client_id}"
  tokenExClientSecret: "${portal.client_secret}"
  tokenExScope: 
    - "portal.w"
    - "portal.r"
```

## Client Identity Mapping

A critical component is mapping the **External Client ID** (from Okta) to an **Internal Function ID**.

### Recommended Approach: Database (Shadow Client)

Maintain a "Shadow Client" record in the `light-oauth2` database for each external partner.

1.  **Registration**: Create a client in `light-oauth2` where `client_id` matches the Okta Client ID.
2.  **Mapping**: Populate the `custom_claims` column for this client with the internal identity:
    ```json
    { "functionId": "acme-integration-user", "internalRole": "partner" }
    ```
3.  **Execution**: When `light-oauth2` performs the exchange, it looks up this client record and automatically injects these custom claims into the new token.

**Pros**:
*   Scalable: Manage thousands of partners via Portal UI/Database without restarts.
*   Standard: Uses existing `light-oauth2` features.
*   Decoupled: The handler code remains generic.

### Alternative Approach: Configuration

Map IDs locally in the handler configuration.

```yaml
tokenExchangeMapping:
  "okta-client-id-1": "internal-id-1"
```

**Pros**: Simple for MVP.
**Cons**: Requires restart to add partners; hard to manage at scale.

## Security Considerations

1.  **Trust**: The internal `light-oauth2` must trust the Portal Client to perform exchanges.
2.  **Validation**: The `TokenExchangeHandler` *must* validate the external token before attempting exchange to prevent garbage requests from reaching the OAuth server.
3.  **Scope**: The exchanged token should have scopes limited to what the external partner is allowed to do.
