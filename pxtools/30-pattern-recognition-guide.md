# Guide to Recognising Patterns in Existing WebPanels

## Purpose

This guide lets an AI analyse **hand-written WebPanels** (built without patterns) and decide **which PXTools pattern (or combination of patterns)** each of them could be migrated to.

## Analysis method

For each WebPanel, analyse:
1. **Layout structure** (grids, forms, tabs, popups)
2. **Behaviour** (CRUD, query, flow, composition)
3. **Relationship with transactions** (base table, navigation)
4. **Available actions** (buttons, links, exports)
5. **Input/output parameters**

> **Important about grids**: the mere presence of a grid does not imply PXWorkWith. There are "phantom" grids used only to persist SDT variables (legacy GeneXus Evo1) that must be ignored when classifying. See [13-grid-webpanel-semantics.md](13-grid-webpanel-semantics.md) for the detail of how to tell real grids from phantoms, multiple grids, and For Each Line.

---

## Recognition signals per pattern

### PXWorkWith — Master-Detail CRUD

PXWorkWith generates three kinds of screen: **Selection**, **View** and **Prompt**. Telling Selection from Prompt is essential because both have a grid but their purpose differs.

#### PXWorkWith Selection

The main CRUD listing screen. The user browses, filters, and runs actions on records.

**Signals in the Form (.gxForm):**
- A main data grid
- A search button/image
- MaxRows on the grid (paging)
- CRUD actions (Insert, Update, Delete, Display)

**Signals in the Source (.gxSource):**
- No `out:` parameters in Parm (it returns no values to the caller)
- Actions with a Link to a Transaction for Insert/Update
- It may have a Delete with confirmation
- A title describing the listing (e.g. "Invoices", "Documents")

**Signals in the code:**
```
// Signal: a For each over a transaction's base table
For each
    where <filters>
    order <attributes>
    // Loads the grid
EndFor

// Signal: a Link to a transaction with Insert/Update mode
Link(TrnInvoice, InvoiceId)
```

#### PXWorkWith Prompt

A search/lookup popup the user opens to **select a record and return values** to the caller. It is a record selector, not a data entry.

**Signals in the Form (.gxForm):**
- A data grid (same as Selection)
- A search button/image
- `MaxRows` on the grid (paging) — this rules out an auxiliary PXParameterRequest grid
- A column with `eventGX="Enter"`, or a per-row selection image

**Signals in the Source (.gxSource):**
- A `Parm` with `out:` parameters — **the definitive indicator** that it returns values to the caller
- An Enter event that loads the Out variables and does a `Return`
- A title like "Select \<Entity\>" — it is a record selector
- No CRUD actions (no Insert, no Update, no Delete)

**Signals in the code:**
```
// DEFINITIVE Prompt signal: a Parm with out
Parm(in:&CompanyCode, out:&BankCode, out:&BankName);

// DEFINITIVE signal: an Enter event loading Out and returning
Event Enter
    &BankCode = BankCode
    &BankName = BankName
    Return
EndEvent
```

#### PXWorkWith Prompt — special case: a grid loaded from an SDT/WebSession

A Prompt may have its grid loaded **manually** with `Grid.Load` from an SDT or from data in WebSession, instead of reading a base table directly. This is very common in selectors filtered by prior context (by employee, by warehouse, by third party, by batch, and so on) and in multi-row selectors where the caller leaves data in WebSession before invoking the popup.

**Do not confuse it with an auxiliary PXParameterRequest grid.** It is still a Prompt when:
- The WebPanel's main element is the grid (not a tabular form of editable fields)
- The purpose is to **select one or more records and return to the caller**
- The grid may have `Rows`/`MaxRows` and filters, even though its data comes from an SDT/WebSession
- The title is usually "Select…", "Choose…", "List of…"

**Signals in the code:**
```
// Signal: the grid is loaded manually from an SDT/WebSession
&SDTItems.FromJson(&WebSession.Get("Items"))
For &i = 1 to &SDTItems.Count
    Grid1.Load
EndFor

// Signal: flag the row(s) and return through WebSession (see the "Prompt with no Parm out" case)
Event 'Accept'
    &WebSession.Set("Items", &SDTItems.ToJson())
    &Window.Close()
EndEvent
```

#### PXWorkWith Prompt — special case: returning through WebSession (no Parm out)

In legacy KBs (especially ones migrated from Evolution 1) it is common for a Prompt **not to use `Parm` with `out:`** and instead leave the selection's result in **WebSession**, for the caller to read when it refreshes as the popup closes. This is typical of selectors invoked with `target=New` (popup) that end with `&Window.Close()` or by closing the form.

