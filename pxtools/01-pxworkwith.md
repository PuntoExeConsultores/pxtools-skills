# PXWorkWith - Pattern CRUD Maestro-Detalle

## Metadata

| Campo | Valor |
|-------|-------|
| Pattern | PXWorkWith |
| Tipo de asociacion | Transaction o (None) |
| Archivo de definicion | PXWorkWith.pattern |
| Archivo de instancia | PXWorkWithInstance.xml (~439 KB) |
| Archivo de settings | PXWorkWithSettings.xml (~166 KB) |
| Plataformas | Web Desktop, Web Responsive, Smart Devices |

---

## 1. Que es y para que sirve

PXWorkWith es el pattern principal de PXTools para generacion automatica de interfaces CRUD (Create, Read, Update, Delete) con arquitectura maestro-detalle. A partir de una unica instancia de configuracion XML, genera multiples objetos GeneXus que cubren:

- **Selection**: grilla de seleccion con filtros, ordenes, acciones y exportacion.
- **View**: vista de detalle con pestanas (tabs) tabulares y/o grillas.
- **Prompt**: dialogo de busqueda/lookup reutilizable.
- **Transaction**: transaccion con formulario HTML o Abstract.
- **Controller**: controlador de navegacion entre Selection, View y Transaction.
- **Exportacion**: procedimientos de exportacion a Excel y generacion de graficos.

Cada objeto se genera en version Desktop (HTML WebForm) y Responsive (Abstract Form), controlado por propiedades independientes.

### Estadisticas de uso real

En la KB pFacturas se registran **179 instancias** de PXWorkWith. El pattern se usa extensivamente en los modulos internos de PXTools: Security, Alerts, CloudTasks, FileStorage, entre otros.

---

## 2. Objetos que genera

Una unica instancia de PXWorkWith genera los siguientes objetos GeneXus:

| Object ID | Tipo GX | Patron de nombre | XPath del elemento | Descripcion |
|---|---|---|---|---|
| TransactionDesktop | Transaction | `D{Instance.Name}` | `//transaction` | Transaction con HTML WebForm |
| TransactionResponsive | Transaction | `R{Instance.Name}` | `//transaction` | Transaction con Abstract Form |
| SelectionSDT | SDT | `PXWW{Instance.Name}Rows` | `instance/level/selection` | SDT de filas seleccionadas |
| PromptSDT | SDT | `PXWW{Instance.Name}Rows` | `instance/level/prompt` | SDT de filas del prompt |
| TabGridSDT | SDT | `PXWW{Instance.Name}Rows` | `instance/level/view/sections//section[@type='Grid']` | SDT de filas de tab grid |
| Selection | WebPanel | `WW{Instance.Name}` | `instance/level/selection` | Grilla de seleccion (Desktop) |
| SelectionResponsive | WebPanel | `RWW{Instance.Name}` | `instance/level/selection` | Grilla de seleccion (Responsive) |
| View | WebPanel | `View{Instance.Name}` | `instance/level/view` | Vista de detalle (Desktop) |
| ViewResponsive | WebPanel | `RView{Instance.Name}` | `instance/level/view` | Vista de detalle (Responsive) |
| TabTabular | WebComponent | `{Element.wcname}` | `instance/level/view/sections//section[@type='Tabular']` | Tab tabular (Desktop) |
| TabTabularResponsive | WebComponent | `{Element.wcname}` | idem | Tab tabular (Responsive) |
| TabGrid | WebComponent | `{Element.wcname}` | `instance/level/view/sections//section[@type='Grid']` | Tab grilla (Desktop) |
| TabGridResponsive | WebComponent | `{Element.wcname}` | idem | Tab grilla (Responsive) |
| Prompt | WebPanel | `Pr{Instance.Name}` | `instance/level/prompt` | Prompt/lookup (Desktop) |
| PromptResponsive | WebPanel | `RPr{Instance.Name}` | `instance/level/prompt` | Prompt/lookup (Responsive) |
| LevelController | WebPanel | `Ct{Instance.Name}` | `instance/level[../transaction[@afterTrn='Call Levels Controllers']]` | Controlador de nivel (Desktop) |
| LevelControllerResponsive | WebPanel | `RCt{Instance.Name}` | idem | Controlador de nivel (Responsive) |
| InstanceController | WebPanel | `Ct{Instance.Name}` | `instance/controller` | Controlador de instancia (Desktop) |
| InstanceControllerResponsive | WebPanel | `RCt{Instance.Name}` | idem | Controlador de instancia (Responsive) |
| SelectionExportExcel | Procedure | `Ex{Instance.Name}` | `instance/level/selection/modes` | Exportacion a Excel |
| SelectionExportExcelSDT | SDT | `Ex{Instance.Name}ExcelSDT` | idem | SDT para exportacion Excel |
| SelectionChart | Procedure | `Ex{Instance.Name}` | `instance/level/selection/modes` | Generacion de graficos |
| SelectionUpdateGridRowsProcedure | Procedure | `Upd{Instance.Name}GridRows` | `instance/level/selection/modes` | Actualizacion masiva de filas |
| ChkChosenValueSelected | Procedure | `Chk{Instance.Name}ChosenValueSelected` | `instance//variable/controlInfo[@controlType='Chosen']` | Validacion de control Chosen |
| RetChosenValues | DataProvider | `Ret{Instance.Name}ChosenValues` | idem | Retorna valores Chosen |
| RetChosenResults | Procedure | `Ret{Instance.Name}ChosenResults` | idem | Retorna resultados Chosen |
| SelectionGridHandlerDataProvider | DataProvider | `PXWW{Instance.Name}Rows` | `instance/level/selection` | Grid handler para Selection |
| PromptGridHandlerDataProvider | DataProvider | `PXWW{Instance.Name}Rows` | `instance/level/prompt` | Grid handler para Prompt |
| TabGridGridHandlerDataProvider | DataProvider | `PXWW{Instance.Name}Rows` | `instance/level/view/sections//section[@type='Grid']` | Grid handler para Tab Grid |

