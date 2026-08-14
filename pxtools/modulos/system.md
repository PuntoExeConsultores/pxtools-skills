# Módulo @System — Catálogo de Objetos del Sistema

> Comportamiento del módulo `@PXTools/@System`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@System/`
  - `APIs/` — core (transacciones catálogo + APIs de registro). Intocable.
  - `Personalized/` — los procs de "seed" que se regeneran por proyecto.
- Cualificador: `PXTools.System`.
- **Depende de:** `@APIs` (base). Es **infraestructura compartida**: otros módulos (@Security, @OAV, @ControlPreferences) dependen de @System, no al revés.

## 1. Qué provee

Un módulo núcleo que mantiene el **catálogo interno de objetos y módulos del sistema**: un registro persistido de los objetos GeneXus (transacciones, web panels, procedures, tablas) agrupados por módulo. Ese catálogo es la **tabla "padre"** sobre la que otros módulos del framework cuelgan sus datos por objeto: **@Security** (permisos por objeto/acción/party), **@OAV** (atributos/clases dinámicos por objeto) y **@ControlPreferences** (preferencias de control por objeto) referencian todos a `SystemObjectName`.

## 2. Concepto central: `TSystemObjects` como tabla raíz

`TSystemObjects` es el catálogo central de "objetos del sistema" y la **raíz** de varios subsistemas: cualquier funcionalidad que necesite colgar datos "por objeto GeneXus" lo hace con una FK a `SystemObjectName`.

```
                 TSystemModules (PK SystemModuleName)
                        ▲ (SystemModuleName)
                        │
                 TSystemObjects  (PK SystemObjectName · self-FK SystemObjectParent)
                        ▲
     ┌──────────────────┼───────────────────┬─────────────────────┐
 SecurityObjectAccess   SystemObjectOAV*    ControlPreferences   (…)
 (@Security)            (@OAV)              (@ControlPreferences)
```

## 3. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **TSystemObjects** (BC) | `SystemObjectName` (dom. `ObjectName, GeneXus`) | **Catálogo de objetos.** Atributos: `SystemObjectType` (`ObjectType`, restringido a `T/H/D`), `SystemObjectDescription`, `SystemObjectParent` (self-FK jerárquica, vía Group `SystemObjectParent`), `SystemObjectUTL` (Boolean), `SystemObjectVersionChequed`, `SystemObjectOAVDeclaration` (Boolean, marca participación en OAV), `SystemModuleName` (FK a `TSystemModules`). Tabla física `#Tables/SystemObjects`. |
| **TSystemModules** (BC) | `SystemModuleName` (`ObjectName`) | Catálogo de módulos. `SystemModuleDescription`. |

Objetos auxiliares:
- **SDT `SystemObject`** (`APIs/SystemObject.StructuredDataType.gxSource`) — contrato de entrada de `AddSystemObject`: `Name`, `Type`, `Description`, `Parent`, `UTL`, `OAVDeclaration`, `ModuleName`.
- **Group `SystemObjectParent`** — subtipo `SystemObjectParent : SystemObjectName` (resuelve la auto-relación padre/hijo).

## 4. Dominios del módulo

**Propio (root-legacy — vive en `#Domains/` raíz por ser previo a los dominios por módulo):**
- **`ObjectType`** (`Character(1)`): `Transaction=T`, `WebPanel=H`, `DataBase=D`, `Procedure=P` (las transacciones restringen el ValueRange a `T/H/D`). Es de @System (lo define su `TSystemObjects`), aunque también lo leen @APIs y @Security.

**Usa de @APIs base / GeneXus:** `ObjectName` / `ObjectDescription` / `Version` (namespace `GeneXus`), `Links`, `MaxMem`, `Window*`.

## 5. Mecanismo de registro

