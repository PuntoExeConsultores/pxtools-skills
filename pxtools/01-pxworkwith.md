# PXWorkWith — the master-detail CRUD pattern

## Metadata

| Field | Value |
|-------|-------|
| Pattern | PXWorkWith |
| Association type | Transaction or (None) |
| Definition file | PXWorkWith.pattern |
| Instance file | PXWorkWithInstance.xml (~439 KB) |
| Settings file | PXWorkWithSettings.xml (~166 KB) |
| Platforms | Web Desktop, Web Responsive, Smart Devices |

---

## 1. What it is and what it is for

PXWorkWith is PXTools' main pattern for automatically generating CRUD (Create, Read, Update, Delete) interfaces with a master-detail architecture. From a single XML configuration instance it generates several GeneXus objects covering:

- **Selection**: a selection grid with filters, orders, actions and export.
- **View**: a detail view with tabular tabs and/or grids.
- **Prompt**: a reusable search/lookup dialog.
- **Transaction**: a transaction with an HTML or Abstract form.
- **Controller**: a navigation controller between Selection, View and Transaction.
- **Export**: procedures for exporting to Excel and generating charts.

Every object is generated in a Desktop (HTML WebForm) and a Responsive (Abstract Form) version, controlled by independent properties.

### Real-world usage statistics

PXWorkWith is the framework's most used pattern: it appears extensively in the @PXTools modules themselves (Security, Alerts, CloudTasks, FileStorage, among others) and in the applications integrating them.

---

## 2. Objects it generates

A single PXWorkWith instance generates the following GeneXus objects:

| Object ID | GX type | Name pattern | Element XPath | Description |
|---|---|---|---|---|
| TransactionDesktop | Transaction | `D{Instance.Name}` | `//transaction` | A transaction with an HTML WebForm |
| TransactionResponsive | Transaction | `R{Instance.Name}` | `//transaction` | A transaction with an Abstract Form |
| SelectionSDT | SDT | `PXWW{Instance.Name}Rows` | `instance/level/selection` | The SDT of the selected rows |
| PromptSDT | SDT | `PXWW{Instance.Name}Rows` | `instance/level/prompt` | The SDT of the prompt's rows |
| TabGridSDT | SDT | `PXWW{Instance.Name}Rows` | `instance/level/view/sections//section[@type='Grid']` | The SDT of the tab grid's rows |
| Selection | WebPanel | `WW{Instance.Name}` | `instance/level/selection` | The selection grid (Desktop) |
| SelectionResponsive | WebPanel | `RWW{Instance.Name}` | `instance/level/selection` | The selection grid (Responsive) |
| View | WebPanel | `View{Instance.Name}` | `instance/level/view` | The detail view (Desktop) |
| ViewResponsive | WebPanel | `RView{Instance.Name}` | `instance/level/view` | The detail view (Responsive) |
| TabTabular | WebComponent | `{Element.wcname}` | `instance/level/view/sections//section[@type='Tabular']` | A tabular tab (Desktop) |
| TabTabularResponsive | WebComponent | `{Element.wcname}` | idem | A tabular tab (Responsive) |
| TabGrid | WebComponent | `{Element.wcname}` | `instance/level/view/sections//section[@type='Grid']` | A grid tab (Desktop) |
| TabGridResponsive | WebComponent | `{Element.wcname}` | idem | A grid tab (Responsive) |
| Prompt | WebPanel | `Pr{Instance.Name}` | `instance/level/prompt` | The prompt/lookup (Desktop) |
| PromptResponsive | WebPanel | `RPr{Instance.Name}` | `instance/level/prompt` | The prompt/lookup (Responsive) |
| LevelController | WebPanel | `Ct{Instance.Name}` | `instance/level[../transaction[@afterTrn='Call Levels Controllers']]` | The level controller (Desktop) |
| LevelControllerResponsive | WebPanel | `RCt{Instance.Name}` | idem | The level controller (Responsive) |
| InstanceController | WebPanel | `Ct{Instance.Name}` | `instance/controller` | The instance controller (Desktop) |
| InstanceControllerResponsive | WebPanel | `RCt{Instance.Name}` | idem | The instance controller (Responsive) |
| SelectionExportExcel | Procedure | `Ex{Instance.Name}` | `instance/level/selection/modes` | Export to Excel |
| SelectionExportExcelSDT | SDT | `Ex{Instance.Name}ExcelSDT` | idem | The SDT for the Excel export |
| SelectionChart | Procedure | `Ex{Instance.Name}` | `instance/level/selection/modes` | Chart generation |
| SelectionUpdateGridRowsProcedure | Procedure | `Upd{Instance.Name}GridRows` | `instance/level/selection/modes` | Bulk row update |
| ChkChosenValueSelected | Procedure | `Chk{Instance.Name}ChosenValueSelected` | `instance//variable/controlInfo[@controlType='Chosen']` | Chosen control validation |
| RetChosenValues | DataProvider | `Ret{Instance.Name}ChosenValues` | idem | Returns the Chosen values |
| RetChosenResults | Procedure | `Ret{Instance.Name}ChosenResults` | idem | Returns the Chosen results |
| SelectionGridHandlerDataProvider | DataProvider | `PXWW{Instance.Name}Rows` | `instance/level/selection` | The Selection's grid handler |
| PromptGridHandlerDataProvider | DataProvider | `PXWW{Instance.Name}Rows` | `instance/level/prompt` | The Prompt's grid handler |
| TabGridGridHandlerDataProvider | DataProvider | `PXWW{Instance.Name}Rows` | `instance/level/view/sections//section[@type='Grid']` | The Tab Grid's grid handler |

> **Note**: objects sharing the same name pattern (for example `PXWW{Instance.Name}Rows`) are told apart by the XPath context of the element originating them inside the instance.

---

## 3. XML structure of the instance

### 3.1 The main hierarchy

