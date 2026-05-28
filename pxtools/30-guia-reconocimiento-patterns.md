# Guía de Reconocimiento de Patterns en WebPanels Existentes

## Propósito

Esta guía permite a una IA analizar **WebPanels desarrollados a mano** (sin patterns) y determinar **a qué pattern (o combinación de patterns) de PXTools** se podría migrar cada uno.

## Metodología de análisis

Para cada WebPanel, analizar:
1. **Estructura del layout** (grillas, formularios, tabs, popups)
2. **Comportamiento** (CRUD, consulta, flujo, composición)
3. **Relación con transacciones** (tabla base, navegación)
4. **Acciones disponibles** (botones, links, exports)
5. **Parámetros de entrada/salida**

> **Importante sobre grids**: la sola presencia de un grid no implica PXWorkWith. Hay grids "fantasma" usados solo para perdurar variables SDT (legacy GeneXus Evo1) que deben ignorarse al clasificar. Ver [13-semantica-grids-webpanels.md](13-semantica-grids-webpanels.md) para el detalle de cómo identificar grids reales vs fantasma, multiples grids, y For Each Line.

---

## Señales de reconocimiento por pattern

### PXWorkWith — CRUD Maestro-Detalle

PXWorkWith genera tres tipos de pantalla: **Selection**, **View** y **Prompt**. Es fundamental distinguir Selection de Prompt porque ambos tienen grilla pero su propósito es diferente.

#### PXWorkWith Selection

Pantalla principal de listado CRUD. El usuario navega, filtra, y ejecuta acciones sobre registros.

**Señales en el Form (.gxForm):**
- Grilla principal con datos
- Botón/imagen de búsqueda
- MaxRows en la grilla (paginación)
- Acciones CRUD (Insert, Update, Delete, Display)

**Señales en el Source (.gxSource):**
- No tiene parámetros `out:` en Parm (no retorna valores al llamador)
- Acciones con Link a Transaction para Insert/Update
- Puede tener Delete con confirmación
- Titulo descriptivo del listado (ej: "Facturas", "Comprobantes")

**Señales en el código:**
```
// Señal: For each sobre la tabla base de una transacción
For each
    where <filtros>
    order <atributos>
    // Carga grilla
EndFor

// Señal: Link a transacción con modo Insert/Update
Link(TrnFactura, FacturaId)
```

#### PXWorkWith Prompt

Popup de búsqueda/lookup que el usuario abre para **seleccionar un registro y retornar valores** al llamador. Es un selector de registros, no un data entry.

**Señales en el Form (.gxForm):**
- Grilla con datos (igual que Selection)
- Botón/imagen de búsqueda
- `MaxRows` en la grilla (paginación) — descarta que sea grilla auxiliar de PXParameterRequest
- Una columna con `eventGX="Enter"` o imagen de selección por fila

**Señales en el Source (.gxSource):**
- `Parm` con parámetros `out:` — **indicador definitivo** de que retorna valores al llamador
- Event Enter que carga variables Out y hace `Return`
- Titulo como "Seleccionar \<Entidad\>" — indica selector de registros
- No tiene acciones CRUD (no Insert, no Update, no Delete)

**Señales en el código:**
```
// Señal DEFINITIVA de Prompt: Parm con out
Parm(in:&EmpCod, out:&BanCod, out:&BanDes);

// Señal DEFINITIVA: Event Enter carga Out y retorna
Event Enter
    &BanCod = BanCod
    &BanDes = BanDes
    Return
EndEvent
```

#### PXWorkWith Prompt — Caso especial: grilla cargada desde SDT/WebSession

Un Prompt puede tener su grilla cargada **manualmente** con `Grid.Load` a partir de un SDT o de datos en WebSession, en vez de leer directamente de una tabla base. Esto es muy frecuente en selectores filtrados por contexto previo (por funcionario, por almacén, por tercero, por lote, etc.) y en selectores multi-fila donde el llamador deja datos en WebSession antes de invocar al popup.

