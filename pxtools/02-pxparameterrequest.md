# PXParameterRequest — Documentación Completa del Pattern

## 1. Qué es y para qué sirve

PXParameterRequest es un pattern de PXTools que genera **WebPanels modales/popup para captura de parámetros**. Su propósito principal es crear formularios que:

- Capturan datos del usuario antes de ejecutar una acción (reportes, procesos, exportaciones)
- Muestran diálogos de confirmación ("¿Está seguro que desea eliminar?")
- Proveen formularios de login, cambio de contraseña, registro
- Abren popups de ingreso de datos que retornan valores al llamador
- Permiten selección de archivos para importación

Se asocia a objetos de tipo: **WebPanel**, **Procedure**, **Report**, **Transaction** o **(None)**.

### Diferencia clave con PXWorkWith

Mientras PXWorkWith genera pantallas completas con grillas de selección, vistas y edición de datos basados en transacciones, PXParameterRequest genera **formularios de captura simples** que típicamente se abren como popups/modales y retornan valores al llamador.

## 2. Objetos que genera

| Object ID | Tipo GeneXus | Patrón de nombre | Element (XPath) | Descripción |
|-----------|-------------|-------------------|-----------------|-------------|
| Level | WebPanel | `Wb{Instance.Name}` | `instance/level` | Formulario de parámetros (Web Desktop, template HTML) |
| LevelResponsive | WebPanel | `Wb{Instance.Name}` | `instance/level` | Formulario de parámetros (Web Responsive, layout abstracto) |
| ChkChosenValueSelected | Procedure | `Chk{Instance.Name}ChosenValueSelected` | `instance//variable/controlInfo[@controlType='Chosen']` | Validación de controles Chosen |
| RetChosenValues | DataProvider | `Ret{Instance.Name}ChosenValues` | (mismo que anterior) | DataProvider de valores Chosen |
| RetChosenResults | Procedure | `Ret{Instance.Name}ChosenResults` | (mismo que anterior) | Procedimiento de resultados Chosen |

### Nomenclatura dual-platform: caso especial

A diferencia de PXWorkWith que usa prefijos diferentes para cada plataforma (`WW` vs `RWW`, `View` vs `RView`), PXParameterRequest usa el **MISMO nombre** para ambas versiones:

```
PXWorkWith:          WWFactura (Desktop)  /  RWWFactura (Responsive)
PXParameterRequest:  WbLogin   (Desktop)  /  WbLogin    (Responsive)  ← MISMO NOMBRE
```

Esto significa que en un KB dado, se genera solo una versión activa (Desktop O Responsive), controlada por las propiedades `generateWeb` y `generateWebResponsive`.

## 3. Tipos de behaviour

El nodo `behaviour` define cómo se presenta y comporta el formulario. Es la propiedad más importante del level.

```
level/behaviour
├── type: enum
├── returnType: enum{Link;Message}
└── closeOnReturn: bool
```

### Tabla de tipos de behaviour

| Tipo | Presentación | Uso típico | Retorno de valores |
|------|-------------|------------|-------------------|
| `None` | Panel simple sin comportamiento especial | Formularios embebidos como WebComponents | No aplica |
| `Panel` | Panel estándar | Formularios independientes que no actúan como popup | Via parámetros |
| `PopupParameterRequest` | **Popup modal** (ventana emergente) | Confirmaciones, captura rápida de datos, selección | Retorna valores al llamador via popup |
| `ParameterRequest` | Página completa de parámetros | Captura de parámetros antes de ejecutar reportes/procesos | Redirige al objeto destino con los valores |
| `FloatingParameterRequest` | Overlay flotante sobre la página actual | Captura rápida sin perder contexto de la pantalla principal | Retorna valores sin navegar |

### returnType y closeOnReturn

```
returnType = Link     → Al confirmar, navega a la URL/objeto destino
returnType = Message  → Al confirmar, envía un mensaje al llamador (útil para WebComponents)

closeOnReturn = true  → Cierra el popup automáticamente al confirmar
closeOnReturn = false → Permanece abierto (útil para formularios de ingreso repetitivo)
```

### Diagrama de flujo según behaviour

```
                    Usuario hace clic en acción
                              │
                              ▼
              ┌───────────────────────────────┐
              │  ¿Qué behaviour tiene el      │
              │  PXParameterRequest?           │
              └───────────────┬───────────────┘
         ┌────────┬───────────┼───────────┬──────────────┐
         ▼        ▼           ▼           ▼              ▼
      ┌──────┐ ┌──────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
      │ None │ │Panel │ │ Popup    │ │Parameter │ │  Floating    │
      │      │ │      │ │Parameter │ │ Request  │ │  Parameter   │
      │      │ │      │ │ Request  │ │          │ │  Request     │
      └──┬───┘ └──┬───┘ └────┬─────┘ └────┬─────┘ └──────┬───────┘
         │        │          │             │              │
         ▼        ▼          ▼             ▼              ▼
      Embebido  Página    Popup modal   Página        Overlay
      en otro   indepen-  se abre      completa      flotante
      WebPanel  diente    sobre caller  reemplaza     sobre la
                          ┌────────┐   la actual      página
                          │Captura │                   actual
                          │datos   │
                          └───┬────┘
                              │
                          ┌───▼────┐
                          │Retorna │
                          │valores │
                          │al      │
                          │caller  │
                          └────────┘
```