**It is still a PXWorkWith Prompt** — where the return travels (Parm out vs WebSession) does NOT determine the pattern; what matters is the **purpose**: the WebPanel exists so the user can pick a record (or several) and come back to the caller with that choice.

**Signals:**
- Invoked with `target=New` from another WebPanel (a popup)
- It has no `Parm` with `out:`, or no `Parm` at all
- It has a main grid of records (from a base table, an SDT or WebSession)
- An "Accept"/"Select" action storing into WebSession and calling `&Window.Close()`, or closing the form
- The caller reads WebSession in its Refresh/Start

**Migrate it as a Prompt**, with the "store the selection in WebSession" logic in the selection action's `actionPostCode`.

#### Naming heuristics (a secondary reference)

As a **supporting signal** (never decisive on its own), the common prefixes in GeneXus KBs usually indicate:

| Prefix | Likely pattern |
|--------|----------------|
| `Sel*`, `prmt*`, `Prompt*`, `Gx00*` | PXWorkWith Prompt |
| `Con*`, `Vis*`, `Ver*` | PXWorkWith Selection (read-only viewer) |
| `Reg*`, `New*`, `Mod*`, `Cam*`, `Ins*` | PXParameterRequest |
| `WW*`, `WB*`, `Wrk*` | PXWorkWith Selection (CRUD) |

**Important**: these panels were hand-written, so the naming may well be **inconsistent**. The prefix heuristic only serves to **confirm** a classification obtained by analysing the Form/Source, never to replace it. And there are explicit exceptions: `SelRanRep*` ("Select Range for Report"), for instance, is a PXParameterRequest because it captures parameters (dates), not a record.

**The key Selection vs Prompt difference:**

| Criterion | Selection | Prompt |
|-----------|-----------|--------|
| Parm with `out:` | NO | **YES** |
| An Enter event returning values | NO | **YES** |
| CRUD actions (Insert/Update/Delete) | YES | NO |
| Title | Describes the listing | "Select…" |
| Purpose | Browse and operate on data | Pick a record and return |

#### The matching PXWorkWith nodes

| What you see in the WebPanel | PXWorkWith node |
|------------------------------|-----------------|
| A grid of records (Selection) | `selection/grid` |
| A grid of records (Prompt) | `prompt/grid` |
| A quick search field | `filter/search` |
| Advanced filters (combos, ranges) | `filter/advancedSearch` |
| Fixed conditions (WHERE) | `filter/conditions` |
| Orderings | `orders` |
| A "New" button | `selection/actions` (Insert) |
| An "Edit" button in the grid | `selection/actions` (Update) |
| A "Delete" button | `selection/actions` (Delete) |
| An "Export to Excel" button | `selection/modes/exportExcel` |
| A detail screen with tabs | `view` + `view/sections` |
| A tab with tabular data | `view/sections/section[@type='Tabular']` |
| A tab with a sub-grid | `view/sections/section[@type='Grid']` |

**Migratable as PXWorkWith when:**
- [x] It has a grid with transaction data
- [x] It has filters/search
- [x] If it has a Parm out + an Enter that returns → **Prompt**
- [x] If it has CRUD actions → **Selection**
- [x] If it has a read-only grid (non-editable variables/fields) + a per-row navigation action → **Selection** (a data viewer with a detail)

#### PXWorkWith Selection — special case: a read-only viewer with a grid

A WebPanel with a **read-only** grid (all fields readonly/display) is also a PXWorkWith Selection, not a PXParameterRequest. The key is that its main element is a **grid of records**, not a data entry form.

**Definitive signals of a read-only Selection:**
- A main grid with `Rows` or `MaxRows` (paging)
- Every field/variable in the grid is **readonly** (the user cannot edit them)
- It may have tabular data above the grid (a header), also **readonly**
- A per-row action navigating to a detail (e.g. `View_Record`, `View_Detail`)
- Only `Parm` with `In:` (it returns nothing, it only queries)
- An Exit/Close button as the only global action
- The data may come from a base table (with `#Conditions`) or from an SDT/WebSession (with a manual `Grid.Load`)

