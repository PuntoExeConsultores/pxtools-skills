# Actions in the UI Patterns — Cross-Cutting Reference

## Applies to

This document describes the **action system** shared by the three PXTools UI patterns:

- **PXWorkWith** — actions in Selection, View, Prompt, Tabs
- **PXParameterRequest** — actions in the form (Accept, Cancel, custom)
- **PXComposer** — actions in the composed panel

The `action` node's structure is **practically identical** in all three patterns. The specific differences are documented where they apply.

---

## 1. XML structure of the Action node

```xml
<action name="MyAction"
        caption="Visible text"
        tooltip="Optional tooltip"
        image="ButtonImage"
        type="Standard|Custom"
        position="Top|Bottom|Both|Grid"

        conditionPreviousCode="// Code run before evaluating the condition"
        condition="&IsValid = true"
        evaluateCondition="Event|Load"

        actionPreviousCode="// Code run BEFORE the main invocation"

        callType="Link|Call|Prompt|External Link|Client Text Print|Subroutine|Event|Submit|None|Return"
        linkType="GXObject|PXInstance"
        gxObject="ObjectName"
        target="Self|New"
        popupWidth="400"
        popupHeight="300"

        actionPostCode="// Code run AFTER the main invocation"
        refreshAction="..."

        confirm="Are you sure?"
        closeWindowControl="..."
        closeWindowControlCondition="..."
        hasPostCode="true|false">

  <!-- Parameters of the invoked object -->
  <parameters>
    <parameter name="Param1" />
    <parameter name="Param2" />
  </parameters>

  <!-- Security -->
  <security object="MyObject" operation="Execute" />

  <!-- Conditional invocations (see the ConditionalCalls section) -->
  <conditionalCalls>
    <conditionalCall condition="..." gxObject="..." />
  </conditionalCalls>
</action>
```

---

## 2. Action properties

### Identity and presentation

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | The action's unique identifier |
| `caption` | string | The text shown on the button/link |
| `tooltip` | string | Tooltip on hover |
| `image` | reference | The button's image |
| `type` | enum | `Standard` (Accept/Cancel) or `Custom` |
| `position` | enum | `Top`, `Bottom`, `Both`, `Grid` (inside each row) |

### Visibility/execution condition

| Property | Type | Description |
|----------|------|-------------|
| `conditionPreviousCode` | code | Procedural code run to prepare the condition's evaluation. It lets you compute auxiliary variables the condition needs |
| `condition` | expression | A GeneXus expression determining whether the action is shown/executed. If it evaluates to `false`, the action does not fire |
| `evaluateCondition` | enum | `Event`: evaluates the condition on click. `Load`: evaluates it for every grid row (allowing per-row conditional visibility) |

### Pre-invocation code

| Property | Type | Description |
|----------|------|-------------|
| `actionPreviousCode` | code | Procedural code run **before** the main invocation. Typical use: extra validations, preparing data, prior computations |

### The main invocation

| Property | Type | Description |
|----------|------|-------------|
| `callType` | enum | The kind of invocation (see the detailed table below) |
| `linkType` | enum | Only for `callType="Link"`: `GXObject` or `PXInstance` |
| `gxObject` | reference | The target GeneXus object (for GXObject) |
| `target` | enum | `Self` = the same window, `New` = a popup/new window |
| `popupWidth` | int | The popup's width (only when target=New) |
| `popupHeight` | int | The popup's height (only when target=New) |

### PXInstance properties (for callType Link + linkType PXInstance)

| Property | Type | Description |
|----------|------|-------------|
| `instanceObject` | reference | The name of the target pattern instance |
| `instanceLevel` | string | The level's name inside that instance |
| `instanceLevelNode` | enum | The node to invoke: `Selection`, `View`, `Prompt`, `Transaction`, `Level` |
| `instanceLevelViewSection` | string | (View only) A specific tab/section. Omitted, it goes to the main tab |

### Post-invocation code

| Property | Type | Description |
|----------|------|-------------|
| `actionPostCode` | code | Procedural code run **after** the main invocation. Typical use: updating states, storing into WebSession, confirmation messages |
| `refreshAction` | enum | How to refresh the screen/grid after the action |

### Other properties

| Property | Type | Description |
|----------|------|-------------|
| `confirm` | string/bool | A confirmation dialog before executing |
| `closeWindowControl` | string | The control closing the popup |
| `closeWindowControlCondition` | expression | The condition for accepting the close |
| `hasPostCode` | bool | Enables code that runs after the popup closes |
| `security` | subnode | Access control: `object` + `operation` |

---

## 3. callType kinds

