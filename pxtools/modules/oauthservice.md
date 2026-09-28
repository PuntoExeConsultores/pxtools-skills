# @OAuthService Module — OAuth 2.0 / OIDC Authorization Server

Path: `@PXTools/@OAuthService/`
Qualified name: `PXTools.OAuthService`

Subfolders: `APIs/Basic/` (engine + SDTs), `APIs/WS/` (the HTTP endpoints), `APIs/Tasks/`,
`Personalized/`, `#Domains/`.

## 1. What it provides

An OAuth 2.0 / OpenID Connect **authorization server implemented inside the Knowledge Base** — not a
client of somebody else's. It stores clients, authorizations and tokens in its own four tables and
exposes the standard endpoints: authorize, token, introspect (RFC 7662), revoke (RFC 7009), userinfo,
dynamic client registration (RFC 7591) and discovery.

Grants supported: `authorization_code` (with PKCE), `refresh_token` and `client_credentials`. ID
tokens are JWTs signed HS256 with the client secret; there is no asymmetric signing and no JWKS
endpoint.

It also issues authorizations **to the system itself on a person's behalf**, with no browser involved.
That is what lets a non-web surface — a chat channel, a device — act as a user with bounded scopes.

## 2. Core concept: the authorization is the grant, the token is disposable

A row of `OAuthServiceAuthorization` means *"this account consented that this client may act with
these scopes"*. Tokens hang off it and **carry no identity of their own**: client, user, scopes and
account reference are all subtypes resolved through the foreign key.

Everything else follows from that:

- Revoking the authorization kills every token it issued, at once.
- Token validation re-reads the client and the authorization on every call, so disabling a client
  invalidates its live tokens immediately — no waiting for expiry.
- Issuing a new authorization for the same (client, account) **silently revokes the previous one**.
  Re-authorizing invalidates everything outstanding from the earlier grant.

**`AccountReference` is the identity anchor.** It is what the login writes, what becomes the OIDC
subject, and what a resource server gets back from validation. It is the same across all of one
person's authorizations — so when a consumer issues one authorization per connection (one per linked
device, say), `AuthorizationId` is what tells the connections apart, not `AccountReference`.

## 3. Module transactions (4)

| Transaction | What a row is |
|---|---|
| `OAuthServiceTokenPolicy` | A named duration profile: access and refresh lifetimes, or "never expires" |
| `OAuthServiceClient` | A registered application: public id, secret, one redirect URI, status, policy |
| `OAuthServiceAuthorization` | One account's consent to one client: code, scopes, PKCE challenge, status |
| `OAuthServiceToken` | One issued credential: code, use type (access/refresh), format, status |

A client has **exactly one** redirect URI. Client secrets are stored in clear, protected by the access
control on the screen.

## 4. Module domains

Owned: `AuthorizationGrantType`, `AuthorizationStatus`, `ClientStatus`, `OAuthServiceScope` (the
scopes an installation defines), `OAuthServiceClientId`.
Used from the PXTools root: `OAuthTokenFormatType`, `OAuthTokenUseType`, `OAuthTokenStatus`.

## 5. The flows

**Authorization code (+ PKCE)** — `Authorization` (a WebPanel in `Personalized/`, because the consent
screen is naturally per-installation) validates the client and the redirect URI, captures the PKCE
challenge, authenticates the person through the `CheckUserLoginData` hook, and calls
`CreateNewAuthorization`, which revokes the previous consent and creates the row with a code that
lives 10 minutes. The browser is redirected back with the code. `Token` then verifies the PKCE
verifier, the client credentials and the normalized redirect URI, loads the policy and calls
`CreateTokens`.

**Refresh** — the same endpoint, looking the code up among refresh tokens. The refresh token is **not
rotated**.

**Client credentials** — the client authenticates and a *synthetic* authorization is created with the
client id as its account reference, to keep the relational model consistent.

**Internal / headless** — `CreateInternalAuthorization` (`Personalized/`) does the same in-process:
no browser, no PKCE, because the code never travels. Consumers then ask `RetTokenForAuthorization`
for a still-valid access token, which renews from the refresh token when needed. This is the path a
chat channel or a device uses.

**Validation** — a resource server calls `RetDataFromToken` and then `HasScope`. `HasScope` pads with
spaces on both sides, so `write` does not match `write:invoices`.

**Purge** — a TaskManager task deletes expired and revoked rows.

## 6. APIs vs Personalized

