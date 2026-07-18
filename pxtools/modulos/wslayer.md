# Módulo @WSLayer — Capa de Web Services

> Comportamiento del módulo `@PXTools/@WSLayer`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@WSLayer` (`APIs/` core + `Personalized/`).
- Patterns de generación: `Patterns/PXWS*` (PXWSLayer / PXWSQuery / PXWSData / PXWSTransaction).
- Cualificador: `PXTools.WSLayer`.
- **Depende de:** `@APIs` (base). Infra: `@Menus`.

## 1. Qué provee

Dos cosas complementarias:
1. **Framework de generación por patterns** (`PXWS*`): expone transacciones/consultas como servicios **SOAP / REST / API**, versionados, con envelope estándar y OpenAPI opcional.
2. **Runtime**: **control de acceso por whitelist de IPs** (por categoría), **envelope de conexión/respuesta** (credenciales de entrada + `Succeed`/`Response`/`Error`) y **conversión de mensajes** GeneXus → formato de la API.

El logging de invocaciones es **ortogonal** y vive en el módulo hermano **@WebServicesLog**.

## 2. Concepto central: whitelist por categoría

`WSWhiteList` guarda, **por categoría**, reglas **Accept** (lista blanca) o **Deny** (lista negra) con un rango **CIDR**. La semántica:
- Si la categoría tiene **alguna** regla Accept → es **whitelist estricta**: el IP debe matchear un Accept y no matchear un Deny.
- Si **no** tiene reglas Accept → funciona como **blacklist**: permite todo salvo lo denegado.

## 3. Transacción `WSWhiteList`

Un solo nivel; BC.

| Atributo | Tipo | Notas |
|---|---|---|
| `WSWhiteListId` (PK) | `IdFirstLevel` | autonum |
| `WSWhiteListCategory` | `PXToolsWSLayerCategory` | agrupa reglas por categoría de servicio |
| `WSWhiteListAcceptDeny` | `PXToolsWSLayerWhiteListAcceptDeny` | ACC / DEN |
| `WSWhiteListIPVersion` | `PXToolsWSLayerWhiteListIPVersion` | 4 / 6 |
| `WSWhiteListIPRange` | `MaxStr` | rango CIDR (IPv4/IPv6) |

Índices: `IWSWhiteList` (unique, Id), `UWSWhiteList` (duplicate, Category+Id).

## 4. Dominios del módulo

Todos `PXToolsWSLayer*` → @WSLayer (nombre de módulo tras `PXTools`), **root-legacy** (viven en el `#Domains/` raíz). Excepción: `WhiteListReason` vive en el `#Domains/` de paquete pero lo usa **solo** @WSLayer → también es de @WSLayer:

| Dominio | Valores |
|---|---|
| **PXToolsWSLayerWhiteListAcceptDeny** | Accept=`ACC`, Deny=`DEN` |
| **PXToolsWSLayerWhiteListIPVersion** | IPv4=`4`, IPv6=`6` |
| **PXToolsWSLayerMessageType** | Warning, Error, Info, Debug |
| **PXToolsWSLayerConnectionErrorCode** | catálogo de errores de autenticación de conexión (código/clave de empresa/usuario inválidos, sin acceso, sin suscripción, acceso denegado, …) |
| **PXToolsWSLayerConnectionMessage** | Character(100) — mensaje de conexión |
| **PXToolsWSLayerErrorCodes** | RecordDoesNotExists, InvalidDelete, SaveError, InsertIsNotAllowed, UpdateIsNotAllowed (+ `…LoadErrorCode`/`…SaveErrorCode`) |
| **PXToolsWSLayerCategory** | enum de **categorías de servicio** — la clave de agrupación de reglas de whitelist / control de acceso por API (los valores concretos los define cada aplicación). |
| **WhiteListReason** (`@PXTools/#Domains/`) | Denied, NotAccepted, NotDenied, Accepted |

## 5. Mecanismo

### 5.1 Generación (patterns `PXWS*`)
Cuatro patterns cuyo `ParentObject` es una Transaction:
- **PXWSTransaction** → métodos CRUD por versión (`WSTransaction<Obj>V<n>Load/Save/Delete`) + SDT In/Out.
- **PXWSQuery** → consultas paginadas (`WSQuery<Obj>V<n>`, filtros Range/Like/Contains, paginación).
- **PXWSData** → servicios de datos (`WSData<Obj>V<n>Method` + SDT DataIn/DataOut).
- **PXWSLayer** → envuelve lo anterior como **SOAP** (`SOAP<Obj>`), **REST** (`REST<Obj>`) y/o **API** (`API<Obj>`), versionado, con OpenAPI opcional.