```
instance
 |
 +-- transaction ................. The transaction's configuration
 |    +-- afterTrn .............. Post-transaction behaviour
 |    +-- generateTransaction ... Whether or not to generate the transaction
 |
 +-- level ....................... A level (there can be several)
 |    +-- name, description
 |    +-- masterPage, theme
 |    +-- generateWeb ........... Generate Desktop
 |    +-- generateWebResponsive . Generate Responsive
 |    +-- generateSD ............ Generate Smart Devices
 |    |
 |    +-- controller ............ The controller's configuration
 |    |
 |    +-- selection ............. The selection grid
 |    |    +-- grid ............. The grid's configuration
 |    |    |    +-- attributes .. Columns
 |    |    +-- filter ........... Filters
 |    |    |    +-- search ...... Quick search
 |    |    |    +-- advancedSearch Advanced filters
 |    |    |    +-- conditions .. Fixed conditions
 |    |    +-- orders ........... Sort orders
 |    |    +-- actions .......... Actions/buttons
 |    |    +-- modes ............ Export Excel, Charts, Update
 |    |    +-- events ........... Custom event code
 |    |    +-- codes ............ Hooks: Start, Refresh, Load, Sub
 |    |    +-- variables ........ Custom variables
 |    |    +-- parameters ....... The WebPanel's parameters
 |    |    +-- fixedData ........ Rows of fixed data
 |    |
 |    +-- view .................. The detail view with tabs
 |    |    +-- form ............. The attribute layout
 |    |    +-- sections ......... Tabs
 |    |    |    +-- section ..... type: Tabular | Grid
 |    |    +-- actions, events, codes, variables, parameters
 |    |
 |    +-- prompt ................ The search/lookup dialog
 |    |    +-- (a structure similar to selection)
 |    |
 |    +-- navigation ............ Selection/View/Edit navigation
 |
 +-- ...
```

### 3.2 A simplified XML example

```xml
<instance parentObject="Invoice">
  <transaction afterTrn="Call Controller" generateTransaction="True">
    <!-- The transaction's configuration -->
  </transaction>

  <level name="Invoice" description="Invoices"
         masterPage="PXMasterPage" theme="PXTheme"
         generateWeb="True" generateWebResponsive="True" generateSD="False">

    <controller />

    <selection>
      <grid>
        <attributes>
          <attribute name="InvoiceId" description="Invoice no." visible="True" />
          <attribute name="InvoiceDate" description="Date" visible="True" />
          <attribute name="CustomerName" description="Customer" visible="True" />
          <attribute name="InvoiceTotal" description="Total" visible="True" />
        </attributes>
      </grid>

      <filter>
        <search>
          <attribute name="InvoiceId" />
          <attribute name="CustomerName" />
        </search>
        <advancedSearch>
          <attribute name="InvoiceDate" type="Range" />
          <attribute name="CustomerId" />
        </advancedSearch>
        <conditions>
          <condition expression="InvoiceCancelled = false" />
        </conditions>
      </filter>

      <orders>
        <order name="Date Desc" expression="InvoiceDate DESC" default="True" />
        <order name="Customer" expression="CustomerName" />
      </orders>

      <actions>
        <action name="Insert" caption="Insert" callType="Link" />
        <action name="Update" caption="Update" callType="Link" />
        <action name="Delete" caption="Delete" callType="Link" />
        <action name="ExportExcel" caption="Export" callType="Link" />
        <action name="Print" caption="Print"
                callType="Client Text Print" />
      </actions>

      <modes>
        <exportExcel enabled="True" />
        <chart enabled="False" />
        <updateGridRows enabled="False" />
      </modes>

      <events>
        <!-- Custom GeneXus event code -->
      </events>

      <codes>
        <start>/* Code run at startup */</start>
        <refresh>/* Code run in refresh */</refresh>
        <load>/* Code run in the load of each row */</load>
        <subroutine>/* Reusable subroutines */</subroutine>
      </codes>

      <variables>
        <variable name="&amp;MyVariable" type="Numeric" length="4" />
      </variables>

      <parameters>
        <parameter name="CustomerId" />
      </parameters>

      <fixedData>
        <!-- Optional rows of fixed data -->
      </fixedData>
    </selection>

    <view>
      <form>
        <attribute name="InvoiceId" />
        <attribute name="InvoiceDate" />
        <attribute name="CustomerName" />
      </form>

      <sections>
        <section name="Lines" type="Grid" wcname="WWInvoiceLines">
          <grid>
            <attributes>
              <attribute name="ProductName" />
              <attribute name="InvoiceLineQuantity" />
              <attribute name="InvoiceLinePrice" />
            </attributes>
          </grid>
          <filter>
            <search />
          </filter>
          <orders />
          <actions />
        </section>

        <section name="Notes" type="Tabular" wcname="WWInvoiceNotes">
          <form>
            <attribute name="InvoiceNotes" />
          </form>
        </section>

        <section name="Attachments" type="Grid"
                 externalComponent="true" wcname="WWInvoiceAtt">
          <!-- An embedded external component -->
        </section>
      </sections>

      <actions>
        <action name="Edit" caption="Edit" callType="Link" />
      </actions>
    </view>

    <prompt>
      <grid>
        <attributes>
          <attribute name="InvoiceId" />
          <attribute name="CustomerName" />
          <attribute name="InvoiceDate" />
        </attributes>
      </grid>
      <filter>
        <search>
          <attribute name="InvoiceId" />
        </search>
      </filter>
      <orders>
        <order name="Default" expression="InvoiceId DESC" default="True" />
      </orders>
    </prompt>

    <navigation />
  </level>
</instance>
```

---

## 4. The main nodes

### 4.1 Selection

The `selection` node generates the main selection grid (the master listing). It produces the objects `WW{Instance.Name}` (Desktop) and `RWW{Instance.Name}` (Responsive).

**Subnodes:**