## 4. Estructura XML de la instancia

Esquema completo del nodo `instance` definido en `PXParameterRequestInstance.xml` (203 KB):

```xml
<instance>
  <level name="Main" description="Formulario principal"
         masterPage="MasterPage" theme="PXTheme"
         generateWeb="true" generateWebResponsive="true" generateSD="false"
         platform="Any">

    <!-- Comportamiento del formulario -->
    <behaviour type="PopupParameterRequest"
               returnType="Link"
               closeOnReturn="true" />

    <!-- Layout del formulario -->
    <form>
      <attributes>
        <!-- Campos del formulario basados en atributos de la KB -->
      </attributes>
      <variables>
        <!-- Variables custom en el formulario -->
      </variables>
    </form>

    <!-- Acciones (botones) -->
    <actions>
      <action name="Confirm" caption="Aceptar" />
      <action name="Cancel" caption="Cancelar" />
    </actions>

    <!-- Eventos personalizados -->
    <events>
      <event name="OnConfirm">
        <!-- Código GeneXus -->
      </event>
    </events>

    <!-- Hooks de código en puntos específicos del ciclo de vida -->
    <codes>
      <code section="Start">/* Código al iniciar */</code>
      <code section="Refresh">/* Código en Refresh */</code>
      <code section="Load">/* Código en Load */</code>
      <code section="Subroutine">/* Subrutinas auxiliares */</code>
    </codes>

    <!-- Parámetros de entrada/salida del WebPanel -->
    <parameters>
      <parameter name="CustomerId" type="in" />
      <parameter name="SelectedDate" type="out" />
    </parameters>

    <!-- Variables adicionales -->
    <variables>
      <variable name="&Total" type="Numeric" length="12" decimals="2" />
    </variables>

    <!-- Condiciones (filtros, where) -->
    <conditions>
      <condition>CustomerStatus = 'A'</condition>
    </conditions>

    <!-- Grilla opcional dentro del formulario -->
    <grid>
      <attributes />
      <filter />
      <orders />
      <actions />
      <modes />
    </grid>

    <!-- Llamadas al confirmar (qué objeto se invoca con los parámetros) -->
    <calls>
      <call object="RptVentas" parameters="..." />
    </calls>

    <!-- Campos ocultos -->
    <hidden>
      <attribute name="CompanyId" />
    </hidden>

    <!-- Layouts condicionales -->
    <layouts>
      <layout condition="&Mode = 'Edit'" />
    </layouts>

    <!-- Documentación y ayuda -->
    <documentation />
    <help />
  </level>
</instance>
```

### Diagrama jerárquico completo

```
instance
├── level (puede haber múltiples levels)
│   ├── @name              → Nombre del level
│   ├── @description       → Descripción
│   ├── @masterPage         → MasterPage GeneXus a usar
│   ├── @theme              → Theme GeneXus
│   ├── @generateWeb        → bool: generar versión Desktop
│   ├── @generateWebResponsive → bool: generar versión Responsive
│   ├── @generateSD         → bool: generar versión Smart Devices
│   ├── @platform           → enum{Any;Web Desktop}
│   │
│   ├── behaviour
│   │   ├── @type           → enum{None;Panel;PopupParameterRequest;
│   │   │                          ParameterRequest;FloatingParameterRequest}
│   │   ├── @returnType     → enum{Link;Message}
│   │   └── @closeOnReturn  → bool
│   │
│   ├── form
│   │   ├── attributes[]    → Campos basados en atributos GeneXus
│   │   │   └── attribute
│   │   │       ├── @name, @description, @readonly, @visible
│   │   │       └── controlInfo → tipo de control (Edit, Combo, Chosen, etc.)
│   │   └── variables[]     → Variables custom en el form
│   │       └── variable
│   │           ├── @name, @domain, @type, @length
│   │           └── controlInfo
│   │               └── @controlType → enum{Edit;Combo;CheckBox;
│   │                                       RadioButton;Chosen;...}
│   │
│   ├── actions[]
│   │   └── action
│   │       ├── @name, @caption, @tooltip
│   │       ├── @type       → enum{Standard;Custom}
│   │       ├── @position   → enum{Top;Bottom;Both}
│   │       └── @condition  → expresión de visibilidad
│   │
│   ├── events[]
│   │   └── event
│   │       ├── @name       → Nombre del evento
│   │       └── (código GeneXus inline)
│   │
│   ├── codes
│   │   ├── code[@section='Start']
│   │   ├── code[@section='Refresh']
│   │   ├── code[@section='Load']
│   │   └── code[@section='Subroutine']
│   │
│   ├── parameters[]
│   │   └── parameter
│   │       ├── @name       → Nombre del parámetro
│   │       └── @type       → enum{in;out;inout}
│   │
│   ├── variables[]         → Variables del WebPanel (no del form)
│   ├── conditions[]        → Condiciones/Where
│   │
│   ├── grid (opcional)
│   │   ├── attributes[]    → Columnas de la grilla
│   │   ├── filter          → Filtros de la grilla
│   │   ├── orders[]        → Órdenes disponibles
│   │   ├── actions[]       → Acciones por fila
│   │   └── modes           → Insert/Update/Delete/Display
│   │
│   ├── calls[]             → Objetos a invocar al confirmar
│   │   └── call
│   │       ├── @object     → Nombre del objeto GeneXus destino
│   │       └── @parameters → Mapeo de parámetros
│   │
│   ├── hidden[]            → Campos ocultos (se pasan pero no se muestran)
│   ├── layouts[]           → Layouts condicionales
│   ├── documentation       → Documentación del level
│   └── help                → Ayuda contextual
```

