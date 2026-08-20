# @DynamicCallReferences Module — Dynamic Invocation by Name

> Behaviour of the `@PXTools/@DynamicCallReferences` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@DynamicCallReferences` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.DynamicCallReferences`.
- **Depends on:** `@APIs` (base), `@Menus` (`RetMenusDynamicCallReferences`).

## 1. What it provides

A registry associating a **logical code** (`DynamicCallReferenceCode`) with the **runtime identity of a GeneXus object** (`Object.Type`, stored in `DynamicCallReferenceURL`). That way other modules invoke objects **"by name"** with `Call(&url, …)` **without hard-coding** which object they call. It is the cross-cutting piece decoupling "what code to run" from the concrete object — used by @TaskManager (`ExecutionCode`), @TableCleaner, @Alerts, @CloudTasks, and others.

## 2. Core concept: code → object

- The `TDynamicCallReferences` table maps `Code → URL` (where `URL = RealObject.Type`).
- At runtime: `&url = PPEXE_DeDynamicCallReferenceURL.Udp(code)` → `Call(&url, params)` (an **indirect/polymorphic** call resolved at runtime).
- The table is **seeded declaratively**: each module contributes a `RetDynamicCallReference<X>` DataProvider returning its references.

## 3. The `TDynamicCallReferences` transaction

A single level; a BC.

| Attribute | Type | Role |
|---|---|---|
| `DynamicCallReferenceCode` (PK) | `DynamicCallReferenceCode` (Char(50) enum) | Logical key of the "call point". |
| `DynamicCallReferenceType` | `ReferenceType` (Char(50) enum) | Category/subsystem (it defines which contract the `Call` expects). |
| `DynamicCallReferenceDescription` | `ObjectDescription` | Human-readable description. |
| `DynamicCallReferenceURL` | `Links` | Runtime identity of the target object (`Object.Type`). |

The mirror SDT `SDTDynamicCallReferences` (a collection of those 4 members) is the **seeding contract**.

## 4. Module domains

The three `DynamicCallReference*` domains plus `ReferenceType` belong to @DynamicCallReferences (by naming), all **root-legacy** (they live in the root `#Domains/`). Note: even though in this KB only @TaskManager consumes `DynamicCallReferenceParameters`, **it belongs to this module** by naming.

- **`DynamicCallReferenceCode`** (Char(50)): the **catalogue of logical "names"** of invocable objects (tasks, table cleaners, generators, DPs, OAV prompts…). Each module adds its values.
- **`ReferenceType`** (Char(50)): classifies **which subsystem** the reference serves (and therefore the expected signature). Framework values: `TaskManagerExecution` (`TskExecution`), `TaskManagerVisualization`, `TableCleanerProcess`, `StatisticLogProcess`, `StatisticIdCompositionDP`, `OAVAttributeValuesPrompt`, `OAVAttributeValueValidation`, `OAVSrvDynamicReadOnly`, `FormularioPDFPublico`/`…Privado`. The consumer filters the table by this type.
- **`DynamicCallReferenceParameters`** (Char(50)): a parameter value passed to a dynamically invoked object.

> The target object is stored as `Object.Type` (its qualified name) in `DynamicCallReferenceURL`: using the **qualified name** keeps the reference stable when the object is renamed.

## 5. How it works

### 5.1 Resolution — `PPEXE_DeDynamicCallReferenceURL`
`Parm(in: &DynamicCallReferenceId, out: &DynamicCallReferenceURL)` — `For Each Where Code = &Id → &URL = DynamicCallReferenceURL`. It returns the object's `.Type`.

### 5.2 Invocation
`&url` (of type `Attribute:DynamicCallReferenceURL`, domain `Links`) holds the `.Type`; `Call(&url, params)` is the indirect call. Examples:
- **@TaskManager** (`PrcTaskManagerExecution`): `&Process = PPEXE_DeDynamicCallReferenceURL.Udp(TaskManagerExecutionCode)` → `Call(&Process, TaskManagerId, &Error, &ErrorMessage)` (expected type `TaskManagerExecution`).
- **@TableCleaner**: resolves through **subtypes** (it denormalises the URL into `TableCleanerConfiguration` via the `TableCleanerDynamicCallReference` group) → `Call(TableCleanerProcessURL, …)`.

### 5.3 Seeding (upsert)
Each module contributes a `RetDynamicCallReference<X>` (`Output = SDTDynamicCallReferences`) with items:
```
SDTDynamicCallReferencesItem {
    DynamicCallReferenceCode        = DynamicCallReferenceCode.TskXxx
    DynamicCallReferenceType        = ReferenceType.TaskManagerExecution
    DynamicCallReferenceDescription = "Task Xxx"
    DynamicCallReferenceURL         = TskXxx.Type      // ← the real object's .Type
}
```
The upsert is done by `AddDynamicCallReference(in: &SDTDynamicCallReferences)` (`New … When Duplicate …`).

## 6. APIs vs Personalized

- **`APIs/`** (core): the transaction, `PPEXE_DeDynamicCallReferenceURL` (the resolver), `AddDynamicCallReference` (upsert), `DelDynamicCallReferences`.
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | **`AddDynamicCallReferences`** (Procedure) | The **master aggregator**: it calls `.Udp()` on every module's `RetDynamicCallReference<X>`, upserts them, and **prunes** the rows whose code no provider declares any more. This is where the project registers its DataProviders. |
  | `RetDynamicCallReferenceExample` (DataProvider) | A commented template showing how to declare references. |
  | `RetMenusDynamicCallReferences` (DataProvider) | The WW's menu entry. |

## 7. Pattern instance

**PXWorkWithTDynamicCallReferences** — a WW over the transaction (grid filtered by `ReferenceType`). The **"Update References"** action → `DelDynamicCallReferences` + `AddDynamicCallReferences(True)` (manual re-seeding from the UI).

## 8. Key procedures / APIs

| Proc | `Parm()` | Purpose |
|---|---|---|
| `PPEXE_DeDynamicCallReferenceURL` | `in: &Code; out: &URL` | **Resolver** code → `.Type` for `Call(&url, …)`. |
| `AddDynamicCallReference` | `in: &SDTDynamicCallReferences` | Upserts a set of references. |
| `DelDynamicCallReferences` | — | Empties the table (before re-seeding). |
| `AddDynamicCallReferences` (Personalized) | `in: &ShowMessages` | Full re-seed + pruning of obsolete rows. |

### How-to: register a new reference
1. Add the value to the `DynamicCallReferenceCode` domain.
2. Create/edit a `RetDynamicCallReference<X>` (`Output = SDTDynamicCallReferences`) returning `{ Code, Type (ReferenceType), Description, URL = RealObject.Type }`.
3. Add its `.Udp()` to the `Personalized/AddDynamicCallReferences` aggregator.
4. Run "Update References" (or call the aggregator) to persist it.

From then on any consumer invokes the object through `PPEXE_DeDynamicCallReferenceURL.Udp(code)` + `Call(…)`, or by subtype if the module denormalises the URL (as @TableCleaner does).

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- Consumers: [taskmanager.md](taskmanager.md) (`ExecutionCode`), the @TableCleaner module (`TableCleanerProcess`), [alerts.md](alerts.md), [cloudtasks.md](cloudtasks.md), [webserviceslog.md](webserviceslog.md).
- [menus.md](menus.md) — the menu entry is declared in `RetMenusDynamicCallReferences`.
