# @WebServicesLog Module — Web Service Log and Statistics

> Behaviour of the `@PXTools/@WebServicesLog` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@WebServicesLog` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.WebServicesLog`.
- **Depends on:** `@APIs` (base), `@TableCleaner` (log purging, through `RetCountRowsToDeleteDB`), `@SystemParameters`, `@DynamicCallReferences`, `@Menus`.

## 1. What it provides

It records **every inbound web service invocation** (request/response, timings, status, IP) in a raw log, and then **aggregates** those records into statistics tables through a batch task.

## 2. Core concept: raw log → statistics

- **`WebServicesLog`** = one row per call (with `Duration` computed by a formula).
- A batch task with an **incremental cursor** buckets the log into N-minute intervals and accumulates it (`counter += 1`) into:
  - **`WebServicesStatistics`** (by FilterData) — a header plus a detail per RemoteAddress/WS/Method/Status.
  - **`WebServicesRAStatistics`** — by **Remote Address** (IP).

> **`RA` = Remote Address** (per-IP statistics), **not** "rolling average". The engine is incremental with fixed-interval bucketing, not moving averages.

## 3. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **WebServicesLog** | `WebServiceLogId` | **Raw log**: `StartDateTime`/`EndDateTime`, `RemoteAddress` (IP), `WSName`, `MethodName`, `MethodInParameters`/`MethodOutParameters` (MaxMem ~30 KB, JSON), `FilterData` (a free business key), `Status` (`WebServiceLogStatus`), `Duration` (formula `tdiff(End, Start)`). BC: `WebServicesLogBC`. |
| **WebServicesStatistics** | `FilterData, DateTime` (+ a Detail level: `RemoteAddress, WSName, MethodName, Status`) | Aggregates by FilterData; counters. |
| **WebServicesRAStatistics** | `RemoteAddress, DateTime` | Aggregates by IP; counter. |

## 4. Module domains

All `WebService*`/`WebServices*` domains belong to @WebServicesLog (by naming). The status ones are **root-legacy** (root `#Domains/`); the data and panel types are already **module-scoped** (`@WebServicesLog/#Domains/`):

| Domain | Scope | Values |
|---|---|---|
| **WebServiceLogStatus** | root-legacy | WithoutResponse=`WOR`, Success=`SUC`, Failed=`FAI` |
| **WebServiceStatisticDetailStatus** | root-legacy | + Denied=`DEN` (in the aggregation) |
| **WebServicesStatisticsCounterType** | root-legacy | FilterData=`FDA`, CounterType=`FCO` |
| **WebServicesStatisticsPanelType** | module | FilterData=`FDA`, RemoteAddress=`RAS` (view selector) |
| **WebServiceFilterData / …MethodName / …RemoteAddress / …WSName** | module | Base types, VarChar(40) |
| **WebServiceStatisticCounter** | module | Numeric(10.0) — the counter's value |
| **StatisticFilterDuring** | module | *(placeholder — a domain declared with no body)* |

## 5. How it works

### 5.1 Logging an invocation
It is invoked by **each consuming WS object** (NOT by the @WSLayer layer), typically with a `Stub`:
```
Stub Method(...)
    &WebServiceLogId = AddWebServiceLog.Udp(&Pgmname, !"Method", &in.ToJson(), &filter)
    ...  // logic
    UpdWebServiceLog.Call(&WebServiceLogId, WebServiceLogStatus.Success, &out.ToJson())
EndStub
```
- `AddWebServiceLog(in: &WSName, &MethodName, &MethodInParameters, &FilterData; out: &WebServiceLogId)` — a `New` with `StartDateTime=ServerNow()`, `RemoteAddress=RetHTTPRemoteAddress()`, the inputs, and `Status=WithoutResponse`.
- `UpdWebServiceLog(in: &WebServiceLogId, &Status, &MethodOutParameters)` — `EndDateTime`, outputs and final status. `Duration` computes itself.
- Variants: `UpdWebServiceLogWithFilterData`, `UpdWebServiceLogFilterData`. `RetWebServicesLogStatusFromRestCode` maps a REST code → Success/Failed.

