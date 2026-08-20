# @System Module — System Object Catalogue

> Behaviour of the `@PXTools/@System` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@System/`
  - `APIs/` — core (catalogue transactions + registration APIs). Not to be touched.
  - `Personalized/` — the "seed" procedures that get regenerated per project.
- Qualifier: `PXTools.System`.
- **Depends on:** `@APIs` (base). It is **shared infrastructure**: other modules (@Security, @OAV, @ControlPreferences) depend on @System, not the other way round.

## 1. What it provides

A core module maintaining the **internal catalogue of system objects and modules**: a persisted registry of the GeneXus objects (transactions, web panels, procedures, tables) grouped by module. That catalogue is the **"parent" table** other framework modules hang their per-object data from: **@Security** (permissions per object/action/party), **@OAV** (dynamic attributes/classes per object) and **@ControlPreferences** (control preferences per object) all reference `SystemObjectName`.

## 2. Core concept: `TSystemObjects` as the root table

`TSystemObjects` is the central catalogue of "system objects" and the **root** of several subsystems: any feature needing to hang data "per GeneXus object" does so with an FK to `SystemObjectName`.

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

## 3. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **TSystemObjects** (BC) | `SystemObjectName` (domain `ObjectName, GeneXus`) | **Object catalogue.** Attributes: `SystemObjectType` (`ObjectType`, restricted to `T/H/D`), `SystemObjectDescription`, `SystemObjectParent` (hierarchical self-FK, through the `SystemObjectParent` Group), `SystemObjectUTL` (Boolean), `SystemObjectVersionChequed`, `SystemObjectOAVDeclaration` (Boolean, marks participation in OAV), `SystemModuleName` (FK to `TSystemModules`). Physical table `#Tables/SystemObjects`. |
| **TSystemModules** (BC) | `SystemModuleName` (`ObjectName`) | Module catalogue. `SystemModuleDescription`. |

Supporting objects:
- **SDT `SystemObject`** (`APIs/SystemObject.StructuredDataType.gxSource`) — the input contract of `AddSystemObject`: `Name`, `Type`, `Description`, `Parent`, `UTL`, `OAVDeclaration`, `ModuleName`.
- **Group `SystemObjectParent`** — the subtype `SystemObjectParent : SystemObjectName` (resolves the parent/child self-relationship).

## 4. Module domains

**Its own (root-legacy — it lives in the root `#Domains/` because it predates per-module domains):**
- **`ObjectType`** (`Character(1)`): `Transaction=T`, `WebPanel=H`, `DataBase=D`, `Procedure=P` (the transactions restrict the ValueRange to `T/H/D`). It belongs to @System (its `TSystemObjects` defines it), even though @APIs and @Security read it too.

**Used from @APIs base / GeneXus:** `ObjectName` / `ObjectDescription` / `Version` (the `GeneXus` namespace), `Links`, `MaxMem`, `Window*`.

## 5. The registration mechanism

### Low-level APIs (`APIs/`, idempotent)
| Proc | `Parm()` | What it does |
|---|---|---|
| `AddSystemModule` | `in: &SystemModuleName` | Idempotent module creation (`For Each … When None → New`); the description is the name. |
| `AddSystemObject` | `in: &SystemObject` | Idempotent object creation (it calls `AddSystemModule` first); `CommitOnExit='No'`. |
| `DelSystemObject` / `DelSystemObjects` | `in: &SystemObjectName` / — | Delete one object / purge everything. |
| `DelSystemModules` | — | Purge all modules. |
| `CheckSystemModules` | — | **Lazy bootstrap**: if there are 0 modules, it invokes `SaveSystemModules`. |

### "Seed" procedures (`Personalized/`)
- **`SaveSystemModules`** — rebuilds the module catalogue (`DelSystemModules` + N `AddSystemModule` — the module list is project-specific and gets regenerated as modules are added). ⚠️ **Adding the module here is a prerequisite for its menus to be seeded**: `AddMenusRecursive` discards any item whose `Module` is not in the catalogue, and `AddDefaultMenus` `RollBack`s the entire run. See [`menus.md`](menus.md) → *The module has to be in the `SystemModules` catalogue*.
- **`SaveSystemObjects`** — the registration point for concrete objects (customized/regenerated per project).

### Who populates the catalogue
- The **name** of each object (`SystemObjectName`) comes from the object itself: its generated events set `&SecurityObjectStructure.Name = &Pgmname` and call `PAddSecurityContext` (which lives in `@APIs/Personalized/SecurityConnector`, **not** in @System, and only builds the in-memory `SecurityContext` — it does not write the catalogue).
- The **persistence** is done by `AddSystemObject` / `SaveSystemObjects` and by consumers such as `@OAV` (`SaveOAVSystemObjects`, which sets `OAVDeclaration`) and `@ProcessMonitor` (`StartProcessStatus` → `AddSystemObject`).

## 6. APIs vs Personalized

- **`APIs/`** (core): the catalogue transactions (`TSystemObjects`, `TSystemModules`), the `SystemObject` SDT, the Group, and the `Add*/Del*/CheckSystemModules` APIs.
- **`Personalized/`** (regenerated per project): `SaveSystemModules` (the project's module list) and `SaveSystemObjects` (object registration), each with its triggering PXParameterRequest instance.

## 7. Pattern instances

| Instance | What it is |
|---|---|
| **PXParameterRequestSaveSystemModules** | Web panel (`WbSaveSystemModules`) with a "Save" button → `Call SaveSystemModules`. |
| **PXParameterRequestSaveSystemObjects** | Web panel (`WbSaveSystemObjects`) with a "Save" button → `Call SaveSystemObjects`. |

> The `TSystemObjects`/`TSystemModules` transactions do use the **PXWorkWith** pattern, but their generated objects live outside the module (under `@Security/APIs/Objects/`).

## 8. Key procedures / APIs

| Proc | `Parm()` | Purpose |
|---|---|---|
| `AddSystemObject` | `in: &SystemObject` | Idempotent creation of an object (also creates its module). |
| `AddSystemModule` | `in: &SystemModuleName` | Idempotent creation of a module. |
| `DelSystemObject` | `in: &SystemObjectName` | Deletes an object. |
| `CheckSystemModules` | — | Bootstraps the module catalogue. |
| `SaveSystemModules` / `SaveSystemObjects` | — | Rebuilds the catalogue (modules / objects). |

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [security.md](security.md) — `SecurityObjectAccess` (and row-level security) are FKs to `TSystemObjects`; the ACL hangs off the catalogue.
- The **@OAV** and **@ControlPreferences** modules — they also hang off `SystemObjectName`.
- `modules/apis.md` — `PAddSecurityContext` (SecurityConnector), which contributes the object's name to the security context.