> **Nota**: los objetos con el mismo patron de nombre (por ejemplo `PXWW{Instance.Name}Rows`) se distinguen por el contexto XPath del elemento que los origina dentro de la instancia.

---

## 3. Estructura XML de la instancia

### 3.1 Jerarquia principal

```
instance
 |
 +-- transaction ................. Configuracion de la transaccion
 |    +-- afterTrn .............. Comportamiento post-transaccion
 |    +-- generateTransaction ... Genera o no la transaccion
 |
 +-- level ....................... Nivel (puede haber multiples)
 |    +-- name, description
 |    +-- masterPage, theme
 |    +-- generateWeb ........... Genera Desktop
 |    +-- generateWebResponsive . Genera Responsive
 |    +-- generateSD ............ Genera Smart Devices
 |    |
 |    +-- controller ............ Configuracion del controlador
 |    |
 |    +-- selection ............. Grilla de seleccion
 |    |    +-- grid ............. Configuracion de grilla
 |    |    |    +-- attributes .. Columnas
 |    |    +-- filter ........... Filtros
 |    |    |    +-- search ...... Busqueda rapida
 |    |    |    +-- advancedSearch Filtros avanzados
 |    |    |    +-- conditions .. Condiciones fijas
 |    |    +-- orders ........... Ordenes de clasificacion
 |    |    +-- actions .......... Acciones/botones
 |    |    +-- modes ............ Export Excel, Charts, Update
 |    |    +-- events ........... Codigo de eventos custom
 |    |    +-- codes ............ Hooks: Start, Refresh, Load, Sub
 |    |    +-- variables ........ Variables custom
 |    |    +-- parameters ....... Parametros del WebPanel
 |    |    +-- fixedData ........ Filas de datos fijos
 |    |
 |    +-- view .................. Vista de detalle con tabs
 |    |    +-- form ............. Layout de atributos
 |    |    +-- sections ......... Pestanas
 |    |    |    +-- section ..... type: Tabular | Grid
 |    |    +-- actions, events, codes, variables, parameters
 |    |
 |    +-- prompt ................ Dialogo de busqueda/lookup
 |    |    +-- (estructura similar a selection)
 |    |
 |    +-- navigation ............ Navegacion Selection/View/Edit
 |
 +-- ...
```

### 3.2 Ejemplo XML simplificado