| callType | Description | Typical use |
|----------|-------------|-------------|
| `Link` | Navigates to another WebPanel/Transaction | Opening a detail screen, an editor, another pattern |
| `Call` | Invokes a Procedure (no UI) | Processing, computing, updating records |
| `Prompt` | Opens a modal dialog (popup) | Opening a PXParameterRequest or a lookup |
| `External Link` | Opens a WebPanel/URL not generated by PXTools | Opening a hand-written WebPanel or an external URL |
| `Client Text Print` | Client-side printing | Printing a document |
| `Subroutine` | Runs a local subroutine | Reusable internal logic defined in `codes/Subroutine` |
| `Event` | Runs inline code (no main object) | When all the logic is procedural code with no external invocation |
| `Submit` | Submits a batch process | Running an asynchronous task |
| `None` | Does nothing | Only the pre/post code matters |
| `Return` | Returns/closes the popup | A Cancel or Exit button |

---

## 4. Critical rule: linkType and Responsive migration

**Only actions with `callType="Link"` need `linkType="PXInstance"`** in order to be platform-independent.

When generating for Responsive, the names of pattern-generated objects change prefix:

```
Desktop:    WWInvoice,  ViewInvoice,  PrInvoice,  TrInvoice
Responsive: RWWInvoice, RViewInvoice, RPrInvoice, RInvoice
```

If an action uses `linkType="GXObject"` with `gxObject="TrInvoice"`, enabling Responsive changes the name and the reference breaks.

With `linkType="PXInstance"`, the generator automatically resolves the right name per platform:

```xml
<!-- WRONG (it breaks when the platform changes) -->
<action callType="Link" linkType="GXObject" gxObject="TrInvoice" />

<!-- RIGHT (platform-independent) -->
<action callType="Link" linkType="PXInstance"
        instanceObject="PXWorkWithInvoice"
        instanceLevel="Invoice"
        instanceLevelNode="Transaction" />
```

**The other callTypes do NOT need PXInstance**, because they invoke Procedures, subroutines or inline code whose names do not change between platforms:

```xml
<!-- OK: Call invokes a Procedure (the name does not change) -->
<action callType="Call" gxObject="ProcessRecord" />

<!-- OK: Event runs inline code (it depends on no names) -->
<action callType="Event" />

<!-- OK: Subroutine is local to the object (it depends on no names) -->
<action callType="Subroutine" subroutine="ValidateData" />
```

---

## 5. ConditionalCalls — conditional invocation

ConditionalCalls lets a single action invoke **different objects/interfaces depending on a condition**, working as a declarative `Do Case` for navigation:

```xml
<action name="ViewDetail"
        caption="View detail"
        callType="Link">
  <conditionalCalls>
    <conditionalCall condition="InvoiceType = 'Domestic'"
                     gxObject="ViewDomesticInvoice"
                     linkType="PXInstance"
                     instanceObject="PXWorkWithDomesticInvoice"
                     instanceLevel="Invoice"
                     instanceLevelNode="View" />
    <conditionalCall condition="InvoiceType = 'Export'"
                     gxObject="ViewExportInvoice"
                     linkType="PXInstance"
                     instanceObject="PXWorkWithExportInvoice"
                     instanceLevel="Invoice"
                     instanceLevelNode="View" />
  </conditionalCalls>
</action>
```

### Behaviour

1. On clicking the action, each `conditionalCall`'s condition is evaluated in order
2. The first condition that is `true` determines the object/instance to invoke
3. If no condition is `true`, the parent action's `gxObject` is used (when there is one)

### Use cases

| Scenario | Example |
|----------|---------|
| A different detail screen depending on the record's type | Viewing a domestic vs an export invoice |
| A different edit form depending on the status | Editing a draft vs an approved record |
| A different flow depending on the user's role | Administrator flow vs operator flow |
| Different views depending on configuration | Detailed view vs summary view |

### ConditionalCalls and PXInstance

Each `conditionalCall` supports the same `linkType`/`instanceObject` properties as the parent action. For Responsive migration, every conditionalCall must use `linkType="PXInstance"` when it points at a pattern-generated object.

---

## 5.bis Confirms — a confirmation dialog with dynamic text

The action's `confirm` property (§2) accepts **fixed text only**. When the question has to name the concrete record ("Delete customer X?"), use the **`confirms`** node, a sibling of `actions` within the same level.

