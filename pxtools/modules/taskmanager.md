# Módulo @TaskManager — Gestor de Tareas de PXTools

> Comportamiento del módulo `@PXTools/@TaskManager`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@TaskManager/`
  - `APIs/` — framework (transacciones, runner, procs). **Intocable.**
  - `Personalized/` — customización del proyecto (colas habilitadas, tareas invocables, menú, table cleaner).
  - `#Domains/` — un dominio propio (`FindSubjectIn`); el resto de los dominios `TaskManager*` y `Cycle*` están en el `#Domains/` **raíz** de la KB.
- El objeto compilado del runner es `APrcTaskManagerExecution` (de `PrcTaskManagerExecution`).
- **Depende de:** `@APIs` (base), `@DynamicCallReferences` (resolución `code → objeto`), `@ProcessMonitor` (lock/log), `@SystemParameters` (bloqueos por cola), `@Menus` (`RetMenusTaskManager`), `@SendMails` (referencia calificada en esta KB). En el grafo canónico también `@System`.

## 1. Qué provee

Un **planificador/ejecutor de tareas por cola** (job scheduler + runner). Permite:

- Definir **tareas** puntuales (una vez) o **cíclicas** (Daily/Weekly/Monthly, con repetición intra-ciclo).
- **Encolar** ejecuciones y que un **runner de línea de comandos** (uno por cola) las dispare cuando vencen.
- Resolver **qué objeto GeneXus ejecutar** de forma desacoplada, vía un **código** (`DynamicCallReferenceCode`) que el módulo `DynamicCallReferences` mapea al objeto real.
- **Reintentos**, **ciclos**, **repeticiones**, **estados**, **jerarquía padre/hijo** y **logging por ejecución** (integrado con `@ProcessMonitor`).

Es la infraestructura sobre la que corren los **procesos batch** de la aplicación (envío/recepción de correos, tareas programadas, limpieza de datos, integraciones con servicios externos, etc.).

## 2. Concepto central: tarea → ejecuciones, y código → objeto

Dos ideas clave:

1. **Una tarea (`TaskManager`) genera N ejecuciones (`TaskManagerExecutions`).** La tarea es la *definición* (qué, con qué parámetros, en qué cola, con qué ciclo/reintentos); cada corrida concreta es una fila de ejecución con su estado, fechas y mensaje. La PK de la ejecución es compuesta: `(TaskManagerId, CycleId, RepeatId, ExecutionId)` — modela ciclos y repeticiones dentro del ciclo.

2. **El objeto a ejecutar NO está hardcodeado**: la tarea guarda un `TaskManagerExecutionCode` (dominio `DynamicCallReferenceCode`); el runner lo resuelve al objeto GeneXus real vía el módulo **`DynamicCallReferences`** (`PPEXE_DeDynamicCallReferenceURL`), que se alimenta de DataProviders `RetDynamicCallReference*` — el del framework es `Personalized/RetDynamicCallReferenceTaskManager` y cada aplicación agrega el suyo con sus propias tareas.

> **Contrato de toda tarea invocable:** `Parm(in: &TaskManagerId, out: &Error TaskManagerExecutionResponse, out: &ErrorMessage)`. El runner llama al objeto con el `TaskManagerId` y espera de vuelta un `TaskManagerExecutionResponse` (`Succeed`/`Retry`/`Fail`) + un mensaje. El objeto lee sus parámetros con `RetTaskManagerParameter*`.

### 2.1 Tipos de tarea: Once, Cíclica y A demanda

Hay tres formas de usar una tarea, según **cómo y quién** la crea:

- **Una sola vez (`Once`)** — se agenda para una fecha/hora y se ejecuta una única vez. Se crea **desde la interfaz gráfica** (el WW de tareas): se ingresa una vez y luego se evalúa/monitorea desde su ejecución.
- **Cíclica** — se repite según un patrón de calendario. También se crea **desde la interfaz gráfica**, donde se define todo el proceso de repetición de las ejecuciones (tipo de ciclo, días/semanas/meses, repetición intra-ciclo, vencimiento). **El diseño de esa configuración se basó en el formato del *Programador de tareas de Windows* (Task Scheduler).**
- **A demanda** — la crea **el propio programa**, llamando a la API `AddTaskManagerSDT` (§5.1). Ahí se define qué objeto ejecutar (`ExecutionCode`), qué parámetros pasar, cuándo ejecutarla (`Date`), y la cantidad de reintentos y el tiempo entre reintentos si falla. Es la forma de "disparar trabajo en background" desde el código.