- **`APIs/Basic/`** — the engine: the transactions, `CreateNewAuthorization`, `CreateTokens`,
  `DisableAuthorizationTokens`, `RetDataFromToken`, `RevokeAuthorization`, `RevokeToken`,
  `SilentAuthorization`, `PurgeExpiredTokens`, plus pure helpers (`HasScope`, `NormalizeRedirectUri`,
  `GenerateIdToken`, `ToBase64Url`) and the SDTs.
- **`APIs/WS/`** — the six HTTP endpoints.
- **`Personalized/`** — the hooks. Three ship as **stubs and must be implemented** before the server
  faces anything hostile:

  | Object | Ships as |
  |---|---|
  | `ChkOAuthRateLimit` | allows everything |
  | `RetUserInfo` | returns only the subject |
  | `RetOAuthServiceIssuer` | reads a system parameter that starts empty |
  | `SendTokenToThirdParty` | a message, no delivery |

  Also here: `Authorization` (the consent screen), `CheckUserLoginData`,
  `CreateInternalAuthorization`, `RetTokenForAuthorization`, and the module's three registrations.

## 7. Pattern instances

`PXWorkWithOAuthServiceClient`, `…TokenPolicy`, `…Authorization`, `…Token` — CRUD.
`PXParameterRequestOAuthMethods` — a test screen for silent authorization and revocation.

The client screen carries a **non-standard delete action** calling `DelOAuthServiceClient`, because
referential integrity forbids deleting a client from the transaction: the procedure removes tokens,
then authorizations, then the client.

## 8. Traps

- **Duration `0` means "never expires", not "no token".** It is declared in two places and is the
  first thing to check when a token outlives its welcome.
- **Form-encoded values must be URL-decoded or nothing matches.** An undecoded redirect URI never
  equals the registered one and the exchange fails as `invalid_grant`.
- **These endpoints deliberately avoid `Parm(out:)`.** A GeneXus web service wraps the body in the
  SDT's name and no OAuth client understands that shape; they are `IsMain` + `CallProtocol='HTTP'`
  and write the JSON themselves.
- **Discovery URLs are built with `.Link()`**, never by hand, because the published object name
  embeds the model namespace.
- **`/.well-known/openid-configuration` needs a URL rewrite in the host**: the spec looks for it at
  the domain root and GeneXus publishes under the application path.
- **Dynamic registration is open by design** — the risk is bounded because a fresh client holds no
  authorization. Self-registered clients get a name prefix so they can be listed and purged.
- **Seeding overwrites hand-edited token policies.** An installation that needs its own durations
  declares its own policy instead of editing the seeded ones.

## 9. Known gaps

The module is not finished. What follows was verified in the code and matters before it faces
anything but trusted callers:

1. **The authorization code is not consumed when it is exchanged.** Nothing marks the authorization
   as used, so the same code can be replayed for its full 10-minute window, minting fresh token pairs
   each time. RFC 6749 requires single use.
2. **The purge deletes authorizations that still own live tokens.** An authorization's expiration is
   always creation + 10 minutes — for client-credentials and internal grants too — while the default
   policy issues hour-long access and month-long refresh tokens. The column is doing duty as both
   "code TTL" and "row retention", and they are not the same number.
3. **The authorize endpoint does not check the client's status**, while token, introspect and revoke
   all do. A disabled client can still obtain a code.
4. **Refresh is impossible under a never-expiring policy**: the lookup requires an expiration greater
   than now, which a null never satisfies. Two other readers of the same table handle the null case
   correctly; this one does not.
5. **The redirect URI is compared raw in one place and normalized in another**, so a URI can pass one
   gate and fail the other.
6. **The module's own domains carry values belonging to one installation** — concrete client
   instances and a concrete scope baked into a generic module. They have to move out before the module
   can be reused. The procedures themselves are clean.
7. Exposure style is inconsistent: three endpoints write raw JSON and three are GeneXus web services
   with an out parameter — the very shape the first three explain breaks OAuth clients.

## References
- [21-oauth-service.md](../21-oauth-service.md) — **the integration guide**: the HTTP endpoints one by
  one, the hooks a host KB must implement, the configuration parameters and the import procedure. This
  document covers how the module works; that one covers how to put it into a Knowledge Base.
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [security.md](security.md) — where an account reference becomes a user context.
- [mcpserver.md](mcpserver.md) — the heaviest consumer of token validation.
- [messaging.md](messaging.md) — issues one internal authorization per linked chat.
