# PXComposer — Pattern de Composicion de Pantallas

## Que es y para que sirve

PXComposer es un pattern de PXTools que genera **WebPanels compuestos** (tipo dashboard) embebiendo otros patterns y objetos GeneXus como WebComponents. A diferencia de PXWorkWith o PXParameterRequest, PXComposer **no tiene Parent Object** — es siempre standalone (`ParentObject Type="(None)"`).

Su proposito principal es permitir la construccion declarativa de pantallas que combinan multiples componentes visuales: grillas de PXWorkWith, formularios de PXParameterRequest, otros PXComposer anidados, controles de usuario o HTML crudo, todo definido en XML sin escribir codigo manual.

```
┌─────────────────────────────────────────────────────────┐
│  PXComposer: WbDashboardVentas                          │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │  WebComponent        │  │  WebComponent             │  │
│  │  PXWorkWith          │  │  PXWorkWith               │  │
│  │  Selection: Facturas │  │  Selection: Clientes      │  │
│  └─────────────────────┘  └──────────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  WebComponent                                     │   │
│  │  PXParameterRequest: FiltroReporte               │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │  HTML                │  │  UserControl              │  │
│  │  <div>info</div>     │  │  GraficoBarras            │  │
│  └─────────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Objetos que genera

PXComposer genera exactamente **dos objetos** por cada level de la instancia, uno para cada plataforma web:

| Object ID | Tipo GeneXus | Patron de nombre | Element | Descripcion |
|-----------|-------------|------------------|---------|-------------|
| `Level` | WebPanel | `Wb{Istance.Name}` | `instance/level` | Panel compuesto para Web Desktop (layout HTML) |
| `LevelResponsive` | WebPanel | `RWb{Istance.Name}` | `instance/level` | Panel compuesto para Web Responsive (layout abstracto) |

> **Nota sobre nomenclatura:** El patron de nombre usa `{Istance.Name}` (con "I" mayuscula y sin la "n" — "Istance" en vez de "Instance"). Esto es un typo historico en el archivo `.Pattern` original, pero es el naming real utilizado por el generador.

### Ejemplos de nombres generados

| Nombre de instancia | Desktop | Responsive |
|---------------------|---------|------------|
| `DashboardVentas` | `WbDashboardVentas` | `RWbDashboardVentas` |
| `FileStorage` | `WbFileStorage` | `RWbFileStorage` |
| `SecurityObjectAccess` | `WbSecurityObjectAccess` | `RWbSecurityObjectAccess` |

### Control de generacion

Cada level tiene propiedades booleanas que controlan que plataformas se generan:

| Propiedad | Default | Efecto |
|-----------|---------|--------|
| `generateWeb` | `true` | Genera el WebPanel Desktop (`Wb{Istance.Name}`) |
| `generateWebResponsive` | `true` | Genera el WebPanel Responsive (`RWb{Istance.Name}`) |
| `generateSD` | `false` | Reservado para Smart Devices (no implementado actualmente) |

## Estructura XML de la instancia

La instancia de PXComposer sigue el schema definido en `PXComposerInstance.xml` (108KB). Estructura completa:

```xml
<instance>
  <level>
    <!-- Propiedades del level -->
    <name>MiDashboard</name>
    <description>Panel principal de ventas</description>
    <masterPage>PXMasterPage</masterPage>
    <theme>PXTheme</theme>

    <!-- Control de generacion -->
    <generateWeb>true</generateWeb>
    <generateWebResponsive>true</generateWebResponsive>
    <generateSD>false</generateSD>

    <!-- Forms con componentes (seccion principal) -->
    <forms>
      <form>
        <name>FormDefault</name>
        <platform>Any</platform>
        <components>
          <component>...</component>
          <component>...</component>
        </components>
      </form>
      <form>
        <name>FormResponsive</name>
        <platform>Web Responsive</platform>
        <components>
          <component>...</component>
        </components>
      </form>
    </forms>

    <!-- Acciones disponibles -->
    <actions>
      <action>
        <name>Volver</name>
        <gxobject>MenuPrincipal</gxobject>
        <!-- ... -->
      </action>
    </actions>

    <!-- Eventos personalizados -->
    <events>
      <event>
        <name>MiEvento</name>
        <code>/* codigo GeneXus */</code>
      </event>
    </events>

    <!-- Hooks de codigo -->
    <codes>
      <code>
        <name>Start</name>
        <code>/* codigo en evento Start */</code>
      </code>
      <code>
        <name>Refresh</name>
        <code>/* codigo en evento Refresh */</code>
      </code>
      <code>
        <name>Load</name>
        <code>/* codigo en evento Load */</code>
      </code>
      <code>
        <name>Subroutine</name>
        <code>/* subrutina reutilizable */</code>
      </code>
    </codes>

    <!-- Parametros del WebPanel -->
    <parameters>
      <parameter>
        <name>ClienteId</name>
        <domain>Numeric</domain>
      </parameter>
    </parameters>

    <!-- Variables adicionales -->
    <variables>
      <variable>
        <name>MiVariable</name>
        <domain>Character</domain>
      </variable>
    </variables>

    <!-- Condiciones/reglas -->
    <conditions>...</conditions>

    <!-- Documentacion -->
    <documentation>Descripcion para desarrolladores</documentation>
  </level>