> Las tareas **Once** y **cíclicas** se dan de alta y administran por **UI**; las tareas **a demanda** nacen del **código** y su seguimiento se hace desde el mismo WW (por cola, estado y — clave — por `FilterData`, ver §2.2).

### 2.2 `FilterData` — identificar tareas a demanda por entidad

`TaskManagerFilterData` (`VarChar(40)`) es un campo pensado para **buscar tareas a demanda dirigidas a una entidad concreta** del sistema. Como una misma tarea (mismo `ExecutionCode`) suele encolarse muchas veces —una por cada registro/entidad—, **el programador que invoca `AddTaskManagerSDT` debe setear el `FilterData`** con un valor que identifique la entidad destino de esa ejecución. Así, los administradores pueden luego **filtrar en el WW** y diferenciar las N instancias de la misma tarea asociadas a distintas entidades/registros del sistema.

> (La cola `ServerProcesses` usa además el `FilterData` para el `ProcessServerId`; ver §5.2.)

## 3. Transacciones del módulo

| Transacción | PK | Rol en el modelo |
|---|---|---|
| **TaskManager** | `TaskManagerId` (autonum) | **Tabla maestra de tareas.** Definición: subject, execution/visualization code + parámetros (JSON), config de reintentos, config de ciclo, cola, filter data, estado agregado, jerarquía (`TaskManagerParentId` auto-FK). |
| **TaskManagerExecutions** | `TaskManagerId, TaskManagerExecutionCycleId, TaskManagerExecutionRepeatId, TaskManagerExecutionId` | **Bitácora de ejecuciones** (subordinada). Estado, tipo (Task/Retry/Cycle/Repeat), fechas planned/start/end, `TaskManagerExecutionMessage` (log), cola. |
| **TaskManagerQueues** | `TaskManagerQueueId` (dom. `TaskManagerQueue`) | **Registro de colas.** Asocia cada cola a un `ProcessServer` (`TaskManagerQueueProcessServerId`, subtype group → `@ProcessMonitor`). |
| **TaskManagerReferences** | `TaskManagerId, TaskManagerReferenceId` | Pares **id/valor** adjuntos a una tarea (API `AddTaskManagerReferenceValue` / `RetTaskManagerReferenceValue`). |
| **TaskManagerBC** / **TaskManagerExecutionsBC** | (BC) | Business Components sobre las **mismas tablas físicas**, con niveles anidados; se usan para insert/delete programático (p. ej. desde `PrcTableCleanerTaskManager`). |

### 3.1 Atributos relevantes de `TaskManager`

- **Ejecución:** `TaskManagerExecutionCode` (`DynamicCallReferenceCode` → objeto a correr), `TaskManagerExecutionParameters` (`MaxStr`, JSON de parámetros). Análogos `…Visualization…` para el objeto de "ver resultado".
- **Reintentos:** `TaskManagerRetry` (Boolean), `TaskManagerRetryDay/Hour/Minute/Second` (lapso), `TaskManagerRetryTimes` (máx.).
- **Ciclo:** `TaskManagerCycleType` (`CycleType` Once/Daily/Weekly/Monthly), `…CycleTimes`, `…CycleWeekDays/Months/MonthDays/MonthsWeeks` (JSON), `…CycleRepeat` + `…CycleRepeatEveryMinutes/EveryHours/ForMinutes/ForHours` (repetición intra-ciclo), `…CycleExpire`.
- **Encolado/estado:** `TaskManagerQueue` (Nullable; **null = Main**), `TaskManagerFilterData` (`VarChar(40)`; identifica la **entidad destino** de una tarea a demanda para poder buscarla — ver §2.2), `TaskManagerStatus`, `TaskManagerEnable`, `TaskManagerCreatedBy` (System/User), formulas `TaskManagerExecuting`/`…CountExecuting`.
- **Histórico:** `TaskManagerHistoryClear`, `…HistoryWithMessage`/`…WithoutMessage` (días de retención, default 90).

