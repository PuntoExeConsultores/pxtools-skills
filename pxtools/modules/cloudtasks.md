# @CloudTasks Module — Scheduled Server-Side Tasks

> Behaviour of the `@PXTools/@CloudTasks` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@CloudTasks/` (`APIs/` with one subfolder per type: `General`, `Certificate`, `DeleteFiles`, `PorcessMonitor` *(sic — a typo on disk)*, `Debug`; plus `Personalized/`).
- Qualifier: `PXTools.CloudTasks`.
- **Depends on:** `@APIs` (base), `@ProcessMonitor` (process watching), `@FileStorage`, `@Alerts` (failure notifications), `@DynamicCallReferences`, `@SystemParameters`, `@Menus`. It is triggered by `@TaskManager`.

## 1. What it provides

A framework of typed **recurring server-side tasks**: checking/installing **cron** lines, checking/creating **directories** and **files**, installing/validating **certificates** in keystores, **monitoring** hung processes, and **cleaning up** temporary files. Each task has a status, a validity window and an optional alert. It does not run by itself: **@TaskManager** triggers it.

## 2. Core concept

A "cloud task" is a row in `CloudTasks` **typed by `CloudTaskType`**. A **dispatcher** (`PrcExecuteTask`) runs a `Do Case` over the type and delegates to the matching checker. The **trigger** is external (two executors registered in @TaskManager: one daily, one high-frequency).

## 3. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **CloudTasks** (BC) | `CloudTaskId` (autonumbered) | The task's **base table** (common attributes + Cron and Certificate subtypes). |
| **CloudTasksBC** | `CloudTaskId` | A "headless" BC over the same table (programmatic use). |
| **CloudTaskDeleteFiles** / **CloudTasksCertificates** | `CloudTaskId` | Views/extensions of the base table pinning `CloudTaskType` (DeleteFiles / CheckCertificateToSign). |
| **KeyStores** | `KeyStoreId` (+ a `StoreCertificate` level) | Key store: file, password (`PswEnc`), type (JKS/PFX), JSON cache, and the detail of installed certificates (alias, validity, serial, issuer). |
| **CloudTasksCertificatesEstructure** | `CloudTaskId` + `KeyStore` | N–N link table between a certificate task and a keystore. |
| **CloudTasksProcessMonitor** | `CloudTaskProcessMonitorId` | Process-watching task. |

**Key `CloudTasks` attributes:** `CloudTaskType`, `CloudTaskStatus`, `CloudTaskStatusMessage`, `CloudTaskPendingExecution` (execution deferred to the next batch), `CloudTaskStartDateTime`/`…EndDateTime` (validity), `CloudTaskSendAlert`, `CloudTaskPath`, `CloudTaskFileStorageId`; the **Cron** group (`…CronSpecialString/User/ExecuteAs/Minutes/Hours/Days/Months/Weekdays`); the **Certificate** group (`…CertKeyStoreId`, `…CertPrivateKeyAlias`, `…CertPrivateKeyPassword` PswEnc, `…CertContent`, `…CertPublicKey…`).

```
CloudTasks ──┬─ CloudTaskFileStorageId ─▶ FileStorage        (@FileStorage)
             └─ CloudTaskCertKeyStoreId ─▶ KeyStores
KeyStores 1──N StoreCertificate                              (installed certificates)
CloudTasksProcessMonitor ─ UserCode+ProcessCode ─▶ ProcessStatus  (@ProcessMonitor)
```

## 4. Module domains

All `CloudTask*`/`Cron*`/`KeyStoreType` domains belong to @CloudTasks (by naming). The table's own are **root-legacy** (root `#Domains/`); only `DeleteFilesProcessOperation` is already **module-scoped** (`@CloudTasks/#Domains/`):

