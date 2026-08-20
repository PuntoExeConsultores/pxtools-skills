# @ProcessMonitor Module — Process Monitor

> Behaviour of the `@PXTools/@ProcessMonitor` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@ProcessMonitor` (`APIs/Basic/`, `APIs/Advanced/`, `Personalized/`).
- Qualifier: `PXTools.ProcessMonitor`.
- The **lock/kill** of hung processes lives in the sibling module `@TaskManager` (its main consumer).
- **Depends on:** `@APIs` (base), `@System`, `@SystemParameters`, `@SendMails`, `@FileStorage`, `@TaskManager` (kill-process). Infrastructure: `@Menus`.

## 1. What it provides

A **registry of running processes** serving as a **concurrency mutex**, a persistent **progress log**, **detection and killing of hung processes**, and partitioning by **server/node**. It is the foundation batch runners (the @TaskManager one, for instance) use to avoid double execution, report progress, and be cancelled or killed.

## 2. Core concept: `ProcessStatus` as a mutex

`ProcessStatus` = "one running process". Its PK (`UserCode + Code`) identifies the process; only **one** row with that PK can be `Running` at a time → that is the **lock**. Each process carries its own log (`ProcessStatusMessages`) and optionally a `ServerId`.

```
Life cycle (ProcessStatusStatus):
  [New] ─Start─▶ Running
   Running ─End────────────▶ Ended      ─Dlt▶
   Running ─Cancel─────────▶ Cancelled  ─Dlt▶
   Running ─RequestKill────▶ KillRequested ─Kill(async)▶ Killed ─Dlt▶
   Running ─Verification (process not alive in the OS)─▶ Ended (forced)
```

## 3. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **ProcessStatus** | `ProcessStatusUserCode, ProcessStatusCode` (empty UserCode = global) | The **mutex**. `ObjectName`/`Description` (subtypes of `SystemObjectName`), `Status` (`ProcessStatusStatus`), `ChangeStatusDateTime` (the basis for detecting hangs), `URL` (output), `IsCommandLine`, `AllowCancelationRequest`, `Parameters`, `ServerId` (FK), `FileStorageId` (blob output, Advanced). |
| **ProcessStatusMessages** | `UserCode, Code, MessageId` | Persistent log (1:N): `DateTime`, `Description` (MaxMem). |
| **ProcessServers** | `ProcessServerId` | Catalogue of execution servers/nodes (per-server queues). `ProcessServerName`. |

## 4. Module domains

Its own (named `ProcessStatus*`), both **root-legacy** (they live in the root `#Domains/`). There is no `ProcessServer*` domain — `ProcessServers` is a transaction:

| Domain | Values |
|---|---|
| **ProcessStatusStatus** | Running=`RUN`, Ended=`END`, Cancelled=`CAN`, Killed=`KIL`, KillRequested=`KRQ` (final states: END, KIL) |
| **ProcessStatusErrorCode** | ProcessNotFoundRunning=`NFR`, CannotKillProcess=`CKP`, Other=`OTH` (also consumed by @TaskManager) |

## 5. How it works

### 5.1 Registration + lock — `StartProcessStatus[SDT]`
On startup, the runner calls `StartProcessStatus`/`StartProcessStatusSDT`:
- It checks `ChkProcessStatusFinalized`: if a previous, non-finalised record exists, the `_SDT` variant refuses (`Result=False`, "Duplicate process running") — **that is the mutex**.
- It creates/updates the row to `Running`, stamps `ChangeStatusDateTime`, clears the URL and **purges the previous messages** (a clean log). `…SDT` additionally stores ObjectName, Parameters, IsCommandLine and ServerId, and registers the object (`AddSystemObject`).

### 5.2 Progress (log) and closing
- `AddProcessStatusMessage(UserCode, Code, text)` — appends a line to the persistent log.
- `EndProcessStatus(UserCode, Code, URL)` → `Ended`, computes the duration, adds a closing message and **sends mail** to the user. `CancelProcessStatus` → `Cancelled` (no mail).