</instance>
```

### Arbol jerarquico resumido

```
instance
└── level (puede haber multiples)
    ├── name, description
    ├── masterPage, theme
    ├── generateWeb, generateWebResponsive, generateSD
    ├── forms
    │   └── form* (multiples forms para distintas plataformas)
    │       ├── name
    │       ├── platform: {Any|Web Desktop|Web Responsive|Smart Devices}
    │       └── components
    │           └── component* (cada pieza embebida)
    │               ├── type: {WebComponent|UserControl|HTML|Action|...}
    │               ├── [WebComponent] callType, gxObject, instanceObject...
    │               ├── [UserControl] controlName, properties
    │               └── [HTML] html content
    ├── actions
    │   └── action* (name, gxobject, image, tooltip, ...)
    ├── events
    │   └── event* (name, code)
    ├── codes
    │   └── code* (name: Start|Refresh|Load|Subroutine, code)
    ├── parameters
    │   └── parameter* (name, domain)
    ├── variables
    │   └── variable* (name, domain)
    ├── conditions
    └── documentation
```

## Sistema de forms y componentes

### Concepto de form

Cada level contiene un nodo `<forms>` que agrupa uno o mas `<form>`. Cada form define un layout de componentes para una plataforma especifica. Esto permite que un mismo PXComposer genere layouts diferentes segun la plataforma destino.

```
forms
├── form (platform="Any")           --> usado si no hay form especifico
├── form (platform="Web Desktop")   --> layout solo para Desktop
├── form (platform="Web Responsive")--> layout solo para Responsive
└── form (platform="Smart Devices") --> layout solo para SD
```

### Resolucion de form por plataforma

El generador selecciona el form a usar con esta logica:

```
Al generar Wb{Name} (Desktop):
  1. Buscar form con platform="Web Desktop"
  2. Si no existe, usar form con platform="Any"

Al generar RWb{Name} (Responsive):
  1. Buscar form con platform="Web Responsive"
  2. Si no existe, usar form con platform="Any"
```

### Componentes dentro de un form

Cada form contiene un nodo `<components>` con multiples `<component>`. Los componentes se renderizan en el WebPanel generado en el orden en que aparecen en el XML.

## Tipos de componentes

### WebComponent

El tipo mas comun e importante. Embebe un WebPanel como WebComponent dentro del panel compuesto.

```xml
<component>
  <type>WebComponent</type>

  <!-- Opcion A: referencia directa a un objeto GeneXus -->
  <callType>GXObject</callType>
  <gxObject>WWFactura</gxObject>

  <!-- Opcion B: referencia a una instancia de pattern -->
  <callType>PXInstance</callType>
  <instanceObject>PXWorkWithFactura</instanceObject>
  <instanceLevel>Factura</instanceLevel>
  <instanceLevelNode>Selection</instanceLevelNode>

  <!-- Parametros opcionales para el WebComponent -->
  <parameters>
    <parameter>
      <name>ClienteId</name>
      <value>&amp;ClienteId</value>
    </parameter>
  </parameters>