| Domain | Scope | Values |
|---|---|---|
| **CloudTaskType** | root-legacy | CheckCronTask=`CronTask`, CheckDirectory=`CheckDir`, CheckFileExistance=`CheckFile`, CheckCertificateToSign=`CheckCSign`, ProcessMonitor=`ProcessMonitor`, DeleteFiles=`DeleteFiles` |
| **CloudTaskStatus** | root-legacy | Enabled=`ENA`, Disabled=`DIS`, Finalized=`FIN`, Suspended=`SUS`, Uninstall=`UNI`, WithError=`ERR` |
| **CloudTaskLogLevel** | root-legacy | None, Resume, Detail |
| **CloudTaskCerticateAction** | root-legacy | Add=`ADD`, Remove=`REM`, NA=`NA` |
| **CloudTaskDeleteFilesRemoveType** | root-legacy | Internal=`INT`, CommandLine=`CMD` |
| **CloudTaskErrorCode** | root-legacy | FileNotFound, KeyStoreBadPass, CertBadPassword, MissingKeyStoreInfo, CronTask*, CantCreateDirectory, DirectoryNotFound, ProcessMonitorKill/Check, Other, … |
| **KeyStoreType** | root-legacy | JKS, PFX |
| **CronSpecialString** | root-legacy | @reboot, @yearly, @monthly, @weekly, @daily, @midnight, @hourly, … |
| **CronUser** | root-legacy | *(no enum — the crontab user)* |
| **DeleteFilesProcessOperation** | **module** | Select, Delete |

## 5. How it works

### 5.1 Creation
Through the UI (the WW routes to the specific transaction according to the type) or `AddCloudTask(in: &SDTCloudTask, …, out: &CloudTaskId)`. Depending on the `InstallCertificateOnBatch` system parameter, the task is born `Suspended` with `CloudTaskPendingExecution=True` (to be executed in the next batch) or runs immediately (`ExecuteTaskOnDemand`).

