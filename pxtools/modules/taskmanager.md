# @TaskManager Module — The PXTools Task Manager

> Behaviour of the `@PXTools/@TaskManager` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@TaskManager/`
  - `APIs/` — the framework (transactions, runner, procedures). **Not to be touched.**
  - `Personalized/` — the project's customization (enabled queues, invocable tasks, menu, table cleaner).
  - `#Domains/` — one domain of its own (`FindSubjectIn`); the rest of the `TaskManager*` and `Cycle*` domains live in the KB's **root** `#Domains/`.
- The runner's compiled object is `APrcTaskManagerExecution` (from `PrcTaskManagerExecution`).
- **Depends on:** `@APIs` (base), `@DynamicCallReferences` (`code → object` resolution), `@ProcessMonitor` (lock/log), `@SystemParameters` (per-queue blocks), `@Menus` (`RetMenusTaskManager`), `@SendMails` (a qualified reference in this KB). In the canonical graph also `@System`.

## 1. What it provides

A **per-queue task scheduler and runner** (job scheduler + runner). It lets you:

- Define **one-off** tasks or **cyclic** ones (Daily/Weekly/Monthly, with intra-cycle repetition).
- **Enqueue** executions and have a **command-line runner** (one per queue) fire them when they fall due.
- Resolve **which GeneXus object to run** in a decoupled way, through a **code** (`DynamicCallReferenceCode`) that the `DynamicCallReferences` module maps to the real object.
- **Retries**, **cycles**, **repetitions**, **statuses**, a **parent/child hierarchy** and **per-execution logging** (integrated with `@ProcessMonitor`).

It is the infrastructure the application's **batch processes** run on (sending/receiving mail, scheduled tasks, data cleanup, integrations with external services, and so on).

## 2. Core concept: task → executions, and code → object

Two key ideas:

1. **One task (`TaskManager`) generates N executions (`TaskManagerExecutions`).** The task is the *definition* (what, with which parameters, in which queue, with which cycle/retries); each concrete run is an execution row with its status, dates and message. The execution's PK is composite: `(TaskManagerId, CycleId, RepeatId, ExecutionId)` — it models cycles and repetitions within a cycle.

2. **The object to run is NOT hard-coded**: the task stores a `TaskManagerExecutionCode` (domain `DynamicCallReferenceCode`); the runner resolves it into the real GeneXus object through the **`DynamicCallReferences`** module (`PPEXE_DeDynamicCallReferenceURL`), which is fed by `RetDynamicCallReference*` DataProviders — the framework's own is `Personalized/RetDynamicCallReferenceTaskManager`, and each application adds its own with its own tasks.

> **The contract of every invocable task:** `Parm(in: &TaskManagerId, out: &Error TaskManagerExecutionResponse, out: &ErrorMessage)`. The runner calls the object with the `TaskManagerId` and expects back a `TaskManagerExecutionResponse` (`Succeed`/`Retry`/`Fail`) plus a message. The object reads its parameters with `RetTaskManagerParameter*`.

### 2.1 Kinds of task: Once, Cyclic and On demand

There are three ways to use a task, depending on **how and who** creates it:

- **Once** — scheduled for a date/time and executed a single time. It is created **through the UI** (the tasks WW): entered once and then evaluated/monitored through its execution.
- **Cyclic** — repeats following a calendar pattern. Also created **through the UI**, where the whole execution repetition scheme is defined (cycle type, days/weeks/months, intra-cycle repetition, expiry). **That configuration's design was modelled on the *Windows Task Scheduler* format.**
- **On demand** — created by **the program itself**, calling the `AddTaskManagerSDT` API (§5.1). There you define which object to run (`ExecutionCode`), which parameters to pass, when to run it (`Date`), and how many retries and how long between them if it fails. This is how you "fire background work" from code.