### 3.2 Diagrama de relaciones

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

## 4. Dominios del módulo

Enum = **valor almacenado**. **Nota (legacy):** salvo `FindSubjectIn` (module-scoped), todos los dominios propios viven en el `#Domains/` **raíz** de la KB — legacy de versiones GeneXus (Evo1/2/3) previas a los dominios asociados a módulo; conceptualmente pertenecen a @TaskManager (solo objetos de @TaskManager los referencian vía `DataType`).

**Propios — enumerados:**

| Dominio | Tipo | Valores |
|---|---|---|
| **TaskManagerExecutionResponse** | Char(20) | `Succeed=SUC`, `Retry=RET`, `Fail=FAI` — **contrato de retorno de toda tarea**. Lo consumen los ~10 módulos que implementan tareas (todos dependen de @TaskManager); la ref en `@APIs/…/PDummy` es un *dummy* de tipos, no ownership. |
| **TaskManagerStatus** | Char(3) | Active=`ACT`, Suspended=`SUS`, Succeed=`SUC`, Fail=`FAI` (estado agregado de la tarea) |
| **TaskManagerExecutionStatus** | Char(3) | Executing=`EXE`, Suspended=`SUS`, Succeed=`SUC`, Fail=`FAI`, KillRequested=`KIR` |
| **TaskManagerExecutionType** | Char(3) | Task=`TSK`, Retry=`RTY`, Cycle=`CYC`, Repeat=`REP` (no hay valor "OnDemand": *a demanda* mapea a `Task`) |
| **TaskManagerCreatedBy** | Char(3) | SystemTask=`SYS`, UserTask=`USR` |
| **TaskManagerParameterType** | Char(3) | Execution=`EXE`, Visualization=`VIS` |
| **TaskManagerTime** | VarChar(40) | Day=`DAY`, Hour=`HRS`, Minute=`MIN`, Second=`SEC` |
| **CycleType** | Char(3) | Once=`ONE`, Daily=`DAI`, Weekly=`WEE`, Monthly=`MON` |
| **CycleMonthsWeeksOrDays** | Char(3) | Days=`DAY`, Weeks=`WEE` (modo del ciclo mensual: día fijo vs. semana-ordinal + día) |
| **CycleDays** | Num(1) | Sunday=`1` … Saturday=`7` |
| **CycleWeeks** | Num(1) | First=`1` … Latest=`5` (ordinal de semana del mes) |
| **CycleMonths** | Num(2) | January=`1` … December=`12` |
| **CycleMonthsDays** | Num(2) | `1` … `31`, Last_Day=`33` (día del mes, incl. "último día") |
| **FindSubjectIn** *(module-scoped)* | Char(20) | `Task`, `Execution` — el WW busca el texto en el subject de la tarea o en el mensaje |
| **TaskManagerQueue** | Char(20) | Conjunto de colas disponibles. Estándar del framework: `Main`, `SendMails`, `ReceiveMails` (+ `…WithAlias`/`…WithoutAlias`), `Statistics`, `TableCleaner`, `Infrastructure`, `Auxiliary`, `ImportExport`, `ServerProcesses`, `Free`; más las colas propias de cada aplicación (las habilita el DataProvider `RetTaskManagerQueues`, ver §6). |

**Propios — de valor (sin enum):**

| Dominio | Tipo | Rol |
|---|---|---|
| **TaskManagerLog** | LongVarChar(2M) | Texto del log/mensaje de una ejecución. |

**Usa dominios de otros módulos** (documentados en el doc de su dueño):

