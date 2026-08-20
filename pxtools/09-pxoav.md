# PXOAV — Object Attribute Values Pattern (Dynamic Attributes)

## What it is

PXOAV implements the **EAV (Entity-Attribute-Value)** pattern in GeneXus. It lets you add **dynamic attributes** to any entity without changing the structure of the base transaction. Attributes are defined at runtime (not at design time) and can have different data types, validations, formulas and controls.

## Parent Objects

- `Transaction` — the transaction the dynamic attributes are added to

## Objects it generates

PXOAV generates more objects than any other pattern. From a single instance:

### GeneXus attributes (24+)

| Category | Generated attributes |
|----------|----------------------|
| **WRI values** (With Referential Integrity) | AttOAVAttributeValuesWRIId, ValuesWRIOAVAttributeId, ValuesWRIOAVAttributeValueCode, OAVAttributeValuesWRIAttributeName (×N), OAVAttributeValuesWRIAttributeDescription (×N) |
| **WORI values** (Without Referential Integrity) | AttOAVAttributeValuesWORIId, ValuesWORIOAVAttributeId, OAVAttributeValuesWORIString, OAVAttributeValuesWORIMemo, OAVAttributeValuesWORIBlob, OAVAttributeValuesWORIAttributeName (×N), OAVAttributeValuesWORIAttributeDescription (×N) |
| **Definition** | OAVAttributeDefinitionId, OAVAttributeDefinitionParentId, OAVAttributeDefinitionParentValue, OAVAttributeDefinitionOrder, OAVAttributeDefinitionFormula, OAVAttributeDefinitionFormulaOrder, OAVAttributeDefinitionDefaultId, OAVAttributeDefinitionCategory, OAVAttributeDefinitionRequired, OAVAttributeDefinitionReadOnly, OAVAttributeDefinitionDynamicReadOnlyId/URL, OAVAttributeDefinitionValidationId/URL, OAVAttributeDefinitionAttributeName/Description (×N) |
| **OAV subtype** | OAVAttributeSubtypeId, OAVAttributeSubtypeCode, OAVAttributeSubtypeDescription, OAVAttributeSubtypeLargeDescription, OAVAttributeSubtypeDataType, OAVAttributeSubtypeDataLength, OAVAttributeSubtypeDataDecimals, OAVAttributeSubtypeDataDescription, OAVAttributeSubtypeDataFrom/To, OAVAttributeSubtypeControlType, OAVAttributeSubtypeControlShowDescriptions, OAVAttributeSubtypeControlNullDescription, OAVAttributeSubtypeControlCheckBoxSelected/Unselected, OAVAttributeSubtypeOrderedValues, OAVAttributeSubtypeDefaultValues, OAVAttributeSubtypePicture, OAVAttributeSubtypeReferentialIntegrity, OAVAttributeSubtypeValidationId/URL, OAVAttributeSubtypePromptId/URL |

### Transactions (3)

| Object | Naming | Description |
|--------|--------|-------------|
| WRITransaction | `WRITransaction` | Values transaction with referential integrity |
| WORITransaction | `WORITransaction` | Values transaction without referential integrity |
| DefinitionTransaction | `DefinitionTransaction` | OAV attribute definition transaction |

### GeneXus Groups (9)

| Object | Description |
|--------|-------------|
| DefinitionOAVAttributeGroup | OAV subtype definition group |
| DefinitionValidationGroup | Validation group |
| DefinitionDynamicReadOnlyGroup | Dynamic read-only group |
| DefinitionDefaultGroup | Default values group |
| DefinitionAttributeGroup (×N) | One group per defined attribute |
| ValuesWRIAttributeGroup (×N) | WRI attributes group |
| ValuesWORIAttributeGroup (×N) | WORI attributes group |
| ValuesWRIOAVAttributeGroup | OAV group for WRI attributes |
| ValuesWORIOAVAttributeGroup | OAV group for WORI attributes |

### Procedures (10+)