> **Once** and **cyclic** tasks are created and administered through the **UI**; **on demand** tasks are born in **code** and are tracked from the same WW (by queue, by status and — crucially — by `FilterData`, see §2.2).

### 2.2 `FilterData` — identifying on-demand tasks by entity

`TaskManagerFilterData` (`VarChar(40)`) is a field meant for **finding on-demand tasks aimed at a specific entity** of the system. Since the same task (the same `ExecutionCode`) is usually enqueued many times — once per record/entity — **the programmer calling `AddTaskManagerSDT` should set `FilterData`** to a value identifying that execution's target entity. Administrators can then **filter in the WW** and tell apart the N instances of the same task associated with different entities/records.

> (The `ServerProcesses` queue additionally uses `FilterData` for the `ProcessServerId`; see §5.2.)

## 3. Module transactions

| Transaction | PK | Role in the model |
|---|---|---|
| **TaskManager** | `TaskManagerId` (autonumbered) | **The master task table.** The definition: subject, execution/visualization code + parameters (JSON), retry configuration, cycle configuration, queue, filter data, aggregate status, hierarchy (`TaskManagerParentId` self-FK). |
| **TaskManagerExecutions** | `TaskManagerId, TaskManagerExecutionCycleId, TaskManagerExecutionRepeatId, TaskManagerExecutionId` | **The execution log** (subordinate level). Status, type (Task/Retry/Cycle/Repeat), planned/start/end dates, `TaskManagerExecutionMessage` (the log), queue. |
| **TaskManagerQueues** | `TaskManagerQueueId` (domain `TaskManagerQueue`) | **The queue registry.** It associates each queue with a `ProcessServer` (`TaskManagerQueueProcessServerId`, subtype group → `@ProcessMonitor`). |
| **TaskManagerReferences** | `TaskManagerId, TaskManagerReferenceId` | **id/value** pairs attached to a task (the `AddTaskManagerReferenceValue` / `RetTaskManagerReferenceValue` API). |
| **TaskManagerBC** / **TaskManagerExecutionsBC** | (BC) | Business Components over the **same physical tables**, with nested levels; used for programmatic insert/delete (from `PrcTableCleanerTaskManager`, for instance). |

### 3.1 Relevant `TaskManager` attributes

- **Execution:** `TaskManagerExecutionCode` (`DynamicCallReferenceCode` → the object to run), `TaskManagerExecutionParameters` (`MaxStr`, a JSON of parameters). Their `…Visualization…` counterparts serve the "view result" object.
- **Retries:** `TaskManagerRetry` (Boolean), `TaskManagerRetryDay/Hour/Minute/Second` (the interval), `TaskManagerRetryTimes` (the maximum).
- **Cycle:** `TaskManagerCycleType` (`CycleType` Once/Daily/Weekly/Monthly), `…CycleTimes`, `…CycleWeekDays/Months/MonthDays/MonthsWeeks` (JSON), `…CycleRepeat` + `…CycleRepeatEveryMinutes/EveryHours/ForMinutes/ForHours` (intra-cycle repetition), `…CycleExpire`.
- **Queueing/status:** `TaskManagerQueue` (Nullable; **null = Main**), `TaskManagerFilterData` (`VarChar(40)`; identifies the **target entity** of an on-demand task so it can be found — see §2.2), `TaskManagerStatus`, `TaskManagerEnable`, `TaskManagerCreatedBy` (System/User), the `TaskManagerExecuting`/`…CountExecuting` formulas.
- **History:** `TaskManagerHistoryClear`, `…HistoryWithMessage`/`…WithoutMessage` (retention days, default 90).

### 3.2 Relationship diagram

