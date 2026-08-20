# Grid Semantics in WebPanels — For Pattern Recognition

This document describes **how to read the presence of grids in a WebPanel** in order to decide which PXTools pattern to migrate it to. A grid does not always mean PXWorkWith — there are important distinctions to learn.

## Applies to

- Pattern recognition when migrating hand-written WebPanels (the PuntoExe process)
- Design decisions when modelling a new screen with PXTools
- Analysis of legacy GeneXus Evolution 1 / 2 / 3 code

## Kinds of grid by purpose

### 1. Listing / selection grid (the main UI)

**Purpose**: show the user a list of records to view, filter, search, sort, select, or run actions on.

**Signals**:
- It is the dominant visual element of the form
- It has `MaxRows` (paging)
- It has a search button/image
- It loads records with a `For Each` over the database, or by iterating an SDT
- The user interacts with it (clicking rows, scrolling, paging)

**Target pattern**:
- If it has a `Parm out:` + an Enter event that returns → **PXWorkWith Prompt**
- If it has CRUD actions (Insert/Update/Delete) → **PXWorkWith Selection**
- If it is a read-only grid used as a viewer with per-row navigation → **PXWorkWith Selection (viewer)**

### 2. Auxiliary grid inside a Data Entry

**Purpose**: complement a tabular form with a list of items (invoice detail, selected items, errors to display, etc.).

**Signals**:
- The main element of the form IS a tabular form with editable fields
- The grid is a bounded area inside the form; it does not dominate visually
- It is usually coupled to the form: changes in form fields affect the grid or vice versa
- The main action (Accept/Confirm) processes the data of BOTH the form and the grid

**Target pattern**: **PXParameterRequest** (with an auxiliary grid inside the `grid` node)

Example: `WnNewInvoice` — invoice header as a form + item detail as a grid.

### 3. "Phantom" grid used to persist SDT variables (legacy GeneXus)

**Original purpose**: in older GeneXus versions (Evolution 1 and earlier), the **only way to persist the state of an SDT/collection between events** within the same WebPanel was to declare the variables as columns of a grid, usually hidden.

**Signals**:
- The grid is **hidden from the user** — it uses `Class="Hidden"`, or `Grid.Visible = 0` / `Grid.Visible = False` is set in `Event Start`
- Its "columns" are SDT variables, collections, or primitive types with no real display
- The user does not interact with it — no scrolling, no search button, no paging
- The code accesses those variables as panel state (for instance in a For Each Line to iterate them)

**Important**: in modern GeneXus versions, panel variables already keep their state between events without a grid. **A phantom SDT-persistence grid is legacy code and must NOT be counted as a functional grid when classifying the WebPanel.**

**Consequence for classification**:
- A WebPanel with a single phantom grid + a form of editable fields → **PXParameterRequest** (NOT PXWorkWith Prompt)
- A WebPanel with several grids where some are phantoms → count only the real ones to decide Selection vs auxiliary

### 4. Grid used only as layout / container

**Purpose**: using a grid's tabular structure purely as a visual container for other controls (far less common).

These usually have no data rows. Ignore them when classifying the pattern.

## How to spot a phantom grid

### Signals in the `.gxForm`

```xml
<grid name="GridSDT" class="Hidden" ...>
   <!-- columns that are SDT variables -->
   <columns>
     <column ...>
       <variable name="MySDT" ... />
     </column>
   </columns>
</grid>
```

Or: a `class` that in the theme maps to `display:none` / `visibility:hidden`.

### Signals in the `.gxSource`

```genexus
Event Start
    Grid1.Visible = False    // or Visible = 0
EndEvent
```

> **CRITICAL — tell a phantom apart from conditional visibility**: a `phantom` grid (pure SDT persistence) has `.Visible = False` and **NEVER** `.Visible = True` anywhere in the code. If a grid shows **both patterns** (`= False` under some conditions and `= True` under others), it is a grid with **conditional visibility** — the user does see it sometimes, so it is NOT a phantom. It counts as a real grid.
>
> **Phantom example (filter it out)**:
> ```genexus
> Event Start
>     GridSDT.Visible = False
> EndEvent
> ```
>
> **Conditional example (do NOT filter — it is a real grid)**:
> ```genexus
> Event Start
>     If &Mode = "Edit"
>         Documents.Visible = True
>     Else
>         Documents.Visible = False
>     EndIf
> EndEvent
> ```
>
> The detector must strip comments (`//`, `/* */`) before looking for the Visible patterns, so commented-out code is not counted.

