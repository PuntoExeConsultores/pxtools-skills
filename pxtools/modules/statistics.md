# @Statistics Module — System Statistics

> Behaviour of the `@PXTools/@Statistics` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@Statistics` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.Statistics`.
- **Depends on:** `@APIs` (base), `@DynamicCallReferences` (dispatching each statistic's process), `@Menus`. It is triggered by `@TaskManager`; purged through `@TableCleaner`.

## 1. What it provides

A framework for **defining and recording numeric metrics per period**: each statistic is declared (metadata + a process computing it), and a batch task runs those processes to populate a time series of values, queryable by range or as a snapshot.

## 2. Core concept

- **`StatisticDefinition`** = the metadata: code + description + the **process** computing the values + an **Id composition** labelling the sub-dimension.
- **`StatisticLog`** = the time series: one numeric value per **(type, subject=ReferenceCode, moment=DateTime, sub-metric=Ids)**.
- Each definition's process is resolved **by name** (through @DynamicCallReferences) and triggered by @TaskManager.

## 3. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **StatisticDefinition** | `StatisticDefinitionCode` (+ an `IdComposition` level) | Metadata: `Description`, `ProcessCode`/`ProcessURL` (the procedure recording the values, a dynamic reference). The `IdComposition` level (position → label, with its DataProvider). |
| **StatisticLog** | `StatisticLogCode, StatisticLogReferenceCode, StatisticLogDateTime, StatisticLogIds` | The **value** (`StatisticLogValue`, Numeric(18.4)) per period (minute granularity). |
| **StatisticLogBC** (BC) | (same) | Programmatic CRUD over the logs. |

The `StatisticLogCode : StatisticDefinitionCode` group (Log ⋈ Definition); plus groups integrating `ProcessCode`/`IdDataProviderCode` with `DynamicCallReferenceCode`.

## 4. Module domains

All `Statistic*` domains belong to @Statistics (by naming), all **root-legacy** (they live in the root `#Domains/`):

| Domain | Values |
|---|---|
| **StatisticType** | Range=`RAN`, Last=`LAS` (the query mode in `RetQuery`) |
| **StatisticDefinitionCode** (Char(50)) | An **extensible** catalogue of statistic types; each application adds its own values. |
| **StatisticMailAccount** (Numeric(1)) | BothEnabled=`0`, POP3Disabled=`1`, SMTPDisabled=`2`, BothDisabled=`3` — a metric of mail account state. Despite describing accounts, it belongs to @Statistics (by naming and by use). |

## 5. How it works

### 5.1 Defining a statistic
1. Add the value to the `StatisticDefinitionCode` enum.
2. Write a "process" procedure (in `Personalized/`) that computes and stores the logs; register it as `ReferenceType.StatisticLogProcess` in @DynamicCallReferences.
3. Register the `StatisticDefinition` row — the `RetStatisticDefintion` seed DataProvider declares it and `AddStatisticDefinition` inserts it (or do it through the WW).
4. Optional: a composition DataProvider (→ `SDTComposition`) giving readable labels to each `Ids`.

### 5.2 Recording a value (from the process procedure)
There is no dedicated "increment" API; the procedure accumulates and **upserts** into `StatisticLog`:
```
New
    StatisticLogReferenceCode = ...
    StatisticLogDateTime      = &ServerNow
    StatisticLogCode          = &StatisticType
    StatisticLogIds           = &SubMetric
    StatisticLogValue         = &Value
When Duplicate
    For Each
        StatisticLogValue = &Value   // update if it already exists
EndNew
```

### 5.3 Execution — through @TaskManager
**`TskStatistics`** (`Parm(in: &TaskManagerId, out: &Error TaskManagerExecutionResponse, out: &ErrorMessage)`) walks **every** `StatisticDefinition` and issues a `Call(StatisticDefinitionProcessURL)` (dynamic call) for each process. It is registered as `ReferenceType.TaskManagerExecution`; the scheduler triggers it. `PrcTableCleanerStatisticDefinition` hooks into @TableCleaner to purge old logs.

### 5.4 Querying
`RetQuery(in: &Query SDTStatisticsQueryIn, out: &Data SDTStatisticsQueryOut)`: filters by `DefinitionCode`+`ReferenceCode` and, depending on `Type`, returns `Last` (most recent snapshot) or `Range` (between `RangeStart`/`RangeEnd`); it enriches the result with the Ids labels (`RetSDTComposition`).

## 6. APIs vs Personalized

- **`APIs/`** (core): the transactions, `RetQuery`, `AddStatisticDefinition`, `TskStatistics`, `RetStatisticDefintion` (seed), `PrcTableCleanerStatisticDefinition`, and the SDTs.
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | `RetDynamicCallReferenceStatisticCodes` (DataProvider) | Registers each of the project's process procedures (`StatisticLogProcess`). |
  | `RetDynamicCallReferenceStatistics` (DataProvider) | Registers `TskStatistics` + the composition DataProviders + the TableCleaner. |
  | `RetMenusStatistics` (DataProvider) | Menu (Definitions / Logs). |
  | The `Prc…` process procedure(s) + composition DataProvider(s) + `RetSDTComposition` | **The concrete logic** computing the metrics and their labels. |

## 7. Pattern instances

- **PXWorkWithStatisticDefinition** — the catalogue WW (a "General" view + an "Id Composition" grid section).
- **PXWorkWithStatisticLog** — the value series WW (filters by Code/Reference/range/Ids/Value).

## 8. Key procedures / APIs

| Object | `Parm()` | Purpose |
|---|---|---|
| `RetQuery` | `in: &Query; out: &Data` | Query (Last/Range). |
| `AddStatisticDefinition` | — | Populates `StatisticDefinition`+IdComposition from the seed. |
| `RetSDTComposition` | `in: &IdDataProviderCode; out: &SDTComposition` | Labels for the Ids dimension. |
| `TskStatistics` | `in: &TaskManagerId, out…` | @TaskManager job: runs each definition's process. |
| `PrcTableCleanerStatisticDefinition` | `(cleaner)` | Purges old logs (@TableCleaner hook). |

**SDTs**: `SDTStatisticDefinition`, `SDTStatisticsQueryIn` (DefinitionCode/ReferenceCode/Type/RangeStart/RangeEnd), `SDTStatisticsQueryOut` (DateTime → Values[Id, Value]), `SDTComposition` (Position/Reference).

> **End-to-end flow:** @TaskManager triggers `TskStatistics` → it walks `StatisticDefinition` → dynamic `Call()` of each `ProcessURL` → the procedures upsert into `StatisticLog` → consumers read through `RetQuery` → @TableCleaner clears the history.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [dynamiccallreferences.md](dynamiccallreferences.md) — resolves `ProcessCode` → procedure.
- [taskmanager.md](taskmanager.md) — runs `TskStatistics`.
- The @TableCleaner module (log purging).