### 5.3 Cancelling/killing hung processes (lives in @TaskManager)
- **Cooperative**: the runner checks `ChkProcessStatusCanceled` on each iteration and exits on its own.
- **Hang detection**: `PrcProcessMonitorVerification` (Main, CommandLine) walks the server's `Running` `IsCommandLine` rows, checks in the OS (Linux `ps -aux | grep java`) whether the process is alive; if it is not, it forces `Ended` + `PrcProcessMonitorAfterEnds`.
- **Explicit kill (2 phases, async)**:
  1. `RequestKillProcess` (@TaskManager) → marks `KillRequested` and enqueues `TskProcessMonitorKillProcess` (the `ServerProcesses` queue if there is a ServerId, otherwise `Infrastructure`).
  2. `KillProcessStatus` (@TaskManager, running on the server) → locates the PID (`ps -aux`), `kill -9`, re-checks, marks `Killed`; it uses `ProcessStatusErrorCode`.

### 5.4 How the @TaskManager runner uses it
`PrcTaskManagerExecution` builds an `SDTProcessStatus` with `Code = RetTaskManagerProcessStatusCode(object, queue)`, `IsCommandLine=True`, `ServerId = RetTaskManagerQueueProcessServerId(queue)`; `StartProcessStatusSDT` → if `Result` it processes the queue (reporting through `AddProcessStatusMessage`) and calls `EndProcessStatus` when done; if `Result=False` it logs "is running yet" (preventing a second runner over the same queue).

## 6. APIs vs Personalized

- **`APIs/Basic/`** + **`APIs/Advanced/`** (core): the transactions, `Start/End/Cancel/Del`, `AddProcessStatusMessage`, the `Chk*` procedures, and the `…WithStorage` variants (output to @FileStorage).
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | `PrcProcessMonitorAfterEnds` (Procedure) | **Hook** invoked when the verification forces a process to `Ended`. Default: `TestAndKillTaskManager` (hands it back to the TaskManager). |
  | `RetMenusProcessMonitor` (DataProvider) | Menu items (Servers / Monitor). |

## 7. Pattern instances

| Instance | What it is |
|---|---|
| **PXWorkWithProcessStatus** | WW over ProcessStatus. |
| **PXWorkWithProcessStatusMessages** | Popup (`PuProcessStatusMessages`) — viewer for a process's log. |
| **PXWorkWithProcessServers** | WW for the servers. |
| **PXWorkWithProcessStatusAdvanced** | The **operational monitor**: a grid with a My/Global filter (Global for admins only, `PIsAdministrator`), OpenDocument (downloads the output) and ViewMessages actions, and a polymorphic **CancelProcess** (Cancel / RequestKill / End / Delete depending on status, type and permission). |

## 8. Key APIs

**Basic**: `StartProcessStatus(UserCode, Code, ObjectName, ObjectDescription, out &Result)`, `StartProcessStatusSDT(in &SDTProcessStatus, out &Result)` (**recommended**), `EndProcessStatus(UserCode, Code, URL)`, `CancelProcessStatus`, `AddProcessStatusMessage`, `ChkProcessStatusCanceled` (cooperative cancellation), `ChkProcessStatusFinalized`, `DltProcessStatus`.

**Advanced**: `StartProcessStatusSDTWithStorage`, `EndProcessStatusWithStorage(… &StorageItemCollection)` (output to @FileStorage), `ChkProcessStatusHasMessages`, `RetProcessStatusFileStorageCount`.

**In @TaskManager**: `RequestKillProcess(UserCode, Code, out &SDTProcessStatusResponse)`, `KillProcessStatus(…)`, `RetTaskManagerProcessStatusCode(ProgramName, Queue, out &Code)`.

**SDTs**: `SDTProcessStatus` (Code, ObjectName, Parameters, IsCommandLine, AllowCancelationRequest, ServerId) — the startup DTO; `SDTProcessStatusResponse` (HasError, ErrorCode, ErrorDescription).

> The `Sample*`/`ProcessIterationSample` procedures are **reference examples** of the runner pattern (global/per-user, with/without CommandLine, cancellable), not production code.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [taskmanager.md](taskmanager.md) — its runner uses `StartProcessStatusSDT`/`EndProcessStatus` as a per-queue lock; the kill (`RequestKillProcess`/`KillProcessStatus`) lives there.
- The **@FileStorage** (blob output) and **@SystemParameters** modules, and `modules/system.md` (`SystemObjectName`).
