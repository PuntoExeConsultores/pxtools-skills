# @OAuthService — OAuth 2.0 + OpenID Connect Authorization Server

## What it is

A PXTools module implementing an **OAuth 2.0 Authorization Server** (RFC 6749) with extensions:
- **PKCE** (RFC 7636) — code_challenge S256 + plain
- **Token Introspection** (RFC 7662)
- **Token Revocation** (RFC 7009)
- **OpenID Connect Core 1.0** — HS256 JWT id_token, /userinfo, /.well-known/openid-configuration

It provides HTTP REST endpoints so client applications (web, mobile, M2M) can obtain access tokens authorized by users of the host system.

## Location in the KB

```
@PXTools/@OAuthService/
├── #Domains/                       ← AuthorizationGrantType, AuthorizationStatus, ClientStatus, OAuthServiceClientId
├── APIs/Basic/                     ← the module's core API (public API + transactions + SDTs)
│   ├── SDT/                        ← SDTs of the public API and the HTTP responses
│   ├── Transactions               (OAuthServiceClient, OAuthServiceAuthorization, OAuthServiceToken, OAuthServiceTokenPolicy)
│   ├── Procedures                 (CreateNewAuthorization, CreateTokens, RetDataFromToken, RevokeToken, ...)
│   ├── JWT/PKCE helpers           (GenerateIdToken, ToBase64Url, NormalizeRedirectUri, DateTimeToUnixTimestamp)
│   └── TskOAuthServicePurgeExpiredTokens.Procedure  ← a TaskManager task
├── APIs/WS/                        ← HTTP endpoints (WebService=True). GX names are PascalCase; GeneXus lowercases the path in Java/C# automatically.
│   ├── Token.Procedure             ← POST /pxtools.oauthservice/token
│   ├── Introspect.Procedure        ← POST /pxtools.oauthservice/introspect
│   ├── Revoke.Procedure            ← POST /pxtools.oauthservice/revoke
│   ├── UserInfo.Procedure          ← GET/POST /pxtools.oauthservice/userinfo (Bearer)
│   └── OpenIDConfiguration.Procedure ← GET /pxtools.oauthservice/openidconfiguration (map it to /.well-known/openid-configuration in the web server)
└── Personalized/                   ← hooks the module's host IMPLEMENTS
    ├── Authorization.WebPanel      ← the /authorize UI (login + consent)
    ├── CheckUserLoginData.Procedure ← credential validation against the host's system
    ├── ChkOAuthRateLimit.Procedure ← rate limiting (stub: allows everything)
    ├── RetOAuthServiceIssuer.Procedure ← the AS's public URL (issuer)
    ├── RetUserInfo.Procedure       ← user data for OIDC UserInfo
    ├── SendTokenToThirdParty.Procedure ← optional post-token callback (silent flow)
    └── RetDynamicCallReferencesOAuthService.DataProvider ← registration of Tasks/Cleaners
```

## Tables the module creates

| Table | Purpose |
|---|---|
| `OAuthServiceClient` | Registered client applications (client_id + client_secret + redirect_uri + status) |
| `OAuthServiceAuthorization` | Issued authorizations (code + AccountReference + scope + PKCE challenge + expiry) |
| `OAuthServiceToken` | Issued access and refresh tokens (token + status + expiry + client + authorization) |
| `OAuthServiceTokenPolicy` | Per-client expiry policies (access_token TTL, refresh_token TTL) |

**The host's user is linked to the module through** `OAuthServiceAuthorizationAccountReference` (a generic Character 255). There is NO direct FK to the host's tables.

## HTTP endpoints

| Endpoint | GX Procedure | Spec | Notes |
|---|---|---|---|
| **token** | `APIs/WS/Token.Procedure` | RFC 6749 §3.2 | **main + `CallProtocol='HTTP'`**. grant_type: authorization_code / refresh_token / client_credentials |
| **introspect** | `APIs/WS/Introspect.Procedure` | RFC 7662 | REST. Returns `{"active":true,...}` or `{"active":false}` |
| **revoke** | `APIs/WS/Revoke.Procedure` | RFC 7009 | REST. Accepts either `token` or `code` |
| **userinfo** | `APIs/WS/UserInfo.Procedure` | OIDC Core §5.3 | REST. Bearer token; it validates scope=openid |
| **openidconfiguration** | `APIs/WS/OpenIDConfiguration.Procedure` | OIDC Discovery §3 | REST. The host maps /.well-known/openid-configuration through a URL rewrite in the web server |
| `GET /authorize` (UI) | `Personalized/Authorization.WebPanel` | RFC 6749 §3.1 | A WebPanel, not REST. API-first NOT implemented. |

