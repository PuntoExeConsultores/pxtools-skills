# @OAuthService — Authorization Server OAuth 2.0 + OpenID Connect

## Qué es

Módulo PXTools que implementa un **Authorization Server OAuth 2.0** (RFC 6749) con extensiones:
- **PKCE** (RFC 7636) — code_challenge S256 + plain
- **Token Introspection** (RFC 7662)
- **Token Revocation** (RFC 7009)
- **OpenID Connect Core 1.0** — id_token JWT HS256, /userinfo, /.well-known/openid-configuration

Provee endpoints HTTP REST para que aplicaciones cliente (web, mobile, M2M) obtengan access tokens autorizados por usuarios del sistema host.

## Ubicación en la KB

```
@PXTools/@OAuthService/
├── #Domains/                       ← AuthorizationGrantType, AuthorizationStatus, ClientStatus, OAuthServiceClientId
├── APIs/Basic/                     ← API core del modulo (publica + transacciones + SDTs)
│   ├── SDT/                        ← SDTs del API publica y respuestas HTTP
│   ├── Transactions               (OAuthServiceClient, OAuthServiceAuthorization, OAuthServiceToken, OAuthServiceTokenPolicy)
│   ├── Procedures                 (CreateNewAuthorization, CreateTokens, RetDataFromToken, RevokeToken, ...)
│   ├── Helpers JWT/PKCE           (GenerateIdToken, ToBase64Url, NormalizeRedirectUri, DateTimeToUnixTimestamp)
│   └── TskOAuthServicePurgeExpiredTokens.Procedure  ← Task del TaskManager
├── APIs/WS/                        ← Endpoints HTTP (WebService=True). Nombres GX en PascalCase; GeneXus baja el path a minuscula en Java/C# automaticamente.
│   ├── Token.Procedure             ← POST /pxtools.oauthservice/token
│   ├── Introspect.Procedure        ← POST /pxtools.oauthservice/introspect
│   ├── Revoke.Procedure            ← POST /pxtools.oauthservice/revoke
│   ├── UserInfo.Procedure          ← GET/POST /pxtools.oauthservice/userinfo (Bearer)
│   └── OpenIDConfiguration.Procedure ← GET /pxtools.oauthservice/openidconfiguration (mapear a /.well-known/openid-configuration en el web server)
└── Personalized/                   ← Hooks que el host del modulo IMPLEMENTA
    ├── Authorization.WebPanel      ← UI /authorize (login + consent)
    ├── CheckUserLoginData.Procedure ← Validacion credenciales contra sistema del host
    ├── ChkOAuthRateLimit.Procedure ← Rate limiting (stub: permite todo)
    ├── RetOAuthServiceIssuer.Procedure ← URL publica del AS (issuer)
    ├── RetUserInfo.Procedure       ← Datos del usuario para OIDC UserInfo
    ├── SendTokenToThirdParty.Procedure ← Callback opcional post-token (silent flow)
    └── RetDynamicCallReferencesOAuthService.DataProvider ← Registro de Tasks/Cleaners
```

## Tablas que crea el modulo

| Tabla | Proposito |
|---|---|
| `OAuthServiceClient` | Aplicaciones cliente registradas (client_id + client_secret + redirect_uri + status) |
| `OAuthServiceAuthorization` | Authorizations emitidas (code + AccountReference + scope + PKCE challenge + expiracion) |
| `OAuthServiceToken` | Access tokens y refresh tokens emitidos (token + status + expiracion + cliente + authorization) |
| `OAuthServiceTokenPolicy` | Politicas de expiracion por cliente (access_token TTL, refresh_token TTL) |

**El usuario del host se vincula al modulo via** `OAuthServiceAuthorizationAccountReference` (Character 255 genérico). NO hay FK directa a tablas del host.

## Endpoints HTTP

| Endpoint | Procedure GX | Spec | Notas |
|---|---|---|---|
| **token** | `APIs/WS/Token.Procedure` | RFC 6749 §3.2 | **main + `CallProtocol='HTTP'`**. grant_type: authorization_code / refresh_token / client_credentials |
| **introspect** | `APIs/WS/Introspect.Procedure` | RFC 7662 | REST. Devuelve `{"active":true,...}` o `{"active":false}` |
| **revoke** | `APIs/WS/Revoke.Procedure` | RFC 7009 | REST. Acepta `token` o `code` |
| **userinfo** | `APIs/WS/UserInfo.Procedure` | OIDC Core §5.3 | REST. Bearer token; valida scope=openid |
| **openidconfiguration** | `APIs/WS/OpenIDConfiguration.Procedure` | OIDC Discovery §3 | REST. El host mapea /.well-known/openid-configuration via URL rewrite en el web server |
| `GET /authorize` (UI) | `Personalized/Authorization.WebPanel` | RFC 6749 §3.1 | WebPanel, no REST. API-first NO implementado. |