**No confundir con grilla auxiliar de PXParameterRequest.** Sigue siendo Prompt si:
- El elemento principal del WebPanel es la grilla (no un formulario tabular de campos editables)
- El propósito es **seleccionar uno o varios registros y retornar al llamador**
- La grilla puede tener `Rows`/`MaxRows` y filtros, aunque sus datos vengan de SDT/WebSession
- El título suele ser "Seleccionar...", "Elegir...", "Lista de..."

**Señales en el código:**
```
// Señal: carga manual del grid desde SDT/WebSession
&SDTBajas.FromJson(&WebSession.Get("BajasAF"))
For &i = 1 to &SDTBajas.Count
    Grid1.Load
EndFor

// Señal: marcar fila(s) y retornar via WebSession (ver caso "Prompt sin Parm out")
Event 'Aceptar'
    &WebSession.Set("BajasAF", &SDTBajas.ToJson())
    &Window.Close()
EndEvent
```

#### PXWorkWith Prompt — Caso especial: retorno por WebSession (sin Parm out)

En KBs legacy (especialmente migradas desde Evolution 1) es común que un Prompt **no use `Parm` con `out:`** y en su lugar deje el resultado de la selección en **WebSession**, para que el llamador lo lea al refrescarse cuando el popup se cierra. Esto es típico en selectores invocados con `target=New` (popup) que terminan con `&Window.Close()` o cierre del form.

**Sigue siendo PXWorkWith Prompt** — el origen del retorno (Parm out vs WebSession) NO determina el pattern; lo que importa es el **propósito**: el WebPanel sirve para que el usuario elija un registro (o varios) y vuelva al llamador con esa elección.

**Señales:**
- Invocado con `target=New` desde otro WebPanel (popup)
- No tiene `Parm` con `out:`, o no tiene `Parm` en absoluto
- Tiene grilla principal con registros (de tabla base, SDT o WebSession)
- Acción "Aceptar"/"Seleccionar" que guarda en WebSession y hace `&Window.Close()` o cierra el form
- El llamador lee la WebSession en su Refresh/Start

**Migrar como Prompt** con la lógica de "guardar selección en WebSession" en `actionPostCode` de la acción de selección.

#### Heurísticas de naming (referencia secundaria)

Como **señal de apoyo** (nunca decisiva por sí sola), los prefijos comunes en KBs GeneXus suelen indicar:

| Prefijo | Pattern probable |
|---------|------------------|
| `Sel*`, `prmt*`, `Prompt*`, `Gx00*` | PXWorkWith Prompt |
| `Con*`, `Vis*`, `Ver*` | PXWorkWith Selection (visor readonly) |
| `Reg*`, `New*`, `Mod*`, `Cam*`, `Ins*` | PXParameterRequest |
| `WW*`, `WB*`, `Wrk*` | PXWorkWith Selection (CRUD) |

**Importante**: estos paneles fueron desarrollados a mano, por lo que puede haber **incoherencias en la nomenclatura**. La heurística por prefijo solo sirve para **confirmar** una clasificación obtenida del análisis del Form/Source, nunca para reemplazarla. Y existen excepciones explícitas: por ejemplo `SelRanRep*` ("Seleccionar Rango para Reporte") es un PXParameterRequest porque captura parámetros (fechas), no un registro.

**Diferencia clave Selection vs Prompt:**

| Criterio | Selection | Prompt |
|----------|-----------|--------|
| Parm con `out:` | NO | **SI** |
| Event Enter que retorna valores | NO | **SI** |
| Acciones CRUD (Insert/Update/Delete) | SI | NO |
| Titulo | Descriptivo del listado | "Seleccionar..." |
| Propósito | Navegar y operar sobre datos | Elegir un registro y retornar |

#### Nodos PXWorkWith correspondientes

| Lo que se ve en el WebPanel | Nodo PXWorkWith |
|----------------------------|-----------------|
| Grilla con registros (Selection) | `selection/grid` |
| Grilla con registros (Prompt) | `prompt/grid` |
| Campo de búsqueda rápida | `filter/search` |
| Filtros avanzados (combos, rangos) | `filter/advancedSearch` |
| Condiciones fijas (WHERE) | `filter/conditions` |
| Ordenamientos | `orders` |
| Botón "Nuevo" | `selection/actions` (Insert) |
| Botón "Editar" en grilla | `selection/actions` (Update) |
| Botón "Eliminar" | `selection/actions` (Delete) |
| Botón "Exportar Excel" | `selection/modes/exportExcel` |
| Pantalla de detalle con tabs | `view` + `view/sections` |
| Tab con datos tabulares | `view/sections/section[@type='Tabular']` |
| Tab con sub-grilla | `view/sections/section[@type='Grid']` |