## 5. Formulario y campos

El nodo `form` define los campos visibles del formulario de captura. Los campos pueden ser **atributos** (de la base de datos) o **variables** (definidas localmente).

### Tipos de control disponibles

Cada campo del formulario tiene un `controlInfo` que define cómo se renderiza:

| controlType | Descripción | Ejemplo de uso |
|-------------|-------------|---------------|
| `Edit` | Campo de texto libre | Nombre, descripción, observaciones |
| `Combo` | Lista desplegable | Estado, tipo, categoría |
| `CheckBox` | Casilla de verificación | Activo/Inactivo, aceptar términos |
| `RadioButton` | Botones de opción | Tipo de documento, género |
| `Chosen` | Selector múltiple con búsqueda (genera objetos adicionales) | Tags, categorías múltiples |
| `DatePicker` | Selector de fecha | Fecha desde, fecha hasta |
| `File` | Selector de archivo | Importar archivo CSV/Excel |
| `Image` | Imagen | Logo, foto |
| `TextBlock` | Texto estático (no editable) | Instrucciones, advertencias |

### Controles tipo Chosen: objetos adicionales

Cuando un campo usa `controlType='Chosen'`, el pattern genera tres objetos adicionales automáticamente:

```
Variable con controlType='Chosen'
        │
        ├──► Chk{Instance.Name}ChosenValueSelected  (Procedure)
        │    Valida que se haya seleccionado al menos un valor
        │
        ├──► Ret{Instance.Name}ChosenValues          (DataProvider)
        │    Retorna la lista de valores posibles para el selector
        │
        └──► Ret{Instance.Name}ChosenResults         (Procedure)
             Retorna los resultados seleccionados por el usuario
```

### Ejemplo de form con múltiples tipos de control

```xml
<form>
  <attributes>
    <attribute name="FechaDesde" description="Fecha desde">
      <controlInfo controlType="DatePicker" />
    </attribute>
    <attribute name="FechaHasta" description="Fecha hasta">
      <controlInfo controlType="DatePicker" />
    </attribute>
    <attribute name="ClienteId" description="Cliente">
      <controlInfo controlType="Edit" />
    </attribute>
  </attributes>
  <variables>
    <variable name="&TipoReporte" domain="TipoReporte">
      <controlInfo controlType="Combo" />
    </variable>
    <variable name="&IncluirAnulados">
      <controlInfo controlType="CheckBox" />
    </variable>
    <variable name="&ArchivoImportar">
      <controlInfo controlType="File" />
    </variable>
  </variables>
</form>
```

## 6. Sistema de acciones y calls

### Acciones (botones)

Las acciones definen los botones del formulario. Típicamente un PXParameterRequest tiene al menos dos acciones: **Confirm** y **Cancel**.

```xml
<actions>
  <!-- Acción estándar de confirmación -->
  <action name="Confirm" caption="Aceptar"
          type="Standard" position="Bottom" />

  <!-- Acción estándar de cancelación -->
  <action name="Cancel" caption="Cancelar"
          type="Standard" position="Bottom" />

  <!-- Acción personalizada -->
  <action name="Preview" caption="Vista previa"
          type="Custom" position="Bottom"
          condition="&TipoReporte = 'PDF'" />
</actions>
```

### Calls (invocación al confirmar)

El nodo `calls` define qué objetos se ejecutan cuando el usuario confirma el formulario. Los parámetros capturados en el form se pasan como argumentos.

```xml
<calls>
  <!-- Llama a un reporte con los parámetros capturados -->
  <call object="RptVentasPorPeriodo"
        parameters="&FechaDesde, &FechaHasta, &ClienteId, &TipoReporte" />
</calls>
```

### Flujo de ejecución

```
┌──────────────────────────────┐
│  PXParameterRequest abierto  │
│  (popup/página/floating)     │
│                              │
│  ┌────────────────────────┐  │
│  │  Campos del formulario │  │
│  │  FechaDesde: [       ] │  │
│  │  FechaHasta: [       ] │  │
│  │  Cliente:    [       ] │  │
│  │  Tipo:       [▼ PDF  ] │  │
│  └────────────────────────┘  │
│                              │
│  [Vista previa]  [Aceptar]   │
│                  [Cancelar]  │
└──────────────┬───────────────┘
               │ Clic en "Aceptar"
               ▼
┌──────────────────────────────┐
│  1. Ejecuta validaciones     │
│  2. Ejecuta evento OnConfirm │
│  3. Invoca calls:            │
│     RptVentasPorPeriodo(     │
│       &FechaDesde,           │
│       &FechaHasta,           │
│       &ClienteId,            │
│       &TipoReporte           │
│     )                        │
│  4. Si closeOnReturn=true,   │
│     cierra el popup          │
└──────────────────────────────┘
```

