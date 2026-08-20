# PXParameterRequest — Complete Pattern Documentation

## 1. What it is and what it is for

PXParameterRequest is a PXTools pattern generating **modal/popup WebPanels for capturing parameters**. Its main purpose is to create forms that:

- Capture user data before running an action (reports, processes, exports)
- Show confirmation dialogs ("Are you sure you want to delete this?")
- Provide login, password change and registration forms
- Open data-entry popups returning values to the caller
- Allow selecting a file for import

It attaches to objects of type: **WebPanel**, **Procedure**, **Report**, **Transaction** or **(None)**.

### The key difference from PXWorkWith

While PXWorkWith generates complete screens with selection grids, views and data editing over transactions, PXParameterRequest generates **simple capture forms** typically opened as popups/modals that return values to the caller.

## 2. Objects it generates

| Object ID | GeneXus type | Name pattern | Element (XPath) | Description |
|-----------|--------------|--------------|-----------------|-------------|
| Level | WebPanel | `Wb{Instance.Name}` | `instance/level` | The parameter form (Web Desktop, HTML template) |
| LevelResponsive | WebPanel | `Wb{Instance.Name}` | `instance/level` | The parameter form (Web Responsive, abstract layout) |
| ChkChosenValueSelected | Procedure | `Chk{Instance.Name}ChosenValueSelected` | `instance//variable/controlInfo[@controlType='Chosen']` | Validation of Chosen controls |
| RetChosenValues | DataProvider | `Ret{Instance.Name}ChosenValues` | (same as above) | DataProvider of the Chosen values |
| RetChosenResults | Procedure | `Ret{Instance.Name}ChosenResults` | (same as above) | Procedure returning the Chosen results |

### Dual-platform naming: a special case

Unlike PXWorkWith, which uses a different prefix per platform (`WW` vs `RWW`, `View` vs `RView`), PXParameterRequest uses the **SAME name** for both versions:

```
PXWorkWith:          WWInvoice (Desktop)  /  RWWInvoice (Responsive)
PXParameterRequest:  WbLogin   (Desktop)  /  WbLogin    (Responsive)  ← THE SAME NAME
```

That means a given KB only generates one active version (Desktop OR Responsive), controlled by the `generateWeb` and `generateWebResponsive` properties.

## 3. behaviour types

> ⚠️ **In the `.gxPattern` the behaviour is written as the `<level>`'s `type` attribute, not as a `<behaviour>` node.** The `behaviour` node and the long names in the table below belong to the pattern's **definition**; neither appears in the instances externalized to disk. Verified across the 57 instances of this KB: zero `<behaviour>` nodes.
>
> ```xml
> <level name="MyForm" description="My Form" caption="My Form" type="Popup">
> ```
>
> Values observed in real instances, with their frequency: `Popup` (44), `Component` (8), `Web Panel` (2), `Prompt` (2), `Process` (1).
>
> **This matters when the panel is invoked as a modal**: if the level does not declare `type="Popup"`, the generated WebPanel is not a popup, and invoking it with `.Popup()` shows it without the frame, the title or the action bar — it looks like a stray form pasted onto the page.
>
> The popup's **size** is not declared here: `popupWidth`, `popupHeight` and `popupBehaviour` go on the **caller's action** (alongside `target="Modal"`), because they are properties of that invocation and not of the form. The same form can be opened at different sizes from different places.

The `behaviour` node defines how the form is presented and behaves. It is the level's most important property.

```
level/behaviour
├── type: enum
├── returnType: enum{Link;Message}
└── closeOnReturn: bool
```

### Table of behaviour types

| Type | Presentation | Typical use | Returning values |
|------|--------------|-------------|------------------|
| `None` | A simple panel with no special behaviour | Forms embedded as WebComponents | Not applicable |
| `Panel` | A standard panel | Standalone forms not acting as popups | Through parameters |
| `PopupParameterRequest` | **A modal popup** (a pop-up window) | Confirmations, quick data capture, selection | Returns values to the caller through the popup |
| `ParameterRequest` | A full parameters page | Capturing parameters before running reports/processes | Redirects to the target object with the values |
| `FloatingParameterRequest` | A floating overlay above the current page | Quick capture without losing the main screen's context | Returns values without navigating |

### returnType and closeOnReturn

```
returnType = Link     → On confirming, it navigates to the target URL/object
returnType = Message  → On confirming, it sends a message to the caller (useful for WebComponents)

closeOnReturn = true  → Closes the popup automatically on confirming
closeOnReturn = false → Stays open (useful for repetitive data entry forms)
```

### Flow diagram by behaviour