### 5.2 Computing statistics — `TskWebServicesLogStatistics`
`Parm(in: &TaskManagerId, out: &Error, out: &ErrorMessage)`:
- Gated by the `WebServiceLogGenerateStatistics` SystemParameter; the period in minutes comes from `RetWebServiceLogStatisticCounterPeriod` (default 60); the **incremental cursor** is `WebServiceLogLastStatisticId`.
- `For Each WebServicesLog Where Id > lastId` (new rows only). **Time bucketing**: minutes since midnight / period, rounded, converted back to a DateTime.
- Upsert (`When None New`) of the `WebServicesStatistics` header (FilterData+bucket), the `Detail` (RemoteAddress/WS/Method/Status, mapped to `Denied`/Failed/Success/WithoutResponse) and `WebServicesRAStatistics` (RemoteAddress+bucket); `counter += 1`.
- It stores the new `LastStatisticId`. Helpers: `RetWebServices[RA]StatisticsOtherColumns` (up to 20 previous periods for the "history in columns"). Cleanup: `PrcTableCleanerWebServicesLog`/`…Statistics`.

## 6. APIs vs Personalized

- **`APIs/`** (core): the transactions, `Add/UpdWebServiceLog`, `TskWebServicesLogStatistics`, the `Ret*`/`PrcTableCleaner*` procedures.
- **`Personalized/`** (3 DataProviders):
  | Object | What gets customized |
  |---|---|
  | `RetDynamicCallReferenceWebServicesLog` | Registers in @TaskManager: the statistics task + 2 Table Cleaners. |
  | `RetMenusWebServicesLog` | Menu (Logs / Statistics / Counters by Filter Data). |
  | `RetSystemParametersWebServicesStatistics` | `WebServiceLogGenerateStatistics`, `WebServiceLogLastStatisticId`, `WebServiceLogStatisticCounterPeriod`. |

## 7. Pattern instances

- **PXWorkWithWebServicesLog** — the log grid (filters by date/FilterData/Status/Method + search inside the parameters). The **View** action → `PXParameterRequestWebServiceLogView`.
- **PXParameterRequestWebServiceLogView** — popup with the In/Out Parameters.
- **PXComposerWebServicesLog** — a layout with the selection plus the embedded parameter view.
- **PXWorkWithWebServicesStatistics** / **PXWorkWithWebServicesRAStatistics** — aggregate grids (by FilterData / by IP), with "history in columns" and cross-links between the two.
- **PXComposerWebServicesStatisticCounters** — embeds the counters level.

## 8. Key procedures / APIs

| Proc | `Parm()` | Purpose |
|---|---|---|
| `AddWebServiceLog` | `in: &WSName, &MethodName, &MethodInParameters, &FilterData; out: &WebServiceLogId` | Opens the log entry when the call starts. |
| `UpdWebServiceLog` | `in: &WebServiceLogId, &Status, &MethodOutParameters` | Closes the call (end time, output, final status). |
| `UpdWebServiceLogWithFilterData` | `+ &FilterData` | The same plus FilterData. |
| `RetWebServicesLogStatusFromRestCode` | `in: &RestCode; out: &Status` | Maps a REST code → Success/Failed. |
| `TskWebServicesLogStatistics` | `in: &TaskManagerId; out…` | The aggregation batch. |
| `PrcTableCleanerWebServicesLog` / `…Statistics` | `(cleaner)` | Purges the log / the statistics by date. |

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [taskmanager.md](taskmanager.md) — runs `TskWebServicesLogStatistics`.
- The **@WSLayer** module — the web services layer (whitelist); logging is **orthogonal** to it (each WS object calls it, not @WSLayer).
- The **@SystemParameters** module (aggregation configuration).