| Object | Description |
|--------|-------------|
| ValuesDeleteAttributeValueProcedure | Delete one attribute value |
| ValuesAddAttributeValueDefaultProcedure | Add a default value |
| ValuesUpdateAttributesValuesProcedure | Update values |
| ValuesDeleteAttributeValuesProcedure | Delete several values |
| ValuesAddAttributeValueProcedure | Add a value |
| ValuesAddAttributeBlobProcedure | Add a blob value |
| ValuesAddAttributeStringProcedure | Add a string value |
| ValuesAddAttributeMemoProcedure | Add a memo value |
| + more Procedures for each data type… | |

## Core concepts

### The EAV model (Entity-Attribute-Value)

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│    Entity    │     │    Attribute     │     │    Attribute     │
│  (Customer)  │────►│    Definition    │────►│      Values      │
│              │     │   (OAV Attrs)    │     │   (OAV Values)   │
│  CustomerId  │     │  AttrId          │     │  EntityId        │
│  Name        │     │  Code            │     │  AttrId          │
│  ...         │     │  Description     │     │  Value           │
│              │     │  DataType        │     │  ValueCode (WRI) │
│              │     │  ControlType     │     │  String (WORI)   │
│              │     │  Required        │     │  Memo (WORI)     │
│              │     │  ReadOnly        │     │  Blob (WORI)     │
└──────────────┘     │  Validation      │     └──────────────────┘
                     │  Formula         │
                     │  DefaultValue    │
                     └──────────────────┘
```

### Two kinds of values

| Kind | Table | Description |
|------|-------|-------------|
| **WRI** (With Referential Integrity) | ValuesWRI | Values whose code references the definition table. Guarantees referential integrity. |
| **WORI** (Without Referential Integrity) | ValuesWORI | Free values (string, memo, blob) with no integrity checking. More flexible. |

### Supported data types

Dynamic attributes support several types through the `DataType` attribute:
- String, Memo, Blob (files)
- Numeric (with length and decimals)
- Date, DateTime, Boolean
- With a customizable picture

### Control types

Each attribute can use a different UI control through `ControlType`:
- Input text
- ComboBox (with ShowDescriptions)
- CheckBox (with Selected/Unselected values)
- Prompt (with PromptId/URL)

### Advanced features

- **Formulas**: computed attributes with an evaluation order
- **Validations**: dynamic validation rules (ValidationId/URL)
- **Dynamic ReadOnly**: conditional read-only (DynamicReadOnlyId/URL)
- **Default values**: value preloading (DefaultId)
- **Categories**: attribute grouping
- **Hierarchy**: parent-child attributes (ParentId/ParentValue)
- **Ordering**: display order (Order)
- **Required**: mandatory field

## Instance — main nodes

```
instance
├── definition
│   ├── DefinitionTransaction (definition transaction config)
│   └── Attributes
│       └── Attribute (×N, each entity attribute identifying the "object")
├── values
│   ├── ValuesWRITransaction (WRI table config)
│   ├── ValuesWORITransaction (WORI table config)
│   ├── DeleteAttributeValue (Procedure)
│   ├── AddAttributeValueDefault (Procedure)
│   ├── UpdateAttributesValues (Procedure)
│   ├── DeleteAttributeValues (Procedure)
│   ├── AddAttributeValue (Procedure)
│   ├── AddAttributeBlob (Procedure)
│   ├── AddAttributeString (Procedure)
│   └── AddAttributeMemo (Procedure)
```

## Real-world use

- The **@OAV** module provides the base infrastructure: TOAVAttributes, TOAVAttributeValues, TSystemObjectOAVAttributes with their PXWorkWith CRUD screens
- It is used to add customizable fields to entities without changing the database

## Integration with other patterns

- **PXWorkWith** can show/edit OAV attributes in View tabs
- **PXFlowController** can invoke PXOAV instances
- The **@OAV module** provides the base objects for managing definitions and values
