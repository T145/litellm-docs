---
title: LiteLLM as requesting app
sidebar_label: LiteLLM as requesting app
---

# LiteLLM as requesting app

In this setup LiteLLM reaches an MCP server whose authorization server is separate from the identity provider that signs your users in. LiteLLM takes the user's identity assertion, trades it at the IdP for an ID-JAG, trades the ID-JAG at the MCP server's authorization server for an access token, and calls the MCP server with it. An agent that holds only a LiteLLM virtual key reaches the MCP server as the user the key belongs to, with no consent screen and no per-user secret in LiteLLM.

Okta calls this registration an AI agent with a resource connection to the MCP server's resource app. Set `auth_type: oauth2_id_jag` on the MCP server to enable it.

:::info
This feature needs an enterprise license and a database. The database holds the identity assertion captured at SSO login, which is what lets a virtual key alone drive the exchange.
:::

## How it works

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Agent as Agent (virtual key)
    participant LiteLLM
    participant IdP as IdP org authorization server
    participant RAS as Resource authorization server
    participant MCP as MCP server

    User->>LiteLLM: Sign in through LiteLLM SSO (OIDC or SAML)
    IdP-->>LiteLLM: id_token + refresh token, or SAML assertion
    LiteLLM->>LiteLLM: Store the assertion for this user (encrypted)
    Agent->>LiteLLM: tools/list with x-litellm-api-key
    LiteLLM->>LiteLLM: Resolve user from key, load stored assertion
    Note over LiteLLM,IdP: Leg 1: RFC 8693 token exchange
    LiteLLM->>IdP: token_exchange_endpoint: subject_token=assertion, requested_token_type=id-jag, audience, scope
    IdP-->>LiteLLM: ID-JAG (aud = resource authorization server)
    Note over LiteLLM,RAS: Leg 2: RFC 7523 jwt-bearer grant
    LiteLLM->>RAS: id_jag_resource_token_endpoint: assertion=ID-JAG
    RAS-->>LiteLLM: access_token, expires_in
    LiteLLM->>LiteLLM: Cache access token per user and server
    LiteLLM->>MCP: tools/list with Authorization: Bearer access_token
    MCP-->>LiteLLM: tools
    LiteLLM-->>Agent: tools
```

The access token is cached by subject token and MCP server, so repeated calls from the same user skip both legs until it expires. Two users never share a token.

## Configure the MCP server

```yaml title="config.yaml" showLineNumbers
mcp_servers:
  internal_tools:
    url: "https://mcp.example.com/mcp"
    transport: "http"
    auth_type: oauth2_id_jag

    # Leg 1: IdP org authorization server token endpoint (RFC 8693 token exchange -> ID-JAG)
    token_exchange_endpoint: "https://your-org.okta.com/oauth2/v1/token"

    # Leg 2: resource authorization server token endpoint (RFC 7523 jwt-bearer -> access token)
    id_jag_resource_token_endpoint: "https://as.mcp.example.com/oauth2/v1/token"

    # LiteLLM's registration at the IdP (the AI agent)
    client_id: "<agent-client-id>"
    client_private_key: os.environ/AGENT_PRIVATE_KEY_PEM
    client_private_key_id: "<jwks-kid>"
    client_assertion_signing_alg: "RS256"

    # LiteLLM's registration at the resource authorization server (defaults to client_id and the same client auth)
    id_jag_resource_client_id: "<client-id-at-resource-as>"
    id_jag_resource_client_secret: os.environ/RESOURCE_AS_CLIENT_SECRET

    # Resource authorization server issuer; sent as the leg 1 audience and is the ID-JAG aud
    audience: "https://as.mcp.example.com"

    # Optional RFC 8707 resource indicator for leg 1
    id_jag_resource: "https://mcp.example.com/"

    # Scopes requested for the access token
    scopes:
      - "mcp.tools.read"
      - "mcp.tools.execute"
