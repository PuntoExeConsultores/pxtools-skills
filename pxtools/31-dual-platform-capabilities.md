# Dual-Platform Capabilities — Desktop → Responsive Migration

## Overview

PXTools can generate **two versions of each WebPanel at the same time**:
1. **Web Desktop** — classic HTML layout (tables, divs, absolute positioning)
2. **Web Responsive** — GeneXus Abstract Layout (responsive, mobile-first)

And optionally:
3. **Smart Devices** — layout for native mobile applications

All of it from **a single pattern instance** (.gxPattern), with no duplicated definition.

## How dual generation works

### In the .Pattern

Every generated object has two versions:

```xml
<!-- Desktop version: uses the WebForm template (HTML layout) -->
<Object Type="WebPanel" Id="Selection" Name="WW{Instance.Name}"
        Element="instance/level/selection">
  <Part Type="WebForm" Template="Templates\SelectionWebForm.dll" />
  <!-- ... the same Variables, Events, Rules, Conditions -->
</Object>

<!-- Responsive version: uses the AbstractForm template (abstract layout) -->
<Object Type="WebPanel" Id="SelectionResponsive" Name="RWW{Instance.Name}"
        Element="instance/level/selection">
  <Part Type="WebForm" Template="Templates\SelectionAbstractForm.dll" />
  <!-- ... the same Variables, Events, Rules, Conditions -->
</Object>
```

**Key point**: both versions share exactly the same logic (Variables, Events, Rules, Conditions). Only the **Form generator** (DLL) differs: one produces an HTML layout (Desktop) and the other an abstract layout (Responsive). On top of that, each version can use a different **UI Template** (`templateObject` for Desktop, `templateResponsiveObject` for Responsive), allowing different visual designs per platform.

### Naming prefixes

| Pattern | Desktop | Responsive |
|---------|---------|------------|
| PXWorkWith Selection | `WW{Name}` | `RWW{Name}` |
| PXWorkWith View | `View{Name}` | `RView{Name}` |
| PXWorkWith Prompt | `Pr{Name}` | `RPr{Name}` |
| PXWorkWith Controller | `Ct{Name}` | `RCt{Name}` |
| PXWorkWith Transaction | `D{Name}` | `R{Name}` |
| PXComposer | `Wb{Name}` | `RWb{Name}` |
| PXParameterRequest | `Wb{Name}` | `Wb{Name}` * |
| PXFlowController | `Ct{Name}` | `Ct{Name}` * |

\* PXParameterRequest and PXFlowController use the **same name** for both versions.

## Control properties

Every level/node of the instance has three generation properties:

```xml
<level name="Invoice"
       generateWeb="True"              <!-- Generate the Desktop version -->
       generateWebResponsive="True"    <!-- Generate the Responsive version -->
       generateSD="False">             <!-- Generate the Smart Devices version -->
```

Possible values:
- `<default>` — use the value defined in Settings
- `True` — generate this version
- `False` — do not generate this version

### Platform (in PXParameterRequest)

PXParameterRequest has an extra `platform` property:

```xml
<level platform="Any">          <!-- Generates for every platform -->
<level platform="Web Desktop">  <!-- Generates for Desktop only -->
```

### Platform in forms (in PXComposer)

PXComposer can define **different forms per platform**:

```xml
<forms>
  <form name="Desktop" platform="Web Desktop">
    <components>
      <!-- Desktop-specific layout -->
    </components>
  </form>
  <form name="Responsive" platform="Web Responsive">
    <components>
      <!-- A different layout for Responsive -->
    </components>
  </form>
</forms>
```

## Progressive migration strategy

### Phase 1: coexistence
```
PXWorkWith instance "Invoice"
├── generateWeb = True           ← Desktop keeps working
├── generateWebResponsive = True ← the Responsive version is generated alongside
│
├── WWInvoice (Desktop)          ← current users stay on this
└── RWWInvoice (Responsive)      ← can be tested and validated
```

### Phase 2: transition
- Redirect users to the Responsive versions
- Verify that all functionality is covered
- Adjust responsive templates/themes if needed

### Phase 3: switching Desktop off
```
PXWorkWith instance "Invoice"
├── generateWeb = False          ← Desktop is turned off
├── generateWebResponsive = True ← Responsive only
│
└── RWWInvoice (Responsive)      ← the only version
```

## Advantages of dual generation

1. **Zero logic duplication**: Variables, Events, Rules and Conditions are the same
2. **Risk-free migration**: both versions coexist
3. **Instant rollback**: if something fails in Responsive, Desktop is still there
4. **One source of truth**: a single .gxPattern drives both versions
5. **A/B validation**: the behaviour of both versions can be compared

## What differs between versions

| Aspect | Desktop | Responsive |
|--------|---------|------------|
| Layout | HTML (tables, absolute positioning) | Abstract (responsive, flexbox) |
| MasterPage | Desktop MasterPage | Responsive MasterPage |
| Theme | Desktop Theme | Responsive Theme |
| Controls | Standard web controls | Responsive controls |
| Size | Fixed | Adapts to the screen |
| Touch | No | Yes |

## What does NOT differ between versions

- Business rules (Rules)
- Events (Events code)
- Variables
- Filter conditions
- Orders
- Actions (structure)
- Parameters
- Security

## ResponsiveLayout — controlling responsive positioning

For the Responsive generator, PXTools exposes the **ResponsiveLayout** property, which uses the same native GeneXus component to define how a section's sub-elements are positioned depending on screen size. It lets you:

- Define **breakpoints** by screen size
- Configure the **arrangement of elements** for each breakpoint (horizontal, vertical, columns)
- Adapt the display of filters, form fields, actions and any sub-element
- All declaratively, from the pattern instance

Combined with the **UI Templates** (which give full freedom over the visual design), the responsive layout is entirely customizable without writing CSS by hand.

## Things to keep in mind with dual generation

- Custom third-party controls may have no responsive equivalent
- Desktop-specific UserControls do not work in an abstract layout
- The MasterPage and Theme must exist for both versions
