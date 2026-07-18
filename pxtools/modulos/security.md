# Módulo @Security — Seguridad de PXTools

> Comportamiento del módulo `@PXTools/@Security`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@Security/`
- Conector de control de acceso (los procs que se llaman desde los objetos generados): `Knowledge Base/@PXTools/@APIs/Personalized/SecurityConnector/`
- Dominios enum: `@Security/#Domains/` y el `#Attributes` / `#Domains` raíz de la KB.
- **Depende de:** `@APIs` (base), `@System` (catálogo de objetos), `@SystemParameters`, `@ControlPreferences`, `@SendMails` (registro/recupero de password). Infra: `@Menus`. *(El grafo canónico lista `@WebServicesLog` para los sub-features Silent Sign On/Server, pero en esta KB no hay referencias `PXTools.WebServicesLog` bajo @Security.)*

## 1. Qué provee

Sistema de **autenticación y autorización** con:

- **Usuarios, Roles** y la asignación **N:N** entre ellos.
- **Dominios** (particiones / tenants) con roles por dominio y tipo de usuario.
- **ACL a nivel objeto/acción** (`SecurityObjectAccess`): qué recurso (usuario o rol) puede acceder a qué pantalla o ejecutar qué acción.
- **Auto-registro** de usuarios con confirmación, y **silent sign-on** (SSO).
- Integración con el **contexto de sesión** (`&Context`) que usa el resto de la app (multitenant).

> **Política por defecto: _default-allow_.** Un objeto **sin** filas de ACL es **accesible para todos**. La seguridad se activa recién cuando se registran permisos y se asocian a roles/usuarios (ver §6.5 y §7). Esto explica por qué una pantalla o acción **nueva funciona para todos** sin configurar nada.

## 2. Concepto central: SecurityParty (un recurso = usuario **o** rol)

Un **SecurityParty** es un *recurso de seguridad* que puede ser **un usuario o un rol**. Las claves `SecurityUserId` y `SecurityRoleId` son **subtipos de `SecurityPartyId`** (comparten el dominio `IdFirstLevel` y un único espacio de claves).

El objetivo del concepto: al definir seguridad sobre un recurso (una **pantalla** o una **acción**), poder **asociarla tanto a usuarios como a roles indistintamente** — igual que los permisos de archivos/carpetas de Windows se asignan a usuarios **o** a grupos de usuarios. Esa asociación *recurso ↔ party* vive en la tabla **`SecurityObjectAccess`**.

```
        SecurityParty   (supertabla de identidad)
        PK SecurityPartyId · SecurityPartyType (User / Role / All)
             ▲                         ▲
      subtipo│                   subtipo│
     SecurityUserId            SecurityRoleId
     (tabla SecurityUsers)     (tabla SecurityRoles)
```

**Cómo se comparte el id:** al dar de alta un usuario o un rol, la transacción **no** autonumera su PK. Primero inserta una fila en `SecurityParty` (con `SecurityPartyType` = User/Role) vía `UpdSecurityParty` → `AddSecurityParty` (que hace `max(SecurityPartyId)+1` sobre toda la tabla) y reutiliza ese id como su clave (`SecurityUsers.Transaction` regla `RefCall(UpdSecurityParty, Insert, …, SecurityUserId)`; ídem `SecurityRoles`). Así `SecurityUserId` y `SecurityRoleId` **nunca colisionan**, y `SecurityObjectAccess.SecurityPartyId` puede apuntar a un usuario o a un rol sin ambigüedad.

## 3. Transacciones del módulo (11)

