# Módulo @DynamicCallReferences — Invocación Dinámica por Nombre

> Comportamiento del módulo `@PXTools/@DynamicCallReferences`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@DynamicCallReferences` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.DynamicCallReferences`.
- **Depende de:** `@APIs` (base), `@Menus` (`RetMenusDynamicCallReferences`).

## 1. Qué provee

Un registro que asocia un **código lógico** (`DynamicCallReferenceCode`) a la **identidad de runtime de un objeto GeneXus** (`Objeto.Type`, guardado en `DynamicCallReferenceURL`). Así otros módulos invocan objetos **"por nombre"** con `Call(&url, …)` **sin hardcodear** a qué objeto llaman. Es la pieza transversal que desacopla "qué código ejecutar" del objeto concreto — la usan @TaskManager (`ExecutionCode`), @TableCleaner, @Alerts, @CloudTasks, etc.

## 2. Concepto central: code → objeto

- La tabla `TDynamicCallReferences` mapea `Code → URL` (donde `URL = ObjetoReal.Type`).
- En runtime: `&url = PPEXE_DeDynamicCallReferenceURL.Udp(code)` → `Call(&url, params)` (llamada **indirecta/polimórfica** resuelta en runtime).
- La tabla se **siembra declarativamente**: cada módulo aporta un DataProvider `RetDynamicCallReference<X>` que devuelve sus referencias.

## 3. Transacción `TDynamicCallReferences`

Un solo nivel; BC.

| Atributo | Tipo | Rol |
|---|---|---|
| `DynamicCallReferenceCode` (PK) | `DynamicCallReferenceCode` (Char(50) enum) | Clave lógica del "punto de llamada". |
| `DynamicCallReferenceType` | `ReferenceType` (Char(50) enum) | Categoría/subsistema (define qué contrato espera el `Call`). |
| `DynamicCallReferenceDescription` | `ObjectDescription` | Descripción legible. |
| `DynamicCallReferenceURL` | `Links` | Identidad de runtime del objeto destino (`Objeto.Type`). |

SDT espejo `SDTDynamicCallReferences` (colección de esos 4 miembros) = **contrato de siembra**.

## 4. Dominios del módulo

Los tres dominios `DynamicCallReference*` + `ReferenceType` son de @DynamicCallReferences (por nomenclatura), todos **root-legacy** (viven en el `#Domains/` raíz). Nota: aunque `DynamicCallReferenceParameters` en esta KB solo lo consuma @TaskManager, **es de este módulo** por nomenclatura.

- **`DynamicCallReferenceCode`** (Char(50)): el **catálogo de "nombres"** lógicos de objetos invocables (tareas, table-cleaners, generadores, DPs, prompts OAV…). Cada módulo agrega sus valores.
- **`ReferenceType`** (Char(50)): clasifica **para qué subsistema** sirve la referencia (y por tanto la firma esperada). Valores del framework: `TaskManagerExecution` (`TskExecution`), `TaskManagerVisualization`, `TableCleanerProcess`, `StatisticLogProcess`, `StatisticIdCompositionDP`, `OAVAttributeValuesPrompt`, `OAVAttributeValueValidation`, `OAVSrvDynamicReadOnly`, `FormularioPDFPublico`/`…Privado`. El consumidor filtra la tabla por este tipo.
- **`DynamicCallReferenceParameters`** (Char(50)): valor de parámetro que se pasa a un objeto invocado dinámicamente.

> El objeto destino se guarda como `Objeto.Type` (nombre calificado) en `DynamicCallReferenceURL`: usar el **nombre calificado** mantiene la referencia estable ante renombres del objeto.

## 5. Mecanismo

### 5.1 Resolución — `PPEXE_DeDynamicCallReferenceURL`
`Parm(in: &DynamicCallReferenceId, out: &DynamicCallReferenceURL)` — `For Each Where Code = &Id → &URL = DynamicCallReferenceURL`. Devuelve el `.Type` del objeto.