**Migrable como PXWorkWith si:**
- [x] Tiene grilla con datos de transacción
- [x] Tiene filtros/búsqueda
- [x] Si tiene Parm out + Enter que retorna → **Prompt**
- [x] Si tiene acciones CRUD → **Selection**
- [x] Si tiene grilla readonly (variables/campos no editables) + acción de navegación por fila → **Selection** (visor de datos con detalle)

#### PXWorkWith Selection — Caso especial: Visor readonly con grilla

Un WebPanel con grilla de **solo lectura** (todos los campos readonly/display) también es un PXWorkWith Selection, no un PXParameterRequest. La clave es que el elemento principal es una **grilla con registros**, no un formulario de data entry.

**Señales definitivas de Selection readonly:**
- Grilla principal con `Rows` o `MaxRows` (paginación)
- Todos los campos/variables de la grilla son **readonly** (no editables por el usuario)
- Puede tener datos tabulares arriba de la grilla (encabezado) también **readonly**
- Acción por fila para navegar a detalle (ej: `Ver_Asiento`, `Ver_Detalle`)
- Solo `Parm` con `In:` (no retorna valores, solo consulta)
- Botón Salir/Cerrar como única acción global
- Los datos pueden venir de tabla base (con `#Conditions`) o de SDT/WebSession (con `Grid.Load` manual)

**Señales en el código:**
```
// Señal: Grilla paginada
Grid1.Rows = 10

// Señal: Datos tabulares readonly (encabezado del comprobante)
&EmpNom = EmpNom
&TipoComprobante = CTipComCod.Trim() + " - " + TipComCod.Trim()
&Estado = "Digitado"

// Señal: Acción de navegación por fila (ver detalle)
Event 'Ver_Asiento'
    &Window.Url = Link(PreAsiento, TrnMode.Display, ...)
    &Window.Open()
EndEvent

// Señal: Solo Salir como acción global
Event 'Salir'
    Return
EndEvent
```

**Diferencia clave con PXParameterRequest:**
- PXParameterRequest es un **data entry** — el usuario ingresa/edita datos en campos
- Selection readonly es un **visor** — el usuario solo ve datos y puede navegar a detalle
- Si todos los campos son readonly → NO es data entry → es Selection

---

### PXParameterRequest — Formulario de Parámetros

PXParameterRequest es un **data entry tabular** (formulario), NO una pantalla de búsqueda con grilla.

**Señales principales:**
- WebPanel con formulario tabular de campos (NO grilla principal)
- Botones de Aceptar/Cancelar (o Confirmar/Salir)
- Captura parámetros del usuario o muestra información para confirmar
- Puede tener una grilla **auxiliar** dentro del formulario (pero la grilla NO es el elemento principal)

**Cómo distinguir de PXWorkWith Prompt:**

| Criterio | PXParameterRequest | PXWorkWith Prompt |
|----------|-------------------|-------------------|
| Elemento principal | Formulario tabular (campos) | **Grilla con registros** |
| Grilla | No tiene, o grilla auxiliar pequeña | **Grilla principal con MaxRows y paginación** |
| Botón de búsqueda | No (o busca dentro de grilla auxiliar) | **Sí, busca registros en la grilla principal** |
| Parm out | Puede tener (retorna datos capturados) | **Siempre tiene** (retorna registro seleccionado) |
| Event Enter en grilla | No aplica | **Sí, carga out vars y retorna** |
| Titulo | "Confirmar...", "Ingresar...", "Anular..." | **"Seleccionar..."** |
| Propósito | Capturar datos / confirmar acción | **Buscar y elegir un registro** |