### Final heuristic

A grid is a phantom if **ALL** of these hold:

1. It is hidden via `class="Hidden"` (or similar), or `Visible = False`/`= 0` in Start
2. Its columns are **panel variables** (not transaction attributes) and most of them are SDTs/collections
3. It has NO `MaxRows`, no paging and no associated search button
4. There is NO `For Each` over the database loading it

If any of these fails, it is probably a real grid.

## The For Each Line pattern

`For Each Line` (or `For each line in <Grid>`) is a GeneXus construct that iterates the grid's rows in code. Its presence carries strong implications for classification.

### What it indicates

`For Each Line` is used when the developer needs to **read/write row values programmatically**. That typically happens with editable grids: the user edits values in the rows and the code collects them in an event (e.g. Accept) by walking the grid.

### Implications for the target pattern

| Context | Recommended pattern |
|---------|---------------------|
| For Each Line + editable grid + a form of editable fields outside the grid | **PXParameterRequest** with an auxiliary grid (mixed data entry) |
| For Each Line + editable grid as the main element + a save action | **PXWorkWith Selection** with an editable grid and `modes/updateGridRows` |
| For Each Line over a phantom SDT-persistence grid | Ignore it — it is legacy. Reclassify as if the grid did not exist |
| For Each Line with no visual grid (phantom grid) | The grid is a phantom → determine the pattern from the rest of the form |

### How to spot For Each Line in the code

```genexus
For each line in Grid1
    &Total = &Total + GridColumn1
    If GridColumn2 > 0
        // ...
    EndIf
EndFor
```

Or its variants:
- `For each line` (without a grid name)
- `For each line in <GridName>`

## Several grids in one WebPanel

### Possible cases

1. **Several real, visible grids**: a complex panel with multiple data sections. Usually points to a PXComposer (composition) or to a special, complex migration case.

2. **One main grid + auxiliary grids of related data**: a mixed Selection structure with embedded grids. PXComposer may be the answer, or PXWorkWith with tabs in View.

3. **One real grid + phantom SDT-persistence grids**: count ONLY the real ones. If a single real grid remains, it is PXWorkWith / PXParameterRequest depending on the case.

4. **All grids are phantoms**: it is legacy code; the pattern is determined by the rest of the form, not by the grids.

### Effect on complexity

Several real grids **raise the complexity score**, because they mean:
- More loading code (multiple For Each)
- More events (typically one per grid)
- More dependencies (a change in one can affect the others)

Several phantom grids, however, **do not raise real complexity** — they are simply legacy, rewritten as modern hooks.

## Summary for automatic classification

When analysing a WebPanel with grids, follow this logic:

```
1. Count every <grid> in the .gxForm
2. For each grid: decide whether it is a phantom (hidden + SDT variables + no MaxRows + no search)
3. Compute real_grids = total - phantoms
4. If real_grids == 0:
     -> Treat as a WebPanel without a grid (pure PXParameterRequest, or Manual)
5. If real_grids == 1:
     -> Apply the standard criteria (Selection / Prompt / PR with an auxiliary grid)
6. If real_grids > 1:
     -> Flag as "multi-grid" for review
     -> Possible targets: PXComposer, PXWorkWith with tabs, or a complex case
7. Additionally: if For Each Line is present:
     -> It indicates an editable grid
     -> Combined with an external form -> PXParameterRequest with a grid
     -> Grid only -> PXWorkWith Selection with an editable grid
```

## Useful numeric indicators for complexity

When analysing the `.gxSource` code, these counters help build the complexity score:

| Indicator | Meaning |
|-----------|---------|
| Total lines inside events (Event … EndEvent) | Volume of panel logic |
| Number of events | Number of extension points |
| Number of subroutines | Granularity of the logic |
| Number of For Each (database) | Loops over the database |
| Number of For Each Line (grids) | Iteration over the UI |
| Number of Procedure calls | Coupling with external logic |
| Number of IFs | Conditional complexity |
| Number of Do Case | Multi-branch decisions |
| SDT variables | Structure manipulation |
| Collection variables | In-memory list handling |
| WebSession references | State across screens |
| XML/JSON operations | Serialization |
| Total number of variables | Scope of the panel |
| Number of (real) grids | UI complexity |