```xml
<instance parentObject="Factura">
  <transaction afterTrn="Call Controller" generateTransaction="True">
    <!-- Configuracion de transaccion -->
  </transaction>

  <level name="Factura" description="Facturas"
         masterPage="PXMasterPage" theme="PXTheme"
         generateWeb="True" generateWebResponsive="True" generateSD="False">

    <controller />

    <selection>
      <grid>
        <attributes>
          <attribute name="FacturaId" description="Nro. Factura" visible="True" />
          <attribute name="FacturaFecha" description="Fecha" visible="True" />
          <attribute name="ClienteNombre" description="Cliente" visible="True" />
          <attribute name="FacturaTotal" description="Total" visible="True" />
        </attributes>
      </grid>

      <filter>
        <search>
          <attribute name="FacturaId" />
          <attribute name="ClienteNombre" />
        </search>
        <advancedSearch>
          <attribute name="FacturaFecha" type="Range" />
          <attribute name="ClienteId" />
        </advancedSearch>
        <conditions>
          <condition expression="FacturaAnulada = false" />
        </conditions>
      </filter>

      <orders>
        <order name="Fecha Desc" expression="FacturaFecha DESC" default="True" />
        <order name="Cliente" expression="ClienteNombre" />
      </orders>

      <actions>
        <action name="Insert" caption="Insertar" callType="Link" />
        <action name="Update" caption="Modificar" callType="Link" />
        <action name="Delete" caption="Eliminar" callType="Link" />
        <action name="ExportExcel" caption="Exportar" callType="Link" />
        <action name="Imprimir" caption="Imprimir"
                callType="Client Text Print" />
      </actions>

      <modes>
        <exportExcel enabled="True" />
        <chart enabled="False" />
        <updateGridRows enabled="False" />
      </modes>

      <events>
        <!-- Codigo GeneXus personalizado para eventos -->
      </events>

      <codes>
        <start>/* Codigo ejecutado al inicio */</start>
        <refresh>/* Codigo ejecutado en refresh */</refresh>
        <load>/* Codigo ejecutado en load de cada fila */</load>
        <subroutine>/* Subrutinas reutilizables */</subroutine>
      </codes>

      <variables>
        <variable name="&amp;MiVariable" type="Numeric" length="4" />
      </variables>

      <parameters>
        <parameter name="ClienteId" />
      </parameters>

      <fixedData>
        <!-- Filas de datos fijos opcionales -->
      </fixedData>
    </selection>

    <view>
      <form>
        <attribute name="FacturaId" />
        <attribute name="FacturaFecha" />
        <attribute name="ClienteNombre" />
      </form>

      <sections>
        <section name="Detalle" type="Grid" wcname="WWFacturaDetalle">
          <grid>
            <attributes>
              <attribute name="ProductoNombre" />
              <attribute name="FacturaDetCantidad" />
              <attribute name="FacturaDetPrecio" />
            </attributes>
          </grid>
          <filter>
            <search />
          </filter>
          <orders />
          <actions />
        </section>

        <section name="Observaciones" type="Tabular" wcname="WWFacturaObs">
          <form>
            <attribute name="FacturaObservaciones" />
          </form>
        </section>

        <section name="Adjuntos" type="Grid"
                 externalComponent="true" wcname="WWFacturaAdj">
          <!-- Componente externo embebido -->
        </section>
      </sections>

      <actions>
        <action name="Editar" caption="Editar" callType="Link" />
      </actions>
    </view>

    <prompt>
      <grid>
        <attributes>
          <attribute name="FacturaId" />
          <attribute name="ClienteNombre" />
          <attribute name="FacturaFecha" />
        </attributes>
      </grid>
      <filter>
        <search>
          <attribute name="FacturaId" />
        </search>
      </filter>
      <orders>
        <order name="Default" expression="FacturaId DESC" default="True" />
      </orders>
    </prompt>

    <navigation />
  </level>
</instance>
```

---

## 4. Nodos principales

### 4.1 Selection

El nodo `selection` genera la grilla principal de seleccion (listado maestro). Produce los objetos `WW{Instance.Name}` (Desktop) y `RWW{Instance.Name}` (Responsive).

**Subnodos:**

| Subnodo | Proposito |
|---------|-----------|
| `grid` | Define columnas visibles (attributes y variables) |
| `filter` | Filtros de busqueda rapida, avanzada y condiciones fijas |
| `orders` | Ordenes de clasificacion disponibles |
| `actions` | Botones y acciones del usuario |
| `modes` | Modos especiales: Export Excel, Charts, Update Grid Rows |
| `events` | Codigo GeneXus de eventos personalizados |
| `codes` | Hooks de codigo: Start, Refresh, Load, Subroutine |
| `variables` | Variables personalizadas adicionales |
| `parameters` | Parametros de entrada del WebPanel |
| `fixedData` | Filas de datos fijos (no provienen de la BD) |

### 4.2 View

El nodo `view` genera la pantalla de detalle de un registro. Produce los objetos `View{Instance.Name}` (Desktop) y `RView{Instance.Name}` (Responsive).

**Componentes:**

- **form**: layout tabular de atributos del registro principal.
- **sections**: pestanas (tabs) que pueden ser de tipo `Tabular` o `Grid`. Cada seccion genera un WebComponent independiente. Solo el tab activo está renderizado, por lo que `GlobalEvents` no aplica entre tabs. Para compartir información entre tabs se puede usar **WebSession** (un tab escribe, otro lee en su Start/Refresh), implementándolo en los hooks de código.
- **actions, events, codes, variables, parameters**: misma estructura que Selection.

