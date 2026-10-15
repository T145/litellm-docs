---
title: LiteLLM as resource app
sidebar_label: LiteLLM as resource app
---

# LiteLLM as resource app

In this setup an agent that your identity provider already signed in trades an ID-JAG for a LiteLLM credential. The agent never opens a browser against LiteLLM and never holds a long-lived virtual key. Your IdP decides which agents may reach LiteLLM, and LiteLLM mints the credential for the user the ID-JAG names.

LiteLLM plays two of the draft's roles at once. Its gateway token endpoint `POST /token` is the resource authorization server, and its MCP gateway and API routes are the resource server the minted credential is used on. Okta calls this registration a resource app with Cross App Access enabled.

:::info
This feature needs an enterprise license, a database, and [JWT authentication](../token_auth.md). The grant is advertised in discovery only when all three are configured and at least one requesting app is allowlisted.
:::

## How it works

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Requesting app (agent)
    participant IdP as IdP org authorization server
    participant LiteLLM as LiteLLM (/token, /mcp)

    Note over Agent,IdP: The user already signed in to the agent through the IdP
    Agent->>IdP: Token exchange: subject_token=id_token, requested_token_type=id-jag, audience=https://litellm.example.com/mcp
    IdP->>IdP: Resource connection allows agent -> LiteLLM
    IdP-->>Agent: ID-JAG (typ oauth-id-jag+jwt, aud=https://litellm.example.com/mcp, client_id=agent)
    Agent->>LiteLLM: POST /token jwt-bearer, assertion=ID-JAG, client_id + client_secret
    LiteLLM->>IdP: Fetch JWKS for iss (cached)
    LiteLLM->>LiteLLM: Verify typ, iss, aud, exp, iat, jti, client_id, map sub to a user, settle team
    LiteLLM-->>Agent: access_token, expires_in, scope, user_id, team_id
    Agent->>LiteLLM: POST /mcp tools/list with Authorization: Bearer access_token
    LiteLLM-->>Agent: tools
    Note over Agent,LiteLLM: No refresh token. On expiry the agent gets a fresh ID-JAG from the IdP
```

## What your IdP needs from LiteLLM

The IdP mints the ID-JAG for a named resource authorization server and puts that server's issuer identifier in `aud`. LiteLLM accepts two values for `aud`, the MCP gateway issuer and the plain proxy origin. Use the MCP gateway issuer in the IdP console, since that is what LiteLLM publishes in discovery.

| Value | Where it comes from |
|-------|---------------------|
| Issuer identifier (the ID-JAG `aud`) | `https://litellm.example.com/mcp`. LiteLLM also accepts `https://litellm.example.com`. |
| Token endpoint | `https://litellm.example.com/token` |
| Authorization server metadata | `https://litellm.example.com/.well-known/oauth-authorization-server/mcp` |
| Native client discovery | `https://litellm.example.com/.well-known/litellm-cli-auth` |

The URLs are built from the request's base URL. When LiteLLM runs behind a load balancer or reverse proxy, set `PROXY_BASE_URL` to the public origin so `issuer` and the endpoints name the address your IdP and your agents use. An ID-JAG whose `aud` names a different host is rejected.

Check what LiteLLM advertises before you configure the IdP. The grant appears in `grant_types_supported` only once the configuration below is in place.

```bash
curl -s https://litellm.example.com/.well-known/oauth-authorization-server/mcp | jq '{issuer, token_endpoint, grant_types_supported, token_endpoint_auth_methods_supported}'
```

```json
{
  "issuer": "https://litellm.example.com/mcp",
  "token_endpoint": "https://litellm.example.com/token",
  "grant_types_supported": [
    "authorization_code",
    "refresh_token",
    "urn:ietf:params:oauth:grant-type:token-exchange",
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
  "token_endpoint_auth_methods_supported": ["none", "client_secret_post", "client_secret_basic"]
}
```

## Okta Admin Console: register LiteLLM as the resource app

These steps are Okta specific. Any identity provider that implements the draft needs the same three facts: LiteLLM's issuer identifier, the requesting app's client ID at LiteLLM, and the scopes the connection may request. Okta's own walkthrough is the [OIDC resource app guide](https://developer.okta.com/blog/2026/08/24/xaa-oidc-resource) and the [Configure Cross App Access](https://help.okta.com/oie/en-us/content/topics/apps/apps-cross-app-access.htm) help page.

1. Create an app integration that represents LiteLLM. Under **Applications > Applications > Create App Integration** pick OIDC and a Web Application. The sign-in redirect URI can be any placeholder, since this integration never performs an interactive login. It only holds the Cross App Access configuration and the user assignments.
2. On that app's **Resource Server** tab, enable Cross App Access and set **Issuer URL** to LiteLLM's issuer identifier, `https://litellm.example.com/mcp`. This value becomes the ID-JAG `aud` and cannot be edited later without deleting the connection. Leave **Audience/tenant ID** empty unless you run a multi-tenant Okta.
3. Assign the users who may reach LiteLLM to this app integration. Okta only mints an ID-JAG for an assigned user.
4. Register the agent as an AI agent under **Directory > AI Agents**, or open the one you already have. Under **User access and authentication**, link the OIDC app integration your users sign in to the agent through. The agent's **Client registration** tab shows the client ID and secret it uses for the token exchange.
5. On the agent, add a **Resource connection**. Pick the LiteLLM app integration as the application instance. In **AI agent's client ID registered in this app**, enter the `client_id` you will allowlist in LiteLLM below. The ID-JAG carries this value in its `client_id` claim and LiteLLM checks that the presenting app matches it. Pick the scopes the agent may request, or allow any scope.
6. Activate the AI agent if its status is still STAGED.

The ID-JAG is signed by the Okta org authorization server, so the JWKS LiteLLM needs is `https://<your-org>.okta.com/oauth2/v1/keys`, and the `iss` claim is `https://<your-org>.okta.com`. Do not point LiteLLM at a custom authorization server such as `/oauth2/default` for this.

## Configure LiteLLM

The ID-JAG is verified with the issuer configuration [JWT authentication](../token_auth.md) already uses, so the IdP goes under `litellm_jwtauth.issuers`. The requesting apps that may redeem an ID-JAG go under `id_jag_requesting_apps`. An app with a `client_secret` is a confidential client and must send it. An app without one is a public client identified by its `client_id` alone. Unset or empty `id_jag_requesting_apps` turns the grant off.

```yaml title="config.yaml" showLineNumbers
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL
  enable_jwt_auth: true
  litellm_jwtauth:
    user_id_jwt_field: sub
    issuers:
      - issuer: https://your-org.okta.com
        jwks_url: https://your-org.okta.com/oauth2/v1/keys
        audience: api://litellm
        user_id_jwt_field: sub
        user_email_jwt_field: email
    id_jag_requesting_apps:
      - client_id: 0oa1agentapp
        client_secret: os.environ/AGENT_APP_CLIENT_SECRET
      - client_id: 0oa1publicagent
```

| Key | What it does for ID-JAG |
|-----|-------------------------|
| `enable_jwt_auth` | Required. Turns on the JWT handler that verifies the assertion. |
| `issuers[].issuer` | Must equal the ID-JAG `iss`. An assertion whose `iss` matches no configured issuer is refused. When `issuers` is not set, LiteLLM falls back to `JWT_ISSUER` and `JWT_PUBLIC_KEY_URL`. |
| `issuers[].jwks_url` | The keys the assertion's signature is checked against, matched by `kid`. Optional when the issuer serves `/.well-known/openid-configuration`. JWKS reads are cached and a stale copy is used when the IdP is unreachable. |
| `issuers[].audience` | Used for ordinary bearer tokens from this issuer. The ID-JAG is checked against LiteLLM's own issuer identifier instead, so this value does not have to match the ID-JAG. |
| `issuers[].user_id_jwt_field`, `issuers[].user_email_jwt_field` | Map the assertion's claims to an existing LiteLLM user. The user must already exist. Nothing is created from an ID-JAG. Okta puts its user ID in `sub` and the address in `email`, so most deployments map `user_email_jwt_field: email` or set the user's `user_id` to the Okta `sub`. The team claim mapping of the issuer, if any, is read too. |
| `id_jag_requesting_apps[].client_id` | The requesting app's client ID exactly as the IdP puts it in the ID-JAG `client_id` claim and as the app sends it in the `client_id` form field or HTTP Basic user name. |
| `id_jag_requesting_apps[].client_secret` | Optional. Makes the app a confidential client that must authenticate with this secret. Omit it for a public client. |

The allowlist is config, so adding an app is a config change and a restart.

## Client: the two legs

Leg 1 is between the agent and the IdP. The agent trades the user's `id_token` (or the refresh token from the same login) for an ID-JAG whose `audience` is LiteLLM's issuer identifier. The example uses Okta's org authorization server and `client_secret_post`; Okta also accepts a `private_key_jwt` client assertion.

```bash title="Leg 1: ID-JAG from the IdP" showLineNumbers
curl -s -X POST https://your-org.okta.com/oauth2/v1/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d requested_token_type=urn:ietf:params:oauth:token-type:id-jag \
  -d subject_token_type=urn:ietf:params:oauth:token-type:id_token \
  --data-urlencode "subject_token=$ID_TOKEN" \
  -d audience=https://litellm.example.com/mcp \
  -d scope=mcp \
  -d client_id=0oa1agentapp \
  -d client_secret=$AGENT_APP_CLIENT_SECRET
```

```json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "access_token": "eyJ0eXAiOiJvYXV0aC1pZC1qYWcrand0...",
  "token_type": "N_A",
  "expires_in": 300
}
```

The `access_token` value is the ID-JAG. Decoded, its header and payload look like this.

```json title="ID-JAG header"
{ "typ": "oauth-id-jag+jwt", "alg": "RS256", "kid": "..." }
```

```json title="ID-JAG payload"
{
  "iss": "https://your-org.okta.com",
  "sub": "00u1a2b3c4D5e6F7g8h9",
  "email": "user@example.com",
  "aud": "https://litellm.example.com/mcp",
  "client_id": "0oa1agentapp",
  "scope": "mcp",
  "jti": "id-jag-7f3c9a21b8",
  "iat": 1760000000,
  "exp": 1760000300
}
```

Leg 2 is between the agent and LiteLLM. The agent posts the ID-JAG to `POST /token` with the `jwt-bearer` grant. A confidential app sends its secret either as HTTP Basic or as the `client_secret` form field. A public app sends only `client_id`.

```bash title="Leg 2: LiteLLM credential from the ID-JAG" showLineNumbers
curl -s -X POST https://litellm.example.com/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer \
  -d client_id=0oa1agentapp \
  -d client_secret=$AGENT_APP_CLIENT_SECRET \
  --data-urlencode "assertion=$ID_JAG"
```

```json
{
  "access_token": "LneZuxEFqvemEwK6lRzgg6BN...",
  "token_type": "Bearer",
  "expires_in": 86400,
  "scope": "mcp",
  "user_id": "user@example.com",
  "team_id": "125eca86-fbde-44de-8cb9-cf1432933fcd"
}
```

The same request with HTTP Basic and no `client_id` field also works.

```bash
curl -s -u "0oa1agentapp:$AGENT_APP_CLIENT_SECRET" -X POST https://litellm.example.com/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer \
  --data-urlencode "assertion=$ID_JAG"
```

`scope` echoes the assertion's `scope`; it is recorded, not enforced, and the credential carries the user's normal permissions. There is no refresh token, as [RFC 7521 section 4.1](https://www.rfc-editor.org/rfc/rfc7521#section-4.1) recommends. When the credential expires the agent asks its IdP for a fresh ID-JAG, which keeps the user's standing at the IdP in the loop. Each assertion is redeemed once.

The credential is minted for one team. The `x-litellm-team-id` header (a team ID or alias) wins and must name one of the user's teams, then the team the assertion names through the issuer's mapped team claim, then the user's only team. A user on several teams who names none gets `400 invalid_request`.

```bash
curl -s -X POST https://litellm.example.com/token \
  -H "x-litellm-team-id: engineering" \
  -d grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer \
  -d client_id=0oa1agentapp \
  -d client_secret=$AGENT_APP_CLIENT_SECRET \
  --data-urlencode "assertion=$ID_JAG"
```

## Use the credential

The minted credential is the same one the browser sign-in and token exchange flows mint. On the MCP gateway it opens the aggregate `/mcp` route and the per-server routes.

```bash title="tools/list with the minted credential" showLineNumbers
curl -s -X POST https://litellm.example.com/mcp \
  -H "Authorization: Bearer LneZuxEFqvemEwK6lRzgg6BN..." \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

```text
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"tools":[{"name":"internal_tools-search", ...}]}}
```

Spend is attributed to the `user_id` and `team_id` from the token response. The same bearer is accepted on the proxy's API routes such as `/v1/chat/completions`, exactly like the credential from [browser sign-in with PKCE](../cli_sso.md#browser-sign-in-with-pkce).

The ID-JAG itself is never a bearer token. Sending it in `Authorization` on `/mcp` returns `401` with `{"error":"authentication_required","message":"Authenticate with the gateway to use the MCP endpoint."}`, and the proxy log says `an ID-JAG is an authorization grant, not a bearer token; redeem it at the token endpoint`.

## What LiteLLM validates

| Check | Rule | On failure |
|-------|------|------------|
| Requesting app | `client_id` (form field or HTTP Basic) is in `id_jag_requesting_apps`, and a confidential app's secret matches. | `401 invalid_client` |
| `typ` header | Must be `oauth-id-jag+jwt`. | `400 invalid_grant`, `assertion typ must be oauth-id-jag+jwt` |
| `iss` | Must equal a configured issuer (or `JWT_ISSUER`). The issuer is resolved first, then the signature is checked against that issuer's keys, so an unknown issuer can never be trusted. | `400 invalid_grant`, `assertion was rejected` |
| Signature | Verified with the issuer's JWKS, matched by `kid`. | `400 invalid_grant`, `assertion was rejected`. A JWKS outage with no cached copy is a `503`. |
| `aud` | Must contain LiteLLM's issuer identifier, `https://litellm.example.com/mcp` or `https://litellm.example.com`. | `400 invalid_grant`, `assertion was rejected` |
| `exp`, `iat` | Both required. `exp` must be in the future, within the JWT handler's leeway. | `400 invalid_grant`, `assertion was rejected` |
| `jti` | Required and single use until `exp`. The replay marker is keyed by `iss` and `jti`. | `400 invalid_grant`, `the assertion has already been redeemed` |
| `client_id` claim | Must equal the requesting app that presented the assertion. | `400 invalid_grant`, `assertion was rejected` |
| `sub` | Mapped through `user_id_jwt_field` or `user_email_jwt_field` to a user that already exists. | `400 invalid_grant`, `assertion was rejected` |
| `cnf` | Rejected. LiteLLM checks no proof of possession, so a sender-constrained ID-JAG is refused rather than accepted unconstrained. | `400 invalid_grant`, `assertion was rejected` |