| Transacción | PK | Rol en el modelo |
|---|---|---|
| **SecurityParty** | `SecurityPartyId` | Supertabla de identidad (usuarios + roles). Discriminador `SecurityPartyType`. Fuente del id compartido. |
| **SecurityUsers** | `SecurityUserId` | Usuarios (subtipo de Party). Login, password, dominio, estado, tipo, web-session, SSO, datos personales/LDAP. |
| **SecurityRoles** | `SecurityRoleId` | Roles (subtipo de Party). Portador del flag admin `SecurityRoleAdmin`. |
| **SecurityUserRoles** | `SecurityUserId, SecurityRoleId` | Asignación **N:N** usuario ↔ rol. |
| **SecurityDomains** | `SecurityDomainId` | Dominios (tenants/particiones). Autoreferencia Main/Alias (`SecurityDomainMainId`). |
| **SecurityDomainRoles** | `SecurityDomainId, SecurityDomainRoleUserType, SecurityRoleId` | Roles por defecto de un dominio, segmentados por tipo de usuario (FrontEnd/BackOffice). |
| **SecurityUsersDomains** | `SecurityUserId` | Transacción **alternativa sobre la misma tabla `SecurityUsers`**, para ABM de usuarios acotado al dominio del contexto. |
| **SecurityObjectAccess** | `SystemObjectName, SecurityObjectAccessActionCode, SecurityPartyId` | **ACL objeto/acción → party.** Núcleo de la autorización (lo consultan `PIsAuthorized`/`PIsEnabled`). |
| **SecurityObjectRecordsAccess** | `SystemObjectName, SecurityRecordAccessId1..10, …ActionCode, SecurityPartyId` | Seguridad a nivel de **registro** (row-level, hasta 10 claves). *Definida; su consumo está fuera del `SecurityConnector` — no documentada en detalle acá.* |
| **Registration** | `RegistrationId` | Auto-registro de usuarios con código de confirmación y estado. |
| **SilentSignOnRequests** | `SilentSignOnRequestId` | Solicitudes de login silencioso (SSO) con vencimiento y estado. |

### Detalle de las clave del modelo

- **SecurityUsers** — atributos relevantes: `SecurityUserCode` (login), `SecurityUserQualifiedCode` (fórmula `code@domain`), `SecurityUserName`, `SecurityUserPsw` (`PswEnc`), `SecurityUserDisabled`, `SecurityUserWebSessionDisabled`, `SecurityUserSilentSignOnEnabled`, `SecurityUserType` (`SecurityUserType`), `SecurityUserDomainId` (→ `SecurityDomains`), `SecurityUserWebSessionId`. La aplicación puede además **embeber atributos de negocio** en `SecurityUsers` (p. ej. la entidad / tenant a la que pertenece el usuario), que así quedan disponibles en el contexto de sesión (ver §11).
- **SecurityRoles** — `SecurityRoleName`, **`SecurityRoleAdmin` (Boolean) = flag de administrador**, `SecurityRoleDescription`, `SecurityRoleType` (System/Project).
- **SecurityDomains** — `SecurityDomainName`, `SecurityDomainDisabled`, `SecurityDomainType` (Main/Alias, default Main), `SecurityDomainMainId` (autoref: un Alias apunta a su Main).
- **SecurityObjectAccess** — `SystemObjectName` (→ `TSystemObjects`, ver §3.1), `SecurityObjectAccessActionCode` (dominio plano `SecurityActions`, pero almacena valores de `SecurityActionCode`: `Access`/`Full`/`INS`/`UPD`/`DLT`/`DSP`), `SecurityPartyId` (→ `SecurityParty`).

### 3.1 SystemObjectName vive en @System (no en @Security)

El catálogo de "objetos del sistema" es **`@PXTools/@System/APIs/TSystemObjects.Transaction.gxSource`** (PK `SystemObjectName`, dominio `ObjectName, GeneXus`). `SecurityObjectAccess.SystemObjectName` (y el row-level) son **FK a `TSystemObjects`**: la ACL se cuelga del catálogo de objetos. El mismo catálogo lo referencian `@OAV` y `@ControlPreferences`.

## 4. Modelo (relaciones)

```
                         SecurityParty (SUPERTABLA)
                         PK SecurityPartyId · disc SecurityPartyType
                          ▲ id compartido (AddSecurityParty)
            ┌─────────────┴─────────────┐
      SecurityUsers                SecurityRoles
      PK SecurityUserId            PK SecurityRoleId
      · SecurityUserDomainId ──▶ SecurityDomains   · SecurityRoleAdmin (admin)
      · SecurityUserType (BO/FE/IN)
            │  SecurityUserRoles (SecurityUserId, SecurityRoleId)  ← N:N
            └───────────────────────────┘

      SecurityDomains (PK SecurityDomainId · SecurityDomainMainId self Alias→Main)
            │  SecurityDomainRoles (SecurityDomainId, SecurityDomainRoleUserType, SecurityRoleId)
            └── roles por dominio y tipo de usuario

      TSystemObjects (@System, PK SystemObjectName)        SecurityParty
            ▲                                                   ▲
            │  SecurityObjectAccess (SystemObjectName, ─────────┘
            └─ SecurityObjectAccessActionCode, SecurityPartyId)   ← ACL objeto/acción/party

      Registration.RegistrationUserId ─────▶ SecurityUsers
      SilentSignOnRequests.…SecurityUserId ▶ SecurityUsers
```