```
                    The user clicks an action
                              │
                              ▼
              ┌───────────────────────────────┐
              │  Which behaviour does the     │
              │  PXParameterRequest have?     │
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
      Embedded  Stand-    A modal popup  A full page   A floating
      in another alone    opens over     replaces the  overlay on
      WebPanel  page      the caller     current one   the current
                          ┌────────┐                    page
                          │Captures│
                          │ data   │
                          └───┬────┘
                              │
                          ┌───▼────┐
                          │Returns │
                          │values  │
                          │to the  │
                          │caller  │
                          └────────┘
```

## 4. XML structure of the instance

The complete schema of the `instance` node defined in `PXParameterRequestInstance.xml` (203 KB):

```xml
<instance>
  <level name="Main" description="Main form"
         masterPage="MasterPage" theme="PXTheme"
         generateWeb="true" generateWebResponsive="true" generateSD="false"
         platform="Any">

    <!-- The form's behaviour -->
    <behaviour type="PopupParameterRequest"
               returnType="Link"
               closeOnReturn="true" />

    <!-- The form's layout -->
    <form>
      <attributes>
        <!-- Form fields based on KB attributes -->
      </attributes>
      <variables>
        <!-- Custom variables in the form -->
      </variables>
    </form>

    <!-- Actions (buttons) -->
    <actions>
      <action name="Confirm" caption="Accept" />
      <action name="Cancel" caption="Cancel" />
    </actions>

    <!-- Custom events -->
    <events>
      <event name="OnConfirm">
        <!-- GeneXus code -->
      </event>
    </events>

    <!-- Code hooks at specific points of the life cycle -->
    <codes>
      <code section="Start">/* Code at startup */</code>
      <code section="Refresh">/* Code in Refresh */</code>
      <code section="Load">/* Code in Load */</code>
      <code section="Subroutine">/* Helper subroutines */</code>
    </codes>

    <!-- The WebPanel's input/output parameters -->
    <parameters>
      <parameter name="CustomerId" type="in" />
      <parameter name="SelectedDate" type="out" />
    </parameters>

    <!-- Extra variables -->
    <variables>
      <variable name="&Total" type="Numeric" length="12" decimals="2" />
    </variables>

    <!-- Conditions (filters, where) -->
    <conditions>
      <condition>CustomerStatus = 'A'</condition>
    </conditions>

    <!-- An optional grid inside the form -->
    <grid>
      <attributes />
      <filter />
      <orders />
      <actions />
      <modes />
    </grid>

    <!-- Calls on confirming (which object is invoked with the parameters) -->
    <calls>
      <call object="RptSales" parameters="..." />
    </calls>

    <!-- Hidden fields -->
    <hidden>
      <attribute name="CompanyId" />
    </hidden>

    <!-- Conditional layouts -->
    <layouts>
      <layout condition="&Mode = 'Edit'" />
    </layouts>

    <!-- Documentation and help -->
    <documentation />
    <help />
  </level>
</instance>
```

### The complete hierarchy

```
instance
├── level (there can be several levels)
│   ├── @name              → The level's name
│   ├── @description       → Description
│   ├── @masterPage        → The GeneXus MasterPage to use
│   ├── @theme             → The GeneXus Theme
│   ├── @generateWeb       → bool: generate the Desktop version
│   ├── @generateWebResponsive → bool: generate the Responsive version
│   ├── @generateSD        → bool: generate the Smart Devices version
│   ├── @platform          → enum{Any;Web Desktop}
│   │
│   ├── behaviour
│   │   ├── @type           → enum{None;Panel;PopupParameterRequest;
│   │   │                          ParameterRequest;FloatingParameterRequest}
│   │   ├── @returnType     → enum{Link;Message}
│   │   └── @closeOnReturn  → bool
│   │
│   ├── form
│   │   ├── attributes[]    → Fields based on GeneXus attributes
│   │   │   └── attribute
│   │   │       ├── @name, @description, @readonly, @visible
│   │   │       └── controlInfo → the control type (Edit, Combo, Chosen, etc.)
│   │   └── variables[]     → Custom variables in the form
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
│   │       └── @condition  → a visibility expression
│   │
│   ├── events[]
│   │   └── event
│   │       ├── @name       → The event's name
│   │       └── (inline GeneXus code)
│   │
│   ├── codes
│   │   ├── code[@section='Start']
│   │   ├── code[@section='Refresh']
│   │   ├── code[@section='Load']
│   │   └── code[@section='Subroutine']
│   │
│   ├── parameters[]
│   │   └── parameter
│   │       ├── @name       → The parameter's name
│   │       └── @type       → enum{in;out;inout}
│   │
│   ├── variables[]         → The WebPanel's variables (not the form's)
│   ├── conditions[]        → Conditions/Where
│   │
│   ├── grid (optional)
│   │   ├── attributes[]    → The grid's columns
│   │   ├── filter          → The grid's filters
│   │   ├── orders[]        → Available orderings
│   │   ├── actions[]       → Per-row actions
│   │   └── modes           → Insert/Update/Delete/Display
│   │
│   ├── calls[]             → Objects to invoke on confirming
│   │   └── call
│   │       ├── @object     → The target GeneXus object's name
│   │       └── @parameters → Parameter mapping
│   │
│   ├── hidden[]            → Hidden fields (passed but not displayed)
│   ├── layouts[]           → Conditional layouts
│   ├── documentation       → The level's documentation
│   └── help                → Contextual help
```