```xml
<actions>
  <!-- The action does not execute: it invokes the confirm -->
  <action name="Delete" inGrid="True" callType="Subroutine" subroutine="ConfirmDelete" />

  <!-- The action doing the real work, fired by the affirmative response -->
  <action name="DoDelete" controlType="Event" callType="Call"
          gxObject="DelCustomer, Sales" refreshAction="Grid Refresh">
    <parameters><parameter name="CustomerId" /></parameters>
  </action>
</actions>

<confirms>
  <confirm name="ConfirmDelete" linkType="GXObject" gxObject="HPEXE_Confirm, PXTools.APIs"
           question="&quot;Delete customer &quot; + CustomerName.Trim() + &quot;?&quot;"
           questionType="Variable">
    <responses>
      <response responseValue="True" callType="Subroutine" subroutine="DoDelete" />
    </responses>
  </confirm>
</confirms>
```

| Property | Description |
|---|---|
| `name` | The confirm's name. The action invokes it with `callType="Subroutine" subroutine="<name>"` |
| `question` | The question's text or **expression** |
| `questionType` | `Variable` = `question` is evaluated as an expression. **Without it the text comes out literally** |
| `linkType` / `gxObject` | The dialog to use; the PXTools standard is `HPEXE_Confirm, PXTools.APIs` |
| `responses/response` | `responseValue` (`True`/`False`) + the invocation to run, exactly like an action |

> **The confirm is invoked as if it were a subroutine.** The visible action executes nothing: it only calls the confirm, and it is the **response** that fires the action doing the work. That is why *two* actions are needed: the one the user sees and the one doing the job.

### The typical case: replacing the transaction's delete

When the entity has dependents, the standard delete fails on referential integrity. The replacement has **three parts, and leaving any of them out makes it ineffective**:

1. **`<modes Delete="false" />`** — without this the pattern **keeps generating its own Delete action** against the transaction, and the fix achieves nothing (see [01-pxworkwith.md](01-pxworkwith.md) §4.1).
2. Your own `action` named `Delete` invoking the confirm.
3. A procedure deleting in cascade, from the leaf up to the root, invoked by the affirmative response.

---

## 6. The complete execution cycle

```
┌─────────────────────────────────────────────────────────────┐
│                   AN ACTION'S CYCLE                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. conditionPreviousCode                                   │
│     └─ Code preparing the condition's variables             │
│                                                             │
│  2. condition                                               │
│     └─ If false → the action does NOT run (end)             │
│                                                             │
│  3. confirm (when defined)                                  │
│     └─ Shows an "Are you sure?" dialog → Yes/No             │
│        If No → end                                          │
│                                                             │
│  4. actionPreviousCode                                      │
│     └─ Validations, preparing data                          │
│                                                             │
│  5. THE MAIN INVOCATION (per callType)                      │
│     ├─ Link → navigates to an object/instance               │
│     │   └─ ConditionalCalls: evaluates and picks a target   │
│     ├─ Call → invokes a Procedure                           │
│     ├─ Event → runs inline code                             │
│     ├─ Prompt → opens a modal popup                         │
│     ├─ Subroutine → runs a local sub                        │
│     ├─ Submit → submits a batch                             │
│     ├─ Return → closes/returns                              │
│     └─ None → does nothing                                  │
│                                                             │
│  6. actionPostCode                                          │
│     └─ Follow-up logic (states, WebSession, messages)       │
│                                                             │
│  7. refreshAction                                           │
│     └─ Refreshes the screen/grid where applicable           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Differences per pattern

### PXWorkWith

- Actions can live in `selection/actions`, `view/actions`, `prompt/actions` and in each `section/actions` (tabs)
- When `position="Grid"`, the action is shown on every row and the condition is evaluated in the Load
- It supports predefined standard actions: `Insert`, `Update`, `Delete`, `Display`, `ExportExcel`

### PXParameterRequest

- Actions live in `level/actions`
- It has the extra **`execute`** property controlling which validations are applied before running the action (see section 8)
- The standard types are `Confirm` (Accept) and `Cancel`
- The actions interact with the `calls` node to chain invocations on confirmation

### PXComposer

- Actions live in `level/actions`
- They work as in PXWorkWith but without a grid context (there is no per-row evaluation)
- They are typically global navigation actions of the composed panel

---

## 8. The Execute property in PXParameterRequest

PXParameterRequest has an exclusive **`execute`** property on its actions, controlling **which validations apply** before running the action. It exists because PXParameterRequest's main job is **data entry**, with validations that must run before the data is processed.

### Context: the PXParameterRequest's Conditions node

The PXParameterRequest's `conditions` node defines **general validation conditions** shared by several actions:

```xml
<conditions>
  <condition>not &DateFrom.IsEmpty()</condition>
  <condition>not &DateTo.IsEmpty()</condition>
  <condition>&DateFrom <= &DateTo</condition>