## 5. Dominios enum de seguridad

**Todo dominio `Security*` / `Psw*` / `SilentSignOn*` / `SSOLinkTo` / `Registration*` / `CloseSessionErrorCode` / `EncKey` es de @Security** (por nomenclatura), **aunque otro módulo sea su único consumidor** — p.ej. `SecurityPartyIdCollection` lo usa solo @Alerts y `SecurityObjectCollection` solo @APIs, pero ambos son de @Security. La mayoría son **root-legacy** (`#Domains/` raíz); unos pocos ya son module-scoped (`@Security/#Domains/`: `SecurityRolesNames`, `SecurityUserNameOrQualifiedCode`, `SecurityUserStatus`).

| Dominio | Tipo | Valores (Enum = valor almacenado) |
|---|---|---|
| **SecurityActionCode** | Character(20) | Full=`Full`, Insert=`INS`, Update=`UPD`, Delete=`DLT`, Display=`DSP`, Access=`Access` |
| **SecurityActions** | Character(20) | *(dominio plano — tipo de los atributos `…ActionCode`; almacena valores de `SecurityActionCode`)* |
| **SecurityPartyType** | Numeric(1.0) | User=`1`, Role=`2`, All=`0` |
| **SecurityUserType** | Character(2) | BackOffice=`BO`, FrontEnd=`FE`, Invite=`IN` |
| **SecurityDomainType** | Character(3) | Main=`MAI`, Alias=`ALI` |
| **SecurityRoleType** | Character(3) | System=`SYS`, Project=`PRJ` |
| **SecurityUserStatus** *(module-scoped)* | Character(3) | Enabled=`ENA`, Disabled=`DIS` |
| **SecurityUserNameOrQualifiedCode** *(module-scoped)* | Character(20) | Name / Code / QualifiedCode / EMail |
| **RegistrationStatus** | Character(20) | Created / Registered / Confirmed / Verificated |
| **SilentSignOnRequestStatus** | Character(20) | Pending / Succeed / Fail |
| **SilentSignOnErrorCode** | Character(30) | InvalidUserPassword / NoPrivilegesToSilentSignOn / UserToSignOnNotExists / … |
| **CloseSessionErrorCode** | Character(30) | InvalidUserPassword / UserToCloseSessionNotExists / … |

Ids y colecciones (dominios de valor, sin enum): `SecurityUserId`, `SecurityRoleId`, `SecurityDomainId`, `SecurityUserCode` (Char(100) `@!`), `SecurityUserQualifiedCode`, `SecurityPartyIdCollection`, `SecurityObjectCollection`, `SecurityFunctions`; y las passwords `PswEnc` (Char(128)), `PswIng` (Char(64)), `EncKey` (Char(32)).

## 6. Mecanismo de autorización

Los objetos generados por los patterns invocan estos procs del `SecurityConnector`. Ambos comparten estructura y la **política default-allow**.

### 6.1 `PIsAuthorized` — acceso a **pantallas** (objetos)
`parm(in: &GxObject, out: &Authorized)`

1. `&GxObject = 'HNotAuthorized'` → **True** (la pantalla de "no autorizado" siempre es accesible).
2. Carga contexto (`PLoadContext`): `&SecurityUserId`, `&SecurityDomainId`, `&UserData = RetUserDataFromCode(&Context.SecurityUserQualifiedCode)`.
3. **Bypass admin**: si `&UserData.IsAdministrator` → **True**.
4. **ApplicationBlocked**: si la preferencia `SystemParameterCode.ApplicationBlocked` está activa → **False** (bloqueo global salvo admin).
5. `For Each` sobre **`SecurityObjectAccess`** `Where SystemObjectName = &GxObject` y `ActionCode = Access OR Full` (ver §6.3 la escalada usuario → rol → dominio).
6. **`When None` → True**: si el objeto **no** tiene ninguna fila de ACL, se concede acceso (**default-allow**).

### 6.2 `PIsEnabled` — acceso a **acciones** de pantalla
`parm(in: &GxObject, in: &Function, out: &Authorized)`

Misma estructura y default-allow, con dos diferencias:
1. Recibe `&Function` (código de acción, p. ej. `INS`/`UPD`/`DLT`/`DSP`) y filtra `Where ActionCode = &Function OR Full` → controla a **nivel de función/acción** dentro del objeto, no el acceso global.
2. **No** tiene el caso `HNotAuthorized` ni el gate `ApplicationBlocked`.

