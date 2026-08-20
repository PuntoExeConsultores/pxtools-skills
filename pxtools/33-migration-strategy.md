# Migration Strategy — From a KB Without Patterns to PXTools

## Overview

This document describes the process of migrating a **GeneXus Knowledge Base (KB) built without patterns** to one that uses PXTools. The migration is not all-or-nothing: it can be done progressively, one WebPanel at a time.

## Prerequisites

1. **PXTools installed** in the target KB
2. The **required @PXTools modules** imported (at minimum: @APIs, @System, @Security)
3. **Patterns enabled** in GeneXus (the patterns extension loaded)
4. **Transactions defined** (PXWorkWith requires transactions)

## The migration process in 6 phases

### Phase 1: inventory and classification

```
┌─────────────────────────────────────────────┐
│           WEBPANEL INVENTORY                │
│                                             │
│  For each WebPanel in the KB:               │
│                                             │
│  1. Identify its type (see the pattern      │
│     recognition guide)                      │
│  2. Classify it:                            │
│     ● 100% migratable to a pattern          │
│     ● Partially migratable (>70%)           │
│     ● Not migratable                        │
│  3. Assign the target pattern               │
│  4. Identify its dependencies               │
└─────────────────────────────────────────────┘
```

**Result**: a classified list of WebPanels with their target pattern and priority.

### Phase 2: migrating the infrastructure

Before migrating individual WebPanels, install the @PXTools modules providing the infrastructure:

| Current functionality | @PXTools module | Action |
|-----------------------|-----------------|--------|
| Custom security | @Security | Migrate users/roles/permissions |
| Hand-built menus | @Menus | Migrate the menu structure |
| System parameters | @SystemParameters | Migrate the parameters |
| Sending mail | @SendMails + @MailAccounts | Migrate the configuration |
| File management | @FileStorage | Migrate the storage |
| Logs | @WebServicesLog | Integrate logging |

### Phase 3: migrating simple CRUD screens (PXWorkWith)

Start with the simplest WebPanels: **CRUD screens** over master tables.

#### The process for each WebPanel:

**Step 1 — create the PXWorkWith instance**

Attach it to the matching transaction and configure:

```xml
<instance>
  <level name="MyEntity">
    <selection>
      <grid>
        <!-- Map the current grid's columns -->
        <attributes>
          <attribute name="AttributeId" />
          <attribute name="AttributeName" />
          <attribute name="AttributeStatus" />
        </attributes>
      </grid>
      <filter>
        <!-- Map the existing filters -->
        <search>
          <attribute name="AttributeName" />
        </search>
      </filter>
      <orders>
        <!-- Map the existing orderings -->
        <order name="ByName">
          <orderAttribute name="AttributeName" ascending="true" />
        </order>
      </orders>
      <actions>
        <!-- Map the existing actions/buttons -->
      </actions>
    </selection>
  </level>
</instance>
```

**Step 2 — migrate the custom logic into hooks**

Whatever is not declarative moves into the code hooks:

```xml
<codes>
  <code type="Start">
    <![CDATA[
      // The original WebPanel's initialization code
    ]]>
  </code>
  <code type="Refresh">
    <![CDATA[
      // The original Refresh code
    ]]>
  </code>
  <code type="Load">
    <![CDATA[
      // The original Load code (per-row computations)
    ]]>
  </code>
</codes>
```

**Step 3 — generate and compare**

1. Generate the objects from the instance
2. Compare the generated WebPanel visually against the original
3. Adjust the instance until they match

**Step 4 — replace**

1. Update the references to the original WebPanel so they point at the generated one
2. Disable/delete the original hand-written WebPanel

### Phase 4: migrating forms (PXParameterRequest)

Migrate popups, confirmation dialogs and parameter-capture forms.

#### Mapping criteria:

| Current WebPanel | PXParameterRequest behaviour |
|------------------|------------------------------|
| Popup with "Are you sure?" + Yes/No | `PopupParameterRequest` |
| Modal form capturing data | `ParameterRequest` |
| Floating filter panel | `FloatingParameterRequest` |
| Embedded form | `Panel` |

### Phase 5: migrating composed screens (PXComposer)

Migrate dashboards and screens combining several components.

#### The process:

1. Identify the screen's individual components
2. Verify that each component has already been migrated to a pattern (PXWorkWith, PXParameterRequest)
3. Create a PXComposer instance composing them:

```xml
<instance>
  <level name="Dashboard">
    <forms>
      <form name="Main" platform="Any">
        <components>
          <component type="WebComponent"
                     callType="PXInstance"
                     instanceObject="PXWorkWithInvoices"
                     instanceLevel="Level1"
                     instanceLevelNode="Selection" />
          <component type="WebComponent"
                     callType="PXInstance"
                     instanceObject="PXWorkWithCustomers"
                     instanceLevel="Level1"
                     instanceLevelNode="Selection" />
        </components>
      </form>
    </forms>
  </level>
</instance>
```

### Phase 6: migrating flows (PXFlowController)

Migrate WebPanels implementing step-by-step workflows.

#### The process:

1. Diagram the current flow (steps, decisions, confirmations)
2. Map each step to a PXFlowController "line"
3. Map each decision to an action with a nextLine
4. Map each confirmation to a confirm node with its responses

---

## Simultaneous dual-platform migration

If you are also moving from Desktop to Responsive, use the pattern migration to do it in a single step:

```
Hand-written WebPanel (Desktop)
    │
    ▼ Migrate to a pattern
    │
    ├── generateWeb = True          ← Keep Desktop
    └── generateWebResponsive = True ← Add Responsive
```

That is more efficient than:
1. Migrating to a pattern on Desktop
2. Then enabling Responsive

## Recommended migration order

```
1. @PXTools modules (infrastructure)
   │
2. Simple CRUD screens (master tables)
   │    └── PXWorkWith with Selection only
   │
3. CRUD screens with a detail
   │    └── PXWorkWith with Selection + View + Tabs
   │
4. Forms and popups
   │    └── PXParameterRequest
   │
5. Composed screens
   │    └── PXComposer (its components must already be migrated)
   │
6. Workflows
   │    └── PXFlowController
   │
7. APIs/Web Services
        └── PXWSLayer + PXWSQuery + PXWSData + PXWSTransaction
```

## Progress metrics

| Metric | How to measure it |
|--------|-------------------|
| % of WebPanels migrated | (migrated WPs / total WPs) × 100 |
| Pattern coverage | (WPs with a pattern / migratable WPs) × 100 |
| UI debt | Number of WPs marked "not migratable" |
| Dual-platform | % of instances with generateWebResponsive = True |

## Risks and mitigation

| Risk | Mitigation |
|------|------------|
| Losing functionality | Compare visually before replacing |
| Different performance | Test with realistic data volumes |
| Logic not captured in the hooks | Review the original WP's Events code exhaustively |
| Broken dependencies | Map every reference before deleting the original WP |
| Team resistance | Migrate progressively, starting with the simplest |

## Per-WebPanel migration checklist

- [ ] WebPanel classified (100% / partial / not migratable)
- [ ] Target pattern identified
- [ ] The associated transaction exists
- [ ] Pattern instance created
- [ ] Columns/fields mapped
- [ ] Filters mapped
- [ ] Orderings mapped
- [ ] Actions mapped
- [ ] Custom code migrated into the hooks
- [ ] Objects generated
- [ ] Visual comparison OK
- [ ] Functional tests OK
- [ ] References updated
- [ ] Original WebPanel disabled
- [ ] generateWebResponsive enabled (where applicable)