### 5.2 Invocación
`&url` (tipo `Attribute:DynamicCallReferenceURL`, dominio `Links`) contiene el `.Type`; `Call(&url, params)` es la llamada indirecta. Ejemplos:
- **@TaskManager** (`PrcTaskManagerExecution`): `&Process = PPEXE_DeDynamicCallReferenceURL.Udp(TaskManagerExecutionCode)` → `Call(&Process, TaskManagerId, &Error, &ErrorMessage)` (tipo esperado `TaskManagerExecution`).
- **@TableCleaner**: resuelve por **subtipos** (denormaliza la URL en `TableCleanerConfiguration` vía el Group `TableCleanerDynamicCallReference`) → `Call(TableCleanerProcessURL, …)`.

### 5.3 Siembra (upsert)
Cada módulo aporta `RetDynamicCallReference<X>` (`Output = SDTDynamicCallReferences`) con items:
```
SDTDynamicCallReferencesItem {
    DynamicCallReferenceCode        = DynamicCallReferenceCode.TskXxx
    DynamicCallReferenceType        = ReferenceType.TaskManagerExecution
    DynamicCallReferenceDescription = "Task Xxx"
    DynamicCallReferenceURL         = TskXxx.Type      // ← .Type del objeto real
}
```
El upsert lo hace `AddDynamicCallReference(in: &SDTDynamicCallReferences)` (`New … When Duplicate …`).

## 6. APIs vs Personalized

- **`APIs/`** (core): la transacción, `PPEXE_DeDynamicCallReferenceURL` (resolver), `AddDynamicCallReference` (upsert), `DelDynamicCallReferences`.
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | **`AddDynamicCallReferences`** (Procedure) | El **agregador maestro**: llama `.Udp()` de cada `RetDynamicCallReference<X>` de todos los módulos, hace upsert, y **poda** las filas cuyo código ya no declara ningún provider. Aquí el proyecto registra sus DPs. |
  | `RetDynamicCallReferenceExample` (DataProvider) | Plantilla comentada de cómo declarar referencias. |
  | `RetMenusDynamicCallReferences` (DataProvider) | Entrada de menú del WW. |

## 7. Instancia de pattern

**PXWorkWithTDynamicCallReferences** — WW sobre la transacción (grid filtrado por `ReferenceType`). Acción **"Update References"** → `DelDynamicCallReferences` + `AddDynamicCallReferences(True)` (re-siembra manual desde la UI).

## 8. Procedimientos / APIs clave

| Proc | `Parm()` | Propósito |
|---|---|---|
| `PPEXE_DeDynamicCallReferenceURL` | `in: &Code; out: &URL` | **Resolver** code → `.Type` para `Call(&url, …)`. |
| `AddDynamicCallReference` | `in: &SDTDynamicCallReferences` | Upsert de un conjunto de referencias. |
| `DelDynamicCallReferences` | — | Vacía la tabla (antes de re-sembrar). |
| `AddDynamicCallReferences` (Personalized) | `in: &ShowMessages` | Re-siembra total + poda de obsoletos. |

### How-to: registrar una referencia nueva
1. Agregar el valor al dominio `DynamicCallReferenceCode`.
2. Crear/editar un `RetDynamicCallReference<X>` (`Output = SDTDynamicCallReferences`) que devuelva `{ Code, Type (ReferenceType), Description, URL = ObjetoReal.Type }`.
3. Añadir su `.Udp()` en el agregador `Personalized/AddDynamicCallReferences`.
4. Correr "Update References" (o llamar al agregador) para persistir.

Desde entonces cualquier consumidor invoca el objeto vía `PPEXE_DeDynamicCallReferenceURL.Udp(code)` + `Call(…)`, o por subtipo si el módulo denormaliza la URL (como @TableCleaner).

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- Consumidores: [taskmanager.md](taskmanager.md) (`ExecutionCode`), módulo @TableCleaner (`TableCleanerProcess`), [alerts.md](alerts.md), [cloudtasks.md](cloudtasks.md), [webserviceslog.md](webserviceslog.md).
- [menus.md](menus.md) — la entrada de menú se declara en `RetMenusDynamicCallReferences`.
