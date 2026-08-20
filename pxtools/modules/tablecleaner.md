# Módulo @TableCleaner — Limpieza Programada de Tablas

> Comportamiento del módulo `@PXTools/@TableCleaner`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@TableCleaner` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.TableCleaner`.
- **Depende de:** `@APIs` (base), `@DynamicCallReferences` (despacho `code → cleaner`), `@SystemParameters` (límites de borrado), `@Menus`. Lo dispara `@TaskManager`.

## 1. Qué provee

**Purga programada de datos antiguos por tabla**: una configuración por proceso define la antigüedad de retención; una tarea batch recorre las configuraciones y **despacha dinámicamente** el proceso de limpieza de cada módulo (por nombre, vía @DynamicCallReferences). Es la infraestructura común de "table cleaner" que reutilizan @SendMails, @ReceiveMails, @FileStorage, @Alerts, @WebServicesLog, @TaskManager, etc.

## 2. Concepto central

- **`TableCleanerConfiguration`** = una fila por proceso de limpieza registrado, con su **retención** (Years/Months/Days) y límite de lote.
- Su PK (`TableCleanerProcessCode`) es una **referencia dinámica** de tipo `TableCleanerProcess`; la URL del proceso concreto se hereda de `TDynamicCallReferences`.
- **`TskTableCleaner`** (batch) itera las configuraciones, calcula la **fecha de corte** y llama a cada `PrcTableCleaner<X>` por nombre.

## 3. Transacción `TableCleanerConfiguration`

| Atributo | Tipo | Rol |
|---|---|---|
| `TableCleanerProcessCode` (PK) | `DynamicCallReferenceCode` | FK (subtipo) a `TDynamicCallReferences` (tipo `TableCleanerProcess`). |
| `TableCleanerProcessDescription` / `…URL` | inferidos | Heredados de la referencia dinámica. |
| `TableCleanerYears` / `Months` / `Days` | num | Antigüedad de retención (se computa la fecha de corte). |
| `TableCleanerMaxRows` | `IdFirstLevel` (nullable) | Límite de lote (o el global). |
| `TableCleanerEnableCounter` | Boolean | ¿Reportar conteos? |

> Grupo `TableCleanerDynamicCallReference`: `TableCleanerProcessCode : DynamicCallReferenceCode` (+ Description/URL). La config **no** guarda la URL: la hereda del catálogo de referencias dinámicas.

## 4. Dominios del módulo

@TableCleaner **no define dominios propios** (no existe ningún dominio `TableCleaner*` — `TableCleanerConfiguration` es una transacción). Usa de otros módulos:

| Dominio | Dueño | Uso |
|---|---|---|
| `ReferenceType` (valor `TableCleanerProcess`) | @DynamicCallReferences | Tipo de la referencia del cleaner |
| `DynamicCallReferenceCode` | @DynamicCallReferences | Códigos de cleaner: `TskTableCleaner`, `TableCleanerFileStorage`, `TableCleanerWebServicesLog`, `TableCleanerAlerts`, … (uno por tabla/módulo) |
| `SystemParameterCode` (`TableCleanerMaxRowsToDeleteDB/SDT`) | @SystemParameters | Límites de borrado |
| `SDTCounterType` (Failed=`FAI`, Succeed=`SUC`) | @APIs base | Tipo del contador de resultado (anclado en el SDT `SDTCounter` de @APIs; lo usan muchos módulos) |

## 5. Mecanismo

### 5.1 Ejecución — `TskTableCleaner`
`Parm(in: &TaskManagerId, out: &Error, out: &ErrorMessage)` (registrado como `TaskManagerExecution`):
1. `&InitialDate = ServerNow`; `&MaxRowsToDelete = RetMaxRowsToDeleteDB`.
2. `For Each TableCleanerConfiguration` (⋈ `TDynamicCallReferences`): calcula `&Date = &InitialDate.AddYears(-Years).AddMonths(-Months).AddDays(-Days)`, resuelve el `MaxRows` (fila o global), y hace la **llamada dinámica por nombre**:
   ```
   Call(TableCleanerProcessURL, TableCleanerProcessCode, &Date, &TableCleanerMaxRows, &TableCleanerEnableCounter, &SDTCounter)
   ```
   (`TableCleanerProcessURL` = `.Type` del `PrcTableCleaner<X>`). Acumula el `&SDTCounter` en el reporte.