| Subnode | Purpose |
|---------|---------|
| `grid` | Defines the visible columns (attributes and variables) |
| `filter` | Quick-search, advanced-search and fixed-condition filters |
| `orders` | The available sort orders |
| `actions` | The user's buttons and actions |
| `modes` | Which operations the pattern generates (see below) |
| `confirms` | Confirmation dialogs with dynamic text (see [12-pattern-ui-actions.md](12-pattern-ui-actions.md) §5.bis) |
| `events` | Custom GeneXus event code |
| `codes` | Code hooks: Start, Refresh, Load, Subroutine |
| `variables` | Additional custom variables |
| `parameters` | The WebPanel's input parameters |
| `fixedData` | Rows of fixed data (not coming from the database) |

**Subnode order** (the pattern validates them by sequence; out of order, the instance does not
apply): `modes`, `orders`, `filter`, `layouts`, `codes`, `actions`, `confirms`, `variables`.

#### The `modes` node

It is configured with **attributes, not children**, and controls which operations the pattern generates:

```xml
<modes />                                          <!-- all the defaults -->
<modes Delete="false" />                           <!-- no deletion -->
<modes Insert="false" Update="false" Delete="false" Display="false" Export="true" />
```

| Attribute | What it enables |
|---|---|
| `Insert` / `Update` / `Delete` / `Display` | The four operations on the transaction |
| `Export` | Export to Excel |
| `Chart` (+ `chartWidth`, `chartHeight`) | Charts |
| `AddSelected` / `AddAll` / `RemoveSelected` / `RemoveAll` | Multiple selection |
| `InsertEventCondition` | Makes the insert conditional on an event |

Values: `true` | `false` | `default`.

> **Disabling a mode is what frees the action's name.** To replace `Delete`'s behaviour with your
> own you have to set `Delete="false"` **and also** declare an `action name="Delete"`; with the mode
> enabled the pattern generates its own action against the transaction anyway.

### 4.2 View

The `view` node generates a record's detail screen. It produces the objects `View{Instance.Name}` (Desktop) and `RView{Instance.Name}` (Responsive).

**Components:**

- **form**: the tabular layout of the main record's attributes.
- **layouts/layout/fixedData**: the View's **header** (see below).
- **sections**: tabs, which can be of type `Tabular` or `Grid`. Each section generates an independent WebComponent. Only the active tab is rendered, so `GlobalEvents` does not apply across tabs. To share information between tabs you can use **WebSession** (one tab writes, another reads it in its Start/Refresh), implementing it in the code hooks.
- **actions, events, codes, variables, parameters**: the same structure as Selection.

**The View's header (`fixedData`):** the View should declare a `layouts/layout/fixedData/fixedDataAttributes` (usually with one `<row>`) showing, fixed above the tabs, the **representative data of the positioned record** — typically the values of the **parameters the View received** (or their descriptions/representative attributes). E.g. if the View receives `CustomerId, OrderId`, the row shows `CustomerName`, `OrderDate` and `OrderId`:

```xml
<view caption="&quot;Order &quot; + OrderId.ToString().Trim()" ...>
  <parameters>
    <parameter name="CustomerId" null="True" />
    <parameter name="OrderId" null="True" />
  </parameters>
  <layouts>
    <layout platform="Any">
      <fixedData>
        <fixedDataAttributes>
          <row>
            <attribute name="CustomerName" description="Customer" descriptionPosition="Left" />
            <attribute name="OrderDate" description="Date" descriptionPosition="Left" />
            <attribute name="OrderId" description="Order" descriptionPosition="Left" />
          </row>
        </fixedDataAttributes>
      </fixedData>
    </layout>
  </layouts>
  <sections> ... </sections>
</view>
```

### 4.3 Prompt

The `prompt` node generates a search/lookup dialog. It produces the objects `Pr{Instance.Name}` (Desktop) and `RPr{Instance.Name}` (Responsive). Its internal structure is similar to `selection`'s (grid, filter, orders, and so on) but aimed at selecting one record and returning its key.

### 4.4 Controller

There are two kinds of controller:

| Kind | XPath | Generation condition |
|------|-------|----------------------|
| **InstanceController** | `instance/controller` | Always available |
| **LevelController** | `instance/level` | Only when `transaction[@afterTrn='Call Levels Controllers']` |

The controller manages navigation between Selection, View and Transaction (editing). Both generate `Ct{Instance.Name}` (Desktop) and `RCt{Instance.Name}` (Responsive) objects.

The transaction's `afterTrn` property determines the post-save flow:

| Value | Behaviour |
|-------|-----------|
| `Call Controller` | Returns to the instance controller |
| `Call Levels Controllers` | Returns to the corresponding level's controller |
| `Do Nothing` | Runs no post-transaction navigation |

### 4.5 Transaction (the edit form)

The `transaction/layouts/layout` node defines the **transaction's form**. Left empty (`<layout platform="Any" />`) the pattern generates the default form; with an `<attributes>` node you define **the complete form**: the attributes listed, in that order, with their description and their control.

```xml
<transaction name="Customer, Sales">
  <parameters>...</parameters>
  <layouts>
    <layout platform="Any">
      <attributes>
        <attribute name="CustomerId" description="Id" />
        <attribute name="CustomerName" description="Name" />
        <attribute name="CustomerCountryId" description="Country">
          <controlInfo controlType="Dynamic Combo Box" itemValue="CountryId" itemDescription="CountryName" emptyItem="True" emptyItemText="Select" />
        </attribute>
      </attributes>
    </layout>
  </layouts>
</transaction>
```

`itemValue` / `itemDescription` are attributes **of the foreign table** (not the subtypes), and both must belong to the same table. For a combo fed by a DataProvider instead of by a table, see §6.1.

> **List every attribute.** The `<attributes>` node replaces the whole form: an omitted attribute
> **disappears from the screen**. When adding a single combo you still have to enumerate all the rest.

### 4.6 Where a ControlType is defined

