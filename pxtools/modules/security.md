# @Security Module — PXTools Security

> Behaviour of the `@PXTools/@Security` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@Security/`
- Access-control connector (the procedures the generated objects call): `Knowledge Base/@PXTools/@APIs/Personalized/SecurityConnector/`
- Enumerated domains: `@Security/#Domains/` and the KB's root `#Attributes` / `#Domains`.
- **Depends on:** `@APIs` (base), `@System` (object catalogue), `@SystemParameters`, `@ControlPreferences`, `@SendMails` (registration/password recovery). Infrastructure: `@Menus`. *(The canonical graph lists `@WebServicesLog` for the Silent Sign On/Server sub-features, but in this KB there are no `PXTools.WebServicesLog` references under @Security.)*

## 1. What it provides

An **authentication and authorization** system with:

- **Users, Roles** and the **N:N** assignment between them.
- **Domains** (partitions / tenants) with per-domain roles and user types.
- **Object/action-level ACL** (`SecurityObjectAccess`): which resource (user or role) may access which screen or run which action.
- **Self-registration** of users with confirmation, and **silent sign-on** (SSO).
- Integration with the **session context** (`&Context`) the rest of the app uses (multi-tenancy).

> **Default policy: _default-allow_.** An object **with no** ACL rows is **accessible to everyone**. Security only kicks in once permissions are recorded and associated with roles/users (see §6.5 and §7). That is why a **new** screen or action **works for everybody** with no configuration at all.

## 2. Core concept: SecurityParty (one resource = a user **or** a role)

A **SecurityParty** is a *security resource* that can be **a user or a role**. The keys `SecurityUserId` and `SecurityRoleId` are **subtypes of `SecurityPartyId`** (they share the `IdFirstLevel` domain and a single key space).

The point of the concept: when defining security over a resource (a **screen** or an **action**), you can **associate it with users and roles interchangeably** — much as Windows file/folder permissions are granted to users **or** to groups of users. That *resource ↔ party* association lives in the **`SecurityObjectAccess`** table.

```
        SecurityParty   (identity supertable)
        PK SecurityPartyId · SecurityPartyType (User / Role / All)
             ▲                         ▲
      subtype│                  subtype│
     SecurityUserId            SecurityRoleId
     (SecurityUsers table)     (SecurityRoles table)
```

**How the id is shared:** when a user or a role is created, the transaction does **not** autonumber its PK. It first inserts a row into `SecurityParty` (with `SecurityPartyType` = User/Role) through `UpdSecurityParty` → `AddSecurityParty` (which does `max(SecurityPartyId)+1` over the whole table) and reuses that id as its own key (the `SecurityUsers.Transaction` rule `RefCall(UpdSecurityParty, Insert, …, SecurityUserId)`; the same for `SecurityRoles`). That way `SecurityUserId` and `SecurityRoleId` **never collide**, and `SecurityObjectAccess.SecurityPartyId` can point at a user or a role unambiguously.

## 3. Module transactions (11)

| Transaction | PK | Role in the model |
|---|---|---|
| **SecurityParty** | `SecurityPartyId` | Identity supertable (users + roles). Discriminator `SecurityPartyType`. Source of the shared id. |
| **SecurityUsers** | `SecurityUserId` | Users (a Party subtype). Login, password, domain, status, type, web session, SSO, personal/LDAP data. |
| **SecurityRoles** | `SecurityRoleId` | Roles (a Party subtype). Carrier of the `SecurityRoleAdmin` admin flag. |
| **SecurityUserRoles** | `SecurityUserId, SecurityRoleId` | The **N:N** user ↔ role assignment. |
| **SecurityDomains** | `SecurityDomainId` | Domains (tenants/partitions). Main/Alias self-reference (`SecurityDomainMainId`). |
| **SecurityDomainRoles** | `SecurityDomainId, SecurityDomainRoleUserType, SecurityRoleId` | A domain's default roles, segmented by user type (FrontEnd/BackOffice). |
| **SecurityUsersDomains** | `SecurityUserId` | An **alternative transaction over the same `SecurityUsers` table**, for user CRUD scoped to the context's domain. |
| **SecurityObjectAccess** | `SystemObjectName, SecurityObjectAccessActionCode, SecurityPartyId` | **The object/action → party ACL.** The core of authorization (`PIsAuthorized`/`PIsEnabled` query it). |
| **SecurityObjectRecordsAccess** | `SystemObjectName, SecurityRecordAccessId1..10, …ActionCode, SecurityPartyId` | **Record-level** security (row-level, up to 10 keys). *Defined; its consumption lives outside the `SecurityConnector` — not documented in detail here.* |
| **Registration** | `RegistrationId` | User self-registration with a confirmation code and a status. |
| **SilentSignOnRequests** | `SilentSignOnRequestId` | Silent sign-on (SSO) requests with an expiry and a status. |