</component>
```

#### callType: GXObject vs PXInstance

| callType | Uso | Ventaja |
|----------|-----|---------|
| `GXObject` | Referencia directa a un WebPanel existente | Simple, funciona con cualquier WebPanel |
| `PXInstance` | Referencia a un pattern PXWorkWith, PXParameterRequest o PXComposer | Se resuelve automaticamente al objeto generado correcto segun la plataforma |

Cuando se usa `PXInstance`, el generador resuelve automaticamente el nombre del objeto segun la plataforma:

```
PXInstance: PXWorkWithFactura / level=Factura / node=Selection
  → Desktop genera:    WebComponent = WWFactura
  → Responsive genera: WebComponent = RWWFactura
```

Esto es fundamental para la generacion dual-platform: una sola definicion produce referencias correctas en ambas plataformas.

### UserControl

Embebe un User Control de GeneXus dentro del panel.

```xml
<component>
  <type>UserControl</type>
  <controlName>MiGrafico</controlName>
  <properties>
    <property>
      <name>DataSource</name>
      <value>&amp;DatosGrafico</value>
    </property>
  </properties>
</component>
```

### HTML

Inserta contenido HTML crudo directamente en el WebPanel generado.

```xml
<component>
  <type>HTML</type>
  <html><![CDATA[
    <div class="info-panel">
      <h3>Resumen de ventas</h3>
      <p>Informacion adicional aqui</p>
    </div>
  ]]></html>
</component>
```

### Action

Embebe un boton o enlace de accion.

```xml
<component>
  <type>Action</type>
  <name>VerReporte</name>
  <gxobject>ReporteVentas</gxobject>
  <!-- propiedades de accion -->
</component>
```

## Embedding de otros patterns (PXInstance)

La capacidad mas potente de PXComposer es embeber instancias de otros patterns de forma declarativa. Cuando `callType="PXInstance"`, se configuran tres propiedades que identifican exactamente que objeto embeber:

### Nodos embebibles por pattern

| Pattern origen | instanceLevelNode validos | Objeto resuelto (Desktop) | Objeto resuelto (Responsive) |
|---------------|--------------------------|--------------------------|------------------------------|
| PXWorkWith | `Selection` | `WW{Name}` | `RWW{Name}` |
| PXWorkWith | `View` | `View{Name}` | `RView{Name}` |
| PXWorkWith | `Prompt` | `Pr{Name}` | `RPr{Name}` |
| PXParameterRequest | `Level` | `Wb{Name}` | `RWb{Name}` |
| PXComposer | (level completo) | `Wb{Name}` | `RWb{Name}` |

### Ejemplo de composicion con PXInstance

```xml
<!-- Embeber la Selection de PXWorkWith de Facturas -->
<component>
  <type>WebComponent</type>
  <callType>PXInstance</callType>
  <instanceObject>PXWorkWithFactura</instanceObject>
  <instanceLevel>Factura</instanceLevel>
  <instanceLevelNode>Selection</instanceLevelNode>
</component>

<!-- Embeber un PXParameterRequest de filtros -->
<component>
  <type>WebComponent</type>
  <callType>PXInstance</callType>
  <instanceObject>PXParameterRequestFiltroReporte</instanceObject>
  <instanceLevel>FiltroReporte</instanceLevel>
  <instanceLevelNode>Level</instanceLevelNode>
</component>

<!-- Embeber otro PXComposer (anidamiento) -->
<component>
  <type>WebComponent</type>
  <callType>PXInstance</callType>
  <instanceObject>PXComposerSubPanel</instanceObject>
  <instanceLevel>SubPanel</instanceLevel>
</component>
```

### Diagrama de composicion

```
PXComposer (WbMiDashboard)
│
├── WebComponent [PXInstance]
│   └── PXWorkWith:Factura:Selection ──► WWFactura / RWWFactura
│
├── WebComponent [PXInstance]
│   └── PXWorkWith:Cliente:Selection ──► WWCliente / RWWCliente
│
├── WebComponent [PXInstance]
│   └── PXParameterRequest:Filtro:Level ──► WbFiltro / RWbFiltro
│
├── WebComponent [GXObject]
│   └── MiWebPanelCustom (directo, no pattern)
│
├── HTML
│   └── <div>contenido estatico</div>
│
└── UserControl
    └── GraficoVentas
