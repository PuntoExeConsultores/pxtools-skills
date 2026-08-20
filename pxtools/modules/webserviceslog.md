# Módulo @WebServicesLog — Log y Estadísticas de Web Services

> Comportamiento del módulo `@PXTools/@WebServicesLog`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@WebServicesLog` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.WebServicesLog`.
- **Depende de:** `@APIs` (base), `@TableCleaner` (purga del log, vía `RetCountRowsToDeleteDB`), `@SystemParameters`, `@DynamicCallReferences`, `@Menus`.

## 1. Qué provee

Registra **cada invocación entrante a un web service** (request/response, tiempos, estado, IP) en un log crudo, y luego **agrega** esos registros en tablas de estadísticas mediante una tarea batch.

## 2. Concepto central: log crudo → estadísticas

- **`WebServicesLog`** = una fila por llamada (con `Duration` calculada por fórmula).
- Una tarea batch con **cursor incremental** buckea el log por intervalos de N minutos y lo acumula (`counter += 1`) en:
  - **`WebServicesStatistics`** (por FilterData) — cabecera + detalle por RemoteAddress/WS/Método/Status.
  - **`WebServicesRAStatistics`** — por **Remote Address** (IP).

> **`RA` = Remote Address** (estadística por IP), **no** "rolling average". El motor es incremental con bucketeo por intervalos fijos, no medias móviles.

## 3. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **WebServicesLog** | `WebServiceLogId` | **Log crudo**: `StartDateTime`/`EndDateTime`, `RemoteAddress` (IP), `WSName`, `MethodName`, `MethodInParameters`/`MethodOutParameters` (MaxMem ~30 KB, JSON), `FilterData` (clave de negocio libre), `Status` (`WebServiceLogStatus`), `Duration` (fórmula `tdiff(End, Start)`). BC: `WebServicesLogBC`. |
| **WebServicesStatistics** | `FilterData, DateTime` (+ nivel Detail: `RemoteAddress, WSName, MethodName, Status`) | Agregados por FilterData; contadores. |
| **WebServicesRAStatistics** | `RemoteAddress, DateTime` | Agregados por IP; contador. |

## 4. Dominios del módulo

Todos `WebService*`/`WebServices*` → @WebServicesLog (nomenclatura). Los de estado son **root-legacy** (`#Domains/` raíz); los tipos de dato y de panel ya son **module-scoped** (`@WebServicesLog/#Domains/`):

| Dominio | Scope | Valores |
|---|---|---|
| **WebServiceLogStatus** | root-legacy | WithoutResponse=`WOR`, Success=`SUC`, Failed=`FAI` |
| **WebServiceStatisticDetailStatus** | root-legacy | + Denied=`DEN` (en la agregación) |
| **WebServicesStatisticsCounterType** | root-legacy | FilterData=`FDA`, CounterType=`FCO` |
| **WebServicesStatisticsPanelType** | module | FilterData=`FDA`, RemoteAddress=`RAS` (selector de vista) |
| **WebServiceFilterData / …MethodName / …RemoteAddress / …WSName** | module | Tipos base VarChar(40) |
| **WebServiceStatisticCounter** | module | Numeric(10.0) — valor del contador |
| **StatisticFilterDuring** | module | *(placeholder — dominio declarado sin cuerpo)* |

## 5. Mecanismo

### 5.1 Registrar una invocación
Lo invoca **cada objeto WS consumidor** (NO la capa @WSLayer), típicamente con `Stub`:
```
Stub Metodo(...)
    &WebServiceLogId = AddWebServiceLog.Udp(&Pgmname, !"Metodo", &in.ToJson(), &filter)
    ...  // lógica
    UpdWebServiceLog.Call(&WebServiceLogId, WebServiceLogStatus.Success, &out.ToJson())
EndStub
```
- `AddWebServiceLog(in: &WSName, &MethodName, &MethodInParameters, &FilterData; out: &WebServiceLogId)` — `New` con `StartDateTime=ServerNow()`, `RemoteAddress=RetHTTPRemoteAddress()`, inputs, `Status=WithoutResponse`.
- `UpdWebServiceLog(in: &WebServiceLogId, &Status, &MethodOutParameters)` — `EndDateTime`, outputs y estado final. `Duration` se calcula sola.
- Variantes: `UpdWebServiceLogWithFilterData`, `UpdWebServiceLogFilterData`. `RetWebServicesLogStatusFromRestCode` mapea código REST → Success/Failed.