## 7. Grilla opcional

Un PXParameterRequest puede incluir opcionalmente una **grilla** dentro del formulario. Esto es útil cuando el formulario necesita mostrar una lista de registros para que el usuario seleccione o revise antes de confirmar.

```xml
<grid>
  <!-- Columnas de la grilla -->
  <attributes>
    <attribute name="ProductoId" />
    <attribute name="ProductoDescripcion" />
    <attribute name="ProductoPrecio" />
  </attributes>

  <!-- Filtros de la grilla -->
  <filter>
    <condition>ProductoActivo = true</condition>
  </filter>

  <!-- Órdenes disponibles -->
  <orders>
    <order name="PorNombre" attributes="ProductoDescripcion" />
    <order name="PorPrecio" attributes="ProductoPrecio" />
  </orders>

  <!-- Acciones por fila -->
  <actions>
    <action name="Select" caption="Seleccionar" />
  </actions>

  <!-- Modos permitidos -->
  <modes insert="false" update="false" delete="false" display="true" />
</grid>
```

### Caso de uso típico

```
┌──────────────────────────────────────┐
│  Seleccionar productos a exportar    │
│                                      │
│  Formato: [▼ Excel ]                 │
│                                      │
│  ┌──┬────────────┬──────────┬──────┐ │
│  │✓ │ Producto   │ Precio   │      │ │
│  ├──┼────────────┼──────────┼──────┤ │
│  │☑ │ Widget A   │ $100.00  │ [Sel]│ │
│  │☐ │ Widget B   │ $250.00  │ [Sel]│ │
│  │☑ │ Widget C   │ $75.00   │ [Sel]│ │
│  └──┴────────────┴──────────┴──────┘ │
│                                      │
│         [Exportar]  [Cancelar]       │
└──────────────────────────────────────┘
```

## 8. Hooks de código

El nodo `codes` permite inyectar código GeneXus en puntos específicos del ciclo de vida del WebPanel generado. Esto es fundamental para personalizar el comportamiento sin modificar el código generado directamente.

### Secciones disponibles

| Sección | Momento de ejecución | Uso típico |
|---------|---------------------|------------|
| `Start` | Evento `Start` del WebPanel | Inicializar variables, verificar permisos, cargar valores por defecto |
| `Refresh` | Evento `Refresh` | Recalcular valores, actualizar estado de controles |
| `Load` | Evento `Load` de la grilla (si existe) | Calcular campos derivados por fila |
| `Subroutine` | Se declaran como subrutinas del WebPanel | Lógica reutilizable dentro del formulario |

### Ejemplo de hooks de código

```xml
<codes>
  <code section="Start">
    // Verificar permisos
    if not &PXSession.IsAuthenticated()
      return
    endif
    // Valores por defecto
    &FechaDesde = Today() - 30
    &FechaHasta = Today()
  </code>

  <code section="Refresh">
    // Actualizar total estimado según filtros
    &TotalEstimado = CalcTotalVentas(&FechaDesde, &FechaHasta, &ClienteId)
  </code>

  <code section="Subroutine">
    Sub 'ValidarFechas'
      if &FechaDesde > &FechaHasta
        msg('La fecha desde no puede ser mayor a la fecha hasta')
        &OK = false
      endif
    EndSub
  </code>
</codes>
```

> **Formato e indentación del CDATA**: en el `.gxPattern` real cada hook es `<code type="Start"><![CDATA[…]]></code>`; la primera línea va en **columna 0** y solo el anidamiento de bloques indenta (+1 tab). Regla completa (común a los patterns) en [`00-overview.md`](00-overview.md) → *Hooks de código: formato e indentación del CDATA*.

### Eventos personalizados

Además de los hooks de código, el nodo `events` permite definir eventos GeneXus completos:

```xml
<events>
  <event name="'Validar'">
    Do 'ValidarFechas'
    if &OK
      msg('Parámetros válidos', status)
    endif
  </event>
</events>
```

## 9. Capacidad dual-platform

PXParameterRequest soporta generación para **Web Desktop** y **Web Responsive** desde la misma definición de instancia.

### Propiedades de control de plataforma

```xml
<level name="Login"
       generateWeb="true"              <!-- Genera versión Desktop -->
       generateWebResponsive="true"    <!-- Genera versión Responsive -->
       generateSD="false"              <!-- No genera versión Smart Devices -->
       platform="Any">                 <!-- Any | Web Desktop -->
```

### Propiedad platform

| Valor | Efecto |
|-------|--------|
| `Any` | El level se genera para todas las plataformas habilitadas (Web Desktop + Responsive) |
| `Web Desktop` | El level se genera **solo** para Web Desktop, incluso si `generateWebResponsive=true` |

### Objetos generados según plataforma