### Detail of the model's keys

- **SecurityUsers** — relevant attributes: `SecurityUserCode` (login), `SecurityUserQualifiedCode` (a formula: `code@domain`), `SecurityUserName`, `SecurityUserPsw` (`PswEnc`), `SecurityUserDisabled`, `SecurityUserWebSessionDisabled`, `SecurityUserSilentSignOnEnabled`, `SecurityUserType` (`SecurityUserType`), `SecurityUserDomainId` (→ `SecurityDomains`), `SecurityUserWebSessionId`. The application may also **embed business attributes** in `SecurityUsers` (the entity / tenant the user belongs to, for instance), which then become available in the session context (see §11).
- **SecurityRoles** — `SecurityRoleName`, **`SecurityRoleAdmin` (Boolean) = the administrator flag**, `SecurityRoleDescription`, `SecurityRoleType` (System/Project).
- **SecurityDomains** — `SecurityDomainName`, `SecurityDomainDisabled`, `SecurityDomainType` (Main/Alias, default Main), `SecurityDomainMainId` (self-reference: an Alias points at its Main).
- **SecurityObjectAccess** — `SystemObjectName` (→ `TSystemObjects`, see §3.1), `SecurityObjectAccessActionCode` (the plain `SecurityActions` domain, though it stores `SecurityActionCode` values: `Access`/`Full`/`INS`/`UPD`/`DLT`/`DSP`), `SecurityPartyId` (→ `SecurityParty`).

### 3.1 SystemObjectName lives in @System (not in @Security)

The "system objects" catalogue is **`@PXTools/@System/APIs/TSystemObjects.Transaction.gxSource`** (PK `SystemObjectName`, domain `ObjectName, GeneXus`). `SecurityObjectAccess.SystemObjectName` (and row-level security) are **FKs to `TSystemObjects`**: the ACL hangs off the object catalogue. `@OAV` and `@ControlPreferences` reference the same catalogue.

## 4. The model (relationships)

```
                         SecurityParty (SUPERTABLE)
                         PK SecurityPartyId · disc SecurityPartyType
                          ▲ shared id (AddSecurityParty)
            ┌─────────────┴─────────────┐
      SecurityUsers                SecurityRoles
      PK SecurityUserId            PK SecurityRoleId
      · SecurityUserDomainId ──▶ SecurityDomains   · SecurityRoleAdmin (admin)
      · SecurityUserType (BO/FE/IN)
            │  SecurityUserRoles (SecurityUserId, SecurityRoleId)  ← N:N
            └───────────────────────────┘

      SecurityDomains (PK SecurityDomainId · SecurityDomainMainId self Alias→Main)
            │  SecurityDomainRoles (SecurityDomainId, SecurityDomainRoleUserType, SecurityRoleId)
            └── roles per domain and user type

      TSystemObjects (@System, PK SystemObjectName)        SecurityParty
            ▲                                                   ▲
            │  SecurityObjectAccess (SystemObjectName, ─────────┘
            └─ SecurityObjectAccessActionCode, SecurityPartyId)   ← object/action/party ACL

      Registration.RegistrationUserId ─────▶ SecurityUsers
      SilentSignOnRequests.…SecurityUserId ▶ SecurityUsers
```

## 5. Security enumerated domains

**Every `Security*` / `Psw*` / `SilentSignOn*` / `SSOLinkTo` / `Registration*` / `CloseSessionErrorCode` / `EncKey` domain belongs to @Security** (by naming), **even when another module is its only consumer** — for instance only @Alerts uses `SecurityPartyIdCollection` and only @APIs uses `SecurityObjectCollection`, yet both belong to @Security. Most are **root-legacy** (root `#Domains/`); a few are already module-scoped (`@Security/#Domains/`: `SecurityRolesNames`, `SecurityUserNameOrQualifiedCode`, `SecurityUserStatus`).

| Domain | Type | Values (Enum = the stored value) |
|---|---|---|
| **SecurityActionCode** | Character(20) | Full=`Full`, Insert=`INS`, Update=`UPD`, Delete=`DLT`, Display=`DSP`, Access=`Access` |
| **SecurityActions** | Character(20) | *(a plain domain — the type of the `…ActionCode` attributes; it stores `SecurityActionCode` values)* |
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

Ids and collections (value domains, no enum): `SecurityUserId`, `SecurityRoleId`, `SecurityDomainId`, `SecurityUserCode` (Char(100) `@!`), `SecurityUserQualifiedCode`, `SecurityPartyIdCollection`, `SecurityObjectCollection`, `SecurityFunctions`; plus the passwords `PswEnc` (Char(128)), `PswIng` (Char(64)), `EncKey` (Char(32)).

