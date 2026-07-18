# Módulo @CloudTasks — Tareas Programadas del Servidor

> Comportamiento del módulo `@PXTools/@CloudTasks`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@CloudTasks/` (`APIs/` con subcarpetas por tipo: `General`, `Certificate`, `DeleteFiles`, `PorcessMonitor` *(sic, typo en disco)*, `Debug`; + `Personalized/`).
- Cualificador: `PXTools.CloudTasks`.
- **Depende de:** `@APIs` (base), `@ProcessMonitor` (vigilancia de procesos), `@FileStorage`, `@Alerts` (notifica fallos), `@DynamicCallReferences`, `@SystemParameters`, `@Menus`. Lo dispara `@TaskManager`.

## 1. Qué provee

Un framework de **tareas recurrentes del lado servidor**, tipadas: verificar/instalar líneas **cron**, verificar/crear **directorios** y **archivos**, instalar/validar **certificados** en keystores, **monitorear procesos** colgados, y **limpiar archivos** temporales. Cada tarea tiene estado, ventana de vigencia y opción de alerta. No corre por sí misma: la dispara **@TaskManager**.

## 2. Concepto central

Una "cloud task" es un registro en `CloudTasks` **tipado por `CloudTaskType`**. Un **dispatcher** (`PrcExecuteTask`) hace `Do Case` sobre el tipo y delega en el verificador correspondiente. El **disparo** es externo (dos ejecutores registrados en @TaskManager: uno diario, uno de alta frecuencia).

## 3. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **CloudTasks** (BC) | `CloudTaskId` (autonum) | **Tabla base** de una tarea (atributos comunes + subtipos Cron y Certificado). |
| **CloudTasksBC** | `CloudTaskId` | BC "headless" sobre la misma tabla (uso programático). |
| **CloudTaskDeleteFiles** / **CloudTasksCertificates** | `CloudTaskId` | Vistas/extensiones de la tabla base que fijan `CloudTaskType` (DeleteFiles / CheckCertificateToSign). |
| **KeyStores** | `KeyStoreId` (+ nivel `StoreCertificate`) | Almacén de claves: archivo, password (`PswEnc`), tipo (JKS/PFX), JSON cache, y detalle de certificados instalados (alias, vigencia, serial, issuer). |
| **CloudTasksCertificatesEstructure** | `CloudTaskId` + `KeyStore` | Tabla de vínculo N–N tarea-certificado ↔ keystore. |
| **CloudTasksProcessMonitor** | `CloudTaskProcessMonitorId` | Tarea de vigilancia de procesos. |

**Atributos clave de `CloudTasks`:** `CloudTaskType`, `CloudTaskStatus`, `CloudTaskStatusMessage`, `CloudTaskPendingExecution` (ejecución diferida al próximo batch), `CloudTaskStartDateTime`/`…EndDateTime` (vigencia), `CloudTaskSendAlert`, `CloudTaskPath`, `CloudTaskFileStorageId`; grupo **Cron** (`…CronSpecialString/User/ExecuteAs/Minutes/Hours/Days/Months/Weekdays`); grupo **Certificado** (`…CertKeyStoreId`, `…CertPrivateKeyAlias`, `…CertPrivateKeyPassword` PswEnc, `…CertContent`, `…CertPublicKey…`).

```
CloudTasks ──┬─ CloudTaskFileStorageId ─▶ FileStorage        (@FileStorage)
             └─ CloudTaskCertKeyStoreId ─▶ KeyStores
KeyStores 1──N StoreCertificate                              (certificados instalados)
CloudTasksProcessMonitor ─ UserCode+ProcessCode ─▶ ProcessStatus  (@ProcessMonitor)
```

## 4. Dominios del módulo

Todos `CloudTask*`/`Cron*`/`KeyStoreType` → @CloudTasks (nomenclatura). Los de la tabla son **root-legacy** (`#Domains/` raíz); solo `DeleteFilesProcessOperation` ya es **module-scoped** (`@CloudTasks/#Domains/`):