### APIs de bajo nivel (`APIs/`, idempotentes)
| Proc | `Parm()` | Qué hace |
|---|---|---|
| `AddSystemModule` | `in: &SystemModuleName` | Alta idempotente del módulo (`For Each … When None → New`); descripción = el nombre. |
| `AddSystemObject` | `in: &SystemObject` | Alta idempotente del objeto (llama antes a `AddSystemModule`); `CommitOnExit='No'`. |
| `DelSystemObject` / `DelSystemObjects` | `in: &SystemObjectName` / — | Baja de un objeto / purga total. |
| `DelSystemModules` | — | Purga total de módulos. |
| `CheckSystemModules` | — | **Bootstrap perezoso**: si hay 0 módulos, invoca `SaveSystemModules`. |

### Procs de "seed" (`Personalized/`)
- **`SaveSystemModules`** — reconstruye el catálogo de módulos (`DelSystemModules` + N `AddSystemModule` — la lista de módulos es específica del proyecto, se regenera al agregar módulos). ⚠️ **Agregar acá el módulo es requisito para que sus menús se siembren**: `AddMenusRecursive` descarta el ítem cuyo `Module` no esté en el catálogo y `AddDefaultMenus` hace `RollBack` de toda la corrida. Ver [`menus.md`](menus.md) → *El módulo tiene que estar en el catálogo de `SystemModules`*.
- **`SaveSystemObjects`** — punto de registro de objetos concretos (se personaliza/regenera por proyecto).

### Quién puebla el catálogo
- El **nombre** de cada objeto (`SystemObjectName`) lo aporta el propio objeto: sus eventos generados setean `&SecurityObjectStructure.Name = &Pgmname` y llaman `PAddSecurityContext` (que vive en `@APIs/Personalized/SecurityConnector`, **no** en @System, y solo arma el `SecurityContext` en memoria — no escribe el catálogo).
- La **persistencia** la hacen `AddSystemObject` / `SaveSystemObjects` y consumidores como `@OAV` (`SaveOAVSystemObjects`, que marca `OAVDeclaration`) y `@ProcessMonitor` (`StartProcessStatus` → `AddSystemObject`).

## 6. APIs vs Personalized

- **`APIs/`** (core): las transacciones catálogo (`TSystemObjects`, `TSystemModules`), el SDT `SystemObject`, el Group, y las APIs `Add*/Del*/CheckSystemModules`.
- **`Personalized/`** (regenerable por proyecto): `SaveSystemModules` (lista de módulos del proyecto) y `SaveSystemObjects` (registro de objetos), cada uno con su instancia PXParameterRequest disparadora.

## 7. Instancias de patterns

| Instancia | Qué es |
|---|---|
| **PXParameterRequestSaveSystemModules** | Web panel (`WbSaveSystemModules`) con botón "Save" → `Call SaveSystemModules`. |
| **PXParameterRequestSaveSystemObjects** | Web panel (`WbSaveSystemObjects`) con botón "Save" → `Call SaveSystemObjects`. |

> Las transacciones `TSystemObjects`/`TSystemModules` usan el pattern **PXWorkWith**, pero sus objetos generados residen fuera del módulo (bajo `@Security/APIs/Objects/`).

## 8. Procedimientos / APIs clave

| Proc | `Parm()` | Propósito |
|---|---|---|
| `AddSystemObject` | `in: &SystemObject` | Alta idempotente de un objeto (crea también su módulo). |
| `AddSystemModule` | `in: &SystemModuleName` | Alta idempotente de un módulo. |
| `DelSystemObject` | `in: &SystemObjectName` | Baja de un objeto. |
| `CheckSystemModules` | — | Bootstrap del catálogo de módulos. |
| `SaveSystemModules` / `SaveSystemObjects` | — | Reconstrucción del catálogo (módulos / objetos). |

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [security.md](security.md) — `SecurityObjectAccess` (y row-level) son FK a `TSystemObjects`; la ACL se cuelga del catálogo.
- Módulos **@OAV** y **@ControlPreferences** — también cuelgan de `SystemObjectName`.
- `modulos/apis.md` — `PAddSecurityContext` (SecurityConnector) que aporta el nombre del objeto al contexto de seguridad.