### 4.3 Prompt

El nodo `prompt` genera un dialogo de busqueda/lookup. Produce los objetos `Pr{Instance.Name}` (Desktop) y `RPr{Instance.Name}` (Responsive). Su estructura interna es similar a `selection` (grid, filter, orders, etc.) pero orientada a la seleccion de un registro para retorno de clave.

### 4.4 Controller

Existen dos tipos de controlador:

| Tipo | XPath | Condicion de generacion |
|------|-------|------------------------|
| **InstanceController** | `instance/controller` | Siempre disponible |
| **LevelController** | `instance/level` | Solo cuando `transaction[@afterTrn='Call Levels Controllers']` |

El controlador gestiona la navegacion entre Selection, View y Transaction (edicion). Ambos generan objetos `Ct{Instance.Name}` (Desktop) y `RCt{Instance.Name}` (Responsive).

La propiedad `afterTrn` de la transaccion determina el flujo post-guardado:

| Valor | Comportamiento |
|-------|---------------|
| `Call Controller` | Retorna al controlador de instancia |
| `Call Levels Controllers` | Retorna al controlador del nivel correspondiente |
| `Do Nothing` | No ejecuta navegacion post-transaccion |

---

## 5. Sistema de acciones

El sistema de acciones es compartido por los tres patterns de UI (PXWorkWith, PXParameterRequest, PXComposer). La referencia completa esta en [12-acciones-patterns-ui.md](12-acciones-patterns-ui.md). Aqui se documenta el uso especifico en PXWorkWith.

Cada accion dentro de `actions` tiene la siguiente estructura:

```xml
<action name="MiAccion"
        caption="Texto visible"
        callType="Link"
        linkType="GXObject"
        target="Self"
        condition="FacturaEstado = 'A'"
        confirm="false">
  <security object="MiObjeto" operation="Execute" />
</action>
```

### Propiedades de una accion

| Propiedad | Valores posibles | Descripcion |
|-----------|-----------------|-------------|
| `name` | texto libre | Identificador unico de la accion |
| `caption` | texto libre | Texto visible en el boton/link |
| `conditionPreviousCode` | code | Codigo procedural previo a la evaluacion de la condicion de visibilidad |
| `condition` | expresion GeneXus | Condicion de visibilidad de la accion (evaluada por fila si la accion esta en la grilla) |
| `evaluateCondition` | `Event`, `Load` | Cuando evaluar la condicion: en Load (por cada fila) o en Event (al hacer clic) |
| `actionPreviousCode` | code | Codigo procedural que se ejecuta **antes** de la invocacion principal del objeto |
| `callType` | `Link`, `Call`, `Prompt`, `External Link`, `Client Text Print`, `Subroutine`, `Event`, `Submit`, `None`, `Return` | Tipo de invocacion (ver tabla detallada abajo) |
| `linkType` | `GXObject`, `PXInstance` | Solo para callType `Link`: destino objeto GeneXus directo o instancia PXTools |
| `actionPostCode` | code | Codigo procedural que se ejecuta **despues** de la invocacion principal |
| `refreshAction` | enum | Como refrescar la pantalla/grilla despues de la accion (util cuando la accion modifica datos mostrados en la grilla) |
| `target` | `Self`, `New` | `Self` = misma ventana, `New` = popup/ventana nueva |
| `confirm` | texto o booleano | Dialogo de confirmacion antes de ejecutar |
| `security` | object + operation | Control de acceso (objeto y operacion GAM/PXSecurity) |

### Acciones condicionadas por fila

Las acciones soportan **visibilidad condicional por fila** mediante la propiedad `condition` evaluada en el Load de la grilla. Además, el concepto de **Conditional Calls** permite que una misma acción tenga **comportamientos distintos según el contexto** de información de cada fila:

```xml
<!-- Acción visible solo para facturas pendientes -->
<action name="Aprobar"
        caption="Aprobar"
        callType="Link"
        condition="FacturaEstado = 'Pendiente'"
        evaluateCondition="Load">
</action>

<!-- Conditional Calls: misma acción, diferente destino según contexto -->
<action name="Ver"
        caption="Ver Detalle"
        callType="Link">
  <conditionalCalls>
    <conditionalCall condition="FacturaTipo = 'Nacional'"
                     gxObject="ViewFacturaNacional" />
    <conditionalCall condition="FacturaTipo = 'Exportacion'"
                     gxObject="ViewFacturaExportacion" />
  </conditionalCalls>
</action>
```