| Place | Reach | When |
|---|---|---|
| The pattern's instance (`layout/attributes/attribute/controlInfo`) | That screen only | **Preferred** |
| The transaction's / WebPanel's WebForm | That form only | When the form is `dynamic="false"` and is edited by hand, or when the object has no instance |
| `.gxAttribute` (an attribute's property) | **The whole KB** | Avoid |
| `.gxDomain` (a domain's property) | **Every attribute of the domain** | Avoid |

> **This project's convention**: edit controls are **not** defined on attributes or on domains. It is
> a modality that varies per project — GeneXus allows all four — but here the control always lives in
> the instance, or in the form when the object has no instance.

#### The empty item belongs to the control, never to the data

⚠️ **Never declare the empty value on the domain or on the attribute.** An `(All)`, a `(None)` or a
`(Select...)` is the need of **one particular screen** — a filter meaning "do not filter" — not a
property of the data. Put on the domain, it shows up on every screen using that type, including the
edit forms where that value is not valid.

It goes in the attribute's `controlInfo` inside the instance, in **lower case**:

```xml
<attribute name="MessagingMessageStatus" description="Status" descriptionPosition="Left">
  <controlInfo controlType="Combo Box" emptyItem="True" emptyItemText="(All)" />
</attribute>
```

It works the same on a `Dynamic Combo Box`, alongside its `dataProvider`.

> **Beware the false friend**: the domain *also* accepts `EmptyItem` / `EmptyItemText` (initial
> capital, a different node) and the import takes it without complaint. But the combo is generated
> with `AddEmptyItem="False"` regardless, so the empty item **does not appear** and the symptom is "I
> set the property and nothing happened". The clue that tells them apart is in the generated form: the
> control that really has it comes out with `AddEmptyItem="True"`.

Defining it on the **attribute** propagates it to every appearance of the attribute, **grids included**. A Dynamic Combo Box in a grid column loads every record of the foreign table **for each row**: an unacceptable inefficiency in a listing. It would only be admissible by forcing `Edit` in every grid where the attribute appears, which is fragile.

**The rule for grids: always `Edit`.** If you have to show a piece of data from the foreign entity (typically the name), you do not use a combo: you define a **subtype of the description attribute** in the subtype group and in the transaction, and you show that subtype as `Edit`. The `Id` is left `visible="False"` — it stays available for the actions needing the key — and the name is shown:

```xml
<attribute name="CustomerCountryId" description="Country Id" visible="False" autolink="False">
  <controlInfo controlType="Edit" />
</attribute>
<attribute name="CustomerCountryName" description="Country" visible="True" autolink="False">
  <controlInfo controlType="Edit" />
</attribute>
```

That is: **a combo in the edit form, a name subtype in the grid**. See the group naming convention in [20-pxtools-modules.md](20-pxtools-modules.md).

---

## 5. The action system

The action system is shared by the three UI patterns (PXWorkWith, PXParameterRequest, PXComposer). The complete reference is in [12-pattern-ui-actions.md](12-pattern-ui-actions.md). This section documents the specifics of using it in PXWorkWith.

Each action inside `actions` has the following structure:

```xml
<action name="MyAction"
        caption="Visible text"
        callType="Link"
        linkType="GXObject"
        target="Self"
        condition="InvoiceStatus = 'A'"
        confirm="false">
  <security object="MyObject" operation="Execute" />
</action>
```

### An action's properties

| Property | Possible values | Description |
|----------|-----------------|-------------|
| `name` | free text | The action's unique identifier |
| `caption` | free text | The text visible on the button/link |
| `conditionPreviousCode` | code | Procedural code run before evaluating the visibility condition |
| `condition` | a GeneXus expression | The action's visibility condition (evaluated per row when the action is in the grid) |
| `evaluateCondition` | `Event`, `Load` | When to evaluate the condition: in Load (for each row) or in Event (on click) |
| `actionPreviousCode` | code | Procedural code run **before** the object's main invocation |
| `callType` | `Link`, `Call`, `Prompt`, `External Link`, `Client Text Print`, `Subroutine`, `Event`, `Submit`, `None`, `Return` | The invocation type (see the detailed table below) |
| `linkType` | `GXObject`, `PXInstance` | For callType `Link` only: the target is a direct GeneXus object or a PXTools instance |
| `actionPostCode` | code | Procedural code run **after** the main invocation |
| `refreshAction` | enum | How to refresh the screen/grid after the action (useful when the action modifies data shown in the grid) |
| `target` | `Self`, `New` | `Self` = the same window, `New` = a popup/new window |
| `confirm` | text or boolean | A confirmation dialog before executing |
| `security` | object + operation | Access control (a GAM/PXSecurity object and operation) |

### Per-row conditional actions

Actions support **conditional visibility per row** through the `condition` property evaluated in the grid's Load. Beyond that, the **Conditional Calls** concept lets a single action have **different behaviours depending on the context** of each row's information:

```xml
<!-- An action visible only for pending invoices -->
<action name="Approve"
        caption="Approve"
        callType="Link"
        condition="InvoiceStatus = 'Pending'"
        evaluateCondition="Load">
</action>

<!-- Conditional Calls: the same action, a different target depending on the context -->
<action name="View"
        caption="View detail"
        callType="Link">
  <conditionalCalls>
    <conditionalCall condition="InvoiceType = 'Domestic'"
                     gxObject="ViewDomesticInvoice" />
    <conditionalCall condition="InvoiceType = 'Export'"
                     gxObject="ViewExportInvoice" />
  </conditionalCalls>
</action>
```

This lets you:
- Show/hide buttons according to the status, the type or any piece of data of each record
- Invoke different objects from the same button according to the row's context
- Control whether the condition is evaluated in Load (per row) or in Event (on click)

### Invocation types (callType)

```
+---------------------+------------------------------------------+
| callType            | Behaviour                                |
+---------------------+------------------------------------------+
| Link                | Navigates to another WebPanel/Transaction|
| Call                | Invokes a Procedure (no UI)              |
| Prompt              | Opens a modal dialog (popup)             |
| External Link       | Opens an external URL or a non-PXTools WP|
| Client Text Print   | Printing from the client                 |
| Subroutine          | Runs a subroutine inside the same object |
| Event               | Runs inline code (no main object)        |
| Submit              | Submits a batch process                  |
| None                | Does nothing                             |
| Return              | Only returns (closes the popup/goes back)|
+---------------------+------------------------------------------+
```

### Target types (linkType)

- **GXObject**: directly invokes a GeneXus object (WebPanel, Procedure, Transaction).
- **PXInstance**: invokes another PXTools instance, automatically resolving the corresponding generated object.

**A critical rule for the Responsive migration:** only actions with `callType="Link"` need to use `linkType="PXInstance"` instead of `linkType="GXObject"`, because they are the ones invoking **graphical interfaces generated by patterns**, whose names change with the platform (e.g. `WWInvoice` → `RWWInvoice`). The other callTypes (`Call`, `Submit`, `Event`, `Subroutine`, and so on) invoke Procedures or objects whose names **do not change** between platforms, so `GXObject` is right for them.

### PXInstance properties (for callType Link)

When `linkType="PXInstance"`, these properties identify the generated object:

| Property | Description |
|----------|-------------|
| `instanceObject` | The target pattern instance's name (e.g. `PXWorkWithInvoice`) |
| `instanceLevel` | The level's name inside the instance |
| `instanceLevelNode` | The level node to invoke: `Selection`, `View`, `Prompt`, `Transaction`, `Level` |
| `instanceLevelViewSection` | (View only) The specific section/tab of the View to navigate to. Omitted, it goes to the main tab |

The generator resolves the object's name per platform automatically:
```
PXInstance → PXWorkWithInvoice / Level=Invoice / Node=Selection
  Desktop generates:    a Link to WWInvoice
  Responsive generates: a Link to RWWInvoice
```

### An action's execution cycle

```
1. conditionPreviousCode  → Code preparing the condition's evaluation
2. condition              → If false, the action is not shown/run
3. actionPreviousCode     → Code before the main invocation
4. [The main invocation]  → Per callType (Link/Call/Event/etc.)
5. actionPostCode         → Code after the invocation
6. refreshAction          → Refreshing the screen/grid when applicable
```

---

## 6. Filters and orders

### 6.1 Filters

The `filter` node holds three filtering mechanisms:

```
filter
 +-- search .............. Quick search (the top bar)
 |    +-- attribute ...... The attributes included in the free search
 |
 +-- advancedSearch ...... Advanced filters (the drop-down panel)
 |    +-- attribute ...... Fields with a filter type (Range, Exact, etc.)
 |
 +-- conditions .......... Fixed conditions (not visible to the user)
      +-- condition ...... A GeneXus expression always applied
```

**search (quick search)**: the list of attributes searched with free text. The user types a term and it is filtered against every listed attribute with a LIKE operator.

**advancedSearch (advanced search)**: individual fields with specific controls. Each attribute can have a filter type (a date range, an exact value, a list, and so on).

**conditions (fixed conditions)**: GeneXus expressions always applied as a WHERE filter, which the user cannot change. They are used to restrict data by context (for example, filtering only active records).

**Filter combos fed by a DataProvider**: a filter can be a `variable` with `<controlInfo controlType="Dynamic Combo Box" dataSourceFrom="DataProvider" dataProvider="Ret<Entity>, <module>" dataProviderParameters="&amp;X" dataProviderItemValue="Id" dataProviderItemDescription="Name" emptyItem="True" emptyItemText="(All)" />`, fed by a `Ret<Entity>` returning a collection SDT `{Id, Name}`. For the **DataProvider + SDT format** (Output/Collection, the Output group, the rule that a DataProvider does **not** accept `For Each`, and the combo recipe) see the reference **kbbridge → `genexus-dataprovider.md`**.

### 6.2 Orders

The `orders` node defines the available sorting options:

```xml
<orders>
  <order name="Most recent" expression="InvoiceDate DESC" default="True" />
  <order name="By customer" expression="CustomerName ASC, InvoiceDate DESC" />
  <order name="By amount" expression="InvoiceTotal DESC" />
</orders>
```

| Property | Description |
|----------|-------------|
| `name` | The text visible in the order selector |
| `expression` | The sorting expression (attributes + ASC/DESC) |
| `default` | `True` if it is the order applied by default |

A combo/selector is generated letting the user choose the sort criterion at run time.

---

## 7. Tabs (Grid and Tabular)

The detail view's tabs (`view/sections/section`) can be of two types:

### 7.1 A Grid section

Shows a detail grid (a 1:N relationship). It generates a WebComponent with a structure similar to Selection's:

```xml
<section name="Lines" type="Grid" wcname="WWInvoiceLines">
  <grid>
    <attributes>...</attributes>
  </grid>
  <filter>...</filter>
  <orders>...</orders>
  <actions>...</actions>
  <modes>...</modes>
</section>
```

Objects generated:
- `TabGrid` / `TabGridResponsive` (WebComponent)
- `TabGridSDT` (the rows SDT)
- `TabGridGridHandlerDataProvider` (DataProvider)

### 7.2 A Tabular section

Shows attributes in form format (1:1 data or complementary information):

```xml
<section name="Notes" type="Tabular" wcname="WWInvoiceNotes">
  <form>
    <attribute name="InvoiceNotes" />
    <attribute name="InvoiceRemarks" />
  </form>
</section>
```

Objects generated:
- `TabTabular` / `TabTabularResponsive` (WebComponent)

### 7.3 External components

When `externalComponent="true"`, the section embeds an existing WebComponent instead of generating a new one. That allows integrating manually developed components.

```
+------------------------------------------+
| View: ViewInvoice                        |
|                                          |
|  [Invoice #1234]  [Date: 2026-01-15]     |
|  [Customer: Acme Corp]                   |
|                                          |
|  +------+-------------+-----------+      |
|  |Lines | Notes       | Attachm.  |      |
|  +------+-------------+-----------+      |
|  | Tab Grid           | Tab Tabul | Tab  |
|  | (type="Grid")      | (type=    | ext. |
|  |                    | "Tabular")| comp |
|  | ProductName     Q P| InvNotes  | (ext)|
|  | InvLineQty      5 U| InvRemarks|      |
|  | InvLinePrice  100 U|           |      |
|  +--------------------+-----------+------+
```

---

## 8. Modes (Export, Charts, Update Grid Rows)

The `modes` node inside `selection` enables special features:

### 8.1 Export Excel

```xml
<modes>
  <exportExcel enabled="True" />
</modes>
```

It generates:
- `Ex{Instance.Name}` (Procedure): the export logic.
- `Ex{Instance.Name}ExcelSDT` (SDT): the data structure for the Excel file.

**Excel templates**: PXTools supports using Excel files as **templates** for the export. That lets the end user define custom designs: complex formats, several sheets, logos, formulas, charts, and so on. The export Procedure takes the Excel template as a base and fills it with the grid's data.

### 8.2 Charts

```xml
<modes>
  <chart enabled="True" />
</modes>
```

It generates:
- `Ex{Instance.Name}` (Procedure): the chart-generation logic.

### 8.3 Update Grid Rows (bulk update)

```xml
<modes>
  <updateGridRows enabled="True" />
</modes>
```

It generates:
- `Upd{Instance.Name}GridRows` (Procedure): the logic for bulk-updating the rows selected in the grid.

### 8.4 CRUD modes (attributes) and a Selection with no transaction

In real instances the `modes` node carries the modes as **attributes** (not subnodes): `Insert`, `Update`, `Delete`, `Display` — which interact with the WorkWith's **Transaction** — plus `Export`:

```xml
<modes Insert="false" Update="false" Delete="false" Display="false" Export="true" />
```

**A standalone Selection (with no transaction):** when the PXWorkWith is not associated with a Transaction (its base table comes from inference), the **4** modes interacting with the transaction — `Insert`, `Update`, `Delete`, `Display` — must be set to **`false`** (there is no transaction to navigate to). `Export="true"` is independent and enables the Excel export; it is a valid use of the `modes` node even in a read-only listing. The link to the View is kept through `descriptionAttribute` / `<link>` (it does not depend on the `Display` mode).

---

## 9. Code hooks (Events and Codes)

PXWorkWith provides extension points for injecting custom GeneXus code into the generated objects.

### 9.1 The `events` node

It lets you define complete GeneXus events (Event handlers) injected into the generated object. It is used for logic reacting to the user's actions.

### 9.2 The `codes` node

It defines code blocks run at specific moments of the object's life cycle:

| Hook | When it runs | Typical use |
|------|--------------|-------------|
| `start` | When the WebPanel starts (the Start event) | Initialising variables, checking permissions |
| `refresh` | When the grid refreshes (the Refresh event) | Recomputing filters, updating state |
| `refreshForm` | A refresh of the **WebForm only** (it is not included in the `Ex{Name}` export) | The form's UI commands: `.Enabled`, `.Visible`, control properties (see §9.7) |
| `load` | When each grid row loads (the Load event) | Computing derived fields, applying conditional formatting |
| `subroutine` | Subroutines invocable from the events | Reusable logic inside the object |

```xml
<codes>
  <start>
    &amp;MyVariable = Today()
    If &amp;CustomerId.IsEmpty()
      Return
    EndIf
  </start>
  <refresh>
    &amp;TotalRecords = CountInvoices(&amp;DateFilter)
  </refresh>
  <load>
    If InvoiceTotal > 10000
      &amp;RowClass = "HighValue"
    EndIf
  </load>
  <subroutine>
    Sub 'UpdateTotals'
      &amp;GrandTotal = SumInvoices()
    EndSub
  </subroutine>
</codes>
```

> **CDATA formatting and indentation**: the code's first line goes at **column 0** and only block nesting indents (+1 tab). It is common to every pattern with code nodes — the full rule is in [`00-overview.md`](00-overview.md) → *Code hooks: CDATA formatting and indentation*.

### 9.3 A grid with no base table (Load with no base table)

PXWorkWith supports grids **entirely based on variables**, with no need for a database table. In that mode:

- The grid is defined with **variables** instead of transaction attributes
- The **filters** are declared as variables in the filter section
- The **Load code** defines how each row's data is populated (it can invoke external web services, APIs, in-memory SDTs, and so on)
- Filters and orders work normally because they operate on the declared variables

That makes it possible to build PXWorkWith screens showing data coming from:
- External web services (REST/SOAP)
- Third-party APIs
- In-memory computations
- Any data source reachable from GeneXus code

```xml
<selection>
  <grid>
    <!-- A grid 100% based on variables, with no transaction attributes -->
    <variables>
      <variable name="Code" dataType="Character" length="20" />
      <variable name="Name" dataType="Character" length="100" />
      <variable name="Price" dataType="Numeric" length="12" decimals="2" />
    </variables>
  </grid>
  <filter>
    <variables>
      <variable name="NameFilter" dataType="Character" length="100" />
    </variables>
  </filter>
  <codes>
    <load><![CDATA[
      // Load the data from an external web service
      &SDTProducts = WSGetProducts.Udp(&NameFilter)
      For &Product in &SDTProducts
        &Code = &Product.Code
        &Name = &Product.Name
        &Price = &Product.Price
        Load  // Adds the row to the grid
      EndFor
    ]]></load>
  </codes>
</selection>
```

### 9.4 Custom variables

The `variables` node lets you declare additional variables needed by the code hooks:

```xml
<variables>
  <variable name="&amp;MyVariable" type="Numeric" length="4" />
  <variable name="&amp;RowClass" type="Character" length="20" />
</variables>
```

**Rule — do not duplicate the form's variables:** the `variables` node must contain **only** the variables used in procedural code (`codes`, `previousCode`, `actionPostCode`, and so on) that are **not** already declared in the form. A variable declared as a **filter** (`filter/attributes`) or as a grid **column** (a `variable` inside `attributes`) must **not** be re-declared in `variables`: doing so produces the build error *"Variable X is declared twice"* and **makes the pattern application fail** (`PatternApplicationException`), so the Ww/View objects are not generated. (E.g. `&CustomerId`, used in the Start and in a combo's `dataProviderParameters` but with no control on the form, does go in `variables`; whereas `&CategoryId`/`&StatusId`, which are filter combos, do not.)

### 9.5 Determining the Selection's base table

GeneXus determines a Selection's **base table** **by inference**: it gathers (a) every attribute used as a grid **column** and (b) every attribute referenced inside the **InGrid actions** — the parameters of whatever the action invokes, `previousCode`, `actionPostCode` and a `condition` evaluated in the event (everything evaluated inside the action's event) — and looks for the table having all those attributes as its base or extended table.