## 5. The form and its fields

The `form` node defines the capture form's visible fields. Fields can be **attributes** (from the database) or **variables** (defined locally).

### Available control types

Each form field has a `controlInfo` defining how it is rendered:

| controlType | Description | Example use |
|-------------|-------------|-------------|
| `Edit` | Free text field | Name, description, notes |
| `Combo` | Drop-down list | Status, type, category |
| `CheckBox` | A check box | Active/Inactive, accept terms |
| `RadioButton` | Option buttons | Document type, gender |
| `Chosen` | Multi-selector with search (it generates extra objects) | Tags, multiple categories |
| `DatePicker` | Date selector | Date from, date to |
| `File` | File selector | Importing a CSV/Excel file |
| `Image` | An image | Logo, photo |
| `TextBlock` | Static (non-editable) text | Instructions, warnings |

### Chosen controls: extra objects

When a field uses `controlType='Chosen'`, the pattern automatically generates three extra objects:

```
A variable with controlType='Chosen'
        │
        ├──► Chk{Instance.Name}ChosenValueSelected  (Procedure)
        │    Validates that at least one value was selected
        │
        ├──► Ret{Instance.Name}ChosenValues          (DataProvider)
        │    Returns the list of possible values for the selector
        │
        └──► Ret{Instance.Name}ChosenResults         (Procedure)
             Returns the results the user selected
```

### A form example with several control types

```xml
<form>
  <attributes>
    <attribute name="DateFrom" description="Date from">
      <controlInfo controlType="DatePicker" />
    </attribute>
    <attribute name="DateTo" description="Date to">
      <controlInfo controlType="DatePicker" />
    </attribute>
    <attribute name="CustomerId" description="Customer">
      <controlInfo controlType="Edit" />
    </attribute>
  </attributes>
  <variables>
    <variable name="&ReportType" domain="ReportType">
      <controlInfo controlType="Combo" />
    </variable>
    <variable name="&IncludeCancelled">
      <controlInfo controlType="CheckBox" />
    </variable>
    <variable name="&FileToImport">
      <controlInfo controlType="File" />
    </variable>
  </variables>
</form>
```

## 6. The action and call system

### Actions (buttons)

Actions define the form's buttons. A PXParameterRequest typically has at least two: **Confirm** and **Cancel**.

```xml
<actions>
  <!-- The standard confirmation action -->
  <action name="Confirm" caption="Accept"
          type="Standard" position="Bottom" />

  <!-- The standard cancellation action -->
  <action name="Cancel" caption="Cancel"
          type="Standard" position="Bottom" />

  <!-- A custom action -->
  <action name="Preview" caption="Preview"
          type="Custom" position="Bottom"
          condition="&ReportType = 'PDF'" />
</actions>
```

### Calls (invocation on confirming)

The `calls` node defines which objects run when the user confirms the form. The parameters captured in the form are passed as arguments.

```xml
<calls>
  <!-- Calls a report with the captured parameters -->
  <call object="RptSalesByPeriod"
        parameters="&DateFrom, &DateTo, &CustomerId, &ReportType" />
</calls>
```

### The execution flow

```
┌──────────────────────────────┐
│  PXParameterRequest is open  │
│  (popup/page/floating)       │
│                              │
│  ┌────────────────────────┐  │
│  │  The form's fields     │  │
│  │  Date from:  [       ] │  │
│  │  Date to:    [       ] │  │
│  │  Customer:   [       ] │  │
│  │  Type:       [▼ PDF  ] │  │
│  └────────────────────────┘  │
│                              │
│  [Preview]       [Accept]    │
│                  [Cancel]    │
└──────────────┬───────────────┘
               │ Click on "Accept"
               ▼
┌──────────────────────────────┐
│  1. Runs the validations     │
│  2. Runs the OnConfirm event │
│  3. Invokes the calls:       │
│     RptSalesByPeriod(        │
│       &DateFrom,             │
│       &DateTo,               │
│       &CustomerId,           │
│       &ReportType            │
│     )                        │
│  4. If closeOnReturn=true,   │
│     closes the popup         │
└──────────────────────────────┘
```

## 7. The optional grid