</conditions>
```

These conditions represent **rules about mandatory data** that apply regardless of which action fires (Cancel aside).

### execute values

| Value | Behaviour |
|-------|-----------|
| **General Conditions** | Runs the PXParameterRequest's general `conditions` **and** the action's local conditions (`condition`). Use it for actions requiring every mandatory field to be complete (Accept, Confirm, Generate) |
| **Action Conditions Only** | Runs **only** the action's local conditions (`condition`, `conditionPreviousCode`), ignoring the general `conditions`. Use it for actions that do not require every field validated (a partial Preview, Search, auxiliary actions) |

### Example

```xml
<!-- The form's general conditions (mandatory data) -->
<conditions>
  <condition>not &DateFrom.IsEmpty()</condition>
  <condition>not &CustomerId.IsEmpty()</condition>
</conditions>

<actions>
  <!-- Accept: validates the general conditions + the local ones -->
  <action name="Confirm" caption="Accept"
          execute="General Conditions"
          callType="Call" gxObject="GenerateReport" />

  <!-- Preview: validates its own condition only -->
  <action name="Preview" caption="Preview"
          execute="Action Conditions Only"
          condition="&ReportType = 'PDF'"
          callType="Link" gxObject="PreviewReport" />

  <!-- Cancel: validates nothing -->
  <action name="Cancel" caption="Cancel"
          callType="Return" />
</actions>
```

In this example:
- **Accept** requires DateFrom and CustomerId to be filled in (the general conditions)
- **Preview** only checks that the report type is PDF (its local condition), regardless of the other fields
- **Cancel** runs no validation at all

---

## 9. Mapping hand-written code to Action properties

A guide for migrating hand-written WebPanel events into pattern actions:

| Code in a hand-written WebPanel | Action property |
|---------------------------------|-----------------|
| Validations at the top of the Event (If field.IsEmpty(), Msg, Return) | `conditionPreviousCode` + `condition`, or `execute="General Conditions"` with the `conditions` node |
| `Do 'MySubroutine'` before the invocation | `actionPreviousCode` |
| `MyProcedure.Call(...)` | `callType="Call"`, `gxObject="MyProcedure"` |
| `MyWebPanel.Link(...)` or `Link(MyWebPanel, ...)` | `callType="Link"`, `linkType="PXInstance"` (if it is a pattern) or `linkType="GXObject"` |
| A `Do Case` with different Links per condition | `conditionalCalls` with several `conditionalCall` entries |
| Logic after the invocation (If result, WebSession.Set, etc.) | `actionPostCode` |
| A `Return` at the end of the event | Implicit in the action's cycle |
| Code only, with no object invocation | `callType="Event"` (it all goes in as inline code) |
| A bare return with no logic | `callType="Return"` |

---

## 10. Automatic security: visibility of navigating actions (PIsAuthorized)

When an action **navigates to another object** (`callType="Link"` with an `instanceObject`/`gxObject`, or a column/variable `<link>`), PXTools generates in the generated object's **Start** an authorization check against the **target object**; if the user is not authorized, it **hides the control** (it does not disable it: `Visible = False`):

```
&lEnabled = udp(PIsAuthorized ,!'MyApp.WbApprove')
If &lEnabled = Boolean.False
	btnApprove.Visible = Boolean.False
EndIf
```

**Practical consequences:**

- **This is expected behaviour, not a bug.** If an action "does not appear" when the screen opens — and its `evaluateCondition` is `Event`, which should hide nothing — the usual cause is this gating: the user **has no permission over the target object**.
- The check is against the **generated target object**, whose name depends on the platform: for a PXParameterRequest, `Wb{Name}` (Desktop) and `RWb{Name}` (Responsive); for a PXWorkWith, `WW{Name}`/`RWW{Name}`, `View{Name}`/`RView{Name}`, and so on. **Both must be authorized** (Desktop and Responsive) in PXSecurity.
- **When adding an action opening a new object**, that target is **unauthorized by default** until an admin grants it to the role → until then the button **is invisible**. A mandatory step after creating the action: **authorize the target object(s)**.
- Do not confuse this with `condition`/`evaluateCondition`/`execute` (data validation) or with the `<security object=… operation=…>` subnode (permission to execute the action itself): this **visibility** gating is generated by PXTools **automatically** from the navigation, without being declared in the instance.

> A property discovered by inspecting the generated object (`Tr{Name}`/`RTr{Name}`) — see the method in `00-overview.md` → *"Check the instance against the generated object"*.