**Real URLs** (verified at runtime, Java generator):

| Exposure | URL |
|---|---|
| main + `CallProtocol='HTTP'` | `<base>/<namespace>.pxtools.oauthservice.a<object in lowercase>` — the Java generator prepends the `a` to main procedures |
| Expose as Web Service (REST) | `<base>/rest/PXTools/OAuthService/<Object>` — it **preserves the module's and the object's casing** |

> It is not true that GeneXus lowercases REST paths: it publishes them with the module's casing. That is why the REST URL is **not** OAuth-compliant in shape, even though the body's contract is.

**HTTP responses**: JSON with snake_case field names. In `Token` this is achieved with a per-member `JsonName` on the SDT plus `JsonNullSerialization = 'NoProperty'` (empty fields are not serialized); the other web services still use `XMLName`.

**Status codes**: `HttpResponse` exposes no StatusCode setter in GX17, but the real status **can** be set with `PXTools.APIs.SetHttpStatus` (inline JAVA over `httpContext.getResponse().setStatus(...)`), called **before** the `AddHeader`/`AddString` calls. `Token` already returns real 400/401 this way, in addition to the informational `X-OAuth-Status` header. The other four web services still always answer 200: exposed as REST, the `parm out` wraps the body in `{"SDTResponse":{...}}` and there is no control over the status.

> **Exposure criterion**: when a third party defines the HTTP contract (an RFC, a provider, a protocol) → **main + `CallProtocol='HTTP'`**, which gives byte-exact control of the body and the status. When you define the contract yourself → an **API object**, which brings OpenAPI, per-method verbs and the `&RestCode` variable (which exists **only** in API objects, not in REST procedures).

## The module's public API (procedures the host can invoke)

| Procedure | Parm | Use |
|---|---|---|
| `OAuthService.CreateNewAuthorization.Udp` | in:SDTAuthorizationIn, out:SDTAuthorizationOut | Generates an authorization code with optional PKCE |
| `OAuthService.CreateTokens.Udp` | in:SDTExchangeAuthorizationCodeForAccessTokenIn, out:SDTExchangeAuthorizationCodeForAccessTokenOut | Issues an access + refresh token |
| `OAuthService.RetDataFromToken.Udp` | in:token, out:SDTDataFromToken | Internal token introspection (no auth) |
| `OAuthService.RevokeToken.Udp` | in:token, out:SDTResult | Revokes a token |
| `OAuthService.RevokeAuthorization.Udp` | in:client_id, AccountRef, code, out:SDTResult | Revokes an authorization + cascade |
| `OAuthService.SilentAuthorization.Udp` | in:SDTAuthorizationIn, out:SDTAuthorizationOut | Authorization + token synchronously (pre-authorized machine-to-machine) |
| `OAuthService.PurgeExpiredTokens.Udp` | out:SDTPurgeResult | Cleans expired/revoked tokens (callable directly or from the Task) |
| `OAuthService.GenerateIdToken.Udp` | in:SDTIdTokenClaims, in:client_secret, out:JWT string | Builds the HS256 JWT for the id_token |
| `OAuthService.NormalizeRedirectUri.Udp` | in:url, out:normalized | Normalises a URL for robust comparison |
| `OAuthService.DateTimeToUnixTimestamp.Udp` | in:DateTime, out:seconds | Unix timestamp for JWT claims (uses YMDHMStoT + Difference) |
| `OAuthService.HasScope.Udp` | in:tokenScopes, in:requiredScope, out:hasScope | Checks an OAuth scope with exact space-delimited matching (it prevents false positives such as `write` matching `write:invoices`). Use it with `RetDataFromToken.Udp(token).Scopes` for scope-based authorization. |