**URLs reales** (verificadas en runtime, generador Java):

| Exposicion | URL |
|---|---|
| main + `CallProtocol='HTTP'` | `<base>/<namespace>.pxtools.oauthservice.a<objeto en minuscula>` — el prefijo `a` lo agrega el generador Java a los main procs |
| Expose as Web Service (REST) | `<base>/rest/PXTools/OAuthService/<Objeto>` — **respeta el casing del modulo y del objeto** |

> Es incorrecto que GeneXus baje los paths REST a minuscula: los publica con el casing del modulo. Por eso la URL REST **no** queda OAuth-compliant en su forma, aunque el contrato del body si lo sea.

**Respuestas HTTP**: JSON con field names snake_case. En `Token` esto se logra con `JsonName` por miembro del SDT + `JsonNullSerialization = 'NoProperty'` (los campos vacios no se serializan); los demas WS todavia usan `XMLName`.

**Status codes**: `HttpResponse` no expone setter de StatusCode en GX17, pero el status real **si se puede fijar** con `PXTools.APIs.SetHttpStatus` (JAVA inline sobre `httpContext.getResponse().setStatus(...)`), llamandolo **antes** de los `AddHeader`/`AddString`. `Token` ya devuelve 400/401 reales por esta via, ademas del header informativo `X-OAuth-Status`. Los otros cuatro WS siguen respondiendo 200 siempre: expuestos como REST, el `parm out` envuelve el body en `{"SDTResponse":{...}}` y no hay control del status.

> **Criterio de exposicion**: cuando el contrato HTTP lo define un tercero (un RFC, un proveedor, un protocolo) → **main + `CallProtocol='HTTP'`**, que da control byte-exacto del body y del status. Cuando el contrato lo define uno mismo → **API object**, que aporta OpenAPI, verbos por metodo y la variable `&RestCode` (que **solo** existe en API objects, no en procedures REST).

## API publica del modulo (procedures invocables desde el host)

| Procedure | Parm | Uso |
|---|---|---|
| `OAuthService.CreateNewAuthorization.Udp` | in:SDTAuthorizationIn, out:SDTAuthorizationOut | Genera authorization code con PKCE opcional |
| `OAuthService.CreateTokens.Udp` | in:SDTExchangeAuthorizationCodeForAccessTokenIn, out:SDTExchangeAuthorizationCodeForAccessTokenOut | Emite access + refresh token |
| `OAuthService.RetDataFromToken.Udp` | in:token, out:SDTDataFromToken | Introspect token interno (sin auth) |
| `OAuthService.RevokeToken.Udp` | in:token, out:SDTResult | Revoca un token |
| `OAuthService.RevokeAuthorization.Udp` | in:client_id, AccountRef, code, out:SDTResult | Revoca authorization + cascade |
| `OAuthService.SilentAuthorization.Udp` | in:SDTAuthorizationIn, out:SDTAuthorizationOut | Authorization + token sync (machine-to-machine pre-autorizado) |
| `OAuthService.PurgeExpiredTokens.Udp` | out:SDTPurgeResult | Limpia tokens expirados/revocados (invocable directo o desde Task) |
| `OAuthService.GenerateIdToken.Udp` | in:SDTIdTokenClaims, in:client_secret, out:JWT string | Construye JWT HS256 para id_token |
| `OAuthService.NormalizeRedirectUri.Udp` | in:url, out:normalized | Normaliza URL para comparacion robusta |
| `OAuthService.DateTimeToUnixTimestamp.Udp` | in:DateTime, out:seconds | Timestamp Unix para claims JWT (usa YMDHMStoT + Difference) |
| `OAuthService.HasScope.Udp` | in:tokenScopes, in:requiredScope, out:hasScope | Verifica scope OAuth con match exacto por espacios (evita falsos positivos tipo `write` matchear `write:invoices`). Usar con `RetDataFromToken.Udp(token).Scopes` para autorizacion por scope. |

## Hooks que el host DEBE implementar

