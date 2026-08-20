# Módulo @ProcessMonitor — Monitor de Procesos

> Comportamiento del módulo `@PXTools/@ProcessMonitor`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@ProcessMonitor` (`APIs/Basic/`, `APIs/Advanced/`, `Personalized/`).
- Cualificador: `PXTools.ProcessMonitor`.
- El **lock/kill** de procesos colgados vive en el módulo hermano `@TaskManager` (que es el consumidor principal).
- **Depende de:** `@APIs` (base), `@System`, `@SystemParameters`, `@SendMails`, `@FileStorage`, `@TaskManager` (kill-process). Infra: `@Menus`.

## 1. Qué provee

Un **registro de procesos en ejecución** que sirve de **mutex de concurrencia**, **log persistente** de avance, **detección y kill de procesos colgados**, y particionado por **servidor/nodo**. Es la base sobre la que los runners batch (p. ej. el de @TaskManager) evitan doble ejecución, reportan progreso y pueden cancelarse/matarse.

## 2. Concepto central: `ProcessStatus` como mutex

`ProcessStatus` = "un proceso en ejecución". Su PK (`UserCode + Code`) identifica el proceso; solo **uno** con la misma PK puede estar en `Running` a la vez → es el **lock**. Cada proceso lleva su log (`ProcessStatusMessages`) y opcionalmente un `ServerId`.

```
Ciclo de vida (ProcessStatusStatus):
  [New] ─Start─▶ Running
   Running ─End────────────▶ Ended      ─Dlt▶
   Running ─Cancel─────────▶ Cancelled  ─Dlt▶
   Running ─RequestKill────▶ KillRequested ─Kill(async)▶ Killed ─Dlt▶
   Running ─Verificación (proceso no vivo en el SO)─▶ Ended (forzado)
```

## 3. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **ProcessStatus** | `ProcessStatusUserCode, ProcessStatusCode` (UserCode vacío = global) | El **mutex**. `ObjectName`/`Description` (subtipos de `SystemObjectName`), `Status` (`ProcessStatusStatus`), `ChangeStatusDateTime` (base para detectar cuelgues), `URL` (salida), `IsCommandLine`, `AllowCancelationRequest`, `Parameters`, `ServerId` (FK), `FileStorageId` (salida en blob, Advanced). |
| **ProcessStatusMessages** | `UserCode, Code, MessageId` | Log persistente (1:N): `DateTime`, `Description` (MaxMem). |
| **ProcessServers** | `ProcessServerId` | Catálogo de servidores/nodos de ejecución (colas por servidor). `ProcessServerName`. |

## 4. Dominios del módulo

Propios (nombre `ProcessStatus*`), ambos **root-legacy** (viven en el `#Domains/` raíz). No existe dominio `ProcessServer*` — `ProcessServers` es una transacción:

| Dominio | Valores |
|---|---|
| **ProcessStatusStatus** | Running=`RUN`, Ended=`END`, Cancelled=`CAN`, Killed=`KIL`, KillRequested=`KRQ` (finales: END, KIL) |
| **ProcessStatusErrorCode** | ProcessNotFoundRunning=`NFR`, CannotKillProcess=`CKP`, Other=`OTH` (también lo consume @TaskManager) |

## 5. Mecanismo

### 5.1 Registro + lock — `StartProcessStatus[SDT]`
Al arrancar, el runner llama `StartProcessStatus`/`StartProcessStatusSDT`:
- Verifica `ChkProcessStatusFinalized`: si existe un registro previo NO finalizado, la variante `_SDT` rechaza (`Result=False`, "Duplicate process running") — **es el mutex**.
- Da de alta/actualiza a `Running`, sella `ChangeStatusDateTime`, limpia URL y **purga los mensajes anteriores** (log limpio). `…SDT` guarda además ObjectName, Parameters, IsCommandLine, ServerId, y registra el objeto (`AddSystemObject`).

### 5.2 Avance (log) y cierre
- `AddProcessStatusMessage(UserCode, Code, texto)` — agrega una línea al log persistente.
- `EndProcessStatus(UserCode, Code, URL)` → `Ended`, calcula duración, agrega mensaje de cierre y **envía mail** al usuario. `CancelProcessStatus` → `Cancelled` (sin mail).