**Signals in the code:**
```
// Signal: a paged grid
Grid1.Rows = 10

// Signal: read-only tabular data (the record's header)
&CustomerName = CustomerName
&ItemLabel = CategoryCode.Trim() + " - " + ItemCode.Trim()
&Status = "Pending"

// Signal: a per-row navigation action (view the detail)
Event 'View_Detail'
    &Window.Url = Link(TrnDetail, TrnMode.Display, ...)
    &Window.Open()
EndEvent

// Signal: Exit as the only global action
Event 'Exit'
    Return
EndEvent
```

**The key difference from PXParameterRequest:**
- PXParameterRequest is a **data entry** — the user enters/edits data in fields
- A read-only Selection is a **viewer** — the user only looks at data and can navigate to a detail
- If every field is readonly → it is NOT data entry → it is a Selection

---

### PXParameterRequest — Parameter Form

PXParameterRequest is a **tabular data entry** (a form), NOT a search screen with a grid.

**Main signals:**
- A WebPanel with a tabular form of fields (NOT a main grid)
- Accept/Cancel (or Confirm/Exit) buttons
- It captures parameters from the user, or shows information to be confirmed
- It may have an **auxiliary** grid inside the form (but the grid is not the main element)

**How to tell it apart from a PXWorkWith Prompt:**

| Criterion | PXParameterRequest | PXWorkWith Prompt |
|-----------|--------------------|-------------------|
| Main element | A tabular form (fields) | **A grid of records** |
| Grid | None, or a small auxiliary one | **A main grid with MaxRows and paging** |
| Search button | No (or it searches inside the auxiliary grid) | **Yes, it searches records in the main grid** |
| Parm out | It may have one (returning captured data) | **It always has one** (returning the selected record) |
| Enter event on the grid | Not applicable | **Yes, it loads the out vars and returns** |
| Title | "Confirm…", "Enter…", "Cancel…" | **"Select…"** |
| Purpose | Capture data / confirm an action | **Search for and pick a record** |

**Signals in the code:**
```
// PXParameterRequest signal: Accept/Cancel carrying business logic
Event 'Accept'
    // Validations
    If &Field.IsEmpty()
        msg("Field required")
        Return
    EndIf
    // The main invocation
    MyProcedure.Call(&Param1, &Param2)
    Return
EndEvent

Event 'Cancel'
    Return
EndEvent
```

**Detectable PXParameterRequest types:**

| WebPanel behaviour | behaviour type |
|--------------------|----------------|
| A simple popup with a message + Ok | `PopupParameterRequest` |
| A capture form with Accept/Cancel | `ParameterRequest` |
| A floating, non-blocking panel | `FloatingParameterRequest` |
| An embedded panel with no modal behaviour | `Panel` or `None` |

**Migratable as PXParameterRequest when:**
- [x] It is a tabular form (data entry), not a search grid
- [x] It has Accept/Cancel or Confirm/Exit
- [x] It captures user data or shows information to confirm an action
- [x] It does NOT have a main grid with search and paging (that would be PXWorkWith)

**Special case — PXParameterRequest with an auxiliary grid (mark it with \*):**
If the WebPanel has a data entry form as its main element BUT also includes an auxiliary grid (to show selected items, errors, or a detail, for instance), classify it as PXParameterRequest but mark it with an asterisk (\*) for manual review.

---

### PXComposer — Screen Composition

**Main signals:**
- A WebPanel containing several embedded WebComponents
- A dashboard-style layout with sections
- It combines several features into one screen
- The WebComponents can be grids, forms, or other panels

**Signals in the code:**
```
// Signal: several WebComponents in the layout
// The WebForm holds several WebComponent controls
<WebComponent name="wcGrid1" object="WWInvoices" />
<WebComponent name="wcDetail" object="ViewCustomer" />
<WebComponent name="wcActions" object="WbActions" />
```

**Migratable as PXComposer when:**
- [x] The WebPanel is mainly a container of other panels
- [x] It uses WebComponents for composition
- [x] The embedded components could be PXWorkWith, PXParameterRequest or another PXComposer
- [x] It has no significant business logic of its own

---

### PXFlowController — Workflow

**Main signals:**
- A WebPanel guiding the user through a sequence of steps
- It has several "screens" or states within the same WebPanel
- It uses flow-control variables (&Step, &Line, &State)
- Actions moving to the next/previous step
- Intermediate confirmations
- It may open popups and continue depending on the result

**Signals in the code:**
```
// Signal: flow control with a step variable
Do Case
    Case &Step = 1
        // Show step 1
        Call(WbConfirmation, ...)
    Case &Step = 2
        // Process
        PrcProcess(...)
    Case &Step = 3
        // Result
EndCase

// Signal: if-then logic with popups and continuation
If &Confirm = "Yes"
    // Next step
    &Step = 2
Else
    // Back
    &Step = 1
EndIf
```