```
level con generateWeb=true, generateWebResponsive=true
│
├──► Level (Desktop)
│    Objeto: WebPanel "Wb{Instance.Name}"
│    Form:   HTML layout (template *WebForm.dll)
│    Genera: Formulario con tabla HTML, controles posicionados
│
└──► LevelResponsive (Responsive)
     Objeto: WebPanel "Wb{Instance.Name}"    ← MISMO NOMBRE
     Form:   Abstract layout (template *AbstractForm.dll)
     Genera: Formulario con layout abstracto responsive
```

**Nota importante**: Dado que ambas versiones comparten el mismo nombre, en la práctica solo una puede estar activa en el KB. La propiedad `generateWeb`/`generateWebResponsive` controla cuál se genera.

### Diferencias en templates de generación

| Parte del objeto | Template Desktop | Template Responsive |
|------------------|-----------------|-------------------|
| WebForm | `*WebForm.dll` (HTML) | `*AbstractForm.dll` (Abstract) |
| Variables | Compartido | Compartido |
| Events | Compartido (con ajustes) | Compartido (con ajustes) |
| Rules | Compartido | Compartido |

## 10. Relación con otros patterns

PXParameterRequest se integra con los demás patterns de PXTools de múltiples formas:

### Con PXWorkWith

```
┌────────────────────────────────────────┐
│  PXWorkWith (Selection de Facturas)    │
│                                        │
│  [Nueva] [Eliminar] [Exportar Excel]   │
│  ┌──────────────────────────────────┐  │
│  │  Grilla de facturas              │  │
│  └──────────────────────────────────┘  │
│                                        │
│  Clic en [Exportar Excel] ──────────── │ ──► Abre PXParameterRequest
└────────────────────────────────────────┘     como popup para capturar
                                               FechaDesde, FechaHasta,
                                               Formato de exportación
```

Las acciones de PXWorkWith (en Selection, View, etc.) pueden configurarse para abrir un PXParameterRequest como popup antes de ejecutar la acción real.

### Con PXFlowController

```
┌─────────────────────────────────────────────────┐
│  PXFlowController (Proceso de facturación)      │
│                                                 │
│  Paso 1: Seleccionar cliente ──► PXWorkWith     │
│  Paso 2: Confirmar datos ─────► PXParameterReq. │  ← Confirmación
│  Paso 3: Procesar ────────────► Procedure       │
│  Paso 4: Resultado ──────────► PXParameterReq.  │  ← Muestra resultado
└─────────────────────────────────────────────────┘
```

PXFlowController puede invocar PXParameterRequest como un paso del flujo, típicamente para confirmación de datos o captura de parámetros intermedios.

### Con PXComposer

```
┌────────────────────────────────────────────────┐
│  PXComposer (Dashboard)                        │
│  ┌─────────────────┐  ┌─────────────────────┐  │
│  │  PXWorkWith     │  │  PXParameterRequest  │  │
│  │  (lista resumida│  │  (filtros globales)  │  │  ← Embebido
│  │   de ventas)    │  │  FechaDesde: [    ]  │  │     como
│  │                 │  │  FechaHasta: [    ]  │  │     WebComponent
│  │                 │  │  [Aplicar filtro]    │  │
│  └─────────────────┘  └─────────────────────┘  │
└────────────────────────────────────────────────┘
```

PXComposer puede embeber un PXParameterRequest como WebComponent, útil para paneles de filtros en dashboards.

### Mapa completo de relaciones

```
┌──────────────┐         invoca como popup
│  PXWorkWith  │ ──────────────────────────────► ┌─────────────────────┐
│  (acciones)  │                                 │  PXParameterRequest │
└──────────────┘                                 │                     │
                                                 │  Captura parámetros │
┌──────────────┐         invoca como paso        │  y retorna valores  │
│PXFlowContro- │ ──────────────────────────────► │                     │
│ ller (steps) │                                 └──────────┬──────────┘
└──────────────┘                                            │
                                                            │ calls → invoca
┌──────────────┐         embebe como WebComponent           ▼
│  PXComposer  │ ──────────────────────────────► ┌─────────────────────┐
│  (dashboard) │                                 │  Procedure / Report │
└──────────────┘                                 │  Transaction / WP   │
                                                 └─────────────────────┘
```

## 11. Ejemplos de uso real

PXParameterRequest se usa extensivamente, tanto en los propios módulos @PXTools como en las aplicaciones que los integran.

### Ejemplos por categoría

#### Formularios de login y seguridad

| Instancia | Módulo | Behaviour | Descripción |
|-----------|--------|-----------|-------------|
| `Login` | @Security | Panel | Formulario de inicio de sesión |
| `Registration` | @Security | ParameterRequest | Registro de nuevo usuario |
| `ChangePassword` | @Security | PopupParameterRequest | Cambio de contraseña |
| `RecoverPassword` | @Security | ParameterRequest | Recuperación de contraseña |

#### Diálogos de confirmación

| Instancia | Módulo | Behaviour | Descripción |
|-----------|--------|-----------|-------------|
| `Confirm` | @APIs | PopupParameterRequest | Confirmación genérica "¿Está seguro?" |
| `ConfirmDelete` | @APIs | PopupParameterRequest | Confirmación de eliminación |

#### Captura de parámetros para reportes