| Dominio | Dueño | Rol en @TaskManager |
|---|---|---|
| **DynamicCallReferenceParameters** | [@DynamicCallReferences](dynamiccallreferences.md) | Valor de parámetro pasado a una tarea llamada dinámicamente. Por nomenclatura `DynamicCallReference*` es de @DynamicCallReferences (aunque en esta KB solo lo consuma @TaskManager). |
| **ReferenceType** | [@DynamicCallReferences](dynamiccallreferences.md) | Categoría de la referencia dinámica (`TaskManagerExecution`, `TaskManagerVisualization`, …). |
| **DynamicCallReferenceCode** | [@DynamicCallReferences](dynamiccallreferences.md) | Código lógico del objeto a ejecutar (`ExecutionCode`/`VisualizationCode`). |
| **ProcessStatusErrorCode** | [@ProcessMonitor](processmonitor.md) | Códigos de error del lock de concurrencia. |
| **SystemParameterCode** | [@SystemParameters](systemparameters.md) | Códigos de los parámetros de bloqueo (`TaskManagerBlocked…`). |
| **PXToolsDate** | @APIs | Dominio de fecha (formato largo). Prefijo `PXTools*` = namespace del framework → base @APIs. |
| *(base)* `Boolean`, `MaxStr`, `VarLen`, `IdFirstLevel`, `Links`, `Name`, `Path`, … | @APIs | Tipos base del framework. |

## 5. Modelo de ejecución (el núcleo)

### 5.1 Encolar — `AddTaskManagerSDT`

`Parm(in: &SDTTaskManager, out: &TaskManagerId)`. Hace `New` sobre `TaskManager` (status `Suspended`, `CreatedBy=SystemTask`), serializa parámetros y colecciones de ciclo a JSON, normaliza `Queue` (Main→null), y crea la **primera ejecución** con `AddTaskManagerExecution(&TaskManagerId, 1, 1, &Date, TaskManagerExecutionType.Task, &Date, &Queue)`.

El **SDT `SDTTaskManager`** es la API de encolado: `Id, Date, Subject, ExecutionCode, ExecutionParameters` (colección), `VisualizationCode/…Parameters`, `RetryDay/Hour/Minute/Second/Times`, `ParentId`, `CycleType/…`, `Queue, FilterData`.

`AddTaskManagerExecution` materializa una fila `TaskManagerExecutions` en `Suspended`; si ya hay una `Suspended`, adelanta su fecha si la nueva es mayor.

### 5.2 Runner / daemon — `PrcTaskManagerExecution`

`[IsMain='True', CallProtocol='CommandLine']`, `Parm(in: &TaskManagerExecutionQueue, in: &ProcessServerId)`. Un scheduler externo (cron/servicio) lo invoca **una instancia por cola**, típicamente **cada minuto** — de ahí que la granularidad mínima de agendado sea de 1 minuto. Ciclo:

1. **Bloqueos**: SystemParameter `TaskManagerBlocked` (global) y `TaskManagerBlocked<Cola>` (por cola). Si activo, no procesa.
2. **Lock de concurrencia**: `StartProcessStatusSDT` (`@ProcessMonitor`); si ya corre, aborta.
3. **Selección**: `For Each` TaskManager⋈Executions ordenado por `Queue, Status, PlannedDate`, filtrando la cola (Main = `Queue.IsNull()`; `ServerProcesses` además por `FilterData = &ProcessServerId`), `Enable = True`, `ExecutionStatus = Suspended`, `PlannedDate <= ServerNow` (solo lo vencido).
4. **Marca en curso**: status tarea → `Active`, ejecución → `Executing`.
5. **Resuelve el objeto**: `&Process = PPEXE_DeDynamicCallReferenceURL.Udp(TaskManagerExecutionCode)`.
6. **Invoca** (en try/catch): `Call(&Process, TaskManagerId, &Error, &ErrorMessage)`.
7. **Interpreta** `Do Case &Error`: `Succeed` → marca Succeed y agenda **próxima ejecución** (`RetTaskManagerNextExecution`); `Retry` o excepción → `'Retry Execution'`; `Fail` → marca Fail.
8. **Reintento**: si `RetTaskManagerExecutionCount < TaskManagerRetryTimes`, agenda una ejecución `Retry` a `ServerNow + Retry…`.
9. **Persiste + loguea**: `UpdTaskManagerExecutionStatus` (estado + mensaje + fechas), `ChkTaskManagerStatus` (recalcula estado agregado y propaga al padre), `AddProcessStatusMessage` (log persistente en ProcessMonitor).