Every assertion fault other than `typ` and replay returns the generic description `assertion was rejected`, because PyJWT's wording can quote the token's claims. The exact reason is in the proxy log, prefixed `jwt-bearer grant refused an assertion:`.

## Troubleshooting

| Response | Cause | Fix |
|----------|-------|-----|
| `400 unsupported_grant_type`, `this gateway is not configured to redeem identity assertions` | JWT auth, the database, the license or `id_jag_requesting_apps` is missing. | Complete the configuration above and restart. Discovery lists `urn:ietf:params:oauth:grant-type:jwt-bearer` once it is right. |
| `400 invalid_request`, `client_id is required` | Neither the `client_id` form field nor HTTP Basic was sent. | Send one of the two. |
| `400 invalid_request`, `assertion is required` | The `assertion` form field is missing. | Send the ID-JAG as `assertion`. |
| `401 invalid_client`, `client_id is not an allowlisted requesting app` | The app is not in `id_jag_requesting_apps`. | Add it, matching the `client_id` the IdP writes into the ID-JAG. |
| `401 invalid_client`, `client authentication failed` | Wrong or missing secret for a confidential app. | Fix the secret. A public app must not have a `client_secret` configured. |
| `400 invalid_grant`, `assertion typ must be oauth-id-jag+jwt` | The token is an ordinary JWT, for example an `id_token` or access token. | Run leg 1 and send the token exchange result, not the login token. |
| `400 invalid_grant`, `the assertion has already been redeemed` | Same `jti` presented twice. | Each ID-JAG is one credential. Ask the IdP for a new one. |
| `400 invalid_grant`, `assertion was rejected`, log says `Audience doesn't match` | The IdP's Issuer URL for the resource app differs from LiteLLM's issuer. | Make the Resource Server **Issuer URL** exactly `https://litellm.example.com/mcp`, and set `PROXY_BASE_URL` if LiteLLM sits behind a proxy. Watch for trailing slashes and `http` versus `https`. |
| `400 invalid_grant`, `assertion was rejected`, log says `iss is not an issuer this gateway trusts` | The ID-JAG `iss` matches no `issuers[].issuer`. | Add the IdP org authorization server as an issuer. Okta's is `https://<your-org>.okta.com`, with no `/oauth2/default`. |
| `400 invalid_grant`, `assertion was rejected`, log says `no signing key for the assertion` | `jwks_url` points at the wrong key set. | Use the org authorization server keys, `https://<your-org>.okta.com/oauth2/v1/keys`. |
| `400 invalid_grant`, `assertion was rejected`, log says `the assertion's client_id is not the requesting app that presented it` | The IdP minted the ID-JAG for another client ID. | Make the resource connection's **AI agent's client ID registered in this app** equal the `client_id` the agent sends to LiteLLM. |
| `400 invalid_grant`, `assertion was rejected`, log says `subject lookup failed: User doesn't exist in db` | No LiteLLM user matches the mapped claim. | Create the user, or map `user_email_jwt_field: email` so the Okta `email` claim resolves. |
| `400 invalid_request`, `this user belongs to several teams; send the x-litellm-team-id header naming the team for this credential` | The user is on several teams and none was named. | Send `x-litellm-team-id` with a team ID or alias. |
| `400 invalid_request`, `x-litellm-team-id does not name one of this user's teams` | The header names a team the user is not on. | Use one of the user's teams. |
| `503 temporarily_unavailable`, `the gateway could not read this user's teams; retry` | Database read failed. The assertion was not consumed. | Retry. |
| `401` on `/mcp` with the ID-JAG as bearer | The ID-JAG is a grant, not a bearer token. | Redeem it at `/token` and use the `access_token` from the response. |
| Okta leg 1 returns `invalid_target` or `unauthorized_client` | The `audience` or scopes do not match the resource connection. | In **Directory > AI Agents > Resource connections**, check the Issuer URL and the allowed scopes. |

The `jti` is consumed before the credential is minted, so a database failure while minting burns the assertion. The agent then asks its IdP for a fresh one. Team and membership are settled before the `jti` is consumed, so a fixable request error never burns it.