```

## Forms por plataforma

PXComposer soporta definir layouts completamente diferentes por plataforma dentro de la misma instancia. Esto es util cuando el layout responsive requiere una disposicion distinta al desktop.

### Escenario: layout diferente por plataforma

```xml
<forms>
  <!-- Layout Desktop: dos columnas lado a lado -->
  <form>
    <name>FormDesktop</name>
    <platform>Web Desktop</platform>
    <components>
      <component><!-- Panel izquierdo: lista --></component>
      <component><!-- Panel derecho: detalle --></component>
    </components>
  </form>

  <!-- Layout Responsive: componentes apilados -->
  <form>
    <name>FormResponsive</name>
    <platform>Web Responsive</platform>
    <components>
      <component><!-- Lista (ancho completo) --></component>
      <component><!-- Detalle (ancho completo, debajo) --></component>
    </components>
  </form>
</forms>
```

### Escenario: mismo layout para todas las plataformas

```xml
<forms>
  <form>
    <name>FormUnico</name>
    <platform>Any</platform>
    <components>
      <component><!-- Se usa para Desktop y Responsive --></component>
    </components>
  </form>
</forms>
```

## Hooks de codigo

PXComposer permite inyectar codigo GeneXus personalizado en puntos especificos del WebPanel generado mediante el nodo `<codes>`:

| Hook | Momento de ejecucion | Uso tipico |
|------|---------------------|------------|
| `Start` | Evento Start del WebPanel | Inicializacion de variables, carga de datos iniciales, validaciones de acceso |
| `Refresh` | Evento Refresh del WebPanel | Recarga de datos, actualizacion de componentes |
| `Load` | Evento Load del WebPanel | Logica de carga por registro |
| `Subroutine` | Subrutina reutilizable | Logica compartida entre eventos |

### Ejemplo de hooks

```xml
<codes>
  <code>
    <name>Start</name>
    <code>
      // Validar permisos
      if not PXCheckAccess.Udp("Dashboard")
        Return
      endif
      // Inicializar variables
      &amp;FechaDesde = Today() - 30
      &amp;FechaHasta = Today()
    </code>
  </code>
  <code>
    <name>Refresh</name>
    <code>
      // Recargar datos del dashboard
      PXDashboardData.Call(&amp;FechaDesde, &amp;FechaHasta, &amp;DatosResumen)
    </code>
  </code>
</codes>
```

### Eventos personalizados

Ademas de los hooks predefinidos, se pueden agregar eventos arbitrarios:

```xml
<events>
  <event>
    <name>ClienteSeleccionado</name>
    <code>
      // Manejar seleccion de cliente desde un WebComponent
      &amp;ClienteId = ClienteId.FromString(&amp;Data)
      // Refrescar componentes dependientes
    </code>
  </event>
</events>
```

## Capacidad dual-platform

PXComposer genera simultaneamente versiones Desktop y Responsive del mismo panel compuesto. Esta es una caracteristica compartida con PXWorkWith y PXParameterRequest.

### Flujo de generacion

```
Instancia PXComposer
│
├── generateWeb = true
│   └── Genera: Wb{Istance.Name}
│       ├── WebForm (HTML layout)
│       ├── Variables
│       ├── Events (con hooks)
│       └── Rules
│
├── generateWebResponsive = true
│   └── Genera: RWb{Istance.Name}
│       ├── AbstractForm (layout abstracto GeneXus)
│       ├── Variables
│       ├── Events (con hooks)
│       └── Rules
│
└── generateSD = false (reservado)
```

### Resolucion automatica de referencias

Cuando un componente usa `callType="PXInstance"`, el generador resuelve automaticamente la referencia al objeto correcto segun la plataforma que esta generando:

```
Instancia: PXComposerMiDashboard
  └── component: PXInstance → PXWorkWithFactura / Selection

Generacion Desktop (WbMiDashboard):
  └── WebComponent → WWFactura          (prefijo WW)

Generacion Responsive (RWbMiDashboard):
  └── WebComponent → RWWFactura         (prefijo RWW)
