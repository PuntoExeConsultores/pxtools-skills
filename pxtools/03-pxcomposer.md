# PXComposer — Screen Composition Pattern

## What it is and what it is for

PXComposer is a PXTools pattern generating **composed WebPanels** (dashboard-style) by embedding other patterns and GeneXus objects as WebComponents. Unlike PXWorkWith or PXParameterRequest, PXComposer **has no Parent Object** — it is always standalone (`ParentObject Type="(None)"`).

Its main purpose is to allow the declarative construction of screens combining several visual components: PXWorkWith grids, PXParameterRequest forms, other nested PXComposers, user controls or raw HTML, all defined in XML without writing code by hand.

```
┌─────────────────────────────────────────────────────────┐
│  PXComposer: WbSalesDashboard                           │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │  WebComponent       │  │  WebComponent            │  │
│  │  PXWorkWith         │  │  PXWorkWith              │  │
│  │  Selection: Invoices│  │  Selection: Customers    │  │
│  └─────────────────────┘  └──────────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  WebComponent                                    │   │
│  │  PXParameterRequest: ReportFilter                │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │  HTML               │  │  UserControl             │  │
│  │  <div>info</div>    │  │  BarChart                │  │
│  └─────────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Objects it generates

PXComposer generates exactly **two objects** per instance level, one per web platform:

| Object ID | GeneXus type | Name pattern | Element | Description |
|-----------|--------------|--------------|---------|-------------|
| `Level` | WebPanel | `Wb{Istance.Name}` | `instance/level` | The composed panel for Web Desktop (HTML layout) |
| `LevelResponsive` | WebPanel | `RWb{Istance.Name}` | `instance/level` | The composed panel for Web Responsive (abstract layout) |

> **A note on the naming:** the name pattern uses `{Istance.Name}` (capital "I", missing the "n" — "Istance" instead of "Instance"). That is a historical typo in the original `.Pattern` file, but it is the real naming the generator uses.

### Examples of generated names

| Instance name | Desktop | Responsive |
|---------------|---------|------------|
| `SalesDashboard` | `WbSalesDashboard` | `RWbSalesDashboard` |
| `FileStorage` | `WbFileStorage` | `RWbFileStorage` |
| `SecurityObjectAccess` | `WbSecurityObjectAccess` | `RWbSecurityObjectAccess` |

### Generation control

Each level has boolean properties controlling which platforms are generated:

| Property | Default | Effect |
|----------|---------|--------|
| `generateWeb` | `true` | Generates the Desktop WebPanel (`Wb{Istance.Name}`) |
| `generateWebResponsive` | `true` | Generates the Responsive WebPanel (`RWb{Istance.Name}`) |
| `generateSD` | `false` | Reserved for Smart Devices (not currently implemented) |

## XML structure of the instance

A PXComposer instance follows the schema defined in `PXComposerInstance.xml` (108KB). The complete structure:

```xml
<instance>
  <level>
    <!-- Level properties -->
    <name>MyDashboard</name>
    <description>Main sales panel</description>
    <masterPage>PXMasterPage</masterPage>
    <theme>PXTheme</theme>

    <!-- Generation control -->
    <generateWeb>true</generateWeb>
    <generateWebResponsive>true</generateWebResponsive>
    <generateSD>false</generateSD>

    <!-- Forms with components (the main section) -->
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

    <!-- Available actions -->
    <actions>
      <action>
        <name>Back</name>
        <gxobject>MainMenu</gxobject>
        <!-- ... -->
      </action>
    </actions>

    <!-- Custom events -->
    <events>
      <event>
        <name>MyEvent</name>
        <code>/* GeneXus code */</code>
      </event>
    </events>

    <!-- Code hooks -->
    <codes>
      <code>
        <name>Start</name>
        <code>/* code in the Start event */</code>
      </code>
      <code>
        <name>Refresh</name>
        <code>/* code in the Refresh event */</code>
      </code>
      <code>
        <name>Load</name>
        <code>/* code in the Load event */</code>
      </code>
      <code>
        <name>Subroutine</name>
        <code>/* a reusable subroutine */</code>
      </code>
    </codes>

    <!-- WebPanel parameters -->
    <parameters>
      <parameter>
        <name>CustomerId</name>
        <domain>Numeric</domain>
      </parameter>
    </parameters>

    <!-- Extra variables -->
    <variables>
      <variable>
        <name>MyVariable</name>
        <domain>Character</domain>
      </variable>
    </variables>

    <!-- Conditions/rules -->
    <conditions>...</conditions>

    <!-- Documentation -->
    <documentation>A description for developers</documentation>
  </level>