- **No base table** (every column and every piece of action data is a variable): GeneXus decides the object has no base table ⇒ you have to iterate manually in a `code type="Load"` with the **`Load`** command inside it (see 9.3).
- **With a base table**: the `code Load` is **optional** (GeneXus does the iteration by walking the base table). A `Load` command inside the `code Load` means **conditional loading**: if in one iteration the code runs and does **not** reach a `Load`, that record **is not loaded**. But **a base table plus a `Load` command is very inefficient** with thousands of records (it runs the `code Load` for every record) and it complicates the Selection's total count, forcing `PagingProgrammingStyle = PXTools` (a mass walk, costlier still). ⇒ **Not recommended**: with a base table, use the `code Load` **only to compute variables** (with no `Load` command) and filter through `filter/conditions`.

**A practical rule:** for the Selection to have a base table and an efficient native walk, define the columns and the actions' parameters/conditions as **attributes** (not variables). Per-row computed variables (say a derived id) are computed in a `code Load` **without** a `Load` command.

### 9.6 descriptionAttribute, links by variable, and the multi-tenant rule

**`descriptionAttribute`** — the `<descriptionAttribute name="X" />` node at `level` level establishes that the Selection's `X` column **links to the View**. The `autolink` property is not needed on that column. The `autolink` property is old and using it is **not recommended** (especially in **multi-tenant** projects): instead of auto-linking, declare the links explicitly with `descriptionAttribute` (a column → its own View) or with `<link>` (below).