| Dominio | Scope | Valores |
|---|---|---|
| **CloudTaskType** | root-legacy | CheckCronTask=`CronTask`, CheckDirectory=`CheckDir`, CheckFileExistance=`CheckFile`, CheckCertificateToSign=`CheckCSign`, ProcessMonitor=`ProcessMonitor`, DeleteFiles=`DeleteFiles` |
| **CloudTaskStatus** | root-legacy | Enabled=`ENA`, Disabled=`DIS`, Finalized=`FIN`, Suspended=`SUS`, Uninstall=`UNI`, WithError=`ERR` |
| **CloudTaskLogLevel** | root-legacy | None, Resume, Detail |
| **CloudTaskCerticateAction** | root-legacy | Add=`ADD`, Remove=`REM`, NA=`NA` |
| **CloudTaskDeleteFilesRemoveType** | root-legacy | Internal=`INT`, CommandLine=`CMD` |
| **CloudTaskErrorCode** | root-legacy | FileNotFound, KeyStoreBadPass, CertBadPassword, MissingKeyStoreInfo, CronTask*, CantCreateDirectory, DirectoryNotFound, ProcessMonitorKill/Check, Other, … |
| **KeyStoreType** | root-legacy | JKS, PFX |
| **CronSpecialString** | root-legacy | @reboot, @yearly, @monthly, @weekly, @daily, @midnight, @hourly, … |
| **CronUser** | root-legacy | *(sin enum — usuario del crontab)* |
| **DeleteFilesProcessOperation** | **module** | Select, Delete |

## 5. Mecanismo

### 5.1 Alta
Vía UI (el WW enruta a la transacción específica según el tipo) o `AddCloudTask(in: &SDTCloudTask, …, out: &CloudTaskId)`. Según el system-parameter `InstallCertificateOnBatch`, la tarea nace `Suspended` con `CloudTaskPendingExecution=True` (se ejecuta en el próximo batch) o se ejecuta ya (`ExecuteTaskOnDemand`).

### 5.2 Dispatcher — `PrcExecuteTask`
`Do Case CloudTaskType` → delega al verificador del tipo (patrón uniforme `PrcCheck*` que valida con `Chk*` y aplica con `Add*`):
- **CheckCronTask** → escribe la línea cron en el archivo del usuario dentro del crontab configurado.
- **CheckDirectory / CheckFileExistance** → crea el directorio / verifica el archivo.
- **CheckCertificateToSign** → instala/valida el certificado en el keystore.
- **ProcessMonitor** → detecta procesos que exceden el límite hs/min y pide kill (`RequestKillProcess`/`KillProcessStatus` de @ProcessMonitor).
- **DeleteFiles** → `PrcCheckDeleteFiles` → `PrcDeleteFileOperation`.

Tras ejecutar, actualiza `CloudTaskStatus`/`…StatusMessage` (`WithError` / `Finalized` si expiró / `Enabled`) y limpia `CloudTaskPendingExecution`.

### 5.3 Disparo (scheduler) — vía @TaskManager
No hay daemon propio. `Personalized/RetDynamicCallReferenceCloudTask` registra en @DynamicCallReferences dos ejecutores `TaskManagerExecution` (`parm(in: &TaskManagerId, out: &TaskManagerExecutionResponse, out: &ErrorMessage)`):
- **`TskCloudTasksQuickly`** (alta frecuencia): ProcessMonitor (mata colgados) + procesa certificados `PendingExecution`.
- **`TskCloudTasksDaily`** (diario): habilita `Suspended` cuyo Start llegó, finaliza expiradas, ejecuta todas las `Enabled` (`PrcExecuteTask`), refresca el JSON de keystores y remueve certificados vencidos.

### 5.4 Certificados / keystores
Para tareas de firma, instala un certificado (PFX/JKS) en un `KeyStore` (`CloudTaskCertKeyStoreId`), guarda el detalle en `KeyStores.StoreCertificate` y en la tabla de vínculo, cachea contenido (`KeyStoreContentJSON`), alerta de "por vencer"/"vencido" y permite desinstalar. Passwords cifradas (`PswEnc`).

### 5.5 Limpieza de temporales (DeleteFiles)
`PrcDeleteFileOperation` (`IsMain=True`) selecciona/borra archivos bajo un path (filtro por nombre, antigüedad, recursividad), en modo **Internal** (API `Directory`/`File`) o **CommandLine** (shell `find`). Registra según `CloudTaskLogLevel`.
> ⚠️ **Caveat:** en modo CommandLine el `-delete` del `find` está **comentado** (`APIs/DeleteFiles/PrcDeleteFileOperation.gxSource` ~L185) → hoy **no borra** en ese modo.