## Hooks the host MUST implement

These procedures live in `Personalized/` as **stubs** and the module's host must override them:

### `CheckUserLoginData.Procedure`
**Signature**: `in:&UserCode, in:&UserPassword, out:&IsOk`
**Implement**: credential validation against the host's Users table (or a federated IdP).
**Used in**: `Authorization.WebPanel` when the user logs in.

### `RetOAuthServiceIssuer.Procedure`
**Signature**: `out:&Issuer` (Character 255)
**Implement**: return the Authorization Server's public URL (e.g. `https://accounts.mycompany.com`).
**Used in**: the id_token's `iss` claim, the discovery document, and so on.

### `RetUserInfo.Procedure`
**Signature**: `in:&AccountReference, in:&Scopes, out:&SDTOAuthUserInfo`
**Implement**: obtain the user's claims (sub, name, email, etc.) from the host's Users table.
**Used in**: the `/userinfo` endpoint.
**OIDC rules**:
- scope=openid → sub (always)
- scope=profile → name, given_name, family_name, preferred_username, locale, zoneinfo, updated_at
- scope=email → email, email_verified

### `ChkOAuthRateLimit.Procedure` (OPTIONAL)
**Signature**: `in:&ClientId, in:&IPAddress, in:&Endpoint, out:&Allowed, out:&RetryAfterSecs`
**Implement**: rate limiting (a local table, Redis, a WAF). The stub allows everything.
**Used in**: `/token`, at the start. If `&Allowed=False`, it returns 429 with a Retry-After header.

### The `OAuthServiceScope` domain (customizable through the Personalized XPZ)
**Location**: `Knowledge Base/@PXTools/@OAuthService/#Domains/OAuthServiceScope.gxDomain` — a module domain (GeneXus supports domains inside modules, using `#Domains` as a module subfolder). **Conceptually it is part of the module's Personalized XPZ**: it is included in that XPZ so that, on importing the module, the host replaces/extends the EnumValues with its application's real scopes.

**Initial state**: a single example value, `Example: "example:read"`, as a placeholder.

**The host must replace it**: when integrating the module into a real KB, the host edits the domain's `EnumValues` with its business scopes (e.g. `read:invoices`, `write:invoices`, `admin`, `read:users`, and so on). The enum can be used wherever convenient to pass the required scope to `HasScope.Udp()` or to the PXTools WS Layer pattern.

### `SendTokenToThirdParty.Procedure` (OPTIONAL)
**Signature**: `in:&AccessTokenSDT, out:&ErrorCode, out:&ErrorDescription`
**Implement**: push the access_token to an external system (only if `SilentAuthorization` is used).
**Used in**: `SilentAuthorization.Procedure` (the asynchronous machine-to-machine flow).

## Configuration (SystemParameters)

Defined in `Personalized/RetSystemParametersOAuthService.DataProvider.gxSource`:

| Parameter | Type | Default | Purpose |
|---|---|---|---|
| `OAuthServiceGenerateLog` | Boolean | false | Enables debug `Msg()` calls in the internal procedures |

Retrieve it with `RetSystemParameterPreferenceBoolean.Udp(SystemParameterCode.OAuthServiceGenerateLog)`.

## TaskManager integration

The module registers a Task for automatically purging expired tokens:

