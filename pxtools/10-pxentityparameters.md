# PXEntityParameters — Per-Entity Configurable Parameters Pattern

## What it is

PXEntityParameters generates a **complete infrastructure of configurable parameters** per entity. It lets you define typed parameters (string, numeric, date, boolean, memo, password) stored as key-value pairs, with multi-language support and specialised UI controls (Combo, Chosen).

## Parent Objects

- `Transaction` — the transaction holding the parameters

## Objects it generates

A single instance generates a very large set of objects:

### Domains (2)

| Object | Naming | Description |
|--------|--------|-------------|
| DomainEntityParameterCode | `EntityParameterCode` | Domain for parameter codes |
| DomainEntityParameterCategory | `EntityParameterCategory` | Domain for categories |

### Attributes (7)

| Object | Naming | Description |
|--------|--------|-------------|
| AttributeEntityParameterCode | `EntityParameterCode` | Parameter code |
| AttributeEntityParameterDescription | `EntityParameterDescription` | Description |
| AttributeEntityParameterType | `EntityParameterType` | Data type |
| AttributeEntityParameterCategory | `EntityParameterCategory` | Category |
| AttributeEntityParameterMultiLanguage | `EntityParameterMultiLanguage` | Multi-language flag |
| AttributeEntityParameterPreferenceCode | `EntityParameterPreferenceCode` | Preference code |
| + 3 more for preferences | | Language, Value, MemoValue |

### Transactions (3)

| Object | Naming | Description |
|--------|--------|-------------|
| TransactionEntityParameters | `EntityParameters` | Parameter definitions |
| TransactionEntityParametersPreferences | `EntityParametersPreferences` | Preference values |
| TransactionEntityParametersPreferencesComponent | `EntityParametersPreferencesComponent` | Preferences component |

### Groups (1)

| Object | Naming | Description |
|--------|--------|-------------|
| GroupEntityParametersPreferences | `EntityParametersPreferences` | Structure group |

### SDTs (1)

| Object | Naming | Description |
|--------|--------|-------------|
| SDTEntityParameters | `SDTEntityParameters` | SDT for programmatic handling |

### Procedures (14+)

| Object | Description |
|--------|-------------|
| ProcedureAddEntityParameters | Add several parameters |
| ProcedureReturnEntityParameters | Get an entity's parameters |
| ProcedureAddEntityParameter | Add a single parameter |
| ProcedureCheckEntityParametersExistence | Check existence |
| ProcedureReturnEntityParameterType | Get a parameter's type |
| ProcedureReturnEntityParameterDescription | Get the description |
| ProcedureReturnEntityParameterPreferenceBoolean | Get a boolean value |
| ProcedureReturnEntityParameterPreferenceDate | Get a date value |
| ProcedureReturnEntityParameterPreferenceInteger | Get an integer value |
| ProcedureReturnEntityParameterPreferenceMemo | Get a memo value |
| ProcedureReturnEntityParameterPreferencePassword | Get a password value |
| ProcedureReturnEntityParameterPreferenceString | Get a string value |
| ProcedureReturnEntityParameterPreferenceLanguage* | Multi-language versions (×6) |
| ProcedureUpdateEntityParameterPreferenceChosen | Update a Chosen value |
| ProcedureReturnEntityParameterComboValues | Get combo values |

### DataProviders (2)

| Object | Description |
|--------|-------------|
| DataProviderReturnEntityParametersGeneral | General parameters DP |
| DataProviderReturnEntityParameterChosenValues | Chosen values DP |

Total: **30+ objects** from a single instance.

## Data model

```
┌──────────────────────┐
│  EntityParameters    │  ← Parameter definitions
│  ────────────────    │
│  ParameterCode (PK)  │
│  Description         │
│  Type                │  ← String|Numeric|Date|Boolean|Memo|Password
│  Category            │
│  MultiLanguage       │  ← Multi-language support
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────┐
│  EntityParametersPreferences │  ← The value of each parameter
│  ────────────────────────    │
│  EntityId (PK)               │  ← The entity it belongs to
│  ParameterCode (PK, FK)      │
│  Language (PK)               │  ← For multi-language
│  Value                       │  ← String value
│  MemoValue                   │  ← Memo value (long text)
└──────────────────────────────┘
```

## Parameter types

| Type | Read Procedure | Storage |
|------|----------------|---------|
| `String` | ReturnEntityParameterPreferenceString | Value field |
| `Integer` | ReturnEntityParameterPreferenceInteger | Value field (parsed) |
| `Date` | ReturnEntityParameterPreferenceDate | Value field (parsed) |
| `Boolean` | ReturnEntityParameterPreferenceBoolean | Value field ("True"/"False") |
| `Memo` | ReturnEntityParameterPreferenceMemo | MemoValue field |
| `Password` | ReturnEntityParameterPreferencePassword | Value field (encrypted) |

## Multi-language support

When `MultiLanguage=true`, each parameter can hold a different value per language. The `*Language*` Procedures handle reading and writing with the extra `Language` parameter.

## UI controls

- **Combo**: predefined selectable values (ReturnEntityParameterComboValues)
- **Chosen**: advanced multi-selector (ReturnEntityParameterChosenValues, UpdateEntityParameterPreferenceChosen)

## Real-world use

- The **@SystemParameters** module uses this pattern for global system parameters (PXWorkWithSystemParametersPreferences)

## Difference from @SystemParameters

| Aspect | PXEntityParameters (pattern) | @SystemParameters (module) |
|--------|------------------------------|----------------------------|
| Scope | Parameters per entity | Global system parameters |
| Attachment | Bound to a Transaction | Standalone module |
| Customization | Generates a custom structure | Predefined structure |
