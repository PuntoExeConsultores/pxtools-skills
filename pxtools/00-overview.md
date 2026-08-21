# PXTools — Framework Overview

## What PXTools is

PXTools is a **pattern** framework for [GeneXus](https://www.genexus.com/) that turns declarative XML definitions into complete, working GeneXus code. It was created by **PuntoExe Consultores** (Uruguay) and supports these platforms:

- **Web Desktop** (classic HTML layout)
- **Web Responsive** (GeneXus abstract layout)
- **Smart Devices** (native mobile applications)

## Architecture of the Pattern System

### The three levels of a pattern

```
┌──────────────────────────────────────────────────────────────┐
│  LEVEL 1: PATTERN DEFINITION (.Pattern)                      │
│  ─────────────────────────────────────────                   │
│  XML schema defining:                                        │
│  • Node structure and allowed properties                     │
│  • Which GeneXus objects each node generates                 │
│  • Generator code (compiled DLLs)                            │
│  • Parent Objects (what kind of object it attaches to)       │
│  Location: Patterns/<PatternName>/<PatternName>.Pattern      │
├──────────────────────────────────────────────────────────────┤
│  LEVEL 2: PATTERN INSTANCE (.gxPattern)                      │
│  ─────────────────────────────────────────                   │
│  XML that follows the .Pattern schema                        │
│  Configured by the developer for one entity/feature          │
│  Location: Knowledge Base/<path>/<Name>.gxPattern            │
├──────────────────────────────────────────────────────────────┤
│  LEVEL 3: GENERATED OBJECTS (.Childs/)                       │
│  ─────────────────────────────────────                       │
│  The GeneXus objects the generator produces                  │
│  WebPanels, Procedures, DataProviders, SDTs, APIs, etc.      │
│  Location: Knowledge Base/<path>/.<Name>.gxSource.Childs/    │
└──────────────────────────────────────────────────────────────┘
```

### Supporting files per pattern

Every pattern has additional files in its definition directory:

| File | Purpose |
|------|---------|
| `<Pattern>Instance.xml` | Instance Specification: the complete instance schema (ElementTypes, attributes, child nodes, types, default values) |
| `<Pattern>Settings.xml` | Global pattern preferences/configuration (defaults, prefixes, theme classes, etc.) |
| `<Pattern>CustomTypes.xml` | Definitions of the custom types used in properties |
| `Templates/` | Generator code (compiled DLLs) producing each part of the GeneXus objects. Compiled to protect intellectual property and licensing |
| `Icons/` | Pattern icons for the GeneXus IDE |
| `Resources/` | Additional resources (default settings, initial exports) |

### Anatomy of a .Pattern

Every `.Pattern` file has this XML structure:

```xml
<Pattern Publisher="PuntoExe" Id="GUID" Name="PXWorkWith" Version="10.3.0">
  <Definition>
    <InstanceName>PXWorkWith{0}</InstanceName>              <!-- Naming convention -->
    <InstanceSpecification>PXWorkWithInstance.xml</InstanceSpecification>
    <SettingsSpecification>PXWorkWithSettings.xml</SettingsSpecification>
    <CustomTypeDefinitions>PXWorkWithCustomTypes.xml</CustomTypeDefinitions>
    <Implementation>PuntoExe.Patterns.PXWorkWith.dll</Implementation>
    <ParentObjects>                                         <!-- What it attaches to -->
      <ParentObject Type="Transaction" />
      <ParentObject Type="(None)" />
    </ParentObjects>
  </Definition>

  <Objects>
    <!-- Each Object declares one GeneXus object to generate -->
    <Object Type="WebPanel" Id="Selection" Name="WW{Instance.Name}"
            Element="instance/level/selection">             <!-- Source XML node -->
      <Part Type="WebForm" Template="Templates\SelectionWebForm.dll" />
      <Part Type="Variables" Template="Templates\SelectionVariables.dll" />
      <Part Type="Events" Template="Templates\SelectionEvents.dll" />
      <Part Type="Rules" Template="Templates\SelectionRules.dll" />
    </Object>
    <!-- ... more objects ... -->
  </Objects>
</Pattern>
```

Key points:
- **`Element`** says which XML node of the instance triggers the generation of that object
- **`Name`** uses placeholders such as `{Instance.Name}`, `{Element.name}` for dynamic naming
- **`Count="*"`** means it can generate several objects (one per Element XPath match)
- **`Template`** (inside the .Pattern) points at the **generator code** (DLL) producing each part of the GeneXus object. Not to be confused with the PXTools "UI Templates" (see the next section)

## Pattern Catalogue

### UI patterns (they generate WebPanels)

| Pattern | Generates | Parent Object | Dual-platform |
|---------|-----------|---------------|---------------|
| **PXWorkWith** | Selection, View, Edit, Prompt, Controller, Tabs (Grid/Tabular), Excel export, Charts | Transaction / (None) | Yes (Web + Responsive + SD) |
| **PXParameterRequest** | Modal/popup WebPanel for capturing parameters | WebPanel / Procedure / Report / Transaction / (None) | Yes (Web + Responsive) |
| **PXComposer** | Composer WebPanel embedding WebComponents | (None) | Yes (Web + Responsive) |

### Flow pattern

| Pattern | Generates | Parent Object | Dual-platform |
|---------|-----------|---------------|---------------|
| **PXFlowController** | WebPanel with a step-by-step workflow: actions, confirmations, iterations | Procedure / Transaction / (None) | Yes (Web + Responsive + SD) |

### API / Web Service patterns

| Pattern | Generates | Parent Object |
|---------|-----------|---------------|
| **PXWSLayer** | SOAP Procedures, REST Procedures, API objects (OpenAPI 3.0), In/Out SDTs | Transaction / (None) |
| **PXWSQuery** | DataProvider + Procedure + SDTs + Domain (ordering), with filters, search and paging | Transaction / (None) |
| **PXWSData** | Read Procedure + SDTs (In/Out/Structure), with code hooks | Transaction / (None) |
| **PXWSTransaction** | Load/Save/Delete Procedures + SDTs (Structure/In/Out per method) via Business Components | Transaction / (None) |

### Data / configuration patterns

| Pattern | Generates | Parent Object |
|---------|-----------|---------------|
| **PXOAV** | Attributes, Transactions (WRI/WORI/Definition), Groups, Procedures (value CRUD) | Transaction |
| **PXEntityParameters** | Attributes, Domains, Transactions, Groups, SDTs, Procedures, DataProviders for per-entity configurable parameters | Transaction |
| **PXReportTemplate** | Generates no objects directly; defines template/settings for reports | Procedure |

## How the patterns relate

```
                    ┌──────────────────┐
                    │   PXWSLayer      │ ◄── Orchestrates the REST/SOAP API
                    │   (API Object)   │
                    └────────┬─────────┘
                             │ methods point to:
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ PXWSQuery  │  │ PXWSData   │  │PXWSTransac.│
     │(DataProv.) │  │(Procedure) │  │(Load/Save/ │
     └────────────┘  └────────────┘  │ Delete)    │
                                     └────────────┘

     ┌───────────────────────────────────────────┐
     │              PXComposer                   │
     │  (screen composer)                        │
     │  Embeds as WebComponents:                 │
     │  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
     │  │PXWorkWith│ │PXParam.  │ │PXComposer│   │
     │  │Selection │ │Request   │ │(nested)  │   │
     │  └──────────┘ └──────────┘ └──────────┘   │
     └───────────────────────────────────────────┘

     ┌────────────────────────────────────────────┐
     │          PXFlowController                  │
     │  Actions can invoke:                       │
     │  • PXWorkWith (any node)                   │
     │  • PXParameterRequest                      │
     │  • PXComposer                              │
     │  • PXOAV                                   │
     │  • Other PXFlowControllers                 │
     │  • GeneXus objects directly                │
     └────────────────────────────────────────────┘
```

## Communication between components: GlobalEvents

When several WebComponents are **displayed on the same screen at the same time**, they talk to each other through GeneXus' native **`GlobalEvents` External Object**. That object, imported by default into every GeneXus KB, allows you to:

1. **Define global events** that any panel can raise
2. **Listen for events** in other panels currently rendered, and react (refresh, update data, etc.)

**Applies to**: **PXComposer** components (all visible on screen simultaneously).

**Does not apply to**: **PXWorkWith View** tabs, since only the active tab is loaded/rendered — the other tabs are not listening at that moment. To communicate between tabs use **WebSession**: one tab writes data into WebSession and, when another tab becomes active, it reads that data in its Start/Refresh.

`GlobalEvents` is wired up in the **code hooks** (`codes` and `events` nodes) of the pattern instances, not as a declarative property of the pattern.

## `&WindowSelf`: which kind of window the screen is running in

Every screen PXTools generates receives a `&WindowSelf` parameter, and it is not bookkeeping: it
**declares the kind of window the screen is running in**, and two things are decided from it.

1. **The `RecentLink` history**, which is kept **separately per window kind**. A modal browsing its own
   history does not disturb the history of the main area behind it.
2. **The JavaScript PXTools injects into the HTML** so the window can **size itself**. This is the part
   that bites: hand a screen the wrong value and nothing fails — no error, no warning — the window
   simply comes out the wrong size, and the cause looks nothing like the symptom.

| Value | The window it means |
|---|---|
| *(empty)* | the main area |
| `MD`, `MD1`, `MD2`… | a **modal**, opened with GeneXus' `Popup()` method. The first one is `MD`, **not** `MD0`; a modal that opens another popup passes `MD1`, and so on |
| `PU`, `PU1`, `PU2`… | a **browser popup window**, the old way of opening a screen. Worse than a modal: it does not hold focus well and can end up lost behind the main window if the user clicks there. The first one is `PU`, not `PU0` |
| `EM`, `EM1`… | a screen opened through an **Embedded Page** control, which ends up as an `iFrame` in the HTML |

**The rule is to pass the value that matches how the screen is really being opened**, and to increment
the number when a window of a kind opens another of the same kind. Calling `SomeScreen.Popup()` while
passing an empty `&WindowSelf` says "I am the main area" to a screen that is in fact a modal: the
auto-sizing scripts never get injected, and the modal is left at whatever size it happens to get.

## Code hooks: CDATA formatting and indentation (common to all patterns)

`codes` nodes exist in **PXWorkWith, PXParameterRequest and PXComposer**. In the `.gxPattern`, each hook is serialized with the code inside a CDATA and the **first line flush against the opening** `<![CDATA[`:

```xml
<code type="Start"><![CDATA[&Total = 0
For Each
	Where OrderId = &OrderId
	&Total = &Total + OrderAmount
EndFor]]></code>
```

**Indentation rule** (applies to every pattern with CDATA code nodes):

- The **first line** stays at **column 0** (flush against `<![CDATA[`), and the **following lines also start at column 0** for the base level. A frequent mistake is leaving **one extra tab** from line 2 onwards — it shifts the whole block and leaves `For Each`/`EndFor` (or `If`/`EndIf`) aligned with their own content.
- Only **block nesting** (`For Each`, `If`, `Do While`, `Do Case`, `Sub`) adds **+1 tab**; the closing keywords (`EndFor`, `EndIf`, `EndDo`, `EndCase`, `EndSub`) line up with their opening.
- The **internal alignment** of the `=` signs (tabs mid-line, after the attribute name) is independent of the leading indentation and is preserved.

> **Exception — PXFlowController**: it serializes code as a `data="…"` attribute (not CDATA), so this indentation guidance does not apply there.

## UI Templates — full control over the visual design

### Key concept

The PXTools UI patterns (PXWorkWith, PXParameterRequest, PXComposer, PXFlowController) do not impose a fixed visual design. The design is defined through **Templates**: GeneXus objects the developer creates and customizes freely.

### What a Template is

A Template is a **real GeneXus object** defining the complete visual design of the screen:

| Generator | Template type | Description |
|-----------|---------------|-------------|
| Web Desktop | WebPanel with HTML layout | The developer designs the HTML freely |
| Web Responsive | WebPanel with Abstract Layout | The developer uses the GeneXus abstract layout |
| Smart Devices | Panel for SD | The developer designs the mobile panel |

### Template Elements

Inside the Template the developer places **Template Elements**: simple placeholders where the PXTools generator injects the content specific to each instance (grid, filters, action buttons, form, and so on).

```
┌──────────────────────────────────────────────────────┐
│  TEMPLATE (WebPanel designed by the developer)       │
│                                                      │
│  ┌─ Company logo ─┐  ┌─ Custom breadcrumb ─┐        │
│  └────────────────┘  └─────────────────────┘        │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │  ██ TEMPLATE ELEMENT: Filters ██         │ ◄─ PXTools injects here
│  └──────────────────────────────────────────┘        │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │  ██ TEMPLATE ELEMENT: Grid ██            │ ◄─ PXTools injects here
│  └──────────────────────────────────────────┘        │
│                                                      │
│  ┌─ Custom footer ──────────────────────────┐        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  Everything else (logo, breadcrumb, footer, CSS,     │
│  JavaScript, extra controls) is 100% the             │
│  developer's, with all the flexibility of GeneXus    │
└──────────────────────────────────────────────────────┘
```

### Template Groups

Templates are organised into **Template Groups** (`templatesGroup`) that bundle the templates for each kind of screen a pattern generates. This lets you:

- Keep a "Standard Design" group and a "Minimalist Design" group
- Apply a whole group to one instance or to the entire KB through Settings
- Have each level of an instance use a different Template Group, or an individual Template (`templateObject`, `templateResponsiveObject`, `templateSDObject`)

### Why this matters

Templates are the **single most important capability for shaping the visual design**:

1. The generator produces the logic (variables, events, rules, conditions)
2. The Template defines the complete visual design
3. Template Elements are the insertion points for the generated content
4. The developer has **the full freedom of GeneXus** around the Template Elements: HTML, CSS, JavaScript, third-party controls, images, animations

This means two KBs using the same PXWorkWith pattern can end up with visually completely different screens if they use different Templates.

### Catalogue of Template Elements

The Template Elements available for PXWorkWith Selection (taken from real Templates):

| Template Element (Type) | What the generator injects |
|-------------------------|----------------------------|
| `Grid` | The main data grid |
| `Filters` | Advanced filter panel |
| `UniqueSearchField` | Quick search field |
| `UniqueSearchAction` | Search button |
| `InsertAction` | Insert button |
| `UpdateAction` | Update button |
| `DeleteAction` | Delete button |
| `DisplayAction` | View detail button |
| `EditAction` | Edit button |
| `ImageActions` | Image actions |
| `ButtonActions` | Button actions |
| `StandardButtonActions` | Standard actions (OK, Cancel) |
| `OrderSelector` | Order selector |
| `InsertVariables` | Insert variables |
| `GridPagingStatus` | Paging status |
| `PageJump` | Page jump |
| `PageRowChange` | Rows-per-page change |
| `TopGridFixedData` | Fixed data above the grid |
| `BottomGridFixedData` | Fixed data below the grid |
| `FixedDataSection` | Fixed data section (with subtype: Parameters, Top Grid, Bottom Grid) |
| `HiddenElements` | Functional hidden elements |
| `GridHandlerControl` | Grid handler control |
| `GridHandlerAction` | Grid handler action |
| `AddAllAction` | Select-all action |
| `RemoveAllAction` | Deselect-all action |
| `Search` | Search button |
| `ErrorViewer` | Error viewer |
| `ProgramName` | Program name |

### Templates shipped with PXTools

PXTools includes a collection of predefined Templates in `@PXTools/@APIs/Personalized/Templates/`:

**For PXWorkWith Selection (Desktop):**
- `PXToolsSelectionTemplate` — standard layout
- `PXToolsSelectionGXUITemplate` — with GXUI controls
- `PXToolsSelectionTwoPaneTemplate` — two panes
- `PXToolsSelectionButtonsWithImagesTemplate` — buttons with images
- `PXToolsSelectionComponentLeftRightTemplate` — left/right component
- `PXToolsSelectionComponentRightTemplate` — component on the right
- And more layout variants…

**For PXWorkWith Selection (Responsive):**
- `PXToolsResponsiveSelectionTemplate` — standard responsive layout
- `PXToolsResponsiveSelectionTwoPaneTemplate` — responsive two panes
- `PXToolsResponsiveSelectionExpandComponentTemplate` — with expandable panel

**For other nodes:**
- `PXToolsViewTemplate` / `PXToolsResponsiveViewTemplate` — View
- `PXToolsTransactionTemplate` / `PXToolsResponsiveTransactionTemplate` — Transaction
- `PXToolsTabGridTemplate` / `PXToolsTabGridGXUITemplate` — Tab Grid
- `PXToolsTabTabularTemplate` / `PXToolsResponsiveTabTabularTemplate` — Tab Tabular
- `PXToolsComposerTemplate` / `PXToolsResponsiveComposerTemplate` — Composer
- `PXToolsParameterRequestTemplate` / `PXToolsResponsiveParameterRequestTemplate` — PR
- `PXToolsParameterRequestLoginTemplate` — Login
- `PXToolsSDGridTemplate` and variants — Smart Devices

### Predefined visual styles

PXTools offers visual design variants by colour (Red, Blue, Green, Gray, Violet and others). These variants are applied through the GeneXus Theme and define colour palettes, typography and styles for every element of the generated screens.

## Dual-Platform Generation

The UI patterns (PXWorkWith, PXParameterRequest, PXComposer) and PXFlowController generate **two versions** of each WebPanel:

| Version | Name prefix | Property |
|---------|-------------|----------|
| Web Desktop | `WW`, `Wb`, `Pr`, `Ct`, `View` | `generateWeb` |
| Web Responsive | `RWW`, `RWb`, `RPr`, `RCt`, `RView` | `generateWebResponsive` |
| Smart Devices | (varies) | `generateSD` |

This enables:
- Progressive Desktop → Responsive migration
- Both platforms coexisting at once
- The same instance definition generating both versions

## @PXTools Modules

PXTools ships 25+ reusable modules providing cross-cutting functionality. Each module is a package of GeneXus objects (Procedures, WebPanels, SDTs, DataProviders) installed under `Knowledge Base/@PXTools/@<ModuleName>/`. The modules include their own pattern instances (PXWorkWith for CRUD screens, PXParameterRequest for forms, PXComposer for composed views).

Available modules: @APIs, @Alerts, @CloudTasks, @ControlPreferences, @DynamicCallReferences, @ExportImport, @FileStorage, @MailAccounts, @Menus, @OAV, @ProcessMonitor, @Projects, @ReceiveMails, @Security, @SecurityProjects, @SendMails, @Statistics, @System, @SystemParameters, @TableCleaner, @TaskManager, @WSLayer, @WebServicesLog.

> **Not PXTools modules** (they appear under `@PXTools/` only as glue): `@ResponsiveLayout` and `@SmartMenus` are **GeneXus platform** modules contributing supporting SDTs/APIs to the PXTools User Controls.

See the detail in [20-pxtools-modules.md](20-pxtools-modules.md).

## Naming conventions

### Objects generated by PXWorkWith

| Generated object | Naming | Example (instance "Customer") |
|------------------|--------|-------------------------------|
| Selection (Desktop) | `WW{Name}` | `WWCustomer` |
| Selection (Responsive) | `RWW{Name}` | `RWWCustomer` |
| View (Desktop) | `View{Name}` | `ViewCustomer` |
| View (Responsive) | `RView{Name}` | `RViewCustomer` |
| Prompt (Desktop) | `Pr{Name}` | `PrCustomer` |
| Prompt (Responsive) | `RPr{Name}` | `RPrCustomer` |
| Controller (Desktop) | `Ct{Name}` | `CtCustomer` |
| Controller (Responsive) | `RCt{Name}` | `RCtCustomer` |
| Tab Grid Component | `{wcname}` | (defined in the instance) |
| Tab Tabular Component | `{wcname}` | (defined in the instance) |
| Excel export | `Ex{Name}` | `ExCustomer` |
| Selected Rows SDT | `PXWW{Name}Rows` | `PXWWCustomerRows` |
| Grid Handler DP | `PXWW{Name}Rows` | `PXWWCustomerRows` |
| Transaction (Desktop) | `D{Name}` | `DCustomer` |
| Transaction (Responsive) | `R{Name}` | `RCustomer` |

### Objects generated by the WS patterns

| Pattern | Object | Naming | Example (Trn "Customer", V1) |
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

## Do not declare a property that already carries its default

An instance should contain **decisions**, and nothing else. A property whose value equals the
pattern's default changes nothing when it is written and changes nothing when it is removed — it only
makes the instance longer, and the longer it gets the harder it is to see what was actually decided.
When every filter carries `type="Equal" isRequired="False" allowNullValue="False"`, the one filter that
is genuinely required disappears into the noise, and so does the real content of any diff.

**Where the defaults are.** Each pattern ships its own schema in
`Patterns/<Pattern>/<Pattern>Instance.xml`, and every property declares its own:

```xml
<Attribute Name="ascending" Type="bool" ... DefaultValue="true" />
```

To list them for an element before writing an instance:

```bash
python -c "
import io,re
s=io.open('Patterns/PXWSQuery/PXWSQueryInstance.xml',encoding='utf-8',errors='replace').read()
i=s.find('ElementType Name=\"OrderAttribute\"'); j=s.find('</ElementType>',i)
for m in re.finditer(r'<Attribute Name=\"([^\"]+)\"[^>]*?DefaultValue=\"([^\"]*)\"',s[i:j]):
    print(m.group(1),'->',repr(m.group(2)))
"
```

**It is not only tidiness.** A property written with its default value can be **the thing that breaks
the apply**. Real case in PXWSQuery: `<orderAttribute ascending="True" />` — the default is already
`true`, so the line says nothing — makes the pattern application fail with

```
'System.InvalidCastException'
Unable to cast object of type 'Artech.Common.Diagnostics.GxException'
                      to type 'Artech.Genexus.Common.CustomTypes.EnumValues'
```

while `ascending="False"`, the value that *does* change the behaviour, works and is used in production.
Time spent hunting that is time spent on a line that should never have been written.

> **Reading the other way round:** when an existing instance declares something, assume it was
> deliberate and check the default before removing it. The value that differs from the default is the
> author telling you something.

## Method: check the instance against the generated object

The objects a pattern generates **are written to disk** (externalized KB) in a hidden folder next to the instance:

```
@Module/.../.{Instance}.{Pattern}.gxPattern.Childs/
    TrCustomer.WebPanel.gxSource    <- generated Desktop Selection
    RTrCustomer.WebPanel.gxSource   <- Responsive Selection
    ...
```

When something behaves unexpectedly (an invisible/disabled control, an event that does not fire, data that never arrives), the most direct route is to **read the generated object** and find the code that produces it, rather than guessing at the instance:

- `grep` the generated `.gxSource` for `.Visible`, `.Enabled`, `PIsAuthorized`, `Event '<Action>'`, the control name (`btn<Action>`), or the variable/attribute involved.
- Compare what was generated against the intent of the instance and locate the **property** (or the missing authorization, or the pattern rule) causing the difference.
- Fix it in the **instance** (or in configuration, e.g. security), **never** in the generated object — it gets regenerated and the change is lost.

Every property discovered this way gets **documented** (in the pattern's own doc or in `12-pattern-ui-actions.md`) so that, over time, reading the generated code is **no longer necessary** to write the instance correctly. Examples already catalogued: visibility gating by `PIsAuthorized` on navigating actions (`12-pattern-ui-actions.md` §10); the `RefreshForm` hook not being included in the Excel export (`01-pxworkwith.md` §9.7).