Esto permite:
- Mostrar/ocultar botones según el estado, tipo o cualquier dato de cada registro
- Invocar objetos diferentes desde el mismo botón según el contexto de la fila
- Controlar la evaluación de la condición en Load (por fila) o en Event (al clic)

### Tipos de invocacion (callType)

```
+---------------------+------------------------------------------+
| callType            | Comportamiento                           |
+---------------------+------------------------------------------+
| Link                | Navega a otro WebPanel/Transaction       |
| Call                | Invoca un Procedure (sin interfaz)       |
| Prompt              | Abre dialogo modal (popup)               |
| External Link       | Abre URL externa o WebPanel no-PXTools   |
| Client Text Print   | Impresion desde el cliente               |
| Subroutine          | Ejecuta subrutina dentro del mismo objeto|
| Event               | Ejecuta codigo inline (sin objeto ppal)  |
| Submit              | Somete un proceso batch                  |
| None                | No ejecuta nada                          |
| Return              | Solo retorna (cierra popup/vuelve)       |
+---------------------+------------------------------------------+
```

### Tipos de destino (linkType)

- **GXObject**: invoca directamente un objeto GeneXus (WebPanel, Procedure, Transaction).
- **PXInstance**: invoca otra instancia de PXTools, resolviendo automaticamente el objeto generado correspondiente.

**Regla critica para migracion Responsive:** Solo las acciones con `callType="Link"` necesitan usar `linkType="PXInstance"` en lugar de `linkType="GXObject"`, porque son las que invocan **interfaces graficas generadas por patterns** cuyos nombres cambian segun la plataforma (ej: `WWFactura` → `RWWFactura`). Los demas callTypes (`Call`, `Submit`, `Event`, `Subroutine`, etc.) invocan Procedures u objetos cuyo nombre **no cambia** entre plataformas, por lo que `GXObject` es correcto para ellos.

### Propiedades de PXInstance (para callType Link)

Cuando `linkType="PXInstance"`, se usan estas propiedades para identificar el objeto generado:

| Propiedad | Descripcion |
|-----------|-------------|
| `instanceObject` | Nombre de la instancia de pattern destino (ej: `PXWorkWithFactura`) |
| `instanceLevel` | Nombre del level dentro de la instancia |
| `instanceLevelNode` | Nodo del level a invocar: `Selection`, `View`, `Prompt`, `Transaction`, `Level` |
| `instanceLevelViewSection` | (Solo para View) Seccion/tab especifica del View a la que navegar. Si se omite, va al tab principal |

El generador resuelve automaticamente el nombre del objeto segun la plataforma:
```
PXInstance → PXWorkWithFactura / Level=Factura / Node=Selection
  Desktop genera:    Link a WWFactura
  Responsive genera: Link a RWWFactura
```

### Ciclo de ejecucion de una accion

```
1. conditionPreviousCode  → Codigo previo para evaluar condicion
2. condition              → Si es false, la accion no se muestra/ejecuta
3. actionPreviousCode     → Codigo previo a la invocacion principal
4. [Invocacion principal] → Segun callType (Link/Call/Event/etc.)
5. actionPostCode         → Codigo posterior a la invocacion
6. refreshAction          → Refresco de pantalla/grilla si corresponde
```

---

## 6. Filtros y ordenes

### 6.1 Filtros

El nodo `filter` contiene tres mecanismos de filtrado:

```
filter
 +-- search .............. Busqueda rapida (barra superior)
 |    +-- attribute ...... Atributos incluidos en busqueda libre
 |
 +-- advancedSearch ...... Filtros avanzados (panel desplegable)
 |    +-- attribute ...... Campos con tipo de filtro (Range, Exact, etc.)
 |
 +-- conditions .......... Condiciones fijas (no visibles al usuario)
      +-- condition ...... Expresion GeneXus aplicada siempre
```

**search (busqueda rapida)**: lista de atributos contra los que se busca con texto libre. El usuario escribe un termino y se filtra contra todos los atributos listados con operador LIKE.

**advancedSearch (busqueda avanzada)**: campos individuales con controles especificos. Cada atributo puede tener un tipo de filtro (rango de fechas, valor exacto, lista, etc.).

**conditions (condiciones fijas)**: expresiones GeneXus que se aplican siempre como filtro WHERE sin que el usuario pueda modificarlas. Se usan para restringir datos por contexto (por ejemplo, filtrar solo registros activos).

### 6.2 Ordenes

El nodo `orders` define las opciones de ordenamiento disponibles:

```xml
<orders>
  <order name="Mas recientes" expression="FacturaFecha DESC" default="True" />
  <order name="Por cliente" expression="ClienteNombre ASC, FacturaFecha DESC" />
  <order name="Por monto" expression="FacturaTotal DESC" />
</orders>
```