## 6. The authorization mechanism

The objects the patterns generate invoke these `SecurityConnector` procedures. Both share a structure and the **default-allow policy**.

### 6.1 `PIsAuthorized` — access to **screens** (objects)
`parm(in: &GxObject, out: &Authorized)`

1. `&GxObject = 'HNotAuthorized'` → **True** (the "not authorized" screen is always reachable).
2. Load the context (`PLoadContext`): `&SecurityUserId`, `&SecurityDomainId`, `&UserData = RetUserDataFromCode(&Context.SecurityUserQualifiedCode)`.
3. **Admin bypass**: if `&UserData.IsAdministrator` → **True**.
4. **ApplicationBlocked**: if the `SystemParameterCode.ApplicationBlocked` preference is on → **False** (a global lock, admins aside).
5. `For Each` over **`SecurityObjectAccess`** `Where SystemObjectName = &GxObject` and `ActionCode = Access OR Full` (see §6.3 for the user → role → domain escalation).
6. **`When None` → True**: if the object has no ACL rows at all, access is granted (**default-allow**).

### 6.2 `PIsEnabled` — access to a screen's **actions**
`parm(in: &GxObject, in: &Function, out: &Authorized)`

The same structure and default-allow, with two differences:
1. It receives `&Function` (the action code, e.g. `INS`/`UPD`/`DLT`/`DSP`) and filters `Where ActionCode = &Function OR Full` → it controls the **function/action level** inside the object, not overall access.
2. It has **neither** the `HNotAuthorized` case nor the `ApplicationBlocked` gate.