```xml
<!-- Ejemplo: Parámetros para reporte de ventas -->
<instance>
  <level name="VentasPorPeriodo" description="Reporte de ventas por período"
         generateWeb="true" generateWebResponsive="false">
    <behaviour type="ParameterRequest" returnType="Link" closeOnReturn="true" />
    <form>
      <attributes>
        <attribute name="FechaDesde" />
        <attribute name="FechaHasta" />
      </attributes>
      <variables>
        <variable name="&Categoria">
          <controlInfo controlType="Combo" />
        </variable>
        <variable name="&Etiquetas">
          <controlInfo controlType="Chosen" />
        </variable>
      </variables>
    </form>
    <actions>
      <action name="Confirm" caption="Generar reporte" />
      <action name="Cancel" caption="Cancelar" />
    </actions>
    <calls>
      <call object="RptVentasPorPeriodo" />
    </calls>
  </level>
</instance>
```

#### Importación de archivos

| Instancia | Módulo | Behaviour | Descripción |
|-----------|--------|-----------|-------------|
| `ImportCSV` | @ExportImport | PopupParameterRequest | Selección de archivo CSV para importar |
| `ImportExcel` | @ExportImport | PopupParameterRequest | Selección de archivo Excel |

#### Gestión de tareas y correos

| Instancia | Módulo | Behaviour | Descripción |
|-----------|--------|-----------|-------------|
| `SendMail` | @SendMails | PopupParameterRequest | Formulario para enviar email |
| `TaskCreate` | @TaskManager | PopupParameterRequest | Creación rápida de tarea |

### Módulos @PXTools que usan PXParameterRequest

| Módulo | Cantidad de instancias | Uso principal |
|--------|----------------------|---------------|
| @APIs | Varias | Confirmaciones genéricas |
| @Security | 4+ | Login, registro, cambio/recuperación de contraseña |
| @SendMails | 1+ | Formulario de envío de email |
| @TaskManager | 2+ | Creación y edición rápida de tareas |
| @ExportImport | 2+ | Selección de archivos para importar |

## 12. Settings del pattern

Los settings globales del pattern se definen en `PXParameterRequestSettings.xml` (56 KB) y configuran valores por defecto para todas las instancias.

### Grupos principales de settings

| Grupo | Propósito |
|-------|-----------|
| `Context` | Información de contexto (KB, módulo, prefijos) |
| `Form` | Configuración por defecto del formulario (ancho, alto, estilos) |
| `Grid` | Configuración por defecto de la grilla opcional |
| `Help` | Sistema de ayuda contextual |
| `Images` | Imágenes por defecto para botones y acciones |
| `Labels` | Textos/etiquetas por defecto (Aceptar, Cancelar, etc.) |
| `MasterPages` | MasterPages por defecto por plataforma |
| `PagingButtons` | Configuración de paginación de la grilla |
| `Prefixes` | Prefijos de nombres de objetos generados |
| `Security` | Configuración de seguridad (permisos, autenticación) |
| `StandardActions` | Acciones estándar y sus propiedades por defecto |
| `Tab` | Configuración de tabs si el formulario tiene tabs |
| `Template` | Template GeneXus por defecto |
| `Templates` | Templates de generación de código (DLL/DKT) |
| `ThemeClasses` | Clases de tema CSS por defecto para cada elemento |
| `Default Instance` | Instancia por defecto al crear un nuevo PXParameterRequest |
| `Navigation` | Configuración de navegación (breadcrumbs, menús) |
| `Objects` | Configuración de objetos generados |

## 13. Sistema de acciones

Las acciones de PXParameterRequest comparten la misma estructura que PXWorkWith y PXComposer. La referencia completa del nodo Action (propiedades, callTypes, ConditionalCalls, ciclo de ejecucion, reglas de PXInstance) esta en [12-acciones-patterns-ui.md](12-acciones-patterns-ui.md).

### Propiedad exclusiva: execute

PXParameterRequest tiene una propiedad exclusiva **`execute`** en las acciones que controla que validaciones se aplican antes de ejecutar. Dado que PXParameterRequest tiene funcion principal de **data entry**, pueden existir condiciones de datos obligatorios definidas en el nodo `conditions` del level que son comunes a muchas acciones:

```xml
<!-- Condiciones generales del formulario (datos obligatorios) -->
<conditions>
  <condition>not &FechaDesde.IsEmpty()</condition>
  <condition>not &ClienteId.IsEmpty()</condition>
</conditions>
```

| Valor de execute | Comportamiento |
|------------------|---------------|
| **General Conditions** | Ejecuta las `conditions` generales del level **y** las condiciones locales de la accion. Para acciones que requieren todos los datos completos (Aceptar, Confirmar) |
| **Action Conditions Only** | Ejecuta **solo** las condiciones locales de la accion, ignorando las generales. Para acciones que no requieren validacion de todos los campos (Vista previa, Buscar) |

Ejemplo con `execute`:

```xml
<actions>
  <action name="Confirm" caption="Aceptar"
          execute="General Conditions"
          callType="Call" gxObject="GenerarReporte" />

  <action name="Preview" caption="Vista previa"
          execute="Action Conditions Only"
          condition="&TipoReporte = 'PDF'"
          callType="Link" gxObject="PreviewReporte" />

  <action name="Cancel" caption="Cancelar" callType="Return" />
</actions>
```