A PXParameterRequest can optionally include a **grid** inside the form. That is useful when the form needs to show a list of records for the user to select or review before confirming.

```xml
<grid>
  <!-- The grid's columns -->
  <attributes>
    <attribute name="ProductId" />
    <attribute name="ProductDescription" />
    <attribute name="ProductPrice" />
  </attributes>

  <!-- The grid's filters -->
  <filter>
    <condition>ProductActive = true</condition>
  </filter>

  <!-- Available orderings -->
  <orders>
    <order name="ByName" attributes="ProductDescription" />
    <order name="ByPrice" attributes="ProductPrice" />
  </orders>

  <!-- Per-row actions -->
  <actions>
    <action name="Select" caption="Select" />
  </actions>

  <!-- Allowed modes -->
  <modes insert="false" update="false" delete="false" display="true" />
</grid>
```

### A typical use case

```
┌──────────────────────────────────────┐
│  Select the products to export       │
│                                      │
│  Format: [▼ Excel ]                  │
│                                      │
│  ┌──┬────────────┬──────────┬──────┐ │
│  │✓ │ Product    │ Price    │      │ │
│  ├──┼────────────┼──────────┼──────┤ │
│  │☑ │ Widget A   │ $100.00  │ [Sel]│ │
│  │☐ │ Widget B   │ $250.00  │ [Sel]│ │
│  │☑ │ Widget C   │ $75.00   │ [Sel]│ │
│  └──┴────────────┴──────────┴──────┘ │
│                                      │
│         [Export]    [Cancel]         │
└──────────────────────────────────────┘
```

## 8. Code hooks

The `codes` node lets you inject GeneXus code at specific points of the generated WebPanel's life cycle. This is essential for customizing the behaviour without editing the generated code directly.

### Available sections

| Section | When it runs | Typical use |
|---------|--------------|-------------|
| `Start` | The WebPanel's `Start` event | Initialising variables, checking permissions, loading default values |
| `Refresh` | The `Refresh` event | Recomputing values, updating control states |
| `Load` | The grid's `Load` event (when there is one) | Computing derived per-row fields |
| `Subroutine` | Declared as the WebPanel's subroutines | Reusable logic inside the form |

### `Subroutine`: one node per subroutine, and the `name` IS the name

⚠️ Unlike `Start`, `Refresh` and `Load` — which are single points of the life cycle — the `Subroutine` type is declared **once per subroutine**, with its name in the `name` attribute, and the CDATA carries **only the body**: the pattern generates the `Sub 'Name'` and the `EndSub`.

```xml
<codes>
  <code type="Start"><![CDATA[Do 'Read Data'
Do 'Show Status']]></code>

  <code type="Subroutine" name="Read Data"><![CDATA[&Load = False
For Each
	Where CompanyTaxId = &CompanyTaxId
	&Load = True
	Exit
EndFor]]></code>

  <code type="Subroutine" name="Show Status"><![CDATA[&Message.Visible = &Load]]></code>
</codes>
```

**The easy mistake** is putting every subroutine into a single node, writing the `Sub … EndSub` by hand inside the CDATA. The pattern **does not split them**: you end up with one node with an empty `name`, which the visual editor shows as an unnamed *Code Subroutine*, and the subroutines are never generated.

Names may contain spaces (`name="Find Company"`), and they are invoked as usual: `Do 'Find Company'`.

### A code hooks example

```xml
<codes>
  <code section="Start">
    // Check permissions
    if not &PXSession.IsAuthenticated()
      return
    endif
    // Default values
    &DateFrom = Today() - 30
    &DateTo = Today()
  </code>

  <code section="Refresh">
    // Update the estimated total according to the filters
    &EstimatedTotal = CalcSalesTotal(&DateFrom, &DateTo, &CustomerId)
  </code>

  <code section="Subroutine">
    Sub 'ValidateDates'
      if &DateFrom > &DateTo
        msg('The from date cannot be later than the to date')
        &OK = false
      endif
    EndSub
  </code>
</codes>
```

> **CDATA formatting and indentation**: in the real `.gxPattern` each hook is `<code type="Start"><![CDATA[…]]></code>`; the first line goes at **column 0** and only block nesting indents (+1 tab). The full rule (shared by the patterns) is in [`00-overview.md`](00-overview.md) → *Code hooks: CDATA formatting and indentation*.

### Custom events

Beyond the code hooks, the `events` node lets you define complete GeneXus events:

```xml
<events>
  <event name="'Validate'">
    Do 'ValidateDates'
    if &OK
      msg('Valid parameters', status)
    endif
  </event>
</events>
```

## 9. Dual-platform capability

PXParameterRequest supports generation for **Web Desktop** and **Web Responsive** from the same instance definition.

### Platform control properties