- **Task**: `TskOAuthServicePurgeExpiredTokens.Procedure` — it invokes `PurgeExpiredTokens.Udp()`
- **Registry**: `RetDynamicCallReferencesOAuthService.DataProvider` declares the Task in `DynamicCallReferences` so the TaskManager can discover it
- **Enum**: `DynamicCallReferenceCode.TskOAuthServicePurgeExpiredTokens` (add it to the host's domain if it is not there)

The host configures the frequency from the TaskManager CRUD screen (recommended: daily, outside peak hours).

## Dependent PXTools modules (they must be present in the host KB)

| PXTools module | Use |
|---|---|
| `@WebServicesLog` | Invocation logging (AddWebServiceLog.Udp, UpdWebServiceLog.Call) |
| `@SystemParameters` | Configuration (RetSystemParameterPreferenceBoolean.Udp) |
| `@TaskManager` | To run TskOAuthServicePurgeExpiredTokens (the TaskManagerId attribute, the TaskManagerExecutionResponse domain) |
| `@DynamicCallReferences` | The Task registry (SDTDynamicCallReferences, the DynamicCallReferenceCode domain) |

External GeneXus modules: **GeneXusCryptography** (the Hashing, Hmac EOs) and **SecurityAPICommons** (the Base64Encoder, HexaEncoder EOs). Both are needed for PKCE S256 and HS256 JWT.

## Supported OAuth flows

### Authorization Code Flow (with PKCE)
1. The client opens a browser at `/authorize?response_type=code&client_id=X&redirect_uri=Y&scope=openid+profile&code_challenge=Z&code_challenge_method=S256`
2. `Authorization.WebPanel` shows the login, calls `CheckUserLoginData`, generates the code through `CreateNewAuthorization`, and redirects to the redirect_uri with `?code=...`
3. The client POSTs to `/token` with `grant_type=authorization_code&code=...&client_id=...&client_secret=...&code_verifier=...`
4. `token.Procedure` validates PKCE (SHA256(code_verifier) == the stored code_challenge), validates the normalised redirect_uri, and issues the tokens
5. If the scope includes `openid`, the response also carries an `id_token` (HS256 JWT)

### Client Credentials Flow
1. The client POSTs to `/token` with `grant_type=client_credentials&client_id=X&client_secret=Y&scope=Z`
2. `token.Procedure` validates the client, creates a synthetic authorization and issues an access_token (no refresh)

### Refresh Token Flow
1. The client POSTs to `/token` with `grant_type=refresh_token&refresh_token=X&client_id=Y&client_secret=Z`
2. `token.Procedure` validates the refresh_token and issues a new access_token (the refresh is not rotated)

### Silent (asynchronous machine-to-machine)
1. System A calls `SilentAuthorization.Udp` with a pre-authorized client (the SilentAuthorization=True flag in `OAuthServiceClient`)
2. The module creates the authorization+token and pushes the token to system B through the `SendTokenToThirdParty` hook

## Known gaps / limitations

- **`/authorize` is not a REST API** — it is a WebPanel. Pure SPA clients would need it migrated to REST. Not blocking for server-side apps.
- **`HttpResponse.StatusCode` is not settable** — GeneXus exposes no setter. OAuth errors are signalled in the body (the `error` field); the logical HTTP status is exposed in the `X-OAuth-Status` header so the host can map it if it wants.
- **HS256 only** — the id_token is signed with the client_secret. There is NO RS256 and no JWKS endpoint.
- **`code_challenge_method=plain`** — supported but NOT recommended (use S256).
- **No refresh token rotation** — on refresh, the refresh_token is NOT rotated (the RFC allows it, but we do not require it).
- **No `prompt`, `max_age` or `acr_values`** on the OIDC side — only the basic flow.

## How to import the module into a KB

1. **Check the dependencies in the host KB**:
   - PXTools modules: `@WebServicesLog`, `@SystemParameters`, `@TaskManager`, `@DynamicCallReferences`
   - GeneXus modules: `GeneXusCryptography`, `SecurityAPICommons`
2. **Import the `@PXTools/@OAuthService/` folder** through the Knowledge Manager.
3. **Implement the hooks** in `Personalized/`:
   - `CheckUserLoginData` → your auth system
   - `RetOAuthServiceIssuer` → your public URL (probably from a SystemParameter)
   - `RetUserInfo` → your Users table
   - (Optional) `ChkOAuthRateLimit`, `SendTokenToThirdParty`
4. **Add the `TskOAuthServicePurgeExpiredTokens` enum** value to your KB's `DynamicCallReferenceCode` domain (if it is not there).
5. **Configure URL rewriting** in the web server to map `/.well-known/openid-configuration` to the `openidconfiguration` endpoint.
6. **Register the OAuth clients** through the `OAuthServiceClient` transaction (client_id, client_secret, redirect_uri, status, token policy).
7. **Build All** and verify that the module's 4 tables are created.

## Summary

A working OAuth 2.0 / OIDC module, self-contained in its data (its own tables) and decoupled from the host through hooks (`Personalized/`). Ready to be packaged as a PXTools dependency.