`PCheckSystemAccess` is a wrapper: `PIsAuthorized('hhome')` (access to the system's home).

### 6.3 Escalation user → role → domain role

Inside the `For Each` over `SecurityObjectAccess`, for each row (`&SecurityPartyId`, `SecurityPartyType`):
- **User** (`SecurityPartyType = User`): authorizes if `SecurityPartyId = &SecurityUserId`.
- **Role** (else): it looks in **`SecurityUserRoles`** through the `SecurityUserRolesEnabled(&SecurityUserId, &SecurityPartyId)` DataSelector — does the user hold that (active) role? If so → authorized.
- `When None`: if there is a domain, it looks in **`SecurityDomainRoles`** through `SecurityDomainRolesEnabled(&SecurityDomainId, &UserData.Type, &SecurityPartyId)` — roles inherited from the domain by user type (FrontEnd/BackOffice). If it applies → authorized.

**DataSelectors:**
- `SecurityUserRolesEnabled` — the active `SecurityUserRoles` assignments: `SecurityUserDisabled = False`, the user's domain not disabled, `SecurityUserId = &SecurityUserId`, and `SecurityRoleId = &SecurityRoleId` **only when the role is not empty** (0/empty = "every role of the user").
- `SecurityDomainRolesEnabled` — `SecurityDomainRoles` with `SecurityDomainDisabled = False`, filtering by domain, role and user type.

### 6.4 Admin bypass and ApplicationBlocked
- **Admin**: `SecurityRoles.SecurityRoleAdmin = True`. It is computed by `PIsAdministratorFromCode` (which looks at the user's roles in the Main domain, or the domain's roles by type) and lands in `UserData.IsAdministrator`. An admin passes **every** check.
- **ApplicationBlocked**: a global preference (`SystemParameterCode.ApplicationBlocked`) blocking access for non-admins (maintenance mode).

### 6.5 The default-allow policy (key)

`When None → True` means **"anything unconfigured is public"**. Consequences:
- A new screen/action has **no** restrictions until somebody creates rows in `SecurityObjectAccess`.
- Restricting a resource means **explicitly defining** who may use it (users/roles); as soon as **one** ACL row exists for that object, it stops being public (only the listed parties — directly or through a role/domain — get in).

## 7. How to secure a screen or an action (how-to)

1. **The object catalogues itself.** When they run, the generated objects register the `SystemObjectName` in `TSystemObjects` (through `PAddSecurityContext`). There is nothing to create by hand.
2. **While there is no ACL, it is public** (default-allow, §6.5).
3. **To restrict it**, create rows in **`SecurityObjectAccess`** with:
   - `SystemObjectName` = the object (the screen, or the one running the action),
   - `SecurityObjectAccessActionCode` = `Access`/`Full` (screen) or `INS`/`UPD`/`DLT`/`DSP`/`Full` (action),
   - `SecurityPartyId` = the **user or role** being allowed.
4. **Assign users to roles** in `SecurityUserRoles` (and/or domain roles in `SecurityDomainRoles`) so that a permission granted to a role reaches its users.
5. **Admins** (`SecurityRoleAdmin`) always pass.

> **Relationship with the generated UI:** the patterns generate, in the screen's Start, a `PIsAuthorized(<targetObject>)` that **hides** the button of a navigating action when the user is not authorized for the target (see [12-pattern-ui-actions.md](../12-pattern-ui-actions.md) §10). Because the model is default-allow, a **new** target (with no ACL) returns `True` and the button **is visible to everyone**; it only hides once that target is restricted and the user does not qualify.

## 8. Authentication: login, registration and silent sign-on

**Login** — `HLogin.WebPanel` (a main page) delegates to `WbLogin` (the `PXParameterRequestLogin` instance). Credential validation:
- `PCheckUser` — resolves the domain by name and verifies `SecurityUserCode` + `SecurityUserDomainId` + `SecurityUserPsw` (encrypted) against enabled users.
- `PCheckUserWebSessionEnabled` — additionally requires `not SecurityUserWebSessionDisabled`.
- `AuthenticateWindowsUser` — when `AuthenticationMode = windows`, it takes the user from `Thread.CurrentPrincipal` (integrated Windows/LDAP SSO) and sets the context.
- `CheckWebSession` / `SaveSecurityUserWebSessionId` — web session control; if `CheckUniqueSessionPerUser` is on, it enforces **one session per user** (comparing `UserData.WebSessionId` with `&WebSession.Id`).

**Registration** — the `Registration` transaction: it creates a pending user with a `RegistrationConfirmCode` and a `RegistrationStatus` (Created → Registered → Confirmed → Verificated); on confirmation the `SecurityUsers` record is created and linked through `RegistrationUserId`. Instances: `PXParameterRequestRegistrationBasic`, `PXParameterRequestRegistrationConfirmed`.

**Silent Sign-On (SSO)** — the `SilentSignOnRequests` transaction (a token/`AuthorizationId` with an `UpTo` expiry and a Pending/Succeed/Fail `Status`). `CheckUserSilentSignOnEnabled` returns the `SecurityUserId` if the user has `SecurityUserSilentSignOnEnabled` plus an enabled web session.

## 9. The module's CRUD instances

| Type | Instances |
|---|---|
| **PXWorkWith** | `PXWorkWithSecurityUsers`, `PXWorkWithSecurityUsersDomains`, `PXWorkWithSecurityRoles`, `PXWorkWithSecurityDomains`, `PXWorkWithSecurityObjectAccess`, `PXWorkWithSecurityObjectRecordsAccess`, `PXWorkWithSystemObjects`, `PXWorkWithRegistrations`, `PXWorkWithSilentSignOnRequests` |
| **PXParameterRequest** | `PXParameterRequestChangePassword`, `PXParameterRequestRegistrationBasic`, `PXParameterRequestRegistrationConfirmed` |
| **PXComposer** | `PXComposerSecurityObjectAccess`, `PXComposerSecurityObjectRecordAccess` |

## 10. Key `SecurityConnector` procedures

| Procedure | Purpose |
|---|---|
| `PIsAuthorized` | Access control for **screens** (§6.1). |
| `PIsEnabled` | Access control for a screen's **actions/functions** (§6.2). |
| `PCheckSystemAccess` | `PIsAuthorized('hhome')` — access to the home. |
| `PIsAdministrator` / `PIsAdministratorFromCode` | Computes the admin flag from the roles (`SecurityRoleAdmin`). |
| `RetUserDataFromCode` / `RetUserDataFromId` | Build the `UserData` SDT (including `IsAdministrator`, type, domain, email, SSO, web session). |
| `PCheckUser` / `PCheckUserWebSessionEnabled` | Credential validation (login). |
| `AuthenticateWindowsUser` | Integrated Windows/LDAP authentication. |
| `CheckWebSession` / `SaveSecurityUserWebSessionId` | Web session (optionally unique per user). |
| `CheckUserSilentSignOnEnabled` | Silent sign-on enablement. |
| `PAddSecurityContext` | Registers the object (name, functions, module) in the session's `SecurityContext` and in `TSystemObjects`. |
| `RetUserDomainFromQualifiedCode` / `RetDomainFromName` | Resolve `code@domain` / a domain by name. |
| `CtSecurityUsersExists` | Is there at least one user? (bootstrap / first run). |

## 11. Integration with the context (`&Context`)

`PLoadContext.Call(&Context)` (injected by the patterns) loads the session `Context`, the source of the data used across the app — including the **business attributes** the application may have embedded in `SecurityUsers` (the user's entity / tenant, for instance). That is why FrontEnd screens load such data from the context (the multi-tenancy rule) instead of receiving it as a parameter.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [12-pattern-ui-actions.md](../12-pattern-ui-actions.md) §10 — action visibility gating through `PIsAuthorized`.
- `@PXTools/@System/APIs/TSystemObjects.Transaction.gxSource` — the system object catalogue.