### Estructura del nodo Action (resumen)

Cada accion soporta codigo pre/post invocacion:

```xml
<action name="Aceptar"
        caption="Aceptar"
        type="Standard"
        position="Bottom"
        conditionPreviousCode="// Codigo previo para evaluar condicion"
        condition="&EsValido = true"
        actionPreviousCode="// Codigo que se ejecuta ANTES de la invocacion principal"
        callType="Call"
        gxObject="MiProcedimiento"
        actionPostCode="// Codigo que se ejecuta DESPUES de la invocacion principal"
        refreshAction="...">
  <parameters>
    <parameter name="Param1" />
  </parameters>
</action>
```

### Tipos de callType en acciones

| callType | Descripcion | Ejemplo de uso |
|----------|-------------|----------------|
| `Call` | Invoca un Procedure (sin interfaz) | Contabilizar, procesar, actualizar registros |
| `Link` | Navega a otro WebPanel/Transaction | Abrir pantalla de detalle |
| `External Link` | Abre WebPanel no generado por PXTools | Abrir WebPanel manual |
| `Prompt` | Abre dialogo modal | Abrir otro PXParameterRequest como popup |
| `Subroutine` | Ejecuta subrutina local | Logica interna reutilizable |
| `Event` | Ejecuta codigo inline (sin objeto principal) | Cuando toda la logica es codigo procedural |
| `Submit` | Somete proceso batch | Ejecutar tarea asincrona |
| `None` | No ejecuta nada | Placeholder, solo el pre/post code importa |
| `Return` | Solo retorna/cierra popup | Cancelar, salir |

### Regla critica: linkType y migracion Responsive

Solo las acciones con `callType="Link"` necesitan `linkType="PXInstance"` para ser independientes de la plataforma. Los demas callTypes (`Call`, `Submit`, `Event`, etc.) apuntan a Procedures cuyos nombres no cambian entre Desktop y Responsive.

```
callType="Link"  + linkType="PXInstance" → CORRECTO (independiente de plataforma)
callType="Link"  + linkType="GXObject"  → PROBLEMA (se rompe al cambiar de plataforma)
callType="Call"  + gxObject="MiProc"    → OK (Procedures no cambian de nombre)
callType="Event" + (codigo inline)      → OK (no depende de nombres externos)
```

### Ciclo de ejecucion de una accion

```
1. conditionPreviousCode  → Codigo para preparar evaluacion de condicion
2. condition              → Si es false, la accion no se ejecuta
3. actionPreviousCode     → Validaciones, preparacion de datos
4. [Invocacion principal] → Segun callType (Call/Link/Event/etc.)
5. actionPostCode         → Logica posterior (actualizar estados, WebSession, etc.)
6. refreshAction          → Refresco de pantalla/grilla
```

## 14. Alcance real de PXParameterRequest — Guia ampliada para IAs

### Principio fundamental

**Toda logica de negocio es migrable a hooks.** No importa la cantidad de subrutinas, For Each, validaciones, uso de WebSession, SDTs, o llamadas a Procedures. PXParameterRequest provee hooks en todos los puntos del ciclo de vida del WebPanel:

| Codigo original | Hook destino |
|-----------------|-------------|
| Event Start | `codes/Start` |
| Sub 'MiSubrutina' | `codes/Subroutine` |
| Event Refresh | `codes/Refresh` |
| Event Load (grilla) | `codes/Load` |
| Event 'Aceptar' (validaciones) | Action `actionPreviousCode` |
| Event 'Aceptar' (invocacion principal) | Action `callType` + `gxObject` |
| Event 'Aceptar' (logica post) | Action `actionPostCode` |
| Event 'MiEvento' custom | `events/event` |

### Catalogo ampliado de casos de uso

Basado en analisis de 225 WebPanels manuales de una KB real de contabilidad:

| Caso de uso | behaviour | Elementos clave | Ejemplo real |
|-------------|-----------|-----------------|--------------|
| **Confirmacion simple** | PopupParameterRequest | Mensaje + Aceptar/Cancelar | ConfirmaCambio, ConfirmarBorrarPUC |
| **Confirmacion con causal** | PopupParameterRequest | Mensaje + campo texto + validacion min 10 chars | Anula*, TemIntConfirmaAnula |
| **Activar/Inactivar entidad** | PopupParameterRequest | Start evalua estado, Aceptar llama proc toggle | ConfirmarBanco, ConfirmarCiudad |
| **Activar/Inactivar con validacion arbol** | PopupParameterRequest | Start recorre jerarquia padre/hijos, valida dependencias | ConfirmarCuenta, ConfirmarCentroCosto |
| **Borrar con validaciones** | PopupParameterRequest | Start verifica dependencias, oculta Aceptar si hay | ConfirmarBorrarCuentaEmp |
| **Seleccion simple con grilla** | PopupParameterRequest | Grid + filtros + Enter retorna valor | SelBancos, SelEmpresas, SelPuc |
| **Seleccion con creacion de registros** | PopupParameterRequest | Grid + seleccion + crea registro temporal via proc | SelCuentaxPagar, SelCuentaEmpCruce |
| **Seleccion con validacion compleja** | PopupParameterRequest | Grid + Load con logica NIIF/Local/Mixto, homologos | SelTipComAge, SelTipComPar |
| **Seleccion con checkbox multi-fila** | PopupParameterRequest | Grid con boolean por fila + valor editable | SelDeduccion |
| **Seleccion con creacion inline** | PopupParameterRequest | Grid + form mini para crear nuevo registro | SelPaisCre |
| **Anulacion de proceso** | PopupParameterRequest | Observacion + validacion estado + proc de anulacion | AnulaPagosTes, WPAnularFactura |
| **Reversa de contabilizacion** | PopupParameterRequest | Start valida, Aceptar reversa + comprobantes hermanos | ConfirmarReversarComprobante |
| **Captura de parametros para reporte** | ParameterRequest | Form con fechas/filtros + calls a reporte | PXEstadosFinancieros |
| **Generacion de archivos** | PopupParameterRequest | Form + callType Event + genera TXT/Excel | AutorizarPagosRep |
| **Carga/Upload de archivos** | PopupParameterRequest | Form con controlType File + validacion extension | CarDocAdjTercero, CargarFirmaDigital, CargarLogoEmpresa |
| **Edicion de campo unico** | PopupParameterRequest | Form con un campo editable + Confirmar | ModificaNombreNIIF |
| **Visor de errores con grilla** | PopupParameterRequest | Grid readonly cargada desde SDT/WebSession | ErroresComprobante, ErroresRPComprobante |
| **Visor readonly de datos** | Panel o None | Solo display, Salir | ptDetalleCuenta, WPVerConceptosRP, WPVerFirma |
| **Ajuste/calculo con form** | PopupParameterRequest | Form con inputs + logica de calculo + proc | WpnAjustarCausacionCPS |
| **Asignacion de cheques** | PopupParameterRequest | Start valida consecutivos, Aceptar asigna + contabiliza | ConfirmaChequeTraslado |
| **CRUD de detalle** | PopupParameterRequest | Form INS/UPD/DSP con imagen, fechas, validaciones | CreaModificaDetalleContenidoParametrico |
| **Ejecucion one-shot** | Panel | Un solo boton que dispara un proceso | EjecucionPagos |

### Errores comunes de clasificacion por IAs

**NO marcar como "Manual" un WebPanel solo porque:**

1. **Tiene mucha logica en Start/Aceptar** → Toda va en hooks (`codes/Start`, `actionPreviousCode`, `actionPostCode`)
2. **Usa WebSession** → WebSession es mecanismo estandar de comunicacion entre pantallas, no impide migracion
3. **Tiene muchas subrutinas** → Todas van en `codes/Subroutine`
4. **Crea/modifica registros** → La accion puede usar `callType="Call"` o `callType="Event"`
5. **Tiene grilla con Load complejo** → La grilla es un nodo opcional con su propio hook Load
6. **Oculta/muestra controles en Start** → Logica estandar en hook Start
7. **Usa SDTs para pasar datos** → Totalmente soportado en variables y hooks
8. **Tiene validaciones complejas** → Van en `actionPreviousCode` o `conditionPreviousCode`

**SI marcar como "Manual" solo cuando:**

1. Es una **pantalla de Login** (flujo de autenticacion custom, IsMain=True, sin MasterPage)
2. Es una **utilidad de testing/ejemplo** (no es funcionalidad de produccion)
3. Depende de un **framework externo** (Scheduler, controles de terceros especificos)
4. Es un **redirect automatico** sin interaccion de usuario

## Resumen rapido para IA

```
PXParameterRequest = Formulario modal/popup/panel para CUALQUIER interaccion
                     que no sea un CRUD maestro-detalle (eso es PXWorkWith)

CUANDO USAR:
  - Captura de datos antes de ejecutar algo → ParameterRequest
  - Popup de confirmacion (simple o complejo) → PopupParameterRequest
  - Formulario flotante → FloatingParameterRequest
  - Embeber en otro panel → None o Panel
  - Visor readonly de datos → Panel o None (sin acciones de modificacion)
  - Seleccion con grilla (popup) → PopupParameterRequest + grid node
  - Carga/upload de archivos → PopupParameterRequest + controlType File
  - Generacion de archivos/reportes → ParameterRequest + callType Event
  - Ejecucion de proceso one-shot → Panel + un Action con callType Call/Event

TODA LA LOGICA VA EN HOOKS:
  - Start → codes/Start
  - Subrutinas → codes/Subroutine
  - Refresh → codes/Refresh
  - Load (grilla) → codes/Load
  - Boton Aceptar → Action (actionPreviousCode + callType + actionPostCode)
  - Eventos custom → events/event

QUE GENERA:
  - 1 WebPanel (mismo nombre Desktop y Responsive)
  - 3 objetos extra por cada control Chosen

SE INTEGRA CON:
  - PXWorkWith → acciones abren PXParameterRequest como popup
  - PXFlowController → pasos del flujo usan PXParameterRequest
  - PXComposer → embebe PXParameterRequest como WebComponent
```