**Ciclos**: `RetTaskManagerNextExecution` decide próxima repetición (`RetTaskManagerNextRepeat`) o próximo ciclo (`RetTaskManagerNextCycle`, respetando días/semanas/meses y `CycleExpire`).

### 5.3 Leer parámetros dentro de la tarea

La tarea recibe solo `TaskManagerId`; lee sus parámetros del JSON con `RetTaskManagerParameterString/Integer/Date/Boolean(in: &TaskManagerId, in: &Position, out: &Valor)` (Position 1-based).

## 6. APIs vs Personalized

### `APIs/` (framework — no tocar)
Todo el motor: transacciones (`TaskManager*`), BCs, el runner `PrcTaskManagerExecution`, los `Add*/Ret*/Upd*/Chk*`, las instancias de patterns (ABM), y las tareas de mantenimiento del propio módulo (`TskDeleteTaskManagerExecutions`, `TskTaskManagerForceRetryByQueue`, `TskKillProcess`).

### `Personalized/` (punto de extensión — se customiza por proyecto)
| Objeto | Qué define / cómo se customiza |
|---|---|
| **`RetTaskManagerQueues`** (DataProvider) | **Qué colas están habilitadas** en el proyecto (combos de UI + genera los SystemParameters de bloqueo por cola). Cada proyecto agrega aquí, además de las colas estándar del framework, las colas propias de sus flujos batch. |
| **`RetDynamicCallReferenceTaskManager`** (DataProvider) | **Qué objetos del propio framework son invocables** por el TaskManager (mapeo `code → objeto`): `TskDeleteTaskManagerExecutions`, `TskProcessMonitorKillProcess`→`TskKillProcess`, `TskTaskManagerForceRetryByQueue`, y un `TableCleanerTaskManagerQueue<Cola>` por cola → todos a `PrcTableCleanerTaskManager`. |
| **`RetMenusTaskManager`** (DataProvider) | Menú "Task Manager" bajo `Basic`: **Queues** (`TrTaskManagerQueues`) y **Tasks** (`TrTaskManager`). |
| **`PrcTableCleanerTaskManager`** (Procedure) | **Purgador de histórico** por cola (referenciado por todos los `TableCleanerTaskManagerQueue*`). Borra tareas `Once` vencidas y ejecuciones antiguas (preservando el último ciclo para no romper el Retry); antes de borrar, anula las FKs de negocio que apunten a la tarea. |

> **Nota:** el mapeo `code → objeto` de las **tareas de la aplicación** (no las del framework) NO vive acá: cada aplicación lo declara en su propio DataProvider `Personalized` (por convención `RetDynamicCallReference<App>`), que también alimenta a `DynamicCallReferences` (ver §9).

## 7. Instancias de patterns PXTools

| Instancia | Qué es |
|---|---|
| **PXWorkWithTaskManager** | ABM completo de tareas. Transaction con tabs **General** (Queue, Subject, ExecutionCode como Dynamic Combo filtrado por `ReferenceType.TaskManagerExecution`, parámetros vía el prompt `PXParameterRequestUpdTaskManagerParameters`, bloque Retry) y **Cycle Settings**. Selection en 2 niveles (`TaskManager` básico / `TaskManagerAdvanced` con búsqueda dentro de mensajes). View "Task Information" con grid de **Executions** (Retry, ViewMessage, Kill&Retry, ExecuteNow, Download log) y árbol de **Childs Tasks**. |
| **PXWorkWithTaskManagerQueues** | WW simple (basado en la colección `SDTTaskManagerQueues`) para **asignar un ProcessServer a cada cola** (`UpdTaskManagerQueueProcessServer`). |
| **PXParameterRequestExecutionMessage** | Popup "View Message": muestra el `TaskManagerExecutionMessage` (log) de una ejecución en solo lectura. |
| **PXParameterRequestUpdTaskManagerParameters** | Prompt "Edit parameters": editor de la lista de parámetros `;`-separada (Add/Remove item). |

## 8. Procedimientos / APIs clave