**Señales en el código:**
```
// Señal de PXParameterRequest: Aceptar/Cancelar con lógica de negocio
Event 'Aceptar'
    // Validaciones
    If &Campo.IsEmpty()
        msg("Campo requerido")
        Return
    EndIf
    // Invocación principal
    MiProcedimiento.Call(&Param1, &Param2)
    Return
EndEvent

Event 'Cancelar'
    Return
EndEvent
```

**Tipos de PXParameterRequest detectables:**

| Comportamiento del WebPanel | behaviour type |
|----------------------------|---------------|
| Popup simple con mensaje + Ok | `PopupParameterRequest` |
| Formulario de captura con Aceptar/Cancelar | `ParameterRequest` |
| Panel flotante que no bloquea | `FloatingParameterRequest` |
| Panel embebido sin comportamiento modal | `Panel` o `None` |

**Migrable como PXParameterRequest si:**
- [x] Es un formulario tabular (data entry), no una grilla de búsqueda
- [x] Tiene Aceptar/Cancelar o Confirmar/Salir
- [x] Captura datos del usuario o muestra información para confirmar una acción
- [x] NO tiene grilla principal con búsqueda y paginación (eso es PXWorkWith)

**Caso especial — PXParameterRequest con grilla auxiliar (marcar con \*):**
Si el WebPanel tiene un formulario de data entry como elemento principal PERO además incluye una grilla auxiliar (por ejemplo para mostrar items seleccionados, errores, o detalle), se clasifica como PXParameterRequest pero se marca con asterisco (\*) para revisión manual.

---

### PXComposer — Composición de Pantallas

**Señales principales:**
- WebPanel que contiene múltiples WebComponents embebidos
- Layout tipo dashboard con secciones
- Combina varias funcionalidades en una sola pantalla
- Los WebComponents pueden ser grillas, formularios, u otros paneles

**Señales en el código:**
```
// Señal: Múltiples WebComponents en el layout
// En el WebForm hay varios controles WebComponent
<WebComponent name="wcGrilla1" object="WWFacturas" />
<WebComponent name="wcDetalle" object="ViewCliente" />
<WebComponent name="wcAcciones" object="WbAcciones" />
```

**Migrable como PXComposer si:**
- [x] El WebPanel es principalmente un contenedor de otros paneles
- [x] Usa WebComponents para composición
- [x] Los componentes embebidos podrían ser PXWorkWith, PXParameterRequest u otro PXComposer
- [x] No tiene lógica de negocio propia significativa

---

### PXFlowController — Flujo de Trabajo

**Señales principales:**
- WebPanel que guía al usuario a través de una secuencia de pasos
- Tiene múltiples "pantallas" o estados dentro del mismo WebPanel
- Usa variables de control de flujo (&Step, &Line, &State)
- Acciones que llevan al siguiente/anterior paso
- Confirmaciones intermedias
- Puede abrir popups y continuar según el resultado

**Señales en el código:**
```
// Señal: Control de flujo con variable de paso
Do Case
    Case &Step = 1
        // Mostrar paso 1
        Call(WbConfirmacion, ...)
    Case &Step = 2
        // Procesar
        PrcFacturar(...)
    Case &Step = 3
        // Resultado
EndCase

// Señal: Lógica if-then con popups y continuación
If &Confirmar = "Si"
    // Siguiente paso
    &Step = 2
Else
    // Volver
    &Step = 1
EndIf
```

**Migrable como PXFlowController si:**
- [x] El WebPanel implementa un flujo de pasos secuenciales
- [x] Tiene lógica de "siguiente paso" / "paso anterior"
- [x] Usa confirmaciones entre pasos
- [x] Puede tener iteraciones (repetir pasos)
- [x] La lógica se puede expresar como "líneas" con acciones y destinos

---

## Matriz de decisión rápida (actualizada)