</instance>
```

### The hierarchy, summarised

```
instance
└── level (there can be several)
    ├── name, description
    ├── masterPage, theme
    ├── generateWeb, generateWebResponsive, generateSD
    ├── forms
    │   └── form* (several forms for different platforms)
    │       ├── name
    │       ├── platform: {Any|Web Desktop|Web Responsive|Smart Devices}
    │       └── components
    │           └── component* (each embedded piece)
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

## The form and component system

### The concept of a form

Each level contains a `<forms>` node grouping one or more `<form>` entries. Each form defines a component layout for a specific platform. That lets one PXComposer generate different layouts depending on the target platform.

```
forms
├── form (platform="Any")            --> used when there is no specific form
├── form (platform="Web Desktop")    --> layout for Desktop only
├── form (platform="Web Responsive") --> layout for Responsive only
└── form (platform="Smart Devices")  --> layout for SD only
```

### Resolving the form per platform

The generator picks the form with this logic:

```
When generating Wb{Name} (Desktop):
  1. Look for a form with platform="Web Desktop"
  2. If there is none, use the form with platform="Any"

When generating RWb{Name} (Responsive):
  1. Look for a form with platform="Web Responsive"
  2. If there is none, use the form with platform="Any"
```

### Components inside a form

Each form holds a `<components>` node with several `<component>` entries. The components are rendered in the generated WebPanel in the order they appear in the XML.

## Component types

### WebComponent

The most common and most important type. It embeds a WebPanel as a WebComponent inside the composed panel.

```xml
<component>
  <type>WebComponent</type>

  <!-- Option A: a direct reference to a GeneXus object -->
  <callType>GXObject</callType>
  <gxObject>WWInvoice</gxObject>

  <!-- Option B: a reference to a pattern instance -->
  <callType>PXInstance</callType>
  <instanceObject>PXWorkWithInvoice</instanceObject>
  <instanceLevel>Invoice</instanceLevel>
  <instanceLevelNode>Selection</instanceLevelNode>

  <!-- Optional parameters for the WebComponent -->
  <parameters>
    <parameter>
      <name>CustomerId</name>
      <value>&amp;CustomerId</value>
    </parameter>
  </parameters>
</component>
```

#### callType: GXObject vs PXInstance

| callType | Use | Advantage |
|----------|-----|-----------|
| `GXObject` | A direct reference to an existing WebPanel | Simple, it works with any WebPanel |
| `PXInstance` | A reference to a PXWorkWith, PXParameterRequest or PXComposer pattern | It resolves automatically to the right generated object per platform |

When `PXInstance` is used, the generator resolves the object's name per platform automatically:

```
PXInstance: PXWorkWithInvoice / level=Invoice / node=Selection
  → Desktop generates:    WebComponent = WWInvoice
  → Responsive generates: WebComponent = RWWInvoice
```

This is essential for dual-platform generation: one definition produces correct references on both platforms.

### UserControl

Embeds a GeneXus User Control inside the panel.

```xml
<component>
  <type>UserControl</type>
  <controlName>MyChart</controlName>
  <properties>
    <property>
      <name>DataSource</name>
      <value>&amp;ChartData</value>
    </property>
  </properties>
</component>
```

### HTML

Inserts raw HTML content directly into the generated WebPanel.

```xml
<component>
  <type>HTML</type>
  <html><![CDATA[
    <div class="info-panel">
      <h3>Sales summary</h3>
      <p>Additional information here</p>
    </div>
  ]]></html>
</component>
```

### Action

Embeds an action button or link.

```xml
<component>
  <type>Action</type>
  <name>ViewReport</name>
  <gxobject>SalesReport</gxobject>
  <!-- action properties -->
</component>
```

## Embedding other patterns (PXInstance)

PXComposer's most powerful capability is embedding instances of other patterns declaratively. With `callType="PXInstance"`, three properties identify exactly which object to embed:

### Embeddable nodes per pattern

| Source pattern | Valid instanceLevelNode | Resolved object (Desktop) | Resolved object (Responsive) |
|----------------|-------------------------|---------------------------|------------------------------|
| PXWorkWith | `Selection` | `WW{Name}` | `RWW{Name}` |
| PXWorkWith | `View` | `View{Name}` | `RView{Name}` |
| PXWorkWith | `Prompt` | `Pr{Name}` | `RPr{Name}` |
| PXParameterRequest | `Level` | `Wb{Name}` | `RWb{Name}` |
| PXComposer | (the whole level) | `Wb{Name}` | `RWb{Name}` |

### A composition example with PXInstance