```xml
<level name="Login"
       generateWeb="true"              <!-- Generate the Desktop version -->
       generateWebResponsive="true"    <!-- Generate the Responsive version -->
       generateSD="false"              <!-- Do not generate the Smart Devices version -->
       platform="Any">                 <!-- Any | Web Desktop -->
```

### The platform property

| Value | Effect |
|-------|--------|
| `Any` | The level is generated for every enabled platform (Web Desktop + Responsive) |
| `Web Desktop` | The level is generated for Web Desktop **only**, even when `generateWebResponsive=true` |

### Objects generated per platform

```
a level with generateWeb=true, generateWebResponsive=true
│
├──► Level (Desktop)
│    Object: WebPanel "Wb{Instance.Name}"
│    Form:   HTML layout (the *WebForm.dll template)
│    Result: a form with an HTML table and positioned controls
│
└──► LevelResponsive (Responsive)
     Object: WebPanel "Wb{Instance.Name}"    ← THE SAME NAME
     Form:   Abstract layout (the *AbstractForm.dll template)
     Result: a form with a responsive abstract layout
```

**Important note**: since both versions share the same name, in practice only one can be active in the KB. The `generateWeb`/`generateWebResponsive` property controls which one is generated.

### Differences in the generation templates

| Part of the object | Desktop template | Responsive template |
|--------------------|------------------|---------------------|
| WebForm | `*WebForm.dll` (HTML) | `*AbstractForm.dll` (Abstract) |
| Variables | Shared | Shared |
| Events | Shared (with adjustments) | Shared (with adjustments) |
| Rules | Shared | Shared |

## 10. Relationship with the other patterns

PXParameterRequest integrates with the other PXTools patterns in several ways:

### With PXWorkWith

```
┌────────────────────────────────────────┐
│  PXWorkWith (Invoices Selection)       │
│                                        │
│  [New] [Delete] [Export to Excel]      │
│  ┌──────────────────────────────────┐  │
│  │  The invoices grid               │  │
│  └──────────────────────────────────┘  │
│                                        │
│  Click on [Export to Excel] ────────── │ ──► Opens a PXParameterRequest
└────────────────────────────────────────┘     as a popup to capture
                                               DateFrom, DateTo and
                                               the export format
```

PXWorkWith actions (in Selection, View, and so on) can be configured to open a PXParameterRequest as a popup before running the real action.

### With PXFlowController

```
┌─────────────────────────────────────────────────┐
│  PXFlowController (invoicing process)           │
│                                                 │
│  Step 1: select the customer ──► PXWorkWith     │
│  Step 2: confirm the data ────► PXParameterReq. │  ← Confirmation
│  Step 3: process ─────────────► Procedure       │
│  Step 4: result ─────────────► PXParameterReq.  │  ← Shows the result
└─────────────────────────────────────────────────┘
```

PXFlowController can invoke a PXParameterRequest as a step of the flow, typically to confirm data or capture intermediate parameters.

### With PXComposer

```
┌────────────────────────────────────────────────┐
│  PXComposer (Dashboard)                        │
│  ┌─────────────────┐  ┌─────────────────────┐  │
│  │  PXWorkWith     │  │ PXParameterRequest  │  │
│  │  (summary list  │  │ (global filters)    │  │  ← Embedded
│  │   of sales)     │  │  Date from: [    ]  │  │     as a
│  │                 │  │  Date to:   [    ]  │  │     WebComponent
│  │                 │  │  [Apply filter]     │  │
│  └─────────────────┘  └─────────────────────┘  │
└────────────────────────────────────────────────┘
```

PXComposer can embed a PXParameterRequest as a WebComponent, which is useful for filter panels on dashboards.

### The complete relationship map

```
┌──────────────┐         invokes it as a popup
│  PXWorkWith  │ ──────────────────────────────► ┌─────────────────────┐
│  (actions)   │                                 │  PXParameterRequest │
└──────────────┘                                 │                     │
                                                 │  Captures parameters│
┌──────────────┐         invokes it as a step    │  and returns values │
│PXFlowContro- │ ──────────────────────────────► │                     │
│ ller (steps) │                                 └──────────┬──────────┘
└──────────────┘                                            │
                                                            │ calls → invokes
┌──────────────┐         embeds it as a WebComponent        ▼
│  PXComposer  │ ──────────────────────────────► ┌─────────────────────┐
│  (dashboard) │                                 │  Procedure / Report │
└──────────────┘                                 │  Transaction / WP   │
                                                 └─────────────────────┘
```

## 11. Real-world usage examples

PXParameterRequest is used extensively, both in the @PXTools modules themselves and in the applications integrating them.

### Examples by category

#### Login and security forms