### 5.2 Calcular estadísticas — `TskWebServicesLogStatistics`
`Parm(in: &TaskManagerId, out: &Error, out: &ErrorMessage)`:
- Gate por SystemParameter `WebServiceLogGenerateStatistics`; período en minutos (`RetWebServiceLogStatisticCounterPeriod`, default 60); **cursor incremental** `WebServiceLogLastStatisticId`.
- `For Each WebServicesLog Where Id > lastId` (solo lo nuevo). **Bucketeo temporal**: minutos desde medianoche / período, redondeado, de vuelta a DateTime.
- Upsert (`When None New`) de la cabecera `WebServicesStatistics` (FilterData+bucket), el `Detail` (RemoteAddress/WS/Método/Status, mapeando a `Denied`/Failed/Success/WithoutResponse) y `WebServicesRAStatistics` (RemoteAddress+bucket); `counter += 1`.
- Guarda el nuevo `LastStatisticId`. Auxiliares: `RetWebServices[RA]StatisticsOtherColumns` (hasta 20 períodos previos para el "historial en columnas"). Limpieza: `PrcTableCleanerWebServicesLog`/`…Statistics`.

## 6. APIs vs Personalized

- **`APIs/`** (core): las transacciones, `Add/UpdWebServiceLog`, `TskWebServicesLogStatistics`, los `Ret*`/`PrcTableCleaner*`.
- **`Personalized/`** (3 DataProviders):
  | Objeto | Qué se customiza |
  |---|---|
  | `RetDynamicCallReferenceWebServicesLog` | Registra en @TaskManager: Task de estadísticas + 2 Table Cleaners. |
  | `RetMenusWebServicesLog` | Menú (Logs / Statistics / Counters by Filter Data). |
  | `RetSystemParametersWebServicesStatistics` | `WebServiceLogGenerateStatistics`, `WebServiceLogLastStatisticId`, `WebServiceLogStatisticCounterPeriod`. |

## 7. Instancias de patterns

- **PXWorkWithWebServicesLog** — grilla del log (filtros por fecha/FilterData/Status/Método + búsqueda en parámetros). Acción **View** → `PXParameterRequestWebServiceLogView`.
- **PXParameterRequestWebServiceLogView** — popup con In/Out Parameters.
- **PXComposerWebServicesLog** — layout selección + vista de parámetros embebida.
- **PXWorkWithWebServicesStatistics** / **PXWorkWithWebServicesRAStatistics** — grillas de agregados (por FilterData / por IP), con "historial en columnas" y cross-link entre ambas.
- **PXComposerWebServicesStatisticCounters** — embebe el nivel de counters.

## 8. Procedimientos / APIs clave

| Proc | `Parm()` | Propósito |
|---|---|---|
| `AddWebServiceLog` | `in: &WSName, &MethodName, &MethodInParameters, &FilterData; out: &WebServiceLogId` | Abre el log al iniciar la llamada. |
| `UpdWebServiceLog` | `in: &WebServiceLogId, &Status, &MethodOutParameters` | Cierra la llamada (fin, output, estado). |
| `UpdWebServiceLogWithFilterData` | `+ &FilterData` | Igual + FilterData. |
| `RetWebServicesLogStatusFromRestCode` | `in: &RestCode; out: &Status` | Mapea código REST → Success/Failed. |
| `TskWebServicesLogStatistics` | `in: &TaskManagerId; out…` | Batch de agregación. |
| `PrcTableCleanerWebServicesLog` / `…Statistics` | `(cleaner)` | Purga log / estadísticas por fecha. |

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [taskmanager.md](taskmanager.md) — ejecuta `TskWebServicesLogStatistics`.
- Módulo **@WSLayer** — capa de web services (whitelist); el logging es **ortogonal** (lo llama cada objeto WS, no @WSLayer).
- Módulo **@SystemParameters** (config de agregación).