Estos procs viven en `Personalized/` como **stubs** y el host del modulo debe sobreescribirlos:

### `CheckUserLoginData.Procedure`
**Firma**: `in:&UserCode, in:&UserPassword, out:&IsOk`
**Implementar**: validacion de credenciales contra la tabla Users del host (o IdP federado).
**Usado en**: `Authorization.WebPanel` cuando el usuario hace login.

### `RetOAuthServiceIssuer.Procedure`
**Firma**: `out:&Issuer` (Character 255)
**Implementar**: devolver la URL publica del Authorization Server (ej. `https://accounts.miempresa.com`).
**Usado en**: id_token claim `iss`, discovery doc, etc.

### `RetUserInfo.Procedure`
**Firma**: `in:&AccountReference, in:&Scopes, out:&SDTOAuthUserInfo`
**Implementar**: obtener claims del usuario (sub, name, email, etc.) desde la tabla Users del host.
**Usado en**: endpoint `/userinfo`.
**Reglas OIDC**:
- scope=openid → sub (siempre)
- scope=profile → name, given_name, family_name, preferred_username, locale, zoneinfo, updated_at
- scope=email → email, email_verified

### `ChkOAuthRateLimit.Procedure` (OPCIONAL)
**Firma**: `in:&ClientId, in:&IPAddress, in:&Endpoint, out:&Allowed, out:&RetryAfterSecs`
**Implementar**: rate limiting (tabla local, Redis, WAF). Stub permite todo.
**Usado en**: `/token` al inicio. Si `&Allowed=False`, retorna 429 con header Retry-After.

### Dominio `OAuthServiceScope` (personalizable via XPZ Personalized)
**Ubicacion**: `Knowledge Base/@PXTools/@OAuthService/#Domains/OAuthServiceScope.gxDomain` — dominio del modulo (GeneXus soporta dominios dentro de modulos usando `#Domains` como subfolder del modulo). **Conceptualmente forma parte del XPZ Personalized del modulo**: se incluye en ese XPZ para que al importar el modulo el host reemplace/extienda los EnumValues con los scopes reales de su aplicacion.

**Estado inicial**: un solo valor de ejemplo `Example: "example:read"` como placeholder.

**Host debe reemplazar**: al integrar el modulo en una KB real, el host modifica el `EnumValues` del dominio con la lista de scopes de su dominio de negocio (ej. `read:invoices`, `write:invoices`, `admin`, `read:users`, etc.). Se puede usar el enum donde sea conveniente para pasar el scope requerido a `HasScope.Udp()` o al pattern PXTools WS Layer.

### `SendTokenToThirdParty.Procedure` (OPCIONAL)
**Firma**: `in:&AccessTokenSDT, out:&ErrorCode, out:&ErrorDescription`
**Implementar**: push del access_token a un sistema externo (solo si se usa `SilentAuthorization`).
**Usado en**: `SilentAuthorization.Procedure` (flujo machine-to-machine asincrono).

## Configuracion (SystemParameters)

Definidos en `Personalized/RetSystemParametersOAuthService.DataProvider.gxSource`:

| Parametro | Tipo | Default | Proposito |
|---|---|---|---|
| `OAuthServiceGenerateLog` | Boolean | false | Activa Msg() de debug en procs internos |

Recuperar via `RetSystemParameterPreferenceBoolean.Udp(SystemParameterCode.OAuthServiceGenerateLog)`.

## TaskManager integration

El modulo registra una Task para purga automatica de tokens vencidos:

- **Task**: `TskOAuthServicePurgeExpiredTokens.Procedure` — invoca `PurgeExpiredTokens.Udp()`
- **Registry**: `RetDynamicCallReferencesOAuthService.DataProvider` declara el Task en el `DynamicCallReferences` para que el TaskManager lo descubra
- **Enum**: `DynamicCallReferenceCode.TskOAuthServicePurgeExpiredTokens` (agregar en el dominio del host si no esta)

El host configura la frecuencia desde el ABM de TaskManager (recomendado: diariamente fuera de hora pico).

## Modulos PXTools dependientes (deben estar presentes en la KB host)

