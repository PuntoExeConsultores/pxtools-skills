# Módulo @Statistics — Estadísticas del Sistema

> Comportamiento del módulo `@PXTools/@Statistics`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@Statistics` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.Statistics`.
- **Depende de:** `@APIs` (base), `@DynamicCallReferences` (despacho del proceso de cada estadística), `@Menus`. Lo dispara `@TaskManager`; purga vía `@TableCleaner`.

## 1. Qué provee

Un framework para **definir y registrar métricas numéricas por período**: cada estadística se declara (metadato + un proceso que la calcula), y una tarea batch ejecuta esos procesos para poblar una serie temporal de valores, consultable por rango o snapshot.

## 2. Concepto central

- **`StatisticDefinition`** = metadato: código + descripción + el **proceso** que calcula los valores + una **composición de Ids** que etiqueta la sub-dimensión.
- **`StatisticLog`** = serie temporal: un valor numérico por **(tipo, sujeto=ReferenceCode, momento=DateTime, sub-métrica=Ids)**.
- El proceso de cada definición se resuelve **por nombre** (vía @DynamicCallReferences) y lo dispara @TaskManager.

## 3. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **StatisticDefinition** | `StatisticDefinitionCode` (+ nivel `IdComposition`) | Metadato: `Description`, `ProcessCode`/`ProcessURL` (proc que registra los valores, referencia dinámica). Nivel `IdComposition` (posición → etiqueta, con su DP). |
| **StatisticLog** | `StatisticLogCode, StatisticLogReferenceCode, StatisticLogDateTime, StatisticLogIds` | **Valor** (`StatisticLogValue` Numeric(18.4)) por período (granularidad minutos). |
| **StatisticLogBC** (BC) | (igual) | CRUD programático de logs. |

Grupo `StatisticLogCode : StatisticDefinitionCode` (Log ⋈ Definition); grupos que integran `ProcessCode`/`IdDataProviderCode` con `DynamicCallReferenceCode`.

## 4. Dominios del módulo

Todos `Statistic*` → @Statistics (nomenclatura), **root-legacy** (viven en el `#Domains/` raíz):

| Dominio | Valores |
|---|---|
| **StatisticType** | Range=`RAN`, Last=`LAS` (modo de consulta en `RetQuery`) |
| **StatisticDefinitionCode** (Char(50)) | Catálogo **extensible** de tipos de estadística; cada aplicación agrega sus valores. |
| **StatisticMailAccount** (Numeric(1)) | BothEnabled=`0`, POP3Disabled=`1`, SMTPDisabled=`2`, BothDisabled=`3` — métrica de estado de cuentas de correo. Pese a describir cuentas, es de @Statistics (nomenclatura + uso). |

## 5. Mecanismo

### 5.1 Definir una estadística
1. Agregar el valor al enum `StatisticDefinitionCode`.
2. Escribir un proc "proceso" (en `Personalized/`) que compute y grabe los logs; registrarlo como `ReferenceType.StatisticLogProcess` en @DynamicCallReferences.
3. Registrar la fila de `StatisticDefinition` — la declara el DataProvider seed `RetStatisticDefintion` y la inserta `AddStatisticDefinition` (o vía el WW).
4. Opcional: un DataProvider de composición (→ `SDTComposition`) que da etiquetas legibles a cada `Ids`.

### 5.2 Registrar un valor (desde el proc-proceso)
No hay API de "increment" dedicada; el proc acumula y hace un **upsert** sobre `StatisticLog`:
```
New
    StatisticLogReferenceCode = ...
    StatisticLogDateTime      = &ServerNow
    StatisticLogCode          = &StatisticType
    StatisticLogIds           = &SubMetrica
    StatisticLogValue         = &Valor
When Duplicate
    For Each
        StatisticLogValue = &Valor   // update si ya existe
EndNew
```

### 5.3 Ejecución — vía @TaskManager
**`TskStatistics`** (`Parm(in: &TaskManagerId, out: &Error TaskManagerExecutionResponse, out: &ErrorMessage)`) recorre **todas** las `StatisticDefinition` y hace `Call(StatisticDefinitionProcessURL)` (call dinámico) de cada proceso. Se registra como `ReferenceType.TaskManagerExecution`; el scheduler lo dispara. `PrcTableCleanerStatisticDefinition` se engancha a @TableCleaner para purgar logs viejos.

### 5.4 Consultar
`RetQuery(in: &Query SDTStatisticsQueryIn, out: &Data SDTStatisticsQueryOut)`: filtra por `DefinitionCode`+`ReferenceCode` y según `Type`: `Last` (snapshot más reciente) o `Range` (entre `RangeStart`/`RangeEnd`); enriquece con las etiquetas de Ids (`RetSDTComposition`).

## 6. APIs vs Personalized

- **`APIs/`** (core): las transacciones, `RetQuery`, `AddStatisticDefinition`, `TskStatistics`, `RetStatisticDefintion` (seed), `PrcTableCleanerStatisticDefinition`, los SDTs.
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | `RetDynamicCallReferenceStatisticCodes` (DataProvider) | Registra cada proc-proceso del proyecto (`StatisticLogProcess`). |
  | `RetDynamicCallReferenceStatistics` (DataProvider) | Registra `TskStatistics` + los DP de composición + el TableCleaner. |
  | `RetMenusStatistics` (DataProvider) | Menú (Definitions / Logs). |
  | Proc(s) `Prc…` de proceso + DP(s) de composición + `RetSDTComposition` | **La lógica concreta** de cálculo de métricas y sus etiquetas. |

## 7. Instancias de patterns

- **PXWorkWithStatisticDefinition** — WW del catálogo (view "General" + sección grid "Id Composition").
- **PXWorkWithStatisticLog** — WW de la serie de valores (filtros por Code/Reference/rango/Ids/Value).

## 8. Procedimientos / APIs clave

| Objeto | `Parm()` | Propósito |
|---|---|---|
| `RetQuery` | `in: &Query; out: &Data` | Consulta (Last/Range). |
| `AddStatisticDefinition` | — | Puebla `StatisticDefinition`+IdComposition desde el seed. |
| `RetSDTComposition` | `in: &IdDataProviderCode; out: &SDTComposition` | Etiquetas de la dimensión Ids. |
| `TskStatistics` | `in: &TaskManagerId, out…` | Job @TaskManager: ejecuta el proceso de cada definición. |
| `PrcTableCleanerStatisticDefinition` | `(cleaner)` | Purga logs viejos (hook @TableCleaner). |

**SDTs**: `SDTStatisticDefinition`, `SDTStatisticsQueryIn` (DefinitionCode/ReferenceCode/Type/RangeStart/RangeEnd), `SDTStatisticsQueryOut` (DateTime → Values[Id, Value]), `SDTComposition` (Position/Reference).

> **Flujo end-to-end:** @TaskManager dispara `TskStatistics` → recorre `StatisticDefinition` → `Call()` dinámico de cada `ProcessURL` → los procs hacen upsert en `StatisticLog` → el consumo se hace con `RetQuery` → @TableCleaner limpia históricos.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [dynamiccallreferences.md](dynamiccallreferences.md) — resuelve el `ProcessCode` → proc.
- [taskmanager.md](taskmanager.md) — ejecuta `TskStatistics`.
- Módulo @TableCleaner (purga de logs).