### 5.2 Cómo un módulo registra su cleaner
1. Escribe un `PrcTableCleaner<X>` con el **contrato**: `Parm(in: &DynamicCallReferenceCode, in: &Date, in: &MaxRowsToDeleteDB, in: &EnableCounter, out: &SDTCounter)`. Borra registros con fecha `<= &Date`, lotea por `RetMaxRowsToDeleteSDT` (default 3000), hace `Commit` al superar `MaxRowsToDeleteDB`, y devuelve `SDTCounter`.
2. Provee un `RetDynamicCallReference<X>` (en su `Personalized/`) con un item `{ Code = DynamicCallReferenceCode.TableCleaner<X>, Type = ReferenceType.TableCleanerProcess, URL = PrcTableCleaner<X>.Type }`.
3. `AddDynamicCallReferences` (de @DynamicCallReferences) agrega ese DP y hace upsert en `TDynamicCallReferences`. Desde ahí el proceso se puede configurar/ejecutar.

## 6. APIs vs Personalized

- **`APIs/`** (core): la transacción, `TskTableCleaner` (orquestador), `PrcTableCleaner` (upsert de config), y los helpers de límites (`RetMaxRowsToDeleteDB/SDT`, `RetCountRowsToDeleteDB/SDT`).
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | `RetDynamicCallReferenceTableCleaner` (DataProvider) | Registra la task `TskTableCleaner` en @TaskManager. |
  | `RetSystemParametersTableCleaner` (DataProvider) | `TableCleanerMaxRowsToDeleteDB`/`…SDT`. |
  | `RetMenusTableCleaner` (DataProvider) | Entrada de menú. |

> Cada **módulo consumidor** aporta su propio par `RetDynamicCallReference<X>` + `PrcTableCleaner<X>` en su propio `Personalized/`, no acá.

## 7. Instancia de pattern

**PXWorkWithTableCleanerConfiguration** — WW de configuraciones (filtro por `TableCleanerProcess`; grilla con Years/Months/Days/MaxRows/EnableCounter editables). Acciones: **"Update Processes"** (`DelDynamicCallReferences` + `AddDynamicCallReferences(True)` — refresca el registro de referencias) y **Apply** (`PrcTableCleaner` por fila — upsert de la config).

## 8. Procedimientos / APIs clave

| Proc | `Parm()` | Propósito |
|---|---|---|
| `TskTableCleaner` | `in: &TaskManagerId, out…` | Entry point de @TaskManager; despacha cada cleaner. |
| `PrcTableCleaner` | `in: &Code, &Years, &Months, &Days, &MaxRows, &EnableCounter` | Upsert de una config (la **borra** si Years=Months=Days=0). |
| `RetMaxRowsToDeleteDB` / `…SDT` | `out: &MaxRowsToDelete` | Límites globales (default 100000 / 3000). |
| **Contrato cleaner** `PrcTableCleaner<X>` | `in: &Code, &Date, &MaxRowsToDeleteDB, &EnableCounter; out: &SDTCounter` | Firma que todo cleaner concreto implementa. |

**SDT de retorno** `SDTCounter` (`PXTools.APIs`): colección `{ Table, Type (SDTCounterType), Counter, ErrorMessage }`.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [dynamiccallreferences.md](dynamiccallreferences.md) — el `TableCleanerProcessCode` es una referencia dinámica; la URL sale de ahí.
- [taskmanager.md](taskmanager.md) — ejecuta `TskTableCleaner` (cola `TableCleaner`).
- Consumidores: [sendmails.md](sendmails.md), [filestorage.md](filestorage.md), [alerts.md](alerts.md), [webserviceslog.md](webserviceslog.md), etc.