| Propiedad | Descripcion |
|-----------|-------------|
| `name` | Texto visible en el selector de orden |
| `expression` | Expresion de ordenamiento (atributos + ASC/DESC) |
| `default` | `True` si es el orden aplicado por defecto |

Se genera un combo/selector que permite al usuario elegir el criterio de ordenamiento en tiempo de ejecucion.

---

## 7. Tabs (Grid y Tabular)

Las pestanas de la vista de detalle (`view/sections/section`) pueden ser de dos tipos:

### 7.1 Section tipo Grid

Muestra una grilla de detalle (relacion 1:N). Genera un WebComponent con estructura similar a Selection:

```xml
<section name="Detalle" type="Grid" wcname="WWFacturaDetalle">
  <grid>
    <attributes>...</attributes>
  </grid>
  <filter>...</filter>
  <orders>...</orders>
  <actions>...</actions>
  <modes>...</modes>
</section>
```

Objetos generados:
- `TabGrid` / `TabGridResponsive` (WebComponent)
- `TabGridSDT` (SDT de filas)
- `TabGridGridHandlerDataProvider` (DataProvider)

### 7.2 Section tipo Tabular

Muestra atributos en formato formulario (datos 1:1 o informacion complementaria):

```xml
<section name="Observaciones" type="Tabular" wcname="WWFacturaObs">
  <form>
    <attribute name="FacturaObservaciones" />
    <attribute name="FacturaNotas" />
  </form>
</section>
```

Objetos generados:
- `TabTabular` / `TabTabularResponsive` (WebComponent)

### 7.3 Componentes externos

Cuando `externalComponent="true"`, la seccion embebe un WebComponent existente en lugar de generar uno nuevo. Esto permite integrar componentes desarrollados manualmente.

```
+------------------------------------------+
| View: ViewFactura                        |
|                                          |
|  [Factura #1234]  [Fecha: 2026-01-15]   |
|  [Cliente: Acme Corp]                    |
|                                          |
|  +------+-------------+-----------+      |
|  |Detalle| Observaciones| Adjuntos |     |
|  +------+-------------+-----------+      |
|  | Tab Grid           | Tab Tabul | Tab  |
|  | (type="Grid")      | (type=    | ext. |
|  |                    | "Tabular")| comp |
|  | ProductoNombre  Q P| FacturaObs| (ext)|
|  | FactDetCant     5 U| FactNotas |      |
|  | FactDetPrecio 100 U|           |      |
|  +--------------------+-----------+------+
```

---

## 8. Modes (Export, Charts, Update Grid Rows)

El nodo `modes` dentro de `selection` habilita funcionalidades especiales:

### 8.1 Export Excel

```xml
<modes>
  <exportExcel enabled="True" />
</modes>
```

Genera:
- `Ex{Instance.Name}` (Procedure): logica de exportacion.
- `Ex{Instance.Name}ExcelSDT` (SDT): estructura de datos para el archivo Excel.

**Templates Excel**: PXTools soporta el uso de archivos Excel como **plantillas** para la exportacion. Esto permite que el usuario final defina diseños personalizados: formatos complejos, multiples hojas, logos, formulas, graficos, etc. El Procedure de exportacion toma la plantilla Excel como base y la rellena con los datos de la grilla.

### 8.2 Charts (Graficos)

```xml
<modes>
  <chart enabled="True" />
</modes>
```

Genera:
- `Ex{Instance.Name}` (Procedure): logica de generacion de graficos.

### 8.3 Update Grid Rows (Actualizacion masiva)

```xml
<modes>
  <updateGridRows enabled="True" />
</modes>
```

Genera:
- `Upd{Instance.Name}GridRows` (Procedure): logica de actualizacion masiva de filas seleccionadas en la grilla.

---

## 9. Hooks de codigo (Events y Codes)

PXWorkWith provee puntos de extension para inyectar codigo GeneXus personalizado en los objetos generados.

### 9.1 Nodo `events`

Permite definir eventos GeneXus completos (Event handlers) que se inyectan en el objeto generado. Se usa para logica reactiva a acciones del usuario.

### 9.2 Nodo `codes`

Define bloques de codigo que se ejecutan en momentos especificos del ciclo de vida del objeto:

| Hook | Momento de ejecucion | Uso tipico |
|------|---------------------|------------|
| `start` | Al iniciar el WebPanel (evento Start) | Inicializar variables, validar permisos |
| `refresh` | Al refrescar la grilla (evento Refresh) | Recalcular filtros, actualizar estado |
| `load` | Al cargar cada fila de la grilla (evento Load) | Calcular campos derivados, aplicar formato condicional |
| `subroutine` | Subrutinas invocables desde eventos | Logica reutilizable dentro del objeto |