```xml
<!-- Embed the Selection of the Invoices PXWorkWith -->
<component>
  <type>WebComponent</type>
  <callType>PXInstance</callType>
  <instanceObject>PXWorkWithInvoice</instanceObject>
  <instanceLevel>Invoice</instanceLevel>
  <instanceLevelNode>Selection</instanceLevelNode>
</component>

<!-- Embed a filters PXParameterRequest -->
<component>
  <type>WebComponent</type>
  <callType>PXInstance</callType>
  <instanceObject>PXParameterRequestReportFilter</instanceObject>
  <instanceLevel>ReportFilter</instanceLevel>
  <instanceLevelNode>Level</instanceLevelNode>
</component>

<!-- Embed another PXComposer (nesting) -->
<component>
  <type>WebComponent</type>
  <callType>PXInstance</callType>
  <instanceObject>PXComposerSubPanel</instanceObject>
  <instanceLevel>SubPanel</instanceLevel>
</component>
```

### Composition diagram

```
PXComposer (WbMyDashboard)
│
├── WebComponent [PXInstance]
│   └── PXWorkWith:Invoice:Selection ──► WWInvoice / RWWInvoice
│
├── WebComponent [PXInstance]
│   └── PXWorkWith:Customer:Selection ──► WWCustomer / RWWCustomer
│
├── WebComponent [PXInstance]
│   └── PXParameterRequest:Filter:Level ──► WbFilter / RWbFilter
│
├── WebComponent [GXObject]
│   └── MyCustomWebPanel (direct, not a pattern)
│
├── HTML
│   └── <div>static content</div>
│
└── UserControl
    └── SalesChart
```

## Forms per platform

PXComposer supports defining completely different layouts per platform within the same instance. That is useful when the responsive layout needs a different arrangement from the desktop one.

### Scenario: a different layout per platform

```xml
<forms>
  <!-- Desktop layout: two columns side by side -->
  <form>
    <name>FormDesktop</name>
    <platform>Web Desktop</platform>
    <components>
      <component><!-- Left panel: the list --></component>
      <component><!-- Right panel: the detail --></component>
    </components>
  </form>

  <!-- Responsive layout: stacked components -->
  <form>
    <name>FormResponsive</name>
    <platform>Web Responsive</platform>
    <components>
      <component><!-- The list (full width) --></component>
      <component><!-- The detail (full width, below) --></component>
    </components>
  </form>
</forms>
```

### Scenario: the same layout for every platform

```xml
<forms>
  <form>
    <name>SingleForm</name>
    <platform>Any</platform>
    <components>
      <component><!-- Used for both Desktop and Responsive --></component>
    </components>
  </form>
</forms>
```

## Code hooks

PXComposer lets you inject custom GeneXus code at specific points of the generated WebPanel through the `<codes>` node:

| Hook | When it runs | Typical use |
|------|--------------|-------------|
| `Start` | The WebPanel's Start event | Initialising variables, loading initial data, access validations |
| `Refresh` | The WebPanel's Refresh event | Reloading data, updating components |
| `Load` | The WebPanel's Load event | Per-record loading logic |
| `Subroutine` | A reusable subroutine | Logic shared between events |

### A hooks example

```xml
<codes>
  <code>
    <name>Start</name>
    <code>
      // Check permissions
      if not PXCheckAccess.Udp("Dashboard")
        Return
      endif
      // Initialise variables
      &amp;DateFrom = Today() - 30
      &amp;DateTo = Today()
    </code>
  </code>
  <code>
    <name>Refresh</name>
    <code>
      // Reload the dashboard's data
      PXDashboardData.Call(&amp;DateFrom, &amp;DateTo, &amp;SummaryData)
    </code>
  </code>
</codes>
```

### Custom events

Beyond the predefined hooks, arbitrary events can be added:

```xml
<events>
  <event>
    <name>CustomerSelected</name>
    <code>
      // Handle the customer selection coming from a WebComponent
      &amp;CustomerId = CustomerId.FromString(&amp;Data)
      // Refresh the dependent components
    </code>
  </event>
</events>
```

## Dual-platform capability

PXComposer generates the Desktop and Responsive versions of the same composed panel simultaneously. This is a characteristic shared with PXWorkWith and PXParameterRequest.

### The generation flow

```
PXComposer instance
│
├── generateWeb = true
│   └── Generates: Wb{Istance.Name}
│       ├── WebForm (HTML layout)
│       ├── Variables
│       ├── Events (with the hooks)
│       └── Rules
│
├── generateWebResponsive = true
│   └── Generates: RWb{Istance.Name}
│       ├── AbstractForm (GeneXus abstract layout)
│       ├── Variables
│       ├── Events (with the hooks)
│       └── Rules
│
└── generateSD = false (reserved)
```

### Automatic reference resolution

When a component uses `callType="PXInstance"`, the generator automatically resolves the reference to the right object for the platform it is generating:

```
Instance: PXComposerMyDashboard
  └── component: PXInstance → PXWorkWithInvoice / Selection

Desktop generation (WbMyDashboard):
  └── WebComponent → WWInvoice          (WW prefix)

Responsive generation (RWbMyDashboard):
  └── WebComponent → RWWInvoice         (RWW prefix)
```

This guarantees that the Desktop version embeds Desktop components and the Responsive version embeds Responsive ones, with no additional manual configuration.

## Real-world usage examples

### Instances in the @PXTools modules

| Module | PXComposer instance | Purpose |
|--------|---------------------|---------|
| @Alerts | `PXComposerSystemAlertMessage` | System alert messages panel |
| @Alerts | `PXComposerSystemAlertSchedulerView` | The alert scheduler view |
| @FileStorage | `PXComposerFileStorage` | Stored file management panel |
| @Security | `PXComposerSecurityObjectAccess` | Object access control panel |
| @Security | `PXComposerSecurityObjectRecordAccess` | Record-level access panel |
| @WebServicesLog | `PXComposerWebServicesLog` | Web service logs panel |
| @WebServicesLog | `PXComposerWebServicesStatisticCounters` | WS statistical counters panel |

### Typical use in applications

PXComposer is typically used for dashboards and composed screens combining several instances (a PXWorkWith Selection/View, and so on) in a single view.

### Typical instance locations

| Location | Instances |
|----------|-----------|
| `@<Module>/Forms` | Composed front-end forms |
| `@<Module>/Home` | A home screen with several panels |

### The typical pattern: a composed master-detail panel

```xml
<!-- A common instance: a dashboard with filters + grid + detail -->
<instance>
  <level>
    <name>CustomerManagement</name>
    <forms>
      <form>
        <name>MainForm</name>
        <platform>Any</platform>
        <components>
          <!-- Filters through PXParameterRequest -->
          <component>
            <type>WebComponent</type>
            <callType>PXInstance</callType>
            <instanceObject>PXParameterRequestCustomerFilter</instanceObject>
            <instanceLevel>CustomerFilter</instanceLevel>
            <instanceLevelNode>Level</instanceLevelNode>
          </component>
          <!-- The customers grid through PXWorkWith -->
          <component>
            <type>WebComponent</type>
            <callType>PXInstance</callType>
            <instanceObject>PXWorkWithCustomer</instanceObject>
            <instanceLevel>Customer</instanceLevel>
            <instanceLevelNode>Selection</instanceLevelNode>
          </component>
        </components>
      </form>
    </forms>
  </level>
</instance>
```

## Relationship with the other patterns

### PXComposer as a consumer

PXComposer is the main **consumer** of the other UI patterns. It embeds their generated objects as WebComponents:

```
┌──────────────────────────────────────────────────────┐
│                    PXComposer                        │
│                   (THE CONSUMER)                     │
│                                                      │
│   It can embed:                                      │
│                                                      │
│   ┌──────────────┐  ┌───────────────────┐            │
│   │ PXWorkWith   │  │ PXParameterRequest│            │
│   │ • Selection  │  │ • Level           │            │
│   │ • View       │  │                   │            │
│   │ • Prompt     │  │                   │            │
│   └──────────────┘  └───────────────────┘            │
│                                                      │
│   ┌──────────────┐  ┌───────────────────┐            │
│   │ PXComposer   │  │ GeneXus WebPanel  │            │
│   │ (nested)     │  │ (any of them)     │            │
│   └──────────────┘  └───────────────────┘            │
└──────────────────────────────────────────────────────┘
```

### PXComposer as something embeddable

PXComposer can in turn be embedded by:

| Embedding pattern | How it references it |
|-------------------|----------------------|
| Another PXComposer | `callType="PXInstance"` with `instanceObject` pointing at the PXComposer |
| PXFlowController | As a navigation action or a step of the flow |
| PXWorkWith | As a tab component inside a View |

### Key differences from the other patterns

| Characteristic | PXWorkWith | PXParameterRequest | PXComposer |
|----------------|------------|--------------------|------------|
| Parent Object | Transaction / (None) | WebPanel / Procedure / Transaction / (None) | (None) only |
| Generates grids | Yes | No | No (it embeds them) |
| Generates forms | Yes (View/Edit) | Yes | No (it embeds them) |
| Composes other patterns | No | No | **Yes** |
| Built-in navigation | Selection↔View↔Edit | Modal/Popup | Through the embedded components |
| Dual generation | Desktop + Responsive + SD | Desktop + Responsive | Desktop + Responsive |
| Number of objects | Many (Selection, View, Prompt, Controller, Tabs…) | 2 (Desktop + Responsive) | 2 per level (Desktop + Responsive) |