```

| Field | Required | Description |
|-------|----------|-------------|
| `auth_type` | Yes | Must be `oauth2_id_jag`. |
| `token_exchange_endpoint` | Yes | IdP org authorization server token endpoint for leg 1. For Okta this is `https://<your-org>.okta.com/oauth2/v1/token`, never a custom authorization server. |
| `id_jag_resource_token_endpoint` | Yes | Resource authorization server token endpoint for leg 2. |
| `client_id` | Yes | LiteLLM's client ID at the IdP, the AI agent's client ID in Okta. |
| `client_private_key` | One of the two | PEM private key LiteLLM signs the RFC 7523 `client_assertion` with. Okta's AI agent registration requires `private_key_jwt`. |
| `client_secret` | One of the two | `client_secret_post` client authentication, used when `client_private_key` is not set. |
| `client_private_key_id` | Optional | `kid` placed in the `client_assertion` header. Must match the public key registered on the agent. |
| `client_assertion_signing_alg` | Optional | Signing algorithm for the `client_assertion`. Defaults to `RS256`. |
| `id_jag_resource_client_id` | Optional | LiteLLM's client ID at the resource authorization server, when it differs from `client_id`. |
| `id_jag_resource_client_secret` | Optional | Secret for `id_jag_resource_client_id`. When both are unset, leg 2 authenticates with `client_id` and the same client auth as leg 1. |
| `audience` | Recommended | The resource authorization server's issuer identifier. Sent as the leg 1 `audience` and becomes the ID-JAG `aud`. Must equal the Issuer URL configured on the resource app in the IdP. |
| `id_jag_resource` | Optional | RFC 8707 `resource` sent on leg 1. |
| `scopes` | Optional | Scopes requested on leg 1, joined into `scope`. The IdP only grants scopes the resource connection allows. |
| `subject_token_type` | Optional | Declared type of the leg 1 subject token when it comes from the request's `Authorization` header or an OIDC login. Defaults to `urn:ietf:params:oauth:token-type:id_token`. A stored SAML assertion always declares itself as `urn:ietf:params:oauth:token-type:saml2` regardless of this value. |

A server missing either endpoint, `client_id`, or both client auth secrets never reaches the exchange, so check all of them are set when calls to the server fail without any leg being sent.

## Where the subject token comes from

Leg 1 needs an identity assertion for the user. LiteLLM takes it from one of two places. The user is always the one the virtual key resolves to, and no request field can pick another identity.

If the request carries a bearer in `Authorization`, that token is the subject. Keep the LiteLLM key in `x-litellm-api-key` so `Authorization` is free for the user's `id_token`.

```bash title="Caller supplies the id_token" showLineNumbers
curl -X POST "https://litellm.example.com/internal_tools/mcp" \
  -H "Content-Type: application/json" \
  -H "x-litellm-api-key: Bearer <litellm-api-key>" \
  -H "Authorization: Bearer <user-id-token>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

Otherwise LiteLLM uses the identity assertion it captured when the user last signed in through LiteLLM SSO. This is what lets an agent holding only a virtual key reach the MCP server.

```bash title="Virtual key only" showLineNumbers
curl -X POST "https://litellm.example.com/internal_tools/mcp" \
  -H "Content-Type: application/json" \
  -H "x-litellm-api-key: Bearer <litellm-api-key>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

The assertion is captured at UI and CLI sign-in, encrypted, and written to the database so a login on one pod is visible to every pod. Reads are cached per process for `MCP_SSO_ASSERTION_CACHE_TTL_SECONDS` (default 60). LiteLLM only keeps assertions while at least one `oauth2_id_jag` server is registered. Every SSO provider captures one.

| Login | Env | What is stored | Subject token type on leg 1 | Renewal |
|-------|-----|----------------|-----------------------------|---------|
| Generic OIDC | `GENERIC_CLIENT_ID` | The `id_token` and refresh token from the token response. | `subject_token_type` (default `id_token`) | Refresh token redeemed at `GENERIC_TOKEN_ENDPOINT`. Needs `offline_access` in `GENERIC_SCOPE`. |
| Microsoft Entra | `MICROSOFT_CLIENT_ID` | The `id_token` and refresh token from the token response. | `subject_token_type` (default `id_token`) | Refresh token redeemed at `MICROSOFT_TOKEN_ENDPOINT` or `https://login.microsoftonline.com/<MICROSOFT_TENANT>/oauth2/v2.0/token`. Entra only issues a refresh token when the login asked for `offline_access`. |
| Google | `GOOGLE_CLIENT_ID` | The `id_token` and refresh token from the token response. | `subject_token_type` (default `id_token`) | Refresh token redeemed at `https://oauth2.googleapis.com/token`. Google only issues a refresh token when the authorization request carried `access_type=offline`. |
| SAML | `SAML_IDP_METADATA_URL` or `SAML_IDP_METADATA_XML` | The signed `<saml:Assertion>` element, base64url encoded, valid until its `NotOnOrAfter`. | `urn:ietf:params:oauth:token-type:saml2`, always | None. SAML has no refresh token. The user signs in again. |

