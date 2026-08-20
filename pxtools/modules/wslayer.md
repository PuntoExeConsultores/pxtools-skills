# @WSLayer Module — The Web Services Layer

> Behaviour of the `@PXTools/@WSLayer` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@WSLayer` (`APIs/` core + `Personalized/`).
- Generation patterns: `Patterns/PXWS*` (PXWSLayer / PXWSQuery / PXWSData / PXWSTransaction).
- Qualifier: `PXTools.WSLayer`.
- **Depends on:** `@APIs` (base). Infrastructure: `@Menus`.

## 1. What it provides

Two complementary things:
1. **A pattern-based generation framework** (`PXWS*`): exposes transactions/queries as **SOAP / REST / API** services, versioned, with a standard envelope and optional OpenAPI.
2. **Runtime**: **access control by IP whitelist** (per category), the **connection/response envelope** (inbound credentials + `Succeed`/`Response`/`Error`) and **message conversion** from GeneXus to the API's format.

Logging of invocations is **orthogonal** and lives in the sibling module **@WebServicesLog**.

## 2. Core concept: whitelist per category

`WSWhiteList` stores, **per category**, **Accept** (allow list) or **Deny** (block list) rules with a **CIDR** range. The semantics:
- If the category has **any** Accept rule → it is a **strict whitelist**: the IP must match an Accept and must not match a Deny.
- If it has **no** Accept rules → it behaves as a **blacklist**: everything is allowed except what is denied.

## 3. The `WSWhiteList` transaction

A single level; a BC.

| Attribute | Type | Notes |
|---|---|---|
| `WSWhiteListId` (PK) | `IdFirstLevel` | autonumbered |
| `WSWhiteListCategory` | `PXToolsWSLayerCategory` | groups rules by service category |
| `WSWhiteListAcceptDeny` | `PXToolsWSLayerWhiteListAcceptDeny` | ACC / DEN |
| `WSWhiteListIPVersion` | `PXToolsWSLayerWhiteListIPVersion` | 4 / 6 |
| `WSWhiteListIPRange` | `MaxStr` | CIDR range (IPv4/IPv6) |

Indexes: `IWSWhiteList` (unique, Id), `UWSWhiteList` (duplicate, Category+Id).

## 4. Module domains

All `PXToolsWSLayer*` domains belong to @WSLayer (the module name follows `PXTools`), all **root-legacy** (they live in the root `#Domains/`). One exception: `WhiteListReason` lives in the package's `#Domains/` but is used **only** by @WSLayer → it belongs to @WSLayer as well:

| Domain | Values |
|---|---|
| **PXToolsWSLayerWhiteListAcceptDeny** | Accept=`ACC`, Deny=`DEN` |
| **PXToolsWSLayerWhiteListIPVersion** | IPv4=`4`, IPv6=`6` |
| **PXToolsWSLayerMessageType** | Warning, Error, Info, Debug |
| **PXToolsWSLayerConnectionErrorCode** | Catalogue of connection authentication errors (invalid code/company key/user, no access, no subscription, access denied, …) |
| **PXToolsWSLayerConnectionMessage** | Character(100) — connection message |
| **PXToolsWSLayerErrorCodes** | RecordDoesNotExists, InvalidDelete, SaveError, InsertIsNotAllowed, UpdateIsNotAllowed (+ `…LoadErrorCode`/`…SaveErrorCode`) |
| **PXToolsWSLayerCategory** | An enum of **service categories** — the grouping key for whitelist rules / per-API access control (each application defines the concrete values). |
| **WhiteListReason** (`@PXTools/#Domains/`) | Denied, NotAccepted, NotDenied, Accepted |

## 5. How it works

### 5.1 Generation (the `PXWS*` patterns)
Four patterns whose `ParentObject` is a Transaction:
- **PXWSTransaction** → CRUD methods per version (`WSTransaction<Obj>V<n>Load/Save/Delete`) + In/Out SDTs.
- **PXWSQuery** → paged queries (`WSQuery<Obj>V<n>`, Range/Like/Contains filters, paging).
- **PXWSData** → data services (`WSData<Obj>V<n>Method` + DataIn/DataOut SDTs).
- **PXWSLayer** → wraps all of the above as **SOAP** (`SOAP<Obj>`), **REST** (`REST<Obj>`) and/or **API** (`API<Obj>`), versioned, with optional OpenAPI.

`Patterns/PXWSLayer/PXWSLayerSettings.xml` defines the naming convention and the **envelope**: every request carries a `Connection` level; every response an envelope of `Succeed` + `Response` + `Error{Code, Message, Detail}`. It supports multi-tenant attributes from `SDTConnection`, plus flags to **generate the security procedure** (`ChkWSSecurity`) and to **generate logging** (`GenerateWebServiceLog`).