| Instance | Module | Behaviour | Description |
|----------|--------|-----------|-------------|
| `Login` | @Security | Panel | The sign-in form |
| `Registration` | @Security | ParameterRequest | New user registration |
| `ChangePassword` | @Security | PopupParameterRequest | Password change |
| `RecoverPassword` | @Security | ParameterRequest | Password recovery |

#### Confirmation dialogs

| Instance | Module | Behaviour | Description |
|----------|--------|-----------|-------------|
| `Confirm` | @APIs | PopupParameterRequest | Generic "Are you sure?" confirmation |
| `ConfirmDelete` | @APIs | PopupParameterRequest | Deletion confirmation |

#### Capturing report parameters

```xml
<!-- Example: parameters for a sales report -->
<instance>
  <level name="SalesByPeriod" description="Sales report by period"
         generateWeb="true" generateWebResponsive="false">
    <behaviour type="ParameterRequest" returnType="Link" closeOnReturn="true" />
    <form>
      <attributes>
        <attribute name="DateFrom" />
        <attribute name="DateTo" />
      </attributes>
      <variables>
        <variable name="&Category">
          <controlInfo controlType="Combo" />
        </variable>
        <variable name="&Tags">
          <controlInfo controlType="Chosen" />
        </variable>
      </variables>
    </form>
    <actions>
      <action name="Confirm" caption="Generate report" />
      <action name="Cancel" caption="Cancel" />
    </actions>
    <calls>
      <call object="RptSalesByPeriod" />
    </calls>
  </level>
</instance>
```

#### File import

| Instance | Module | Behaviour | Description |
|----------|--------|-----------|-------------|
| `ImportCSV` | @ExportImport | PopupParameterRequest | Selecting a CSV file to import |
| `ImportExcel` | @ExportImport | PopupParameterRequest | Selecting an Excel file |

#### Task and mail management

| Instance | Module | Behaviour | Description |
|----------|--------|-----------|-------------|
| `SendMail` | @SendMails | PopupParameterRequest | The form for sending an email |
| `TaskCreate` | @TaskManager | PopupParameterRequest | Quick task creation |

### @PXTools modules using PXParameterRequest

| Module | Number of instances | Main use |
|--------|---------------------|----------|
| @APIs | Several | Generic confirmations |
| @Security | 4+ | Login, registration, password change/recovery |
| @SendMails | 1+ | The email sending form |
| @TaskManager | 2+ | Quick task creation and editing |
| @ExportImport | 2+ | Selecting files to import |

## 12. Pattern settings

The pattern's global settings are defined in `PXParameterRequestSettings.xml` (56 KB) and configure defaults for every instance.

### Main settings groups

| Group | Purpose |
|-------|---------|
| `Context` | Context information (KB, module, prefixes) |
| `Form` | The form's defaults (width, height, styles) |
| `Grid` | Defaults for the optional grid |
| `Help` | The contextual help system |
| `Images` | Default images for buttons and actions |
| `Labels` | Default texts/labels (Accept, Cancel, etc.) |
| `MasterPages` | Default MasterPages per platform |
| `PagingButtons` | The grid's paging configuration |
| `Prefixes` | Name prefixes for the generated objects |
| `Security` | Security configuration (permissions, authentication) |
| `StandardActions` | Standard actions and their default properties |
| `Tab` | Tab configuration when the form has tabs |
| `Template` | The default GeneXus template |
| `Templates` | Code generation templates (DLL/DKT) |
| `ThemeClasses` | Default CSS theme classes for each element |
| `Default Instance` | The default instance when creating a new PXParameterRequest |
| `Navigation` | Navigation configuration (breadcrumbs, menus) |
| `Objects` | Configuration of the generated objects |

## 13. The action system

PXParameterRequest's actions share the same structure as PXWorkWith's and PXComposer's. The complete Action node reference (properties, callTypes, ConditionalCalls, execution cycle, PXInstance rules) is in [12-pattern-ui-actions.md](12-pattern-ui-actions.md).

### The exclusive property: execute

PXParameterRequest has an exclusive **`execute`** property on its actions, controlling which validations apply before executing. Since PXParameterRequest's main job is **data entry**, there may be mandatory-data conditions defined in the level's `conditions` node that are shared by many actions:

```xml
<!-- The form's general conditions (mandatory data) -->
<conditions>
  <condition>not &DateFrom.IsEmpty()</condition>
  <condition>not &CustomerId.IsEmpty()</condition>
</conditions>
```

| execute value | Behaviour |
|---------------|-----------|
| **General Conditions** | Runs the level's general `conditions` **and** the action's local ones. For actions requiring all the data to be complete (Accept, Confirm) |
| **Action Conditions Only** | Runs **only** the action's local conditions, ignoring the general ones. For actions that do not require every field validated (Preview, Search) |