The stored `id_token` expires on the IdP's schedule, about an hour on Entra and Google. When a refresh token was captured, LiteLLM renews the assertion shortly before `exp`, redeeming the refresh token against the same client the login used and authenticating the way that login did (HTTP Basic for Google, Microsoft and the non-PKCE generic path, body credentials on the PKCE generic path when `GENERIC_INCLUDE_CLIENT_ID` is set). The renewal is single-flighted per user, since an IdP that rotates refresh tokens treats two concurrent redemptions as replay. The Microsoft and Google logins request their providers' default scopes, so unless you add the scope above the assertion lasts one `id_token` lifetime and the next sign-in replaces it. When no refresh token is available the proxy log names the missing scope.

### SAML logins

With SAML SSO there is no `id_token`. LiteLLM keeps the one signed `<saml:Assertion>` the IdP returned and sends it on leg 1 as `subject_token_type=urn:ietf:params:oauth:token-type:saml2`, which RFC 8693 defines and the IETF draft allows. Nothing changes on the MCP server config.

```text title="Leg 1 with a SAML subject"
POST /oauth2/v1/token
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
subject_token=<base64url saml:Assertion>
subject_token_type=urn:ietf:params:oauth:token-type:saml2
audience=<resource authorization server issuer>
scope=<scopes>
client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
client_assertion=<private_key_jwt>
```

Two things differ from an OIDC login. The assertion is only usable until its `NotOnOrAfter`, which most IdPs set to a few minutes, and SAML has no refresh token, so once it passes the call returns `412` and the user has to sign in through LiteLLM SSO again.