```
   DynamicCallReferences (DynamicCallReferenceCode)
        ▲ ExecutionCode        ▲ VisualizationCode
        │                      │
   ┌────┴──────────────────────┴─────────────┐   self (Parent)
   │              TaskManager                 │◄────────────────┐
   │  PK TaskManagerId                        │  TaskManagerParentId
   │  FK TaskManagerQueue ──► TaskManagerQueues (FK ProcessServerId ► ProcessServer @ProcessMonitor)
   │  TaskManagerFilterData                   │
   └───┬───────────────────────┬─────────────┘
       │ 1:N                    │ 1:N
       ▼                        ▼
 TaskManagerExecutions    TaskManagerReferences
 PK(Id,CycleId,RepeatId,ExecId)   PK(Id,ReferenceId)
```

## 4. Module domains

Enum = **the stored value**. **Note (legacy):** except for `FindSubjectIn` (module-scoped), every domain of its own lives in the KB's **root** `#Domains/` — a legacy of GeneXus versions (Evo1/2/3) predating module-scoped domains; conceptually they belong to @TaskManager (only @TaskManager objects reference them through `DataType`).

**Its own — enumerated:**

| Domain | Type | Values |
|---|---|---|
| **TaskManagerExecutionResponse** | Char(20) | `Succeed=SUC`, `Retry=RET`, `Fail=FAI` — **the return contract of every task**. The ~10 modules implementing tasks consume it (all of them depend on @TaskManager); the reference in `@APIs/…/PDummy` is a type *dummy*, not ownership. |
| **TaskManagerStatus** | Char(3) | Active=`ACT`, Suspended=`SUS`, Succeed=`SUC`, Fail=`FAI` (the task's aggregate status) |
| **TaskManagerExecutionStatus** | Char(3) | Executing=`EXE`, Suspended=`SUS`, Succeed=`SUC`, Fail=`FAI`, KillRequested=`KIR` |
| **TaskManagerExecutionType** | Char(3) | Task=`TSK`, Retry=`RTY`, Cycle=`CYC`, Repeat=`REP` (there is no "OnDemand" value: *on demand* maps to `Task`) |
| **TaskManagerCreatedBy** | Char(3) | SystemTask=`SYS`, UserTask=`USR` |
| **TaskManagerParameterType** | Char(3) | Execution=`EXE`, Visualization=`VIS` |
| **TaskManagerTime** | VarChar(40) | Day=`DAY`, Hour=`HRS`, Minute=`MIN`, Second=`SEC` |
| **CycleType** | Char(3) | Once=`ONE`, Daily=`DAI`, Weekly=`WEE`, Monthly=`MON` |
| **CycleMonthsWeeksOrDays** | Char(3) | Days=`DAY`, Weeks=`WEE` (monthly cycle mode: a fixed day vs. week-ordinal + day) |
| **CycleDays** | Num(1) | Sunday=`1` … Saturday=`7` |
| **CycleWeeks** | Num(1) | First=`1` … Latest=`5` (week ordinal within the month) |
| **CycleMonths** | Num(2) | January=`1` … December=`12` |
| **CycleMonthsDays** | Num(2) | `1` … `31`, Last_Day=`33` (day of the month, including "last day") |
| **FindSubjectIn** *(module-scoped)* | Char(20) | `Task`, `Execution` — the WW searches the text in the task's subject or in the message |
| **TaskManagerQueue** | Char(20) | The set of available queues. Framework standard: `Main`, `SendMails`, `ReceiveMails` (+ `…WithAlias`/`…WithoutAlias`), `Statistics`, `TableCleaner`, `Infrastructure`, `Auxiliary`, `ImportExport`, `ServerProcesses`, `Free`; plus each application's own queues (the `RetTaskManagerQueues` DataProvider enables them, see §6). |

**Its own — value domains (no enum):**

| Domain | Type | Role |
|---|---|---|
| **TaskManagerLog** | LongVarChar(2M) | The text of an execution's log/message. |

**Domains used from other modules** (documented in their owner's doc):

| Domain | Owner | Role in @TaskManager |
|---|---|---|
| **DynamicCallReferenceParameters** | [@DynamicCallReferences](dynamiccallreferences.md) | Parameter value passed to a dynamically called task. By naming, `DynamicCallReference*` belongs to @DynamicCallReferences (even though in this KB only @TaskManager consumes it). |
| **ReferenceType** | [@DynamicCallReferences](dynamiccallreferences.md) | Category of the dynamic reference (`TaskManagerExecution`, `TaskManagerVisualization`, …). |
| **DynamicCallReferenceCode** | [@DynamicCallReferences](dynamiccallreferences.md) | Logical code of the object to run (`ExecutionCode`/`VisualizationCode`). |
| **ProcessStatusErrorCode** | [@ProcessMonitor](processmonitor.md) | Error codes of the concurrency lock. |
| **SystemParameterCode** | [@SystemParameters](systemparameters.md) | Codes of the blocking parameters (`TaskManagerBlocked…`). |
| **PXToolsDate** | @APIs | Date domain (long format). The `PXTools*` prefix = the framework's namespace → base @APIs. |
| *(base)* `Boolean`, `MaxStr`, `VarLen`, `IdFirstLevel`, `Links`, `Name`, `Path`, … | @APIs | The framework's base types. |

## 5. The execution model (the core)

### 5.1 Enqueueing — `AddTaskManagerSDT`

`Parm(in: &SDTTaskManager, out: &TaskManagerId)`. It does a `New` on `TaskManager` (status `Suspended`, `CreatedBy=SystemTask`), serializes the parameters and cycle collections into JSON, normalises `Queue` (Main→null), and creates the **first execution** with `AddTaskManagerExecution(&TaskManagerId, 1, 1, &Date, TaskManagerExecutionType.Task, &Date, &Queue)`.

The **`SDTTaskManager` SDT** is the enqueueing API: `Id, Date, Subject, ExecutionCode, ExecutionParameters` (a collection), `VisualizationCode/…Parameters`, `RetryDay/Hour/Minute/Second/Times`, `ParentId`, `CycleType/…`, `Queue, FilterData`.

`AddTaskManagerExecution` materialises a `TaskManagerExecutions` row as `Suspended`; if one is already `Suspended`, it brings its date forward when the new one is later.

### 5.2 Runner / daemon — `PrcTaskManagerExecution`

`[IsMain='True', CallProtocol='CommandLine']`, `Parm(in: &TaskManagerExecutionQueue, in: &ProcessServerId)`. An external scheduler (cron/service) invokes it **one instance per queue**, typically **every minute** — which is why the minimum scheduling granularity is one minute. The cycle:

1. **Blocks**: the `TaskManagerBlocked` SystemParameter (global) and `TaskManagerBlocked<Queue>` (per queue). If active, it does not process.
2. **Concurrency lock**: `StartProcessStatusSDT` (`@ProcessMonitor`); if one is already running, it aborts.
3. **Selection**: a `For Each` over TaskManager⋈Executions ordered by `Queue, Status, PlannedDate`, filtering the queue (Main = `Queue.IsNull()`; `ServerProcesses` additionally by `FilterData = &ProcessServerId`), `Enable = True`, `ExecutionStatus = Suspended`, `PlannedDate <= ServerNow` (only what is due).
4. **Mark in progress**: the task's status → `Active`, the execution's → `Executing`.
5. **Resolve the object**: `&Process = PPEXE_DeDynamicCallReferenceURL.Udp(TaskManagerExecutionCode)`.
6. **Invoke** (inside a try/catch): `Call(&Process, TaskManagerId, &Error, &ErrorMessage)`.
7. **Interpret** `Do Case &Error`: `Succeed` → mark Succeed and schedule the **next execution** (`RetTaskManagerNextExecution`); `Retry` or an exception → `'Retry Execution'`; `Fail` → mark Fail.
8. **Retry**: if `RetTaskManagerExecutionCount < TaskManagerRetryTimes`, schedule a `Retry` execution at `ServerNow + Retry…`.
9. **Persist + log**: `UpdTaskManagerExecutionStatus` (status + message + dates), `ChkTaskManagerStatus` (recomputes the aggregate status and propagates to the parent), `AddProcessStatusMessage` (persistent log in ProcessMonitor).

**Cycles**: `RetTaskManagerNextExecution` decides the next repetition (`RetTaskManagerNextRepeat`) or the next cycle (`RetTaskManagerNextCycle`, honouring days/weeks/months and `CycleExpire`).

### 5.3 Reading the parameters inside the task

The task only receives the `TaskManagerId`; it reads its parameters out of the JSON with `RetTaskManagerParameterString/Integer/Date/Boolean(in: &TaskManagerId, in: &Position, out: &Value)` (Position is 1-based).

## 6. APIs vs Personalized

### `APIs/` (the framework — do not touch)
The whole engine: the transactions (`TaskManager*`), the BCs, the `PrcTaskManagerExecution` runner, the `Add*/Ret*/Upd*/Chk*` procedures, the pattern instances (CRUD), and the module's own maintenance tasks (`TskDeleteTaskManagerExecutions`, `TskTaskManagerForceRetryByQueue`, `TskKillProcess`).

### `Personalized/` (the extension point — customized per project)
| Object | What it defines / how it is customized |
|---|---|
| **`RetTaskManagerQueues`** (DataProvider) | **Which queues are enabled** in the project (the UI combos + it generates the per-queue blocking SystemParameters). Each project adds its own batch-flow queues here, alongside the framework's standard ones. |
| **`RetDynamicCallReferenceTaskManager`** (DataProvider) | **Which of the framework's own objects are invocable** by the TaskManager (the `code → object` mapping): `TskDeleteTaskManagerExecutions`, `TskProcessMonitorKillProcess`→`TskKillProcess`, `TskTaskManagerForceRetryByQueue`, and one `TableCleanerTaskManagerQueue<Queue>` per queue → all pointing at `PrcTableCleanerTaskManager`. |
| **`RetMenusTaskManager`** (DataProvider) | The "Task Manager" menu under `Basic`: **Queues** (`TrTaskManagerQueues`) and **Tasks** (`TrTaskManager`). |
| **`PrcTableCleanerTaskManager`** (Procedure) | The per-queue **history purger** (referenced by every `TableCleanerTaskManagerQueue*`). It deletes expired `Once` tasks and old executions (preserving the last cycle so Retry does not break); before deleting it nulls out any business FKs pointing at the task. |

> **Note:** the `code → object` mapping of the **application's** tasks (not the framework's) does NOT live here: each application declares it in its own `Personalized` DataProvider (by convention `RetDynamicCallReference<App>`), which also feeds `DynamicCallReferences` (see §9).

## 7. PXTools pattern instances

| Instance | What it is |
|---|---|
| **PXWorkWithTaskManager** | Full task CRUD. A Transaction with **General** (Queue, Subject, ExecutionCode as a Dynamic Combo filtered by `ReferenceType.TaskManagerExecution`, parameters through the `PXParameterRequestUpdTaskManagerParameters` prompt, the Retry block) and **Cycle Settings** tabs. A two-level Selection (basic `TaskManager` / `TaskManagerAdvanced` with search inside messages). A "Task Information" View with a grid of **Executions** (Retry, ViewMessage, Kill&Retry, ExecuteNow, Download log) and a tree of **Child Tasks**. |
| **PXWorkWithTaskManagerQueues** | A simple WW (built on the `SDTTaskManagerQueues` collection) for **assigning a ProcessServer to each queue** (`UpdTaskManagerQueueProcessServer`). |
| **PXParameterRequestExecutionMessage** | The "View Message" popup: shows an execution's `TaskManagerExecutionMessage` (log) read-only. |
| **PXParameterRequestUpdTaskManagerParameters** | The "Edit parameters" prompt: an editor for the `;`-separated parameter list (Add/Remove item). |

## 8. Key procedures / APIs

**Enqueueing / executions**
- `AddTaskManagerSDT(in: &SDTTaskManager, out: &TaskManagerId)` — creates the task + its first execution.
- `AddTaskManagerExecution(...)` — creates/brings forward a `Suspended` execution.
- `AddTaskManagerExecutions` / `AddTaskManagerExecutionsAndReset` — re-enqueues the task (and its failed descendants); the *Reset* variant clears the retry counter. Used by the "Retry" action.
- `AddTaskManagerReferenceValue` — attaches an id/value pair.

**Reading**
- `RetTaskManagerParameterString/Integer/Date/Boolean(in: &TaskManagerId, in: &Position, out)` — parameter N out of the JSON.
- `RetTaskManagerNextExecution` / `…NextCycle` / `…NextRepeat` — scheduling computation.
- `RetTaskManagerExecutionCount` — the real retry count (excluding ignored ones).
- `RetTaskManagerQueue`, `RetTaskManagerReferenceValue`, `RetTaskManagerHasChilds`, `RetTaskManagerExecutionMessageLinkToFile` (dumps the log into a `.txt`).

**Status / control**
- `UpdTaskManagerStatus`, `UpdTaskManagerEnable`, `UpdTaskManagerExecutionStatus` (status + message + dates), `UpdTaskManagerExecutionsNow` (Execute Now).
- `ChkTaskManagerStatus` — recomputes the aggregate status and propagates to the parent.

**Kill / fail-and-retry**
- `RequestKillProcess`, `SetFailAndRetryTaskManager`, `TestAndKillTaskManager`, `KillProcessStatus` (integration with `@ProcessMonitor`).

## 9. The application integration pattern

TaskManager is the system's **batch orchestrator**. An application integrates its processes like this:

1. **Declare the queues** — add your own queues in `Personalized/RetTaskManagerQueues`. Dedicating a queue to one kind of work lets its runner run in parallel/isolated (and be blocked independently with `TaskManagerBlocked<Queue>`).
2. **Register the tasks** — in the application's own `Personalized` DataProvider (`RetDynamicCallReference<App>`), map each `DynamicCallReferenceCode` to its procedure with `DynamicCallReferenceURL = <Proc>.Type` and `ReferenceType.TaskManagerExecution`. This is what the runner resolves at execution time.
3. **Enqueue** — build the `SDTTaskManager` (with `ExecutionCode`, parameters, `Queue`, retry configuration, and optionally `FilterData` for grouping) and call `AddTaskManagerSDT`. A **cyclic** task can also be created from the WW.
4. **Implement the task** — the procedure honours the `Parm(in: &TaskManagerId, out: &Error TaskManagerExecutionResponse, out: &ErrorMessage)` contract: it reads its parameters with `RetTaskManagerParameter*`, does its work, and returns `Succeed` / `Retry` (retry per the configuration) / `Fail`. `&ErrorMessage` is the **log** left on the execution (visible in the WW and in `PXParameterRequestExecutionMessage`).

### How-to: make a task "callable" by the TaskManager
1. Create the procedure with the three-parameter contract (read the parameters with `RetTaskManagerParameter*`).
2. Add a value to the `DynamicCallReferenceCode` domain.
3. Register it in the application's `Personalized` DataProvider (`RetDynamicCallReference<App>`), mapping the code → `<Proc>.Type` with `ReferenceType.TaskManagerExecution`.
4. Enqueue it with `AddTaskManagerSDT` (or create it as a cyclic task in the WW), naming the queue.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- The **DynamicCallReferences** module — the `code → object` mapping (resolved by `PPEXE_DeDynamicCallReferenceURL`).
- The **@ProcessMonitor** module — the concurrency lock, ProcessServers and the persistent log (`AddProcessStatusMessage`).
- `modules/security.md` — the @Security module (the `&Context`, access policies).