### 5.2 Dispatcher — `PrcExecuteTask`
`Do Case CloudTaskType` → delegates to that type's checker (a uniform `PrcCheck*` pattern that validates with `Chk*` and applies with `Add*`):
- **CheckCronTask** → writes the cron line into the user's file inside the configured crontab.
- **CheckDirectory / CheckFileExistance** → creates the directory / verifies the file.
- **CheckCertificateToSign** → installs/validates the certificate in the keystore.
- **ProcessMonitor** → detects processes exceeding the hour/minute limit and requests a kill (@ProcessMonitor's `RequestKillProcess`/`KillProcessStatus`).
- **DeleteFiles** → `PrcCheckDeleteFiles` → `PrcDeleteFileOperation`.

After running, it updates `CloudTaskStatus`/`…StatusMessage` (`WithError` / `Finalized` if expired / `Enabled`) and clears `CloudTaskPendingExecution`.

### 5.3 Triggering (the scheduler) — through @TaskManager
There is no daemon of its own. `Personalized/RetDynamicCallReferenceCloudTask` registers two `TaskManagerExecution` executors in @DynamicCallReferences (`parm(in: &TaskManagerId, out: &TaskManagerExecutionResponse, out: &ErrorMessage)`):
- **`TskCloudTasksQuickly`** (high frequency): ProcessMonitor (kills hung processes) + processes `PendingExecution` certificates.
- **`TskCloudTasksDaily`** (daily): enables `Suspended` tasks whose Start has arrived, finalises expired ones, runs every `Enabled` task (`PrcExecuteTask`), refreshes the keystore JSON and removes expired certificates.

### 5.4 Certificates / keystores
For signing tasks, it installs a certificate (PFX/JKS) into a `KeyStore` (`CloudTaskCertKeyStoreId`), stores the detail in `KeyStores.StoreCertificate` and in the link table, caches the content (`KeyStoreContentJSON`), alerts about "expiring"/"expired" certificates, and allows uninstalling. Passwords are encrypted (`PswEnc`).

### 5.5 Temporary file cleanup (DeleteFiles)
`PrcDeleteFileOperation` (`IsMain=True`) selects/deletes files under a path (filtering by name, age, recursion), in **Internal** mode (the `Directory`/`File` API) or **CommandLine** mode (shell `find`). It logs according to `CloudTaskLogLevel`.
> ⚠️ **Caveat:** in CommandLine mode the `find`'s `-delete` is **commented out** (`APIs/DeleteFiles/PrcDeleteFileOperation.gxSource` ~L185) → today it **does not delete** in that mode.

### 5.6 Debug / simulation
- `PXWorkWithSimulateDeleteFiles` (`PuSimulateDeleteFiles`): a dry run (`Operation=Select`) — it lists what would be deleted.
- `DebugCloudTaskOnDemand` / `DebugCloudTasksUtils` (guarded by `CloudTaskEnableDebugProgramms`): run/delete a task, run Daily/Quickly forcing dates/IDs, and keystore/certificate utilities.

## 6. APIs vs Personalized

- **`APIs/`** (core): the transactions, the `PrcExecuteTask` dispatcher, the per-type checkers (`PrcCheck*`/`Chk*`/`Add*`), the `TskCloudTasks*` executors, the certificate/keystore and DeleteFiles machinery, and the SDTs.
- **`Personalized/`** (the project's DataProviders):
  | Object | What gets customized |
  |---|---|
  | `RetMenuCloudTasks` | Menu items (Tasks / Key Stores / Debug). |
  | `RetSystemParametersCloudTasks` | The module's system parameters (crontab/log/working-dir paths, alert parties, batch/debug flags, log type, days of advance warning…). |
  | `RetSystemAlertCategoryTypesCloudTask` | The alert categories the module publishes to @Alerts. |
  | `RetDynamicCallReferenceCloudTask` | Registration of `TskCloudTasksDaily`/`…Quickly` in @TaskManager. |

## 7. Pattern instances

- **PXWorkWith**: `PXWorkWithCloudTasks` (the main WW — Enable/Disable/Finalize/Run/View Message; routes creation by type), `PXWorkWithCloudTasksCertificates`, `PXWorkWithKeyStores`, `PXWorkWithCloudTasksProcessMonitor`, `PXWorkWithCloudTaskDeleteFiles`, `PXWorkWithSimulateDeleteFiles`.
- **PXParameterRequest**: `PXParameterRequestCloudTaskStatusMessage` (view the long message), `DebugCloudTaskOnDemand`, `DebugCloudTasksUtils`.

## 8. Key procedures / APIs

**Life cycle**: `AddCloudTask` / `UpdCloudTask` / `DelCloudTask` / `RetCloudTask`, `RetCloudTaskStatusAndMessage`, `PrcChangeCloudTaskStatus(in: &CloudTaskId, in: &NewStatus, out…)`, `PrcExecuteTask` (dispatcher), `ExecuteTaskOnDemand`, `TskCloudTasksDaily`/`…Quickly`, `SendAlert` (→ @Alerts), `SetSystemMessages` (logging).

**Per-type checkers** (all `(in: &CloudTaskId, in: &DateTime, out: &SDTCloudTaskResponse)`): `PrcCheckCronTask`/`ChkCronTask`/`AddCronTask`, `PrcCheckDirectory`/…, `PrcCheckFileExistance`/…, `PrcCheckProcessMonitor`, `PrcCheckCertificateToSign`, `PrcCheckDeleteFiles`.

**DeleteFiles**: `PrcDeleteFileOperation(in: &SDTProcessDeleteFilesInput, out: &Resume, out: &PairCollection, out…)`.

**Certificates/keystores**: `AddCertificateTask`, `PrcRemoveCertificateFromKeyStore`, `RemoveCertificateAlreadyExpired`, `RegenerateCertificateAlias`, `UpdKeyStoreCertificatesInfo`, `UpdKeyStoreJson`, signing with `FirmaTextFromPFX`/`FirmaTextFromJKS`, reads through `RetCloudTaskPublicKey`, `RetKeyStorePassword`/`SetKeyStorePassword`, `CheckIfAliasExistOnKeyStoreByCommand`.

**Core SDTs**: `SDTCloudTask` (a payload with Cron/Certificate/Common subgroups), `SDTCloudTaskResponse` (`HasError`, `ErrorCode`, `ErrorDescription`), `CloudTaskPartyData`, `SDTProcessDeleteFilesInput`, the `SDTCertificateInfo*` family.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [taskmanager.md](taskmanager.md) — triggers `TskCloudTasksDaily`/`…Quickly` (through `RetDynamicCallReferenceCloudTask`).
- The **@ProcessMonitor** (killing hung processes), **@Alerts** (certificate/cron alerts), **@FileStorage** (content/certificates) and **@SystemParameters** (configuration) modules.