**A link through `<link>` on a column/variable** — both an `<attribute>` and a `<variable>` of the grid accept a `<link>` subnode generating a **PXInstance**-format link to **another** record's View (or to another instance's):

```xml
<variable name="RelatedOrderId" description="Related order" basedOn="OrderId" readOnly="True">
  <link instanceObject="PXWorkWithOrders, MyApp" instanceLevel="Orders" instanceLevelNode="View"
        condition="not &amp;RelatedOrderId.IsEmpty()">
    <parameters>
      <parameter name="CustomerId" />
      <parameter name="&amp;RelatedOrderId" />
    </parameters>
  </link>
</variable>
```

**The multi-tenant rule — the tenant id never travels as a parameter (except in components).** In FrontEnd interfaces the tenant discriminator (here `TenantId`) is **not** received as a parameter (that prevents it from being forced through the URL). It is loaded through `&Context` (`PLoadContext.Call(&Context)`; the pattern injects it) using `&Context.SecurityUserTenantId`, and it is filtered in the `conditions`:

- **View**: its `parameters` carry only the subordinate keys (say `CustomerId, OrderId`), **not** `TenantId`; the filter is done with `<conditions><condition value="TenantId = &amp;Context.SecurityUserTenantId" /></conditions>`.
- **Sections (tabs)**: if they declare no `parameters`, they inherit the View's. Since Sections are **components** (not reachable through the URL), they **can** receive `TenantId` along with the rest of the key — that is how a child grid is filtered by the parent record's full PK.