**Migratable as PXFlowController when:**
- [x] The WebPanel implements a sequence of steps
- [x] It has "next step" / "previous step" logic
- [x] It uses confirmations between steps
- [x] It may have iterations (repeating steps)
- [x] The logic can be expressed as "lines" with actions and destinations

---

## Quick decision matrix (updated)

```
Is it a Login screen (IsMain=True, no MasterPage, custom auth)?
├── YES → Manual (not migratable)
│
Is it a testing/example utility or an automatic redirect?
├── YES → Manual (not migratable)
│
Does it depend on an external framework (Scheduler, third-party controls)?
├── YES → Manual (not migratable)
│
Does it have a master-detail CRUD grid over a transaction?
├── YES → Does it have detail tabs?
│         ├── YES → PXWorkWith (Selection + View)
│         └── NO  → PXWorkWith (Selection only)
│
Does it have a read-only query/selection grid with no CRUD?
├── YES → Is it a popup/lookup returning values?
│         ├── YES → PXParameterRequest with a grid (or PXWorkWith Prompt)
│         └── NO  → PXWorkWith (query-only Selection)
│
Does it have Accept/Cancel or Confirm/Exit?
├── YES → PXParameterRequest (PopupParameterRequest)
│
Is it a data capture form (with or without a grid)?
├── YES → PXParameterRequest
│
Is it a read-only data viewer (Exit only)?
├── YES → PXParameterRequest (behaviour None or Panel)
│
Does it contain several embedded WebComponents?
├── YES → PXComposer
│
Does it implement a sequence of steps?
├── YES → PXFlowController
│
└── EVERYTHING else → PXParameterRequest (with the logic in hooks)
```

**Fundamental rule:** if the WebPanel has any structure with action buttons and/or a form, it is migratable to PXParameterRequest. **All the business logic** (subroutines, For Each, validations, WebSession, SDTs, Procedure calls) moves into code hooks with no loss of functionality.

## Frequent combinations

| Scenario | Patterns to use |
|----------|-----------------|
| A full CRUD screen with detail tabs | PXWorkWith |
| CRUD with a confirmation popup before deleting | PXWorkWith + PXParameterRequest |
| A dashboard with several grids | PXComposer + PXWorkWith (×N) |
| A guided process with confirmations | PXFlowController + PXParameterRequest |
| A composed security view | PXComposer + PXWorkWith + PXParameterRequest |
| A complete REST API for an entity | PXWSLayer + PXWSQuery + PXWSTransaction |
| A report with parameter selection | PXParameterRequest + PXReportTemplate |
| A selection popup with a grid and filters | PXParameterRequest with a grid |
| A cancellation/reversal popup with a reason | PXParameterRequest (PopupParameterRequest) |
| File/image upload | PXParameterRequest with controlType File |
| A read-only viewer of errors/data | PXParameterRequest (behaviour None/Panel) |
| Generating TXT/Excel files | PXParameterRequest with callType Event |
| Running a one-shot process | PXParameterRequest (behaviour Panel) |

## Signals of non-migration

A WebPanel is **not migratable** to patterns ONLY when:
- It is a **Login screen** with a custom authentication flow (IsMain=True, no MasterPage)
- It is a **testing/example utility**, not production functionality
- It depends on an **external framework** managing its own life cycle (a Scheduler, for instance)
- It is an **automatic redirect** with no user interaction (JavaScript/server redirect only)

### What is NOT a reason to mark it "not migratable"

**None of these factors prevents migration:**
- The amount of business logic (all of it goes into hooks)
- Use of WebSession (a standard mechanism)
- Many subroutines (they go into `codes/Subroutine`)
- Creating/modifying records (callType Call or Event)
- A grid with a complex Load (a grid node + the Load hook)
- Dynamically hiding controls (logic in the Start hook)
- Using SDTs to pass data (supported in variables and hooks)
- Complex validations (actionPreviousCode)
- Several nested For Each (it is all GeneXus code inside hooks)

**Expected outcome:** in an analysis of a real KB (225 WebPanels), only **3.1%** turned out to be genuinely non-migratable (7 out of 225: 3 logins, 2 testing, 1 scheduler, 1 redirect). The remaining **96.9%** were migratable to PXParameterRequest, PXWorkWith or PXComposer.

See [32-limitations-and-gaps.md](32-limitations-and-gaps.md) for the detail of the generator's limitations.
