# @TableCleaner Module — Scheduled Table Cleanup

> Behaviour of the `@PXTools/@TableCleaner` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@TableCleaner` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.TableCleaner`.
- **Depends on:** `@APIs` (base), `@DynamicCallReferences` (`code → cleaner` dispatch), `@SystemParameters` (deletion limits), `@Menus`. It is triggered by `@TaskManager`.

## 1. What it provides

**Scheduled purging of old data per table**: one configuration per process defines the retention age; a batch task walks the configurations and **dynamically dispatches** each module's cleanup process (by name, through @DynamicCallReferences). It is the shared "table cleaner" infrastructure reused by @SendMails, @ReceiveMails, @FileStorage, @Alerts, @WebServicesLog, @TaskManager, and others.

## 2. Core concept

- **`TableCleanerConfiguration`** = one row per registered cleanup process, with its **retention** (Years/Months/Days) and batch limit.
- Its PK (`TableCleanerProcessCode`) is a **dynamic reference** of type `TableCleanerProcess`; the concrete process URL is inherited from `TDynamicCallReferences`.
- **`TskTableCleaner`** (batch) iterates the configurations, computes the **cut-off date** and calls each `PrcTableCleaner<X>` by name.

## 3. The `TableCleanerConfiguration` transaction

| Attribute | Type | Role |
|---|---|---|
| `TableCleanerProcessCode` (PK) | `DynamicCallReferenceCode` | FK (subtype) to `TDynamicCallReferences` (type `TableCleanerProcess`). |
| `TableCleanerProcessDescription` / `…URL` | inferred | Inherited from the dynamic reference. |
| `TableCleanerYears` / `Months` / `Days` | num | Retention age (used to compute the cut-off date). |
| `TableCleanerMaxRows` | `IdFirstLevel` (nullable) | Batch limit (or the global one). |
| `TableCleanerEnableCounter` | Boolean | Report counts? |

> The `TableCleanerDynamicCallReference` group: `TableCleanerProcessCode : DynamicCallReferenceCode` (+ Description/URL). The configuration does **not** store the URL: it inherits it from the dynamic reference catalogue.

## 4. Module domains

@TableCleaner **defines no domains of its own** (there is no `TableCleaner*` domain — `TableCleanerConfiguration` is a transaction). It uses these from other modules:

| Domain | Owner | Use |
|---|---|---|
| `ReferenceType` (value `TableCleanerProcess`) | @DynamicCallReferences | Type of the cleaner's reference |
| `DynamicCallReferenceCode` | @DynamicCallReferences | Cleaner codes: `TskTableCleaner`, `TableCleanerFileStorage`, `TableCleanerWebServicesLog`, `TableCleanerAlerts`, … (one per table/module) |
| `SystemParameterCode` (`TableCleanerMaxRowsToDeleteDB/SDT`) | @SystemParameters | Deletion limits |
| `SDTCounterType` (Failed=`FAI`, Succeed=`SUC`) | @APIs base | Type of the result counter (anchored in @APIs' `SDTCounter` SDT; many modules use it) |

## 5. How it works

### 5.1 Execution — `TskTableCleaner`
`Parm(in: &TaskManagerId, out: &Error, out: &ErrorMessage)` (registered as a `TaskManagerExecution`):
1. `&InitialDate = ServerNow`; `&MaxRowsToDelete = RetMaxRowsToDeleteDB`.
2. `For Each TableCleanerConfiguration` (⋈ `TDynamicCallReferences`): computes `&Date = &InitialDate.AddYears(-Years).AddMonths(-Months).AddDays(-Days)`, resolves `MaxRows` (row-level or global), and makes the **dynamic call by name**:
   ```
   Call(TableCleanerProcessURL, TableCleanerProcessCode, &Date, &TableCleanerMaxRows, &TableCleanerEnableCounter, &SDTCounter)
   ```
   (`TableCleanerProcessURL` = the `.Type` of the `PrcTableCleaner<X>`.) It accumulates `&SDTCounter` into the report.

### 5.2 How a module registers its cleaner
1. Write a `PrcTableCleaner<X>` following the **contract**: `Parm(in: &DynamicCallReferenceCode, in: &Date, in: &MaxRowsToDeleteDB, in: &EnableCounter, out: &SDTCounter)`. It deletes records dated `<= &Date`, batches by `RetMaxRowsToDeleteSDT` (default 3000), issues a `Commit` when `MaxRowsToDeleteDB` is exceeded, and returns `SDTCounter`.
2. Provide a `RetDynamicCallReference<X>` (in its own `Personalized/`) with an item `{ Code = DynamicCallReferenceCode.TableCleaner<X>, Type = ReferenceType.TableCleanerProcess, URL = PrcTableCleaner<X>.Type }`.
3. `AddDynamicCallReferences` (from @DynamicCallReferences) picks that DataProvider up and upserts into `TDynamicCallReferences`. From then on the process can be configured and executed.

## 6. APIs vs Personalized

- **`APIs/`** (core): the transaction, `TskTableCleaner` (the orchestrator), `PrcTableCleaner` (config upsert), and the limit helpers (`RetMaxRowsToDeleteDB/SDT`, `RetCountRowsToDeleteDB/SDT`).
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | `RetDynamicCallReferenceTableCleaner` (DataProvider) | Registers the `TskTableCleaner` task in @TaskManager. |
  | `RetSystemParametersTableCleaner` (DataProvider) | `TableCleanerMaxRowsToDeleteDB`/`…SDT`. |
  | `RetMenusTableCleaner` (DataProvider) | Menu entry. |

> Each **consuming module** contributes its own `RetDynamicCallReference<X>` + `PrcTableCleaner<X>` pair in its own `Personalized/`, not here.

## 7. Pattern instance

**PXWorkWithTableCleanerConfiguration** — the configurations WW (filter by `TableCleanerProcess`; grid with editable Years/Months/Days/MaxRows/EnableCounter). Actions: **"Update Processes"** (`DelDynamicCallReferences` + `AddDynamicCallReferences(True)` — refreshes the reference registry) and **Apply** (`PrcTableCleaner` per row — upserts the configuration).

## 8. Key procedures / APIs

| Proc | `Parm()` | Purpose |
|---|---|---|
| `TskTableCleaner` | `in: &TaskManagerId, out…` | @TaskManager entry point; dispatches every cleaner. |
| `PrcTableCleaner` | `in: &Code, &Years, &Months, &Days, &MaxRows, &EnableCounter` | Upserts one configuration (**deletes** it if Years=Months=Days=0). |
| `RetMaxRowsToDeleteDB` / `…SDT` | `out: &MaxRowsToDelete` | Global limits (defaults 100000 / 3000). |
| **The cleaner contract** `PrcTableCleaner<X>` | `in: &Code, &Date, &MaxRowsToDeleteDB, &EnableCounter; out: &SDTCounter` | The signature every concrete cleaner implements. |

**Return SDT** `SDTCounter` (`PXTools.APIs`): a collection of `{ Table, Type (SDTCounterType), Counter, ErrorMessage }`.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [dynamiccallreferences.md](dynamiccallreferences.md) — `TableCleanerProcessCode` is a dynamic reference; the URL comes from there.
- [taskmanager.md](taskmanager.md) — runs `TskTableCleaner` (the `TableCleaner` queue).
- Consumers: [sendmails.md](sendmails.md), [filestorage.md](filestorage.md), [alerts.md](alerts.md), [webserviceslog.md](webserviceslog.md), and others.