### 5.3 Cancelación/kill de colgados (vive en @TaskManager)
- **Cooperativa**: el runner consulta `ChkProcessStatusCanceled` en sus iteraciones y sale solo.
- **Detección de colgados**: `PrcProcessMonitorVerification` (Main, CommandLine) recorre los `Running` `IsCommandLine` del server, verifica en el SO (Linux `ps -aux | grep java`) si el proceso vive; si no, fuerza `Ended` + `PrcProcessMonitorAfterEnds`.
- **Kill explícito (2 fases, async)**:
  1. `RequestKillProcess` (@TaskManager) → marca `KillRequested` y encola `TskProcessMonitorKillProcess` (cola `ServerProcesses` si hay ServerId, si no `Infrastructure`).
  2. `KillProcessStatus` (@TaskManager, corre en el server) → localiza el PID (`ps -aux`), `kill -9`, reverifica, marca `Killed`; usa `ProcessStatusErrorCode`.

### 5.4 Cómo lo usa el runner de @TaskManager
`PrcTaskManagerExecution` arma `SDTProcessStatus` con `Code = RetTaskManagerProcessStatusCode(objeto, queue)`, `IsCommandLine=True`, `ServerId = RetTaskManagerQueueProcessServerId(queue)`; `StartProcessStatusSDT` → si `Result` procesa la cola (reporta con `AddProcessStatusMessage`), al terminar `EndProcessStatus`; si `Result=False` loguea "is running yet" (evita doble runner sobre la misma cola).

## 6. APIs vs Personalized

- **`APIs/Basic/`** + **`APIs/Advanced/`** (core): las transacciones, `Start/End/Cancel/Del`, `AddProcessStatusMessage`, los `Chk*`, y las variantes `…WithStorage` (salida a @FileStorage).
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | `PrcProcessMonitorAfterEnds` (Procedure) | **Hook** invocado cuando la verificación fuerza un proceso a `Ended`. Default: `TestAndKillTaskManager` (reintegra al TaskManager). |
  | `RetMenusProcessMonitor` (DataProvider) | Ítems de menú (Servers / Monitor). |

## 7. Instancias de patterns

| Instancia | Qué es |
|---|---|
| **PXWorkWithProcessStatus** | WW sobre ProcessStatus. |
| **PXWorkWithProcessStatusMessages** | Popup (`PuProcessStatusMessages`) — visor del log de un proceso. |
| **PXWorkWithProcessServers** | WW de servers. |
| **PXWorkWithProcessStatusAdvanced** | **Monitor operativo**: grid con filtro My/Global (Global solo admin, `PIsAdministrator`), acciones OpenDocument (descarga la salida), ViewMessages, y **CancelProcess** polimórfica (Cancel / RequestKill / End / Delete según estado+tipo+permiso). |

## 8. APIs clave

**Basic**: `StartProcessStatus(UserCode, Code, ObjectName, ObjectDescription, out &Result)`, `StartProcessStatusSDT(in &SDTProcessStatus, out &Result)` (**recomendada**), `EndProcessStatus(UserCode, Code, URL)`, `CancelProcessStatus`, `AddProcessStatusMessage`, `ChkProcessStatusCanceled` (cancelación cooperativa), `ChkProcessStatusFinalized`, `DltProcessStatus`.

**Advanced**: `StartProcessStatusSDTWithStorage`, `EndProcessStatusWithStorage(… &StorageItemCollection)` (salida a @FileStorage), `ChkProcessStatusHasMessages`, `RetProcessStatusFileStorageCount`.

**En @TaskManager**: `RequestKillProcess(UserCode, Code, out &SDTProcessStatusResponse)`, `KillProcessStatus(…)`, `RetTaskManagerProcessStatusCode(ProgramName, Queue, out &Code)`.

**SDTs**: `SDTProcessStatus` (Code, ObjectName, Parameters, IsCommandLine, AllowCancelationRequest, ServerId) — DTO de arranque; `SDTProcessStatusResponse` (HasError, ErrorCode, ErrorDescription).

> Los procs `Sample*`/`ProcessIterationSample` son **ejemplos de referencia** del patrón runner (global/usuario, con/sin CommandLine, cancelable), no producción.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [taskmanager.md](taskmanager.md) — su runner usa `StartProcessStatusSDT`/`EndProcessStatus` como lock por cola; el kill (`RequestKillProcess`/`KillProcessStatus`) vive ahí.
- Módulos **@FileStorage** (salida en blob), **@SystemParameters**, y `modulos/system.md` (`SystemObjectName`).