```

Esto garantiza que la version Desktop embeba componentes Desktop y la version Responsive embeba componentes Responsive, sin configuracion manual adicional.

## Ejemplos de uso real

### Instancias en modulos @PXTools

| Modulo | Instancia PXComposer | Proposito |
|--------|---------------------|-----------|
| @Alerts | `PXComposerSystemAlertMessage` | Panel de mensajes de alertas del sistema |
| @Alerts | `PXComposerSystemAlertSchedulerView` | Vista del scheduler de alertas |
| @FileStorage | `PXComposerFileStorage` | Panel de gestion de archivos almacenados |
| @Security | `PXComposerSecurityObjectAccess` | Panel de control de acceso a objetos |
| @Security | `PXComposerSecurityObjectRecordAccess` | Panel de acceso a nivel de registro |
| @WebServicesLog | `PXComposerWebServicesLog` | Panel de logs de web services |
| @WebServicesLog | `PXComposerWebServicesStatisticCounters` | Panel de contadores estadisticos de WS |

### Uso típico en aplicaciones

PXComposer se usa típicamente para dashboards y pantallas compuestas que combinan varias instancias (Selection/View de PXWorkWith, etc.) en una sola vista.

### Ubicaciones típicas de instancias

| Ubicacion | Instancias |
|-----------|-----------|
| `@<Modulo>/Formularios` | Formularios compuestos del frontend |
| `@<Modulo>/Inicio` | Pantalla de inicio con multiples paneles |

### Patron tipico: panel maestro-detalle compuesto

```xml
<!-- Instancia comun: dashboard con filtros + grilla + detalle -->
<instance>
  <level>
    <name>GestionClientes</name>
    <forms>
      <form>
        <name>FormPrincipal</name>
        <platform>Any</platform>
        <components>
          <!-- Filtros via PXParameterRequest -->
          <component>
            <type>WebComponent</type>
            <callType>PXInstance</callType>
            <instanceObject>PXParameterRequestFiltroClientes</instanceObject>
            <instanceLevel>FiltroClientes</instanceLevel>
            <instanceLevelNode>Level</instanceLevelNode>
          </component>
          <!-- Grilla de clientes via PXWorkWith -->
          <component>
            <type>WebComponent</type>
            <callType>PXInstance</callType>
            <instanceObject>PXWorkWithCliente</instanceObject>
            <instanceLevel>Cliente</instanceLevel>
            <instanceLevelNode>Selection</instanceLevelNode>
          </component>
        </components>
      </form>
    </forms>
  </level>
</instance>
```

## Relacion con otros patterns

### PXComposer como consumidor

PXComposer es el principal **consumidor** de los demas patterns de UI. Embebe sus objetos generados como WebComponents:

```
┌──────────────────────────────────────────────────────┐
│                    PXComposer                         │
│                  (CONSUMIDOR)                         │
│                                                      │
│   Puede embeber:                                     │
│                                                      │
│   ┌──────────────┐  ┌───────────────────┐            │
│   │ PXWorkWith   │  │ PXParameterRequest│            │
│   │ • Selection  │  │ • Level           │            │
│   │ • View       │  │                   │            │
│   │ • Prompt     │  │                   │            │
│   └──────────────┘  └───────────────────┘            │
│                                                      │
│   ┌──────────────┐  ┌───────────────────┐            │
│   │ PXComposer   │  │ WebPanel GeneXus  │            │
│   │ (anidado)    │  │ (cualquiera)      │            │
│   └──────────────┘  └───────────────────┘            │
└──────────────────────────────────────────────────────┘
```

### PXComposer como embebible

PXComposer a su vez puede ser embebido por:

| Pattern que embebe | Como lo referencia |
|--------------------|--------------------|
| Otro PXComposer | `callType="PXInstance"` con `instanceObject` apuntando al PXComposer |
| PXFlowController | Como accion de navegacion o paso del flujo |
| PXWorkWith | Como tab component dentro de un View |

### Diferencias clave con otros patterns

| Caracteristica | PXWorkWith | PXParameterRequest | PXComposer |
|---------------|------------|-------------------|------------|
| Parent Object | Transaction / (None) | WebPanel / Procedure / Transaction / (None) | Solo (None) |
| Genera grids | Si | No | No (los embebe) |
| Genera formularios | Si (View/Edit) | Si | No (los embebe) |
| Compone otros patterns | No | No | **Si** |
| Navegacion integrada | Selection↔View↔Edit | Modal/Popup | Via componentes embebidos |
| Generacion dual | Desktop + Responsive + SD | Desktop + Responsive | Desktop + Responsive |
| Cantidad de objetos | Muchos (Selection, View, Prompt, Controller, Tabs...) | 2 (Desktop + Responsive) | 2 por level (Desktop + Responsive) |