**`evaluateCondition="Refresh"`** — an action whose visibility depends on a **filter variable** (not on a row attribute) must be evaluated in Refresh. E.g. showing an action only when the `&Status` filter has a certain value: `evaluateCondition="Refresh" condition="&amp;Status = OrderStatus.Pending"`.

### 9.7 `Refresh` vs `RefreshForm`: data code vs WebForm code (the Excel export)

The **Excel export** procedure (`Ex{Name}`, §8.1) reuses the Selection's **data logic**: it walks the same base table with the same filters/orders. For that the generator **includes the `code Refresh` inside `Ex{Name}`**, but it does **not** include `RefreshForm`, `events` or the `ControlEvent` code (all of that is exclusive to the WebForm).

**Rule:** any command touching a **form control's property** — `&Var.Enabled`, `&Var.Visible`, `Control.Visible`, `.Class`, focus, and so on — **must go in `RefreshForm`**, never in `Refresh`. Left in `Refresh`, it leaks into the `Ex{Name}` procedure (which **has no WebForm**) and the specification emits warnings about that object:

- `src0224` — `'Enabled' is a non-standard expression and support for non-standard expressions is enabled.`
- `spc0002` — `&Var does not have the 'Enabled' property.`

(They are read in the `Ex{Name}` object's navigation — see kbbridge → `genexus-navigation.md`.)

| In `Refresh` (data — it also runs in `Ex{Name}`) | In `RefreshForm` (the WebForm only) |
|---|---|
| Filter variables / date ranges (`RetDatesFromPeriod.Call(&Period, &From, &To)`) | `&From.Enabled = ...` / `&To.Enabled = ...` |
| Counters, totals, data flags used in `conditions` | `.Visible`, `.Class`, enabling/disabling controls |
| Any logic the export **also** needs in order to filter | Anything that **only** makes sense with the form present |

**Example — a Period/From/To filter split properly:**

```xml
<!-- Data: it computes the range; the export needs it for its Where -->
<code type="Refresh"><![CDATA[If &Period <> Period.Custom
	RetDatesFromPeriod.Call(&Period, &From, &To)
EndIf]]></code>

<!-- WebForm: it only enables/disables the form's controls -->
<code type="RefreshForm"><![CDATA[&From.Enabled = &Period = Period.Custom
&To.Enabled = &Period = Period.Custom]]></code>
```

> **`ControlEvent`** code (say `&Period.Click`) **can** use `.Enabled`/`.Visible`: the events exist only in the WebForm and are not included in `Ex{Name}`. The leak happens **only** when the WebForm command is left in `Refresh`.

**Checklist** (a PXWorkWith with the Excel export enabled): if a line touches a control property (`.Enabled`, `.Visible`, `.Class`, focus) or anything belonging to the form → `RefreshForm`; if it computes data/variables the listing filters or shows → `Refresh`.

---

## 10. Dual-platform capability

PXWorkWith generates objects for two web platforms at once:

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

### Control properties per level

Each `level` node has three independent properties controlling generation:

| Property | Values | Description |
|----------|--------|-------------|
| `generateWeb` | `<default>`, `True`, `False` | Generate the Web Desktop version |
| `generateWebResponsive` | `<default>`, `True`, `False` | Generate the Web Responsive version |
| `generateSD` | `<default>`, `True`, `False` | Generate the Smart Devices version |

The `<default>` value inherits the global PXWorkWithSettings configuration.

### The prefix convention per platform

| Platform | Transaction prefix | Selection prefix | View prefix | Prompt prefix | Controller prefix |
|----------|--------------------|------------------|-------------|---------------|-------------------|
| Desktop | `D` | `WW` | `View` | `Pr` | `Ct` |
| Responsive | `R` | `RWW` | `RView` | `RPr` | `RCt` |

---

## 11. Relationship with the other patterns

PXWorkWith is PXTools' central pattern. It relates to the other patterns like this:

```
+------------------+
|   PXWorkWith     |<-------- The main CRUD pattern
+------------------+
        |
        | generates a Transaction --> Associated with a GX Transaction
        |
        | linkType="PXInstance" ----> Can invoke other PXWorkWith instances
        |
        | security.object ----------> Integrates with the PXSecurity module
        |
        | masterPage, theme --------> References shared UI objects
        |
        | externalComponent --------> Embeds WebComponents from other patterns
        |                             or developed manually
        |
        | modes.exportExcel --------> Generates export Procedures
        | modes.chart --------------> Generates chart Procedures
        |
        +-- controller -------------> Controls navigation between
             |                        Selection <-> View <-> Transaction
             |
             +-- afterTrn ----------> Defines the post-transaction flow
```

### The key interaction: actions with linkType="PXInstance"

When an action has `linkType="PXInstance"`, the pattern automatically resolves the GeneXus object generated by the target PXTools instance. That makes it possible to build navigation between several PXWorkWith instances without hardcoding object names.

---

## 12. Naming conventions

### 12.1 Prefixes per object type

| Prefix | Object type | Example |
|--------|-------------|---------|
| `D` | Transaction Desktop | `DCustomer` |
| `R` | Transaction Responsive | `RCustomer` |
| `WW` | Selection Desktop | `WWCustomer` |
| `RWW` | Selection Responsive | `RWWCustomer` |
| `View` | View Desktop | `ViewCustomer` |
| `RView` | View Responsive | `RViewCustomer` |
| `Pr` | Prompt Desktop | `PrCustomer` |
| `RPr` | Prompt Responsive | `RPrCustomer` |
| `Ct` | Controller Desktop | `CtCustomer` |
| `RCt` | Controller Responsive | `RCtCustomer` |
| `Ex` | Export/Chart Procedure | `ExCustomer` |
| `Upd` | Update Grid Rows Procedure | `UpdCustomerGridRows` |
| `PXWW` | The rows SDT | `PXWWCustomerRows` |
| `Chk` | Chosen validation | `ChkCustomerChosenValueSelected` |
| `Ret` | Return Chosen values | `RetCustomerChosenValues` |

### 12.2 Common suffixes

| Suffix | Use | Example |
|--------|-----|---------|
| `Rows` | The row-collection SDT | `PXWWCustomerRows` |
| `ExcelSDT` | The SDT for the Excel export | `ExCustomerExcelSDT` |
| `GridRows` | The bulk-update Procedure | `UpdCustomerGridRows` |
| `ChosenValueSelected` | Chosen validation | `ChkCustomerChosenValueSelected` |
| `ChosenValues` | The DataProvider of the Chosen values | `RetCustomerChosenValues` |
| `ChosenResults` | The Procedure of the Chosen results | `RetCustomerChosenResults` |

### 12.3 The general rule

The base name is always `{Instance.Name}`, which typically matches the name of the associated Transaction. The prefixes indicate the object type and the platform. The suffixes indicate the specific function.

```
[Platform prefix][Function prefix]{Instance.Name}[Suffix]

Examples:
  R    +  WW   + Customer +        = RWWCustomer       (Selection Responsive)
       +  View + Customer +        = ViewCustomer      (View Desktop)
  R    +  Pr   + Customer +        = RPrCustomer       (Prompt Responsive)
       +  Ex   + Customer + ExcelSDT = ExCustomerExcelSDT (the Excel export SDT)
       +  PXWW + Customer + Rows   = PXWWCustomerRows  (the rows SDT)
```

---

## A quick XPath reference

A reference table for locating elements inside the XML instance:

| Element | XPath |
|---------|-------|
| Transaction | `//transaction` |
| Level | `instance/level` |
| Selection | `instance/level/selection` |
| View | `instance/level/view` |
| Prompt | `instance/level/prompt` |
| Instance controller | `instance/controller` |
| Tab Tabular | `instance/level/view/sections//section[@type='Tabular']` |
| Tab Grid | `instance/level/view/sections//section[@type='Grid']` |
| Modes | `instance/level/selection/modes` |
| Chosen control | `instance//variable/controlInfo[@controlType='Chosen']` |
| Level Controller | `instance/level[../transaction[@afterTrn='Call Levels Controllers']]` |