```
¿Es una pantalla de Login (IsMain=True, sin MasterPage, auth custom)?
├── SÍ → Manual (no migrable)
│
¿Es una utilidad de testing/ejemplo o redirect automático?
├── SÍ → Manual (no migrable)
│
¿Depende de un framework externo (Scheduler, controles de terceros)?
├── SÍ → Manual (no migrable)
│
¿Tiene grilla CRUD maestro-detalle sobre transacción?
├── SÍ → ¿Tiene tabs de detalle?
│         ├── SÍ → PXWorkWith (Selection + View)
│         └── NO → PXWorkWith (solo Selection)
│
¿Tiene grilla de consulta/selección readonly sin CRUD?
├── SÍ → ¿Es un popup/lookup que retorna valores?
│         ├── SÍ → PXParameterRequest con grid (o PXWorkWith Prompt)
│         └── NO → PXWorkWith (Selection solo consulta)
│
¿Tiene Aceptar/Cancelar o Confirmar/Salir?
├── SÍ → PXParameterRequest (PopupParameterRequest)
│
¿Es un formulario de captura de datos (con o sin grilla)?
├── SÍ → PXParameterRequest
│
¿Es un visor readonly de datos (solo Salir)?
├── SÍ → PXParameterRequest (behaviour None o Panel)
│
¿Contiene múltiples WebComponents embebidos?
├── SÍ → PXComposer
│
¿Implementa flujo de pasos secuenciales?
├── SÍ → PXFlowController
│
└── TODOS los demás → PXParameterRequest (con la lógica en hooks)
```

**Regla fundamental:** Si el WebPanel tiene cualquier estructura con botones de acción y/o formulario, es migrable a PXParameterRequest. **Toda la lógica de negocio** (subrutinas, For Each, validaciones, WebSession, SDTs, llamadas a Procedures) se migra a hooks de código sin pérdida de funcionalidad.

## Combinaciones frecuentes

| Escenario | Patterns a usar |
|-----------|----------------|
| ABM completo con tabs de detalle | PXWorkWith |
| ABM con popup de confirmación antes de eliminar | PXWorkWith + PXParameterRequest |
| Dashboard con varias grillas | PXComposer + PXWorkWith (×N) |
| Proceso guiado con confirmaciones | PXFlowController + PXParameterRequest |
| Vista compuesta de seguridad | PXComposer + PXWorkWith + PXParameterRequest |
| API REST completa de una entidad | PXWSLayer + PXWSQuery + PXWSTransaction |
| Reporte con selección de parámetros | PXParameterRequest + PXReportTemplate |
| Popup de selección con grilla y filtros | PXParameterRequest con grid |
| Popup de anulación/reversa con causal | PXParameterRequest (PopupParameterRequest) |
| Upload de archivos/imágenes | PXParameterRequest con controlType File |
| Visor readonly de errores/datos | PXParameterRequest (behaviour None/Panel) |
| Generación de archivos TXT/Excel | PXParameterRequest con callType Event |
| Ejecución de proceso one-shot | PXParameterRequest (behaviour Panel) |

## Señales de NO migración

Un WebPanel **no es migrable** a patterns SOLO si:
- Es una **pantalla de Login** con flujo de autenticación custom (IsMain=True, sin MasterPage)
- Es una **utilidad de testing/ejemplo** que no es funcionalidad de producción
- Depende de un **framework externo** que gestiona su propio ciclo de vida (ej: Scheduler)
- Es un **redirect automático** sin interacción de usuario (solo JavaScript/server redirect)

### Lo que NO es razón para marcar como "no migrable"

**Ninguno de estos factores impide la migración:**
- Cantidad de lógica de negocio (toda va en hooks)
- Uso de WebSession (mecanismo estándar)
- Muchas subrutinas (van en `codes/Subroutine`)
- Creación/modificación de registros (callType Call o Event)
- Grilla con Load complejo (grid node + hook Load)
- Ocultamiento dinámico de controles (lógica en hook Start)
- Uso de SDTs para pasar datos (soportado en variables y hooks)
- Validaciones complejas (actionPreviousCode)
- Múltiples For Each anidados (todo es código GeneXus en hooks)

**Resultado esperado:** En un análisis de KB real (225 WebPanels), solo el **3.1%** resultó verdaderamente no migrable (7 de 225: 3 logins, 2 testing, 1 scheduler, 1 redirect). El **96.9%** restante fue migrable a PXParameterRequest, PXWorkWith o PXComposer.

Ver [32-limitaciones-y-gaps.md](32-limitaciones-y-gaps.md) para detalle de limitaciones del generador.