`PCheckSystemAccess` es un envoltorio: `PIsAuthorized('hhome')` (acceso al home del sistema).

### 6.3 Escalada usuario → rol → rol-por-dominio

Dentro del `For Each` sobre `SecurityObjectAccess`, por cada fila (`&SecurityPartyId`, `SecurityPartyType`):
- **Usuario** (`SecurityPartyType = User`): autoriza si `SecurityPartyId = &SecurityUserId`.
- **Rol** (else): busca en **`SecurityUserRoles`** vía DataSelector `SecurityUserRolesEnabled(&SecurityUserId, &SecurityPartyId)` — ¿el usuario tiene ese rol (activo)? Si sí → autoriza.
- `When None`: si hay dominio, busca en **`SecurityDomainRoles`** vía `SecurityDomainRolesEnabled(&SecurityDomainId, &UserData.Type, &SecurityPartyId)` — roles heredados del dominio por tipo de usuario (FrontEnd/BackOffice). Si aplica → autoriza.

**DataSelectors:**
- `SecurityUserRolesEnabled` — asignaciones activas de `SecurityUserRoles`: `SecurityUserDisabled = False`, dominio del usuario no deshabilitado, `SecurityUserId = &SecurityUserId`, y `SecurityRoleId = &SecurityRoleId` **sólo si el rol no viene vacío** (0/vacío = "todos los roles del usuario").
- `SecurityDomainRolesEnabled` — `SecurityDomainRoles` con `SecurityDomainDisabled = False`, filtrando por dominio, rol y tipo de usuario.

### 6.4 Bypass admin y ApplicationBlocked
- **Admin**: `SecurityRoles.SecurityRoleAdmin = True`. Se calcula con `PIsAdministratorFromCode` (mira los roles del usuario en el dominio Main, o los roles del dominio por tipo) y queda en `UserData.IsAdministrator`. Un admin pasa **todos** los chequeos.
- **ApplicationBlocked**: preferencia global (`SystemParameterCode.ApplicationBlocked`) que bloquea el acceso a no-admins (modo mantenimiento).

### 6.5 Política default-allow (clave)

`When None → True` significa **"todo lo no configurado es público"**. Consecuencias:
- Una pantalla/acción nueva **no** tiene restricciones hasta que alguien crea filas en `SecurityObjectAccess`.
- Restringir un recurso implica **definir explícitamente** quién sí (usuarios/roles); en cuanto existe **una** fila de ACL para ese objeto, deja de ser público (sólo los party listados —directos o vía rol/dominio— acceden).

## 7. Cómo asegurar una pantalla o una acción (how-to)

1. **El objeto se cataloga solo.** Al ejecutarse, los objetos generados registran el `SystemObjectName` en `TSystemObjects` (vía `PAddSecurityContext`). No hay que darlo de alta a mano.
2. **Mientras no haya ACL, es público** (default-allow, §6.5).
3. **Para restringir**, crear filas en **`SecurityObjectAccess`** con:
   - `SystemObjectName` = el objeto (pantalla o el que ejecuta la acción),
   - `SecurityObjectAccessActionCode` = `Access`/`Full` (pantalla) o `INS`/`UPD`/`DLT`/`DSP`/`Full` (acción),
   - `SecurityPartyId` = el **usuario o rol** habilitado.
4. **Asignar usuarios a roles** en `SecurityUserRoles` (y/o roles por dominio en `SecurityDomainRoles`) para que el permiso otorgado a un rol aplique a sus usuarios.
5. Los **admins** (`SecurityRoleAdmin`) siempre pasan.

> **Relación con la UI generada:** los patterns generan en el Start de la pantalla un `PIsAuthorized(<objetoDestino>)` que **oculta** el botón de una acción que navega si el usuario no está autorizado al destino (ver [12-acciones-patterns-ui.md](../12-acciones-patterns-ui.md) §10). Como el modelo es default-allow, un destino **nuevo** (sin ACL) da `True` y el botón **se ve para todos**; se oculta sólo una vez que se restringe ese destino y el usuario no califica.

## 8. Autenticación: login, registro y silent sign-on

