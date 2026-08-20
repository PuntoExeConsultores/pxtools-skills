# Limitations and Gaps — What the Patterns Do NOT Support

## Purpose

This document lists the **UI and logic features the PXTools patterns do not currently support**. It matters for:
1. Identifying WebPanels that cannot be migrated 100% to patterns
2. Deciding what would require work at the generator level
3. Planning which functionality has to be implemented by hand after migration

---

## Limitations per pattern

### PXWorkWith

#### Layout and UI
| Limitation | Description | Impact |
|------------|-------------|--------|
| Custom grid layout | The HTML/layout of each grid row cannot be freely customized | WebPanels with "card"-style grids or non-tabular layouts are not migratable |
| Nested grids in Selection | Selection supports one main grid only, not grids inside grids | Screens with inline master-detail in the same grid |
| Custom third-party controls | Only the controls PXTools predefines are supported (Chosen, DatePicker, etc.) | Custom controls such as WYSIWYG editors, maps, interactive charts |
| Drag-and-drop | Not supported | Reordering screens, kanban boards |
| Several grids in Selection | Only one main grid per Selection | Screens with several grids side by side |

#### Logic
| Limitation | Description | Impact |
|------------|-------------|--------|
| Communication between tabs | The tabs (sections) in View are independent: only the active tab is rendered, so they cannot talk in real time through `GlobalEvents`. They can still share information through **WebSession**: one tab writes to WebSession and, when another tab becomes active, it reads that information in its Start/Refresh | Not a blocking limitation, but synchronising through WebSession requires code in the hooks |
| Workflows in the grid | No visual workflows inside the grid | Boards with states and transitions |

#### Exports
| Limitation | Description | Impact |
|------------|-------------|--------|
| PDF export | No native PDF export is generated | Requires a manual implementation |

### PXParameterRequest

| Limitation | Description | Impact |
|------------|-------------|--------|
| Visual wizard with a stepper | Multi-step forms can be built with a PXWorkWith View plus Tabs (each tab = one step, with conditional visibility and free navigation between completed steps). What **does not exist yet** is: (1) a visual "stepper" component with step numbers and progress state, and (2) declarative validations preventing you from moving on while earlier steps are incomplete. Both are planned | Wizards with a visual progress indicator need the stepper built by hand |
| Built-in file upload | File upload has to be implemented with @FileStorage | Forms with file uploads |
| Asynchronous validation | No real-time (on-blur) field validation | Forms validating as the user types |
| Fully custom layout | The form layout follows a predefined structure | Forms with a very specific design |

### PXComposer

| Limitation | Description | Impact |
|------------|-------------|--------|
| Dynamic layout | The arrangement of components is fixed at design time | Dashboards the user can configure |
| Automatic responsiveness of components | Each component has its own responsive behaviour, but the composition may not adapt well | Complex dashboards on mobile |

### PXFlowController

| Limitation | Description | Impact |
|------------|-------------|--------|
| Visual progress UI | No progress bar or current-step indicator is generated | Long flows where the user needs to see which step they are on |
| State persistence | The flow's state is lost if the browser is closed | Flows that must be resumable |
| Parallel flows | Only sequential flow is supported (one active line) | Flows with parallel branches |
| Undo/rollback | No mechanism to undo completed steps | Flows that need to go back with a data rollback |

### WS patterns (PXWSLayer, PXWSQuery, PXWSData, PXWSTransaction)

| Limitation | Description | Impact |
|------------|-------------|--------|
| GraphQL | Only REST and SOAP are generated, not GraphQL | Modern APIs requiring GraphQL |
| WebSockets | No bidirectional communication | APIs needing push/streaming |
| Batch operations | No batch-operation endpoints are generated (except PXWSTransaction) | APIs needing to create/update several records in one call |
| Custom response codes | HTTP response codes are fixed | APIs needing specific HTTP responses |
| Rate limiting | No API-level rate limiting is generated | Public APIs needing throttling |

### PXOAV

| Limitation | Description | Impact |
|------------|-------------|--------|
| Complex computed attributes | Formulas are limited; they do not support complex joins | Dynamic attributes depending on several tables |
| Performance with many attributes | The EAV model carries a performance overhead compared with native columns | Entities with >100 dynamic attributes |
| Indexing | EAV values are not indexed efficiently | Frequent queries over dynamic attributes |

---

## Cross-cutting limitations

### Code generation
| Limitation | Description |
|------------|-------------|
| Compiled generator | The generator code (DLL) cannot be modified by the user. This is by design: it protects the product's intellectual property and licensing |
| Extending the generator | New node types or properties cannot be added without changing the generator DLL |
| Post-processing | There are no post-generation hooks to modify the generated code |

**Important note**: the visual design IS fully customizable through **UI Templates** (GeneXus objects the developer creates freely). Templates define the complete layout of each screen and leave only "Template Elements" as placeholders where the generator injects the specific content. See [00-overview.md](00-overview.md#ui-templates--full-control-over-the-visual-design).

### Integration
| Limitation | Description |
|------------|-------------|
| CI/CD | No native integration with CI/CD pipelines |
| Testing | No automatic unit tests are generated |
| Instance versioning | Instances are versioned along with the KB; they have no independent versioning |

---

## What would have to be built into the generator

### High priority (frequently requested)
1. **Card/responsive grids** — a grid template with a flexible layout
2. **Multi-step wizard** — an extension of PXParameterRequest, or a new pattern
3. **PDF export** — add a PDF export mode to PXWorkWith

### Medium priority
5. **Progress indicator in flows** — stepper/progress bar UI in PXFlowController
6. **Asynchronous validation** — on-blur validation in forms
7. **Batch API operations** — a multi-operation endpoint in PXWSTransaction
8. **Configurable dashboard** — dynamic layout in PXComposer

### Low priority
9. **GraphQL** — a new pattern or an extension of PXWSLayer
10. **Drag-and-drop** — an extension of PXWorkWith
11. **Automatic tests** — unit test generation

---

## Criteria for deciding whether a WebPanel is 100% migratable

A WebPanel is **100% migratable** if ALL of these hold:

- [ ] Its structure matches a known pattern (see [30-pattern-recognition-guide.md](30-pattern-recognition-guide.md))
- [ ] It uses no third-party controls unsupported by PXTools
- [ ] It has no custom JavaScript wired into the server-side logic
- [ ] Communication between components (if any) can be solved with `GlobalEvents`
- [ ] The layout is not heavily customized
- [ ] It implements nothing listed as a "limitation" above

A **partially migratable** WebPanel (>70%) can be migrated and completed with:
- Code in the event hooks (Events, Codes)
- Custom variables and subroutines
- External WebComponents embedded in tabs

A **non-migratable** WebPanel (<30%) has to stay a hand-written WebPanel.