```xml
<codes>
  <start>
    &amp;MiVariable = Today()
    If &amp;ClienteId.IsEmpty()
      Return
    EndIf
  </start>
  <refresh>
    &amp;TotalRegistros = CountFacturas(&amp;FiltroFecha)
  </refresh>
  <load>
    If FacturaTotal > 10000
      &amp;RowClass = "HighValue"
    EndIf
  </load>
  <subroutine>
    Sub 'ActualizarTotales'
      &amp;GrandTotal = SumFacturas()
    EndSub
  </subroutine>
</codes>
```

### 9.3 Grilla sin tabla base (Load sin Tabla Base)

PXWorkWith soporta grillas **completamente basadas en variables**, sin necesidad de una tabla de base de datos. En este modo:

- La grilla se define con **variables** en lugar de atributos de transacción
- Los **filtros** se declaran como variables en la sección de filtros
- El **code Load** define cómo se populan los datos de cada fila (puede invocar web services externos, APIs, SDTs en memoria, etc.)
- Los filtros y órdenes funcionan normalmente porque operan sobre las variables declaradas

Esto permite crear PXWorkWith que muestran datos provenientes de:
- Web Services externos (REST/SOAP)
- APIs de terceros
- Cálculos en memoria
- Cualquier fuente de datos accesible desde código GeneXus

```xml
<selection>
  <grid>
    <!-- Grilla 100% basada en variables, sin atributos de transacción -->
    <variables>
      <variable name="Codigo" dataType="Character" length="20" />
      <variable name="Nombre" dataType="Character" length="100" />
      <variable name="Precio" dataType="Numeric" length="12" decimals="2" />
    </variables>
  </grid>
  <filter>
    <variables>
      <variable name="FiltroNombre" dataType="Character" length="100" />
    </variables>
  </filter>
  <codes>
    <load><![CDATA[
      // Cargar datos desde un Web Service externo
      &SDTProductos = WSGetProductos.Udp(&FiltroNombre)
      For &Producto in &SDTProductos
        &Codigo = &Producto.Code
        &Nombre = &Producto.Name
        &Precio = &Producto.Price
        Load  // Agrega la fila a la grilla
      EndFor
    ]]></load>
  </codes>
</selection>
```

### 9.4 Variables personalizadas

El nodo `variables` permite declarar variables adicionales necesarias para los hooks de codigo:

```xml
<variables>
  <variable name="&amp;MiVariable" type="Numeric" length="4" />
  <variable name="&amp;RowClass" type="Character" length="20" />
</variables>
```

---

## 10. Capacidad dual-platform

PXWorkWith genera objetos para dos plataformas web simultaneamente:

```
+---------------------------+---------------------------+
|      WEB DESKTOP          |      WEB RESPONSIVE       |
+---------------------------+---------------------------+
| Template: HTML WebForm    | Template: Abstract Form   |
| DLL: *WebForm.dll         | DLL: *AbstractForm.dll    |
+---------------------------+---------------------------+
| WW{Name}                  | RWW{Name}                 |
| View{Name}                | RView{Name}               |
| Pr{Name}                  | RPr{Name}                 |
| Ct{Name}                  | RCt{Name}                 |
| D{Name}                   | R{Name}                   |
| {wcname}                  | {wcname} (Responsive)     |
+---------------------------+---------------------------+
```

### Propiedades de control por nivel

Cada nodo `level` tiene tres propiedades independientes que controlan la generacion:

| Propiedad | Valores | Descripcion |
|-----------|---------|-------------|
| `generateWeb` | `<default>`, `True`, `False` | Genera version Web Desktop |
| `generateWebResponsive` | `<default>`, `True`, `False` | Genera version Web Responsive |
| `generateSD` | `<default>`, `True`, `False` | Genera version Smart Devices |

El valor `<default>` hereda la configuracion del PXWorkWithSettings global.

### Convencion de prefijos por plataforma

| Plataforma | Prefijo Transaction | Prefijo Selection | Prefijo View | Prefijo Prompt | Prefijo Controller |
|------------|--------------------|--------------------|--------------|---------------|--------------------|
| Desktop | `D` | `WW` | `View` | `Pr` | `Ct` |
| Responsive | `R` | `RWW` | `RView` | `RPr` | `RCt` |

---

## 11. Relacion con otros patterns

PXWorkWith es el pattern central de PXTools. Se relaciona con los demas patterns de la siguiente manera:

```
+------------------+
|   PXWorkWith     |<-------- Pattern principal CRUD
+------------------+
        |
        | genera Transaction -----> Asociada a una GX Transaction
        |
        | linkType="PXInstance" --> Puede invocar otras instancias PXWorkWith
        |
        | security.object -------> Integra con modulo PXSecurity
        |
        | masterPage, theme -----> Referencia objetos de UI compartidos
        |
        | externalComponent -----> Embebe WebComponents de otros patterns
        |                          o desarrollados manualmente
        |
        | modes.exportExcel -----> Genera Procedures de exportacion
        | modes.chart -----------> Genera Procedures de graficos
        |
        +-- controller ----------> Controla navegacion entre
             |                     Selection <-> View <-> Transaction
             |
             +-- afterTrn -------> Define flujo post-transaccion
```

### Interaccion clave: acciones con linkType="PXInstance"

Cuando una accion tiene `linkType="PXInstance"`, el pattern resuelve automaticamente el objeto GeneXus generado por la instancia PXTools destino. Esto permite construir navegaciones entre multiples instancias de PXWorkWith sin hardcodear nombres de objetos.

---

## 12. Convenciones de nomenclatura

### 12.1 Prefijos por tipo de objeto

| Prefijo | Tipo de objeto | Ejemplo |
|---------|---------------|---------|
| `D` | Transaction Desktop | `DFactura` |
| `R` | Transaction Responsive | `RFactura` |
| `WW` | Selection Desktop | `WWFactura` |
| `RWW` | Selection Responsive | `RWWFactura` |
| `View` | View Desktop | `ViewFactura` |
| `RView` | View Responsive | `RViewFactura` |
| `Pr` | Prompt Desktop | `PrFactura` |
| `RPr` | Prompt Responsive | `RPrFactura` |
| `Ct` | Controller Desktop | `CtFactura` |
| `RCt` | Controller Responsive | `RCtFactura` |
| `Ex` | Export/Chart Procedure | `ExFactura` |
| `Upd` | Update Grid Rows Procedure | `UpdFacturaGridRows` |
| `PXWW` | SDT de filas | `PXWWFacturaRows` |
| `Chk` | Validacion Chosen | `ChkFacturaChosenValueSelected` |
| `Ret` | Return Chosen values | `RetFacturaChosenValues` |

### 12.2 Sufijos comunes

| Sufijo | Uso | Ejemplo |
|--------|-----|---------|
| `Rows` | SDT de coleccion de filas | `PXWWFacturaRows` |
| `ExcelSDT` | SDT para exportacion Excel | `ExFacturaExcelSDT` |
| `GridRows` | Procedure de actualizacion masiva | `UpdFacturaGridRows` |
| `ChosenValueSelected` | Validacion de Chosen | `ChkFacturaChosenValueSelected` |
| `ChosenValues` | DataProvider de valores Chosen | `RetFacturaChosenValues` |
| `ChosenResults` | Procedure de resultados Chosen | `RetFacturaChosenResults` |

### 12.3 Regla general

El nombre base es siempre `{Instance.Name}`, que tipicamente coincide con el nombre de la Transaction asociada. Los prefijos indican el tipo de objeto y la plataforma. Los sufijos indican la funcion especifica.

```
[Prefijo Plataforma][Prefijo Funcion]{Instance.Name}[Sufijo]

Ejemplos:
  R    +  WW   + Factura +        = RWWFactura      (Selection Responsive)
       +  View + Factura +        = ViewFactura      (View Desktop)
  R    +  Pr   + Factura +        = RPrFactura       (Prompt Responsive)
       +  Ex   + Factura + ExcelSDT = ExFacturaExcelSDT (SDT Export Excel)
       +  PXWW + Factura + Rows   = PXWWFacturaRows  (SDT filas)
```

---

## Referencia rapida de XPaths

Tabla de referencia para localizar elementos dentro de la instancia XML:

| Elemento | XPath |
|----------|-------|
| Transaccion | `//transaction` |
| Nivel | `instance/level` |
| Selection | `instance/level/selection` |
| View | `instance/level/view` |
| Prompt | `instance/level/prompt` |
| Controlador de instancia | `instance/controller` |
| Tab Tabular | `instance/level/view/sections//section[@type='Tabular']` |
| Tab Grid | `instance/level/view/sections//section[@type='Grid']` |
| Modes | `instance/level/selection/modes` |
| Control Chosen | `instance//variable/controlInfo[@controlType='Chosen']` |
| Level Controller | `instance/level[../transaction[@afterTrn='Call Levels Controllers']]` |