**Login** — `HLogin.WebPanel` (página main) deriva a `WbLogin` (instancia `PXParameterRequestLogin`). Validación de credenciales:
- `PCheckUser` — resuelve dominio por nombre y verifica `SecurityUserCode` + `SecurityUserDomainId` + `SecurityUserPsw` (encriptada) contra usuarios habilitados.
- `PCheckUserWebSessionEnabled` — además exige `not SecurityUserWebSessionDisabled`.
- `AuthenticateWindowsUser` — si `AuthenticationMode = windows`, toma el usuario del `Thread.CurrentPrincipal` (SSO integrado Windows/LDAP) y setea el contexto.
- `CheckWebSession` / `SaveSecurityUserWebSessionId` — control de sesión web; si `CheckUniqueSessionPerUser` está activo, exige **una sesión por usuario** (compara `UserData.WebSessionId` con `&WebSession.Id`).

**Registro** — transacción `Registration`: alta de un usuario pendiente con `RegistrationConfirmCode` y `RegistrationStatus` (Created → Registered → Confirmed → Verificated); al confirmar se crea el `SecurityUsers` y se enlaza en `RegistrationUserId`. Instancias: `PXParameterRequestRegistrationBasic`, `PXParameterRequestRegistrationConfirmed`.

**Silent Sign-On (SSO)** — transacción `SilentSignOnRequests` (token/`AuthorizationId` con `UpTo` de vencimiento y `Status` Pending/Succeed/Fail). `CheckUserSilentSignOnEnabled` devuelve el `SecurityUserId` si el usuario tiene `SecurityUserSilentSignOnEnabled` + web-session habilitada.

## 9. Instancias ABM del módulo

| Tipo | Instancias |
|---|---|
| **PXWorkWith** | `PXWorkWithSecurityUsers`, `PXWorkWithSecurityUsersDomains`, `PXWorkWithSecurityRoles`, `PXWorkWithSecurityDomains`, `PXWorkWithSecurityObjectAccess`, `PXWorkWithSecurityObjectRecordsAccess`, `PXWorkWithSystemObjects`, `PXWorkWithRegistrations`, `PXWorkWithSilentSignOnRequests` |
| **PXParameterRequest** | `PXParameterRequestChangePassword`, `PXParameterRequestRegistrationBasic`, `PXParameterRequestRegistrationConfirmed` |
| **PXComposer** | `PXComposerSecurityObjectAccess`, `PXComposerSecurityObjectRecordAccess` |

## 10. Procedimientos clave del `SecurityConnector`

| Procedimiento | Propósito |
|---|---|
| `PIsAuthorized` | Control de acceso a **pantallas** (§6.1). |
| `PIsEnabled` | Control de acceso a **acciones/funciones** de pantalla (§6.2). |
| `PCheckSystemAccess` | `PIsAuthorized('hhome')` — acceso al home. |
| `PIsAdministrator` / `PIsAdministratorFromCode` | Cálculo del flag admin desde los roles (`SecurityRoleAdmin`). |
| `RetUserDataFromCode` / `RetUserDataFromId` | Arman el SDT `UserData` (incluye `IsAdministrator`, tipo, dominio, email, SSO, web-session). |
| `PCheckUser` / `PCheckUserWebSessionEnabled` | Validación de credenciales (login). |
| `AuthenticateWindowsUser` | Autenticación integrada Windows/LDAP. |
| `CheckWebSession` / `SaveSecurityUserWebSessionId` | Sesión web (única por usuario opcional). |
| `CheckUserSilentSignOnEnabled` | Habilitación de silent sign-on. |
| `PAddSecurityContext` | Registra el objeto (nombre, funciones, módulo) en el `SecurityContext` de la sesión y en `TSystemObjects`. |
| `RetUserDomainFromQualifiedCode` / `RetDomainFromName` | Resuelven `code@domain` / dominio por nombre. |
| `CtSecurityUsersExists` | ¿Existe al menos un usuario? (bootstrap / primer arranque). |

## 11. Integración con el contexto (`&Context`)

`PLoadContext.Call(&Context)` (inyectado por los patterns) carga el `Context` de la sesión, del que salen los datos usados en toda la app — incluidos los **atributos de negocio** que la aplicación haya embebido en `SecurityUsers` (p. ej. la entidad / tenant del usuario). Por eso las pantallas FrontEnd cargan esos datos desde el contexto (regla multitenant) en vez de recibirlos por parámetro.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [12-acciones-patterns-ui.md](../12-acciones-patterns-ui.md) §10 — gating de visibilidad de acciones por `PIsAuthorized`.
- `@PXTools/@System/APIs/TSystemObjects.Transaction.gxSource` — catálogo de objetos del sistema.