| Modulo PXTools | Uso |
|---|---|
| `@WebServicesLog` | Log de invocaciones (AddWebServiceLog.Udp, UpdWebServiceLog.Call) |
| `@SystemParameters` | Configuracion (RetSystemParameterPreferenceBoolean.Udp) |
| `@TaskManager` | Para ejecutar TskOAuthServicePurgeExpiredTokens (atributo TaskManagerId, dominio TaskManagerExecutionResponse) |
| `@DynamicCallReferences` | Registry de Tasks (SDTDynamicCallReferences, dominio DynamicCallReferenceCode) |

Modulos GeneXus externos: **GeneXusCryptography** (EOs Hashing, Hmac) y **SecurityAPICommons** (EOs Base64Encoder, HexaEncoder). Necesarios para PKCE S256 y JWT HS256.

## Flujos OAuth soportados

### Authorization Code Flow (con PKCE)
1. Cliente abre browser en `/authorize?response_type=code&client_id=X&redirect_uri=Y&scope=openid+profile&code_challenge=Z&code_challenge_method=S256`
2. `Authorization.WebPanel` muestra login, llama `CheckUserLoginData`, genera code via `CreateNewAuthorization`, redirige al redirect_uri con `?code=...`
3. Cliente POST a `/token` con `grant_type=authorization_code&code=...&client_id=...&client_secret=...&code_verifier=...`
4. `token.Procedure` valida PKCE (SHA256(code_verifier) == stored code_challenge), valida redirect_uri normalizado, emite tokens
5. Si scope incluye `openid`, incluye `id_token` (JWT HS256) en la respuesta

### Client Credentials Flow
1. Cliente POST a `/token` con `grant_type=client_credentials&client_id=X&client_secret=Y&scope=Z`
2. `token.Procedure` valida cliente, crea authorization sintetica, emite access_token (sin refresh)

### Refresh Token Flow
1. Cliente POST a `/token` con `grant_type=refresh_token&refresh_token=X&client_id=Y&client_secret=Z`
2. `token.Procedure` valida refresh_token, emite nuevo access_token (refresh no rotado)

### Silent (machine-to-machine asincrono)
1. Sistema A llama `SilentAuthorization.Udp` con cliente pre-autorizado (flag SilentAuthorization=True en `OAuthServiceClient`)
2. El modulo crea authorization+token y empuja el token al sistema B via hook `SendTokenToThirdParty`

## Gaps conocidos / Limitaciones

- **`/authorize` no es API REST** — es WebPanel. Para clientes SPA puros se necesita migrar a REST. No bloqueante para apps server-side.
- **`HttpResponse.StatusCode` no settable** — GeneXus no expone el setter. Los errores OAuth se senalizan en el body (campo `error`); el status HTTP logico se expone en header `X-OAuth-Status` para que el host lo mapee si quiere.
- **Solo HS256** — id_token firmado con client_secret. NO hay RS256 ni JWKS endpoint.
- **`code_challenge_method=plain`** — soportado pero NO recomendado (usar S256).
- **No hay refresh token rotation** — al refrescar, el refresh_token NO se rota (RFC permite pero no lo exigimos).
- **No hay `prompt`, `max_age`, `acr_values`** del lado OIDC — solo el flujo basico.

## Como importar el modulo a una KB

1. **Verificar dependencias en la KB host**:
   - Modulos PXTools: `@WebServicesLog`, `@SystemParameters`, `@TaskManager`, `@DynamicCallReferences`
   - GeneXus modules: `GeneXusCryptography`, `SecurityAPICommons`
2. **Importar el folder `@PXTools/@OAuthService/`** via Knowledge Manager.
3. **Implementar los hooks** en `Personalized/`:
   - `CheckUserLoginData` → tu sistema de auth
   - `RetOAuthServiceIssuer` → tu URL publica (probablemente desde SystemParameter)
   - `RetUserInfo` → tu tabla de Users
   - (Opcional) `ChkOAuthRateLimit`, `SendTokenToThirdParty`
4. **Agregar el enum `TskOAuthServicePurgeExpiredTokens`** al dominio `DynamicCallReferenceCode` de tu KB (si no esta).
5. **Configurar URL rewriting** del web server para mapear `/.well-known/openid-configuration` al endpoint `openidconfiguration`.
6. **Registrar clientes** OAuth via la transaccion `OAuthServiceClient` (client_id, client_secret, redirect_uri, status, token policy).
7. **Build All** y verificar que las 4 tablas del modulo se creen.

## Resumen

Modulo OAuth 2.0 / OIDC funcional, autocontenido en datos (tablas propias) y desacoplado del host via hooks (`Personalized/`). Listo para empaquetar como dependencia de PXTools.
