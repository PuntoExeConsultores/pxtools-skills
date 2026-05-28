# PXTools — Visión General del Framework

## Qué es PXTools

PXTools es un framework de **patterns (patrones)** para [GeneXus](https://www.genexus.com/) que transforma definiciones declarativas XML en código GeneXus completo y funcional. Fue creado por **PuntoExe Consultores** (Uruguay) y soporta las plataformas:

- **Web Desktop** (layout HTML clásico)
- **Web Responsive** (layout abstracto de GeneXus)
- **Smart Devices** (aplicaciones móviles nativas)

## Arquitectura del Sistema de Patterns

### Tres niveles de un pattern

```
┌──────────────────────────────────────────────────────────────┐
│  NIVEL 1: DEFINICIÓN DEL PATRÓN (.Pattern)                   │
│  ─────────────────────────────────────────                   │
│  Schema XML que define:                                      │
│  • Estructura de nodos y propiedades permitidas              │
│  • Qué objetos GeneXus genera cada nodo                      │
│  • Código del generador (DLL compiladas)                      │
│  • Parent Objects (a qué tipo de objeto se asocia)           │
│  Ubicación: Patterns/<PatternName>/<PatternName>.Pattern     │
├──────────────────────────────────────────────────────────────┤
│  NIVEL 2: INSTANCIA DEL PATRÓN (.gxPattern)                  │
│  ─────────────────────────────────────────                   │
│  XML que sigue el schema del .Pattern                        │
│  Configurado por el desarrollador para una entidad/función   │
│  Ubicación: Knowledge Base/<ruta>/<Nombre>.gxPattern         │
├──────────────────────────────────────────────────────────────┤
│  NIVEL 3: OBJETOS GENERADOS (.Childs/)                       │
│  ─────────────────────────────────────                       │
│  Objetos GeneXus resultantes del generador                   │
│  WebPanels, Procedures, DataProviders, SDTs, APIs, etc.      │
│  Ubicación: Knowledge Base/<ruta>/.<Nombre>.gxSource.Childs/ │
└──────────────────────────────────────────────────────────────┘
```

### Archivos de soporte por pattern

Cada pattern tiene archivos adicionales en su directorio de definición:

| Archivo | Propósito |
|---------|-----------|
| `<Pattern>Instance.xml` | Instance Specification: schema completo de la instancia (ElementTypes, atributos, nodos hijos, tipos, valores por defecto) |
| `<Pattern>Settings.xml` | Preferencias/configuración global del pattern (valores por defecto, prefijos, clases de tema, etc.) |
| `<Pattern>CustomTypes.xml` | Definiciones de tipos personalizados usados en las propiedades |
| `Templates/` | Código del generador (DLL compiladas) que produce cada parte de los objetos GeneXus. Compilado para proteger propiedad intelectual y licenciamiento |
| `Icons/` | Iconos del pattern para el IDE de GeneXus |
| `Resources/` | Recursos adicionales (settings por defecto, exportaciones iniciales) |

### Anatomía de un .Pattern

Cada archivo `.Pattern` tiene esta estructura XML:

```xml
<Pattern Publisher="PuntoExe" Id="GUID" Name="PXWorkWith" Version="10.3.0">
  <Definition>
    <InstanceName>PXWorkWith{0}</InstanceName>              <!-- Naming convention -->
    <InstanceSpecification>PXWorkWithInstance.xml</InstanceSpecification>
    <SettingsSpecification>PXWorkWithSettings.xml</SettingsSpecification>
    <CustomTypeDefinitions>PXWorkWithCustomTypes.xml</CustomTypeDefinitions>
    <Implementation>PuntoExe.Patterns.PXWorkWith.dll</Implementation>
    <ParentObjects>                                         <!-- A qué se asocia -->
      <ParentObject Type="Transaction" />
      <ParentObject Type="(None)" />
    </ParentObjects>
  </Definition>

  <Objects>
    <!-- Cada Object define un objeto GeneXus a generar -->
    <Object Type="WebPanel" Id="Selection" Name="WW{Instance.Name}"
            Element="instance/level/selection">             <!-- Nodo XML fuente -->
      <Part Type="WebForm" Template="Templates\SelectionWebForm.dll" />
      <Part Type="Variables" Template="Templates\SelectionVariables.dll" />
      <Part Type="Events" Template="Templates\SelectionEvents.dll" />
      <Part Type="Rules" Template="Templates\SelectionRules.dll" />
    </Object>
    <!-- ... más objetos ... -->
  </Objects>
</Pattern>
```

Puntos clave:
- **`Element`** indica qué nodo XML de la instancia dispara la generación de ese objeto
- **`Name`** usa placeholders como `{Instance.Name}`, `{Element.name}` para naming dinámico
- **`Count="*"`** indica que puede generar múltiples objetos (uno por cada match del Element XPath)
- **`Template`** (en el .Pattern) apunta al **código del generador** (DLL) que produce cada parte del objeto GeneXus. No confundir con los "Templates de UI" de PXTools (ver sección siguiente)

## Catálogo de Patterns

### Patterns de UI (generan WebPanels)

| Pattern | Genera | Parent Object | Dual-platform |
|---------|--------|--------------|---------------|
| **PXWorkWith** | Selection, View, Edit, Prompt, Controller, Tabs (Grid/Tabular), Export Excel, Charts | Transaction / (None) | Si (Web + Responsive + SD) |
| **PXParameterRequest** | WebPanel modal/popup para captura de parámetros | WebPanel / Procedure / Report / Transaction / (None) | Si (Web + Responsive) |
| **PXComposer** | WebPanel compositor que embebe WebComponents | (None) | Si (Web + Responsive) |

### Pattern de flujo

| Pattern | Genera | Parent Object | Dual-platform |
|---------|--------|--------------|---------------|
| **PXFlowController** | WebPanel con flujo de trabajo paso a paso: acciones, confirmaciones, iteraciones | Procedure / Transaction / (None) | Si (Web + Responsive + SD) |

### Patterns de API/WebServices

| Pattern | Genera | Parent Object |
|---------|--------|--------------|
| **PXWSLayer** | Procedures SOAP, Procedures REST, objetos API (OpenAPI 3.0), SDTs In/Out | Transaction / (None) |
| **PXWSQuery** | DataProvider + Procedure + SDTs + Domain (ordenamiento), con filtros, búsqueda, paginación | Transaction / (None) |
| **PXWSData** | Procedure de lectura + SDTs (In/Out/Structure), con hooks de código | Transaction / (None) |
| **PXWSTransaction** | Procedures Load/Save/Delete + SDTs (Structure/In/Out por método) via Business Components | Transaction / (None) |

### Patterns de datos/configuración

| Pattern | Genera | Parent Object |
|---------|--------|--------------|
| **PXOAV** | Attributes, Transactions (WRI/WORI/Definition), Groups, Procedures (CRUD de valores) | Transaction |
| **PXEntityParameters** | Attributes, Domains, Transactions, Groups, SDTs, Procedures, DataProviders para parámetros configurables por entidad | Transaction |
| **PXReportTemplate** | No genera objetos directamente; define template/settings para reportes | Procedure |

## Relaciones entre Patterns

```
                    ┌──────────────────┐
                    │   PXWSLayer      │ ◄── Orquesta API REST/SOAP
                    │   (API Object)   │
                    └────────┬─────────┘
                             │ methods apuntan a:
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ PXWSQuery  │  │ PXWSData   │  │PXWSTransac.│
     │(DataProv.) │  │(Procedure) │  │(Load/Save/ │
     └────────────┘  └────────────┘  │ Delete)    │
                                     └────────────┘

     ┌───────────────────────────────────────────┐
     │              PXComposer                   │
     │  (compositor de pantalla)                 │
     │  Embebe como WebComponents:               │
     │  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
     │  │PXWorkWith│ │PXParam.  │ │PXComposer│   │
     │  │Selection │ │Request   │ │(anidado) │   │
     │  └──────────┘ └──────────┘ └──────────┘   │
     └───────────────────────────────────────────┘

     ┌────────────────────────────────────────────┐
     │          PXFlowController                  │
     │  Acciones pueden invocar:                  │
     │  • PXWorkWith (cualquier nodo)             │
     │  • PXParameterRequest                      │
     │  • PXComposer                              │
     │  • PXOAV                                   │
     │  • Otros PXFlowController                  │
     │  • Objetos GeneXus directos                │
     └────────────────────────────────────────────┘
```

## Comunicación entre componentes: GlobalEvents

Cuando múltiples WebComponents están **desplegados simultáneamente en la misma pantalla**, la comunicación entre ellos se logra mediante el **External Object nativo `GlobalEvents`** de GeneXus. Este objeto, importado por defecto en toda KB GeneXus, permite:

1. **Definir eventos globales** que cualquier panel puede disparar
2. **Escuchar eventos** en otros paneles que estén renderizados en ese momento y reaccionar (refrescar, actualizar datos, etc.)

**Aplica a**: componentes de **PXComposer** (todos visibles simultáneamente en pantalla).

**No aplica a**: tabs de **PXWorkWith View**, ya que solo el tab activo está cargado/renderizado — los demás tabs no están escuchando eventos en ese momento. Para comunicar entre tabs se usa **WebSession**: un tab escribe datos en WebSession y al activarse otro tab, este los lee en su Start/Refresh.

La configuración de `GlobalEvents` se implementa en los **hooks de código** (nodos `codes` y `events`) de las instancias de patterns, no como propiedad declarativa del pattern.

## Templates de UI — Personalización total del diseño visual

### Concepto clave

Los patterns de UI de PXTools (PXWorkWith, PXParameterRequest, PXComposer, PXFlowController) no imponen un diseño visual fijo. El diseño se define mediante **Templates**: objetos GeneXus que el desarrollador crea y personaliza con total libertad.

### Qué es un Template

Un Template es un **objeto GeneXus real** que define el diseño visual completo de la pantalla:

| Generador | Tipo de Template | Descripción |
|-----------|-----------------|-------------|
| Web Desktop | WebPanel con layout HTML | El desarrollador diseña el HTML con total libertad |
| Web Responsive | WebPanel con Layout Abstracto | El desarrollador usa el layout abstracto de GeneXus |
| Smart Devices | Panel for SD | El desarrollador diseña el panel móvil |

### Template Elements

Dentro del Template, el desarrollador coloca **Template Elements**: placeholders simples donde el generador de PXTools inyecta el contenido específico de cada instancia (grilla, filtros, botones de acción, formulario, etc.).

```
┌──────────────────────────────────────────────────────┐
│  TEMPLATE (WebPanel diseñado por el desarrollador)    │
│                                                      │
│  ┌─ Logo empresa ─┐  ┌─ Breadcrumb custom ─┐        │
│  └────────────────┘  └─────────────────────┘        │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │  ██ TEMPLATE ELEMENT: Filtros ██         │ ◄─ PXTools inyecta aquí
│  └──────────────────────────────────────────┘        │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │  ██ TEMPLATE ELEMENT: Grilla ██          │ ◄─ PXTools inyecta aquí
│  └──────────────────────────────────────────┘        │
│                                                      │
│  ┌─ Footer custom ──────────────────────────┐        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  Todo lo demás (logo, breadcrumb, footer, CSS,       │
│  JavaScript, controles extras) es 100% del           │
│  desarrollador con la flexibilidad de GeneXus        │
└──────────────────────────────────────────────────────┘
```

### Template Groups

Los Templates se organizan en **Template Groups** (`templatesGroup`) que agrupan los templates para cada tipo de pantalla generada por un pattern. Esto permite:

- Tener un grupo "Diseño Estándar" y un grupo "Diseño Minimalista"
- Aplicar un grupo completo a una instancia o a toda la KB vía Settings
- Cada level de una instancia puede usar un Template Group diferente o un Template individual (`templateObject`, `templateResponsiveObject`, `templateSDObject`)

### Por qué es importante

Los Templates son la **capacidad más importante para flexibilizar el diseño visual**:

1. El generador produce la lógica (variables, eventos, reglas, condiciones)
2. El Template define el diseño visual completo
3. Los Template Elements son los puntos de inserción del contenido generado
4. El desarrollador tiene **libertad total de GeneXus** alrededor de los Template Elements: HTML, CSS, JavaScript, controles de terceros, imágenes, animaciones

Esto significa que dos KBs usando el mismo pattern PXWorkWith pueden tener pantallas visualmente completamente diferentes si usan Templates distintos.

### Catálogo de Template Elements

Los Template Elements disponibles para PXWorkWith Selection (extraídos de Templates reales):

| Template Element (Type) | Qué inyecta el generador |
|------------------------|-------------------------|
| `Grid` | La grilla principal con datos |
| `Filters` | Panel de filtros avanzados |
| `UniqueSearchField` | Campo de búsqueda rápida |
| `UniqueSearchAction` | Botón de búsqueda |
| `InsertAction` | Botón de insertar |
| `UpdateAction` | Botón de modificar |
| `DeleteAction` | Botón de eliminar |
| `DisplayAction` | Botón de ver detalle |
| `EditAction` | Botón de editar |
| `ImageActions` | Acciones con imagen |
| `ButtonActions` | Acciones con botón |
| `StandardButtonActions` | Acciones estándar (Aceptar, Cancelar) |
| `OrderSelector` | Selector de orden |
| `InsertVariables` | Variables de inserción |
| `GridPagingStatus` | Estado de paginación |
| `PageJump` | Salto de página |
| `PageRowChange` | Cambio de filas por página |
| `TopGridFixedData` | Datos fijos arriba de la grilla |
| `BottomGridFixedData` | Datos fijos abajo de la grilla |
| `FixedDataSection` | Sección de datos fijos (con subtipo: Parameters, Top Grid, Bottom Grid) |
| `HiddenElements` | Elementos ocultos funcionales |
| `GridHandlerControl` | Control del grid handler |
| `GridHandlerAction` | Acción del grid handler |
| `AddAllAction` | Acción seleccionar todos |
| `RemoveAllAction` | Acción deseleccionar todos |
| `Search` | Botón buscar |
| `ErrorViewer` | Visor de errores |
| `ProgramName` | Nombre del programa |

### Templates predefinidos por PXTools

PXTools incluye una colección de Templates predefinidos en `@PXTools/@APIs/Personalized/Templates/`:

**Para PXWorkWith Selection (Desktop):**
- `PXToolsSelectionTemplate` — Layout estándar
- `PXToolsSelectionGXUITemplate` — Con controles GXUI
- `PXToolsSelectionTwoPaneTemplate` — Dos paneles
- `PXToolsSelectionButtonsWithImagesTemplate` — Botones con imágenes
- `PXToolsSelectionComponentLeftRightTemplate` — Componente izq/der
- `PXToolsSelectionComponentRightTemplate` — Componente a la derecha
- Y más variantes de layout...

**Para PXWorkWith Selection (Responsive):**
- `PXToolsResponsiveSelectionTemplate` — Layout responsivo estándar
- `PXToolsResponsiveSelectionTwoPaneTemplate` — Dos paneles responsivo
- `PXToolsResponsiveSelectionExpandComponentTemplate` — Con panel expandible

**Para otros nodos:**
- `PXToolsViewTemplate` / `PXToolsResponsiveViewTemplate` — View
- `PXToolsTransactionTemplate` / `PXToolsResponsiveTransactionTemplate` — Transaction
- `PXToolsTabGridTemplate` / `PXToolsTabGridGXUITemplate` — Tab Grid
- `PXToolsTabTabularTemplate` / `PXToolsResponsiveTabTabularTemplate` — Tab Tabular
- `PXToolsComposerTemplate` / `PXToolsResponsiveComposerTemplate` — Composer
- `PXToolsParameterRequestTemplate` / `PXToolsResponsiveParameterRequestTemplate` — PR
- `PXToolsParameterRequestLoginTemplate` — Login
- `PXToolsSDGridTemplate` y variantes — Smart Devices

### Estilos visuales predefinidos

PXTools ofrece variantes de diseño visual por color (Red, Blue, Green, Gray, Violet y otros). Estas variantes se aplican mediante el Theme de GeneXus y definen paletas de colores, tipografía y estilos para todos los elementos de las pantallas generadas.

## Generación Dual-Platform

Los patterns de UI (PXWorkWith, PXParameterRequest, PXComposer) y PXFlowController generan **dos versiones** de cada WebPanel:

| Versión | Prefijo nombre | Propiedad |
|---------|---------------|-----------|
| Web Desktop | `WW`, `Wb`, `Pr`, `Ct`, `View` | `generateWeb` |
| Web Responsive | `RWW`, `RWb`, `RPr`, `RCt`, `RView` | `generateWebResponsive` |
| Smart Devices | (varía) | `generateSD` |

Esto permite:
- Migración progresiva Desktop → Responsive
- Convivencia simultánea de ambas plataformas
- Misma definición de instancia genera ambas versiones

## Módulos @PXTools

PXTools incluye 25+ módulos reutilizables que proveen funcionalidad transversal. Cada módulo es un paquete de objetos GeneXus (Procedures, WebPanels, SDTs, DataProviders) que se instala bajo `Knowledge Base/@PXTools/@<NombreModulo>/`. Los módulos incluyen sus propias instancias de patterns (PXWorkWith para ABMs, PXParameterRequest para formularios, PXComposer para vistas compuestas).

Módulos disponibles: @APIs, @Alerts, @Audit, @CloudTasks, @ControlPreferences, @DynamicCallReferences, @ExportImport, @FileStorage, @MailAccounts, @Menus, @OAV, @ProcessMonitor, @Projects, @ReceiveMails, @ResponsiveLayout, @Security, @SecurityProjects, @SendMails, @SmartMenus, @Statistics, @System, @SystemParameters, @TableCleaner, @TaskManager, @WSLayer, @WebServicesLog.

Ver detalle en [20-modulos-pxtools.md](20-modulos-pxtools.md).

## Convenciones de nomenclatura

### Objetos generados por PXWorkWith

| Objeto generado | Naming | Ejemplo (instancia "Factura") |
|-----------------|--------|-------------------------------|
| Selection (Desktop) | `WW{Name}` | `WWFactura` |
| Selection (Responsive) | `RWW{Name}` | `RWWFactura` |
| View (Desktop) | `View{Name}` | `ViewFactura` |
| View (Responsive) | `RView{Name}` | `RViewFactura` |
| Prompt (Desktop) | `Pr{Name}` | `PrFactura` |
| Prompt (Responsive) | `RPr{Name}` | `RPrFactura` |
| Controller (Desktop) | `Ct{Name}` | `CtFactura` |
| Controller (Responsive) | `RCt{Name}` | `RCtFactura` |
| Tab Grid Component | `{wcname}` | (definido en la instancia) |
| Tab Tabular Component | `{wcname}` | (definido en la instancia) |
| Export Excel | `Ex{Name}` | `ExFactura` |
| Selected Rows SDT | `PXWW{Name}Rows` | `PXWWFacturaRows` |
| Grid Handler DP | `PXWW{Name}Rows` | `PXWWFacturaRows` |
| Transaction (Desktop) | `D{Name}` | `DFactura` |
| Transaction (Responsive) | `R{Name}` | `RFactura` |

### Objetos generados por patterns WS

| Pattern | Objeto | Naming | Ejemplo (Trn "Customer", V1) |
|---------|--------|--------|------------------------------|
| PXWSLayer | SOAP Procedure | `SOAPTrnV1` | `SOAPCustomerV1` |
| PXWSLayer | REST Procedure | `RESTTrnV1Method` | `RESTCustomerV1GetData` |
| PXWSLayer | API Object | `APIWS` | `APIWSCustomer` |
| PXWSQuery | DataProvider | `DPQueryTrnV1` | `DPQueryCustomerV1` |
| PXWSQuery | Procedure | `WSQueryTrnV1` | `WSQueryCustomerV1` |
| PXWSQuery | Domain (order) | `WSQueryTrnV1` | `WSQueryCustomerV1` |
| PXWSData | Procedure | `WSDataTrnV1Method` | `WSDataCustomerV1Method` |
| PXWSTransaction | Load Procedure | `WSTransactionTrnV1Load` | `WSTransactionCustomerV1Load` |
| PXWSTransaction | Save Procedure | `WSTransactionTrnV1Save` | `WSTransactionCustomerV1Save` |
| PXWSTransaction | Delete Procedure | `WSTransactionTrnV1Delete` | `WSTransactionCustomerV1Delete` |