An example with `execute`:

```xml
<actions>
  <action name="Confirm" caption="Accept"
          execute="General Conditions"
          callType="Call" gxObject="GenerateReport" />

  <action name="Preview" caption="Preview"
          execute="Action Conditions Only"
          condition="&ReportType = 'PDF'"
          callType="Link" gxObject="PreviewReport" />

  <action name="Cancel" caption="Cancel" callType="Return" />
</actions>
```

### The Action node's structure (summary)

Every action supports pre/post invocation code:

```xml
<action name="Accept"
        caption="Accept"
        type="Standard"
        position="Bottom"
        conditionPreviousCode="// Code run before evaluating the condition"
        condition="&IsValid = true"
        actionPreviousCode="// Code run BEFORE the main invocation"
        callType="Call"
        gxObject="MyProcedure"
        actionPostCode="// Code run AFTER the main invocation"
        refreshAction="...">
  <parameters>
    <parameter name="Param1" />
  </parameters>
</action>
```

### callType kinds in actions

| callType | Description | Example use |
|----------|-------------|-------------|
| `Call` | Invokes a Procedure (no UI) | Processing, computing, updating records |
| `Link` | Navigates to another WebPanel/Transaction | Opening a detail screen |
| `External Link` | Opens a WebPanel not generated by PXTools | Opening a hand-written WebPanel |
| `Prompt` | Opens a modal dialog | Opening another PXParameterRequest as a popup |
| `Subroutine` | Runs a local subroutine | Reusable internal logic |
| `Event` | Runs inline code (no main object) | When all the logic is procedural code |
| `Submit` | Submits a batch process | Running an asynchronous task |
| `None` | Does nothing | A placeholder; only the pre/post code matters |
| `Return` | Only returns/closes the popup | Cancel, exit |

### Critical rule: linkType and Responsive migration

Only actions with `callType="Link"` need `linkType="PXInstance"` to be platform-independent. The other callTypes (`Call`, `Submit`, `Event`, and so on) point at Procedures whose names do not change between Desktop and Responsive.

```
callType="Link"  + linkType="PXInstance" → RIGHT (platform-independent)
callType="Link"  + linkType="GXObject"   → PROBLEM (it breaks when the platform changes)
callType="Call"  + gxObject="MyProc"     → OK (Procedures do not change name)
callType="Event" + (inline code)         → OK (it depends on no external names)
```

### An action's execution cycle

```
1. conditionPreviousCode  → Code preparing the condition's evaluation
2. condition              → If false, the action does not run
3. actionPreviousCode     → Validations, preparing data
4. [The main invocation]  → Per callType (Call/Link/Event/etc.)
5. actionPostCode         → Follow-up logic (updating states, WebSession, etc.)
6. refreshAction          → Refreshing the screen/grid
```

## 14. PXParameterRequest's real reach — an extended guide for AIs

### The fundamental principle

**All business logic can be migrated into hooks.** It does not matter how many subroutines, For Each loops, validations, WebSession uses, SDTs or Procedure calls there are. PXParameterRequest provides hooks at every point of the WebPanel's life cycle:

| Original code | Target hook |
|---------------|-------------|
| Event Start | `codes/Start` |
| Sub 'MySubroutine' | `codes/Subroutine` |
| Event Refresh | `codes/Refresh` |
| Event Load (grid) | `codes/Load` |
| Event 'Accept' (validations) | The action's `actionPreviousCode` |
| Event 'Accept' (main invocation) | The action's `callType` + `gxObject` |
| Event 'Accept' (post logic) | The action's `actionPostCode` |
| A custom Event 'MyEvent' | `events/event` |

### An extended catalogue of use cases

Based on analysing hundreds of hand-written WebPanels from real applications (object names are **illustrative**, following the naming convention):