> In this KB the `PXWS*` patterns are **installed but with no instances applied**; that is why there are no generated `ChkWSSecurity` procedures and the endpoints use the whitelist API directly (§5.3).

### 5.2 Connection/response envelope (runtime)
- `SDTConnection`: inbound credentials (developer/company/user/role codes and keys).
- `SDTConnectionResponse`: `Succeed` + `Error{Code (PXToolsWSLayerConnectionErrorCode), Message}`.
- `MessageDetail` + `RetMessageFromGX`/`RetMessageTypeFromGX` — they convert `Messages, GeneXus.Common` into the API's message format.

### 5.3 Whitelist — `ValWSWhitelistIPWithReason`
`(in: &Category, in: &IP, out: &Result: SDTWhiteListResult{IsValid, Reason})`:
1. It walks the category's **ACC** rules; if the IP falls inside a range → `IsValid=True, Reason=Accepted`.
2. If there was an accept **or there are no ACC rules at all** (open mode): it walks the **DEN** rules; inside a range → `IsValid=False, Reason=Denied`; otherwise → `IsValid=True, Reason=NotDenied`.
3. If there are ACC rules but none matched → `IsValid=False, Reason=NotAccepted`.

Supporting procedures: `ValWSWhiteListIPInRange` (IP containment in a CIDR through the `IPMaskValidation` ExternalObject — Java, in `@APIs/APIs/Network/`; **fail-open** if validation errors), `ValWSWhiteListIPRange` (well-formed CIDR), `RetWSWhiteListIPVersion` (detects v4/v6).

### 5.4 How a consumer integrates

The check is applied **at the point you want to protect**, with the category **hard-coded from the domain**. The IP is not a caller parameter: it is obtained right there.

```genexus
&RemoteAddress = RetHTTPRemoteAddress.Udp()
&IPAllowed     = ValWSWhiteListIP.Udp(PXToolsWSLayerCategory.<Category>, &RemoteAddress)
```

Adding a new consumer: **(1)** add the value to the `PXToolsWSLayerCategory` domain; **(2)** invoke `ValWSWhiteListIP` as early as possible in the endpoint — before parsing the body, because this is a **network** control and must not depend on the request being valid; **(3)** load the rules through the White List screen.

Turning the check on or off **requires no code change**: a category with no rows allows everything (§2), so the gate can be coded from day one and the rules loaded later.

> **Antipattern: an FK to `WSWhiteListId`.** Do not model a foreign key to the whitelist row in the entity you want to protect. A concept needs **N rows** (several Accept ranges coexisting with several Denies) and an FK points at only one; besides, the `Id` is an autonumber — different in every installation, impossible to hard-code — while the domain value is not. **The unit of the concept is the category**, which is why the whole API takes it as a parameter instead of an `Id`.

## 6. APIs vs Personalized

- **`APIs/`** (core): `WSWhiteList`, the `Val*`/`RetWSWhiteList*` family, `AddWSWhiteListSDT`, the envelope (`SDTConnection`/`SDTConnectionResponse`) and the message conversion.
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | `RetMenusWSLayer` (DataProvider) | The "White List" menu entry. |
  | `RetWhiteListDenyReason` (Procedure) | Human-readable text for the rejection reason (localizable). |

## 7. Pattern instances

- **PXWorkWithWSWhiteList** — the whitelist rules WW (filter by category, ordered by Category+Id); it generates `TrWSWhiteList`/`CtWSWhiteList`.
- The `PXWSLayer/PXWSQuery/PXWSData/PXWSTransaction` patterns only have their definition in `Patterns/` (no instances applied in this KB).

## 8. Key procedures / APIs

**Whitelist**: `ValWSWhitelistIPWithReason(in: &Category, in: &IP, out: &SDTWhiteListResult)` (the central one), `ValWSWhiteListIP(… out: &IsValid)` (boolean wrapper), `ValWSWhiteListIPInRange`, `ValWSWhiteListIPRange`, `RetWSWhiteListIPVersion`, `AddWSWhiteListSDT(in: &SDTWhiteList, out…)` *(currently a stub)*.

**Messages**: `RetMessageFromGX`, `RetMessageTypeFromGX`.

**SDTs**: `SDTConnection`, `SDTConnectionResponse`, `SDTWhiteList` (Category/AcceptDeny/IPVersion/IPRange), `SDTWhiteListResult` (IsValid/Reason), `MessageDetail`.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [webserviceslog.md](webserviceslog.md) — invocation logging (orthogonal to this layer).
- `30-pattern-recognition-guide.md` and the `PXWS*` pattern docs (where documented separately) for the detail of SOAP/REST/API generation.
- `modules/apis.md` — the `IPMaskValidation` ExternalObject (Network) used by the whitelist.