Los `Patterns/PXWSLayer/PXWSLayerSettings.xml` definen la convención de nombres y el **envelope**: cada request lleva un nivel `Connection`; cada response un envelope `Succeed` + `Response` + `Error{Code, Message, Detail}`. Soporta atributos multi-tenant desde el `SDTConnection`, y banderas para **generar el proc de seguridad** (`ChkWSSecurity`) y para **generar logging** (`GenerateWebServiceLog`).

> En esta KB los patterns `PXWS*` están **instalados pero sin instancias aplicadas**; por eso no hay procs `ChkWSSecurity` generados y los endpoints reutilizan directamente la API de whitelist (§5.3).

### 5.2 Envelope de conexión/respuesta (runtime)
- `SDTConnection`: credenciales de entrada (códigos/claves de desarrollador/empresa/usuario/rol).
- `SDTConnectionResponse`: `Succeed` + `Error{Code (PXToolsWSLayerConnectionErrorCode), Message}`.
- `MessageDetail` + `RetMessageFromGX`/`RetMessageTypeFromGX` — convierten `Messages, GeneXus.Common` al formato de mensajes de la API.

### 5.3 Whitelist — `ValWSWhitelistIPWithReason`
`(in: &Category, in: &IP, out: &Result: SDTWhiteListResult{IsValid, Reason})`:
1. Recorre reglas **ACC** de la categoría; si el IP cae en un rango → `IsValid=True, Reason=Accepted`.
2. Si hubo accept **o no hay ninguna regla ACC** (modo abierto): recorre **DEN**; en rango → `IsValid=False, Reason=Denied`; si no → `IsValid=True, Reason=NotDenied`.
3. Si hay reglas ACC pero ninguna matcheó → `IsValid=False, Reason=NotAccepted`.

Apoyo: `ValWSWhiteListIPInRange` (contención IP en CIDR vía ExternalObject `IPMaskValidation` — Java, en `@APIs/APIs/Network/`; **fail-open** ante error de validación), `ValWSWhiteListIPRange` (CIDR bien formado), `RetWSWhiteListIPVersion` (detecta v4/v6).

## 6. APIs vs Personalized

- **`APIs/`** (core): `WSWhiteList`, la familia `Val*`/`RetWSWhiteList*`, `AddWSWhiteListSDT`, el envelope (`SDTConnection`/`SDTConnectionResponse`) y la conversión de mensajes.
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | `RetMenusWSLayer` (DataProvider) | Entrada de menú "White List". |
  | `RetWhiteListDenyReason` (Procedure) | Texto legible del motivo de rechazo (localizable). |

## 7. Instancias de patterns

- **PXWorkWithWSWhiteList** — WW de las reglas de whitelist (filtro por categoría, orden Category+Id); genera `TrWSWhiteList`/`CtWSWhiteList`.
- Los patterns `PXWSLayer/PXWSQuery/PXWSData/PXWSTransaction` solo tienen definición en `Patterns/` (sin instancias aplicadas en esta KB).

## 8. Procedimientos / APIs clave

**Whitelist**: `ValWSWhitelistIPWithReason(in: &Category, in: &IP, out: &SDTWhiteListResult)` (central), `ValWSWhiteListIP(… out: &IsValid)` (wrapper booleano), `ValWSWhiteListIPInRange`, `ValWSWhiteListIPRange`, `RetWSWhiteListIPVersion`, `AddWSWhiteListSDT(in: &SDTWhiteList, out…)` *(stub actualmente)*.

**Mensajes**: `RetMessageFromGX`, `RetMessageTypeFromGX`.

**SDTs**: `SDTConnection`, `SDTConnectionResponse`, `SDTWhiteList` (Category/AcceptDeny/IPVersion/IPRange), `SDTWhiteListResult` (IsValid/Reason), `MessageDetail`.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [webserviceslog.md](webserviceslog.md) — logging de invocaciones (ortogonal a esta capa).
- `30-guia-reconocimiento-patterns.md` y los docs de patterns `PXWS*` (si se documentan aparte) para el detalle de la generación SOAP/REST/API.
- `modulos/apis.md` — el ExternalObject `IPMaskValidation` (Network) usado por la whitelist.