**Encolar / ejecuciones**
- `AddTaskManagerSDT(in: &SDTTaskManager, out: &TaskManagerId)` — alta de tarea + primera ejecución.
- `AddTaskManagerExecution(...)` — crea/adelanta una ejecución `Suspended`.
- `AddTaskManagerExecutions` / `AddTaskManagerExecutionsAndReset` — re-encola la tarea (y descendientes fallidas); la variante *Reset* limpia el contador de reintentos. Usadas por la acción "Retry".
- `AddTaskManagerReferenceValue` — adjunta par id/valor.

**Lectura**
- `RetTaskManagerParameterString/Integer/Date/Boolean(in: &TaskManagerId, in: &Position, out)` — parámetro N desde el JSON.
- `RetTaskManagerNextExecution` / `…NextCycle` / `…NextRepeat` — cálculo de scheduling.
- `RetTaskManagerExecutionCount` — reintentos reales (excluye ignorados).
- `RetTaskManagerQueue`, `RetTaskManagerReferenceValue`, `RetTaskManagerHasChilds`, `RetTaskManagerExecutionMessageLinkToFile` (vuelca el log a `.txt`).

**Estado / control**
- `UpdTaskManagerStatus`, `UpdTaskManagerEnable`, `UpdTaskManagerExecutionStatus` (estado + mensaje + fechas), `UpdTaskManagerExecutionsNow` (Execute Now).
- `ChkTaskManagerStatus` — recalcula el estado agregado y propaga al padre.

**Kill / fail-and-retry**
- `RequestKillProcess`, `SetFailAndRetryTaskManager`, `TestAndKillTaskManager`, `KillProcessStatus` (integración con `@ProcessMonitor`).

## 9. Patrón de integración con la aplicación

TaskManager es el **orquestador batch** del sistema. Una aplicación integra sus procesos así:

1. **Declarar colas** — agregar las colas propias en `Personalized/RetTaskManagerQueues`. Dedicar una cola a un tipo de trabajo permite correr su runner en paralelo/aislado (y bloquearla independientemente con `TaskManagerBlocked<Cola>`).
2. **Registrar las tareas** — en el DataProvider `Personalized` de la aplicación (`RetDynamicCallReference<App>`), mapear cada `DynamicCallReferenceCode` a su proc con `DynamicCallReferenceURL = <Proc>.Type` y `ReferenceType.TaskManagerExecution`. Esto es lo que resuelve el runner al ejecutar.
3. **Encolar** — construir el `SDTTaskManager` (con `ExecutionCode`, parámetros, `Queue`, config de reintentos, y opcionalmente `FilterData` para agrupar) y llamar `AddTaskManagerSDT`. También se puede dar de alta una tarea **cíclica** desde el WW.
4. **Implementar la tarea** — el proc cumple el contrato `Parm(in: &TaskManagerId, out: &Error TaskManagerExecutionResponse, out: &ErrorMessage)`: lee sus parámetros con `RetTaskManagerParameter*`, hace su trabajo, y devuelve `Succeed` / `Retry` (reintentar según la config) / `Fail`. El `&ErrorMessage` es el **log** que queda en la ejecución (visible en el WW y en `PXParameterRequestExecutionMessage`).

### How-to: hacer una tarea "llamable" por el TaskManager
1. Crear el proc con el contrato de 3 parámetros (leer parámetros con `RetTaskManagerParameter*`).
2. Dar de alta un valor en el dominio `DynamicCallReferenceCode`.
3. Registrarlo en el DataProvider `Personalized` de la aplicación (`RetDynamicCallReference<App>`) mapeando el code → `<Proc>.Type` con `ReferenceType.TaskManagerExecution`.
4. Encolarlo con `AddTaskManagerSDT` (o darlo de alta como tarea cíclica en el WW), indicando la cola.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- Módulo **DynamicCallReferences** — mapeo `code → objeto` (resuelto por `PPEXE_DeDynamicCallReferenceURL`).
- Módulo **@ProcessMonitor** — lock de concurrencia, ProcessServers y log persistente (`AddProcessStatusMessage`).
- `modulos/security.md` — módulo @Security (contexto `&Context`, políticas de acceso).