| Use case | behaviour | Key elements | Example (illustrative) |
|----------|-----------|--------------|------------------------|
| **Simple confirmation** | PopupParameterRequest | A message + Accept/Cancel | `ConfirmChange`, `ConfirmDeleteCategory` |
| **Confirmation with a reason** | PopupParameterRequest | Message + text field + a min-10-chars validation | `ConfirmCancelWithReason` |
| **Activate/deactivate an entity** | PopupParameterRequest | Start evaluates the state, Accept calls a toggle procedure | `ConfirmToggleCustomer`, `ConfirmToggleCity` |
| **Activate/deactivate with tree validation** | PopupParameterRequest | Start walks the parent/child hierarchy, validating dependencies | `ConfirmToggleNode`, `ConfirmToggleCategory` |
| **Delete with validations** | PopupParameterRequest | Start checks dependencies, hides Accept when there are any | `ConfirmDeleteEntity` |
| **Simple selection with a grid** | PopupParameterRequest | Grid + filters + Enter returns the value | `SelCustomers`, `SelProducts` |
| **Selection that creates records** | PopupParameterRequest | Grid + selection + creates a temporary record through a procedure | `SelWithCreateOrder`, `SelWithCreateItem` |
| **Selection with complex validation** | PopupParameterRequest | Grid + Load with per-configuration conditional logic | `SelTypeByContext` |
| **Selection with multi-row checkboxes** | PopupParameterRequest | Grid with a per-row boolean + an editable value | `SelMultiRowItems` |
| **Selection with inline creation** | PopupParameterRequest | Grid + a mini form to create a new record | `SelWithInlineCreate` |
| **Cancelling a process** | PopupParameterRequest | A note + state validation + the cancellation procedure | `CancelProcess`, `CancelDocument` |
| **Reversing a record** | PopupParameterRequest | Start validates, Accept reverses it plus the related records | `ConfirmReverseEntity` |
| **Capturing report parameters** | ParameterRequest | A form with dates/filters + calls to the report | `PXReportParameters` |
| **Generating files** | PopupParameterRequest | A form + callType Event + generates TXT/Excel | `ExportRecordsFile` |
| **Uploading files** | PopupParameterRequest | A form with controlType File + extension validation | `UploadAttachment`, `UploadSignature`, `UploadLogo` |
| **Editing a single field** | PopupParameterRequest | A form with one editable field + Confirm | `EditFieldName` |
| **Error viewer with a grid** | PopupParameterRequest | A read-only grid loaded from an SDT/WebSession | `ErrorViewer`, `ImportErrorViewer` |
| **Read-only data viewer** | Panel or None | Display only, Exit | `ViewEntityDetail`, `ViewConcepts` |
| **Adjustment/computation with a form** | PopupParameterRequest | A form with inputs + computation logic + a procedure | `AdjustEntity` |
| **Assignment with numbering validation** | PopupParameterRequest | Start validates the numbering, Accept assigns and records | `AssignEntity` |
| **Detail CRUD** | PopupParameterRequest | An INS/UPD/DSP form with an image, dates and validations | `CreateEditDetail` |
| **One-shot execution** | Panel | A single button firing a process | PaymentRun |

### Common classification mistakes by AIs

**Do NOT mark a WebPanel as "Manual" merely because:**

1. **It has a lot of logic in Start/Accept** → all of it goes into hooks (`codes/Start`, `actionPreviousCode`, `actionPostCode`)
2. **It uses WebSession** → WebSession is the standard mechanism for communicating between screens; it does not prevent migration
3. **It has many subroutines** → they all go into `codes/Subroutine`
4. **It creates/modifies records** → the action can use `callType="Call"` or `callType="Event"`
5. **It has a grid with a complex Load** → the grid is an optional node with its own Load hook
6. **It hides/shows controls in Start** → standard logic in the Start hook
7. **It uses SDTs to pass data** → fully supported in variables and hooks
8. **It has complex validations** → they go into `actionPreviousCode` or `conditionPreviousCode`

**DO mark it "Manual" only when:**

1. It is a **Login screen** (a custom authentication flow, IsMain=True, no MasterPage)
2. It is a **testing/example utility** (not production functionality)
3. It depends on an **external framework** (a Scheduler, specific third-party controls)
4. It is an **automatic redirect** with no user interaction

## A quick summary for an AI

```
PXParameterRequest = a modal/popup/panel form for ANY interaction
                     that is not master-detail CRUD (that is PXWorkWith)

WHEN TO USE IT:
  - Capturing data before running something → ParameterRequest
  - A confirmation popup (simple or complex) → PopupParameterRequest
  - A floating form → FloatingParameterRequest
  - Embedding it in another panel → None or Panel
  - A read-only data viewer → Panel or None (no modification actions)
  - Selection with a grid (popup) → PopupParameterRequest + a grid node
  - Uploading files → PopupParameterRequest + controlType File
  - Generating files/reports → ParameterRequest + callType Event
  - Running a one-shot process → Panel + one Action with callType Call/Event

ALL THE LOGIC GOES IN HOOKS:
  - Start → codes/Start
  - Subroutines → codes/Subroutine
  - Refresh → codes/Refresh
  - Load (the grid) → codes/Load
  - The Accept button → an Action (actionPreviousCode + callType + actionPostCode)
  - Custom events → events/event

WHAT IT GENERATES:
  - 1 WebPanel (the same name on Desktop and Responsive)
  - 3 extra objects per Chosen control

IT INTEGRATES WITH:
  - PXWorkWith → its actions open a PXParameterRequest as a popup
  - PXFlowController → the flow's steps use PXParameterRequest
  - PXComposer → it embeds PXParameterRequest as a WebComponent
```