### 5.6 Debug / simulación
- `PXWorkWithSimulateDeleteFiles` (`PuSimulateDeleteFiles`): dry-run (`Operation=Select`) — lista lo que se borraría.
- `DebugCloudTaskOnDemand` / `DebugCloudTasksUtils` (protegidos por `CloudTaskEnableDebugProgramms`): ejecutar/eliminar una tarea, correr Daily/Quickly forzando fecha/IDs, y utilidades de keystore/certificado.

## 6. APIs vs Personalized

- **`APIs/`** (core): las transacciones, el dispatcher `PrcExecuteTask`, los verificadores por tipo (`PrcCheck*`/`Chk*`/`Add*`), los ejecutores `TskCloudTasks*`, la maquinaria de certificados/keystores y de DeleteFiles, los SDTs.
- **`Personalized/`** (DataProviders del proyecto):
  | Objeto | Qué se customiza |
  |---|---|
  | `RetMenuCloudTasks` | Ítems de menú (Tasks / Key Stores / Debug). |
  | `RetSystemParametersCloudTasks` | System-parameters del módulo (paths de crontab/log/working-dir, parties de alerta, flags de batch/debug, tipo de log, días de aviso previo…). |
  | `RetSystemAlertCategoryTypesCloudTask` | Categorías de alerta que el módulo publica a @Alerts. |
  | `RetDynamicCallReferenceCloudTask` | Registro en @TaskManager de `TskCloudTasksDaily`/`…Quickly`. |

## 7. Instancias de patterns

- **PXWorkWith**: `PXWorkWithCloudTasks` (WW principal — Enable/Disable/Finalize/Run/View Message; enruta el alta según tipo), `PXWorkWithCloudTasksCertificates`, `PXWorkWithKeyStores`, `PXWorkWithCloudTasksProcessMonitor`, `PXWorkWithCloudTaskDeleteFiles`, `PXWorkWithSimulateDeleteFiles`.
- **PXParameterRequest**: `PXParameterRequestCloudTaskStatusMessage` (ver el mensaje largo), `DebugCloudTaskOnDemand`, `DebugCloudTasksUtils`.

## 8. Procedimientos / APIs clave

**Ciclo de vida**: `AddCloudTask` / `UpdCloudTask` / `DelCloudTask` / `RetCloudTask`, `RetCloudTaskStatusAndMessage`, `PrcChangeCloudTaskStatus(in: &CloudTaskId, in: &NewStatus, out…)`, `PrcExecuteTask` (dispatcher), `ExecuteTaskOnDemand`, `TskCloudTasksDaily`/`…Quickly`, `SendAlert` (→ @Alerts), `SetSystemMessages` (logging).

**Verificadores por tipo** (todos `(in: &CloudTaskId, in: &DateTime, out: &SDTCloudTaskResponse)`): `PrcCheckCronTask`/`ChkCronTask`/`AddCronTask`, `PrcCheckDirectory`/…, `PrcCheckFileExistance`/…, `PrcCheckProcessMonitor`, `PrcCheckCertificateToSign`, `PrcCheckDeleteFiles`.

**DeleteFiles**: `PrcDeleteFileOperation(in: &SDTProcessDeleteFilesInput, out: &Resume, out: &PairCollection, out…)`.

**Certificados/keystores**: `AddCertificateTask`, `PrcRemoveCertificateFromKeyStore`, `RemoveCertificateAlreadyExpired`, `RegenerateCertificateAlias`, `UpdKeyStoreCertificatesInfo`, `UpdKeyStoreJson`, firma `FirmaTextFromPFX`/`FirmaTextFromJKS`, lecturas `RetCloudTaskPublicKey`, `RetKeyStorePassword`/`SetKeyStorePassword`, `CheckIfAliasExistOnKeyStoreByCommand`.

**SDTs núcleo**: `SDTCloudTask` (payload con subgrupos Cron/Certificate/Common), `SDTCloudTaskResponse` (`HasError`, `ErrorCode`, `ErrorDescription`), `CloudTaskPartyData`, `SDTProcessDeleteFilesInput`, familia `SDTCertificateInfo*`.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [taskmanager.md](taskmanager.md) — dispara `TskCloudTasksDaily`/`…Quickly` (vía `RetDynamicCallReferenceCloudTask`).
- Módulos **@ProcessMonitor** (kill de procesos colgados), **@Alerts** (alertas de certificado/cron), **@FileStorage** (contenido/certificados), **@SystemParameters** (config).