Okta documents a two step variant for SAML requesting apps in its [SAML token exchange guide](https://developer.okta.com/docs/guides/xaa-request-token-ex/saml2/main/): first exchange the SAML assertion for a refresh token (`requested_token_type=urn:ietf:params:oauth:token-type:refresh_token`, `scope=openid offline_access`), then exchange that refresh token for the ID-JAG, which lets the app mint new ID-JAGs for the life of the refresh token. For that exchange Okta maps the SAML `Audience` to the AI agent's client ID, so the SAML app's audience must be the agent's client ID. LiteLLM sends the assertion straight to the ID-JAG exchange, so check what your org authorization server accepts for `subject_token_type=saml2`.

## Okta Admin Console: register LiteLLM as the requesting app

These steps are Okta specific. Any identity provider that implements the draft needs the same facts: a client registration for LiteLLM with its public key, the resource app's authorization server issuer, and a policy that lets LiteLLM obtain an ID-JAG for it. Okta's own walkthroughs are the [OIDC requesting app guide](https://developer.okta.com/blog/2026/08/21/xaa-oidc-requesting), the [SAML requesting app guide](https://developer.okta.com/blog/2026/07/17/xaa-saml-requester) and the [token exchange reference](https://developer.okta.com/docs/guides/xaa-request-token-ex/openidconnect/main/).

1. Make sure your users sign in to LiteLLM through an Okta app integration. That is the [Admin UI SSO](../admin_ui_sso.md) OIDC app or the [SAML](../saml_sso.md) app you already have. For OIDC, enable the **Refresh Token** grant and ask for `offline_access` (`GENERIC_SCOPE: "openid profile email offline_access"`) so the stored assertion can be renewed.
2. Under **Directory > AI Agents**, register LiteLLM as an AI agent. Under **User access and authentication** pick **Select an existing app** and choose the app integration from step 1. This is the app whose `id_token` or SAML assertion LiteLLM will present, and Okta only mints an ID-JAG for a user assigned to it.
3. On the agent's **Client registration** tab, set client authentication to `private_key_jwt` and upload LiteLLM's public key as a JWKS, keeping the private key for `client_private_key`. Note the client ID (`client_id`) and the key's `kid` (`client_private_key_id`).
4. Register the MCP server's resource app, or confirm it exists. Its **Resource Server** tab holds the **Issuer URL** of the MCP server's authorization server. That URL is `audience` in LiteLLM's config. The resource app's token endpoint is `id_jag_resource_token_endpoint`.
5. On the AI agent, add a **Resource connection** to that resource app. In **AI agent's client ID registered in this app**, enter the client ID the MCP server's authorization server issued to LiteLLM (`id_jag_resource_client_id`, or `client_id` when they are the same). Pick the scopes LiteLLM may request, or allow any scope. Only scopes allowed here are granted.
6. Activate the AI agent if it is still STAGED.

Leg 1 goes to the org authorization server, `https://<your-org>.okta.com/oauth2/v1/token`. A custom authorization server does not support `requested_token_type=id-jag`.

## The two legs on the wire

Leg 1 is sent once per uncached user and MCP server pair.

```http
POST /oauth2/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&subject_token=<user-id-token>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&audience=https://as.mcp.example.com
&resource=https://mcp.example.com/
&scope=mcp.tools.read mcp.tools.execute
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<signed-jwt>
```

```json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "access_token": "<id-jag-jwt>",
  "token_type": "N_A",
  "expires_in": 300
}
```

Leg 2 presents the ID-JAG to the resource authorization server.

```http
POST /oauth2/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=<id-jag-jwt>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<signed-jwt>
```

```json
{
  "access_token": "access-token-for-mcp-server",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

LiteLLM then calls the MCP server with `Authorization: Bearer access-token-for-mcp-server`, or the header the server's `upstream_token_header` names. The cache TTL is the leg 2 `expires_in` minus LiteLLM's OAuth expiry buffer, or the default OAuth token cache TTL when `expires_in` is missing.

## Credential errors at connect time

On a single-server route such as `/internal_tools/mcp` or `/mcp/internal_tools`, LiteLLM resolves the credential before it opens the MCP session, so a failure comes back as an HTTP status instead of a session with no tools. These are decided before any authorization server is called.

| Status | Meaning | Fix |
|--------|---------|-----|
| `412 Precondition Failed` | No identity assertion is stored for this user, or the stored one has expired and could not be renewed. The body says which. | The user signs in through LiteLLM SSO. |
| `503 Service Unavailable` | The assertion store (the LiteLLM database) is unreachable. | Check database connectivity. |

```bash
$ curl -s -i -X POST https://litellm.example.com/mcp/internal_tools \
    -H "x-litellm-api-key: Bearer <litellm-api-key>" \
    -H 'Content-Type: application/json' \
    -H 'Accept: application/json, text/event-stream' \
    -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
HTTP/1.1 412 Precondition Failed
content-type: application/json

{"detail":"precondition required: ID-JAG requires an IdP identity assertion for this user and none is stored. Sign in through LiteLLM SSO so the gateway captures one."}
```

The 412 is a plain status with a JSON body and no `WWW-Authenticate` header, because only a LiteLLM SSO login fixes it. On the aggregate `/mcp` route one server's failure must not fail the whole connection, so `tools/list` succeeds, the failing server contributes no tools, and the reason lands in `_meta`.

```json
{"_meta":{"litellm.ai/server_outcomes":{"internal_tools":{"status":"internal","http_status":412}}},"tools":[]}
```

Call the single-server route to see the status and message directly.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| MCP server receives the LiteLLM key | Move the LiteLLM key to `x-litellm-api-key` and keep `Authorization` for the user's `id_token`. |
| `412` on connect, body says none is stored | The user has not signed in through LiteLLM SSO since the `oauth2_id_jag` server was registered. Have them sign in. |
| `412` on connect, body says it has expired | OIDC: no refresh token was captured. Add `offline_access` (Entra, generic OIDC) or `access_type=offline` (Google) to the login, then sign in again. SAML: the assertion passed its `NotOnOrAfter`. Sign in again. |
| Startup log warns about an identity assertion capture gap | No SSO provider is configured, so no assertion is ever captured. Configure Google, Microsoft, generic OIDC or SAML SSO. |
| `503` on connect | The assertion store is unreachable. Check the database connection. |
| `tools/list` returns no tools and `_meta` shows `http_status: 412` | You are on the aggregate `/mcp` route. Call the single-server route for the message. |
| Leg 1 returns `401 invalid_client` | `client_id`, `client_private_key` and `client_private_key_id` do not match the key registered on the agent's **Client registration** tab. |
| Leg 1 returns `invalid_target` | `audience` (and `id_jag_resource`, if set) does not match the Issuer URL on the resource connection. Compare the strings exactly. |
| Leg 1 returns `unauthorized_client` | A requested scope is not allowed on the resource connection. Change `scopes` or the connection. |
| Leg 1 returns `requested_token_type invalid or not supported` | `token_exchange_endpoint` points at a custom authorization server. Use `https://<your-org>.okta.com/oauth2/v1/token`. |
| Leg 1 returns `invalid_grant` for a SAML subject | The assertion expired or the SAML app's audience is not the agent's client ID. Sign in again, and set the SAML audience to the agent's client ID. |
| Leg 2 rejects the assertion | `id_jag_resource_token_endpoint` is not the authorization server named by `audience`, the resource client credentials are wrong, or clocks disagree with the ID-JAG `exp`. |
| Authorization servers are called on every request | Confirm leg 2 returns `expires_in` and that the same user and MCP server are being reused. |
