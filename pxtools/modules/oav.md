# @OAV Module — Object-Attribute-Value (Dynamic Attributes)

> Behaviour of the `@PXTools/@OAV` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@OAV` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.OAV`.
- **Depends on:** `@APIs` (base), `@DynamicCallReferences` (the `DynamicCallReferenceCode` domain for Prompt/Validation/DynamicReadOnly), `@Menus` (`RetMenusOAV`). In the canonical graph also `@System` (because of `TSystemObjects`), although in this KB it does not appear as a qualified reference.

## 1. What it provides

An **EAV (Entity-Attribute-Value)** pattern: it lets you add **dynamic attributes to any entity at runtime, without altering the base transaction's schema**. The `@OAV` module supplies the **definition infrastructure** (attribute catalogue, allowed values, per-object assignment); the **PXOAV** pattern — applied to a consuming transaction — generates the **per-instance value storage** plus the editor.

**Design intent** (in the PXTools manual's terms): to *dynamically associate attributes with system entities* in order to (1) **minimise structural impact** on the database when adding attributes, (2) **decouple the user from the developer** — new attributes can be defined without touching code — and (3) build **flexible, adaptable** systems (a *metadata-driven* approach: configuration over code). Pattern concepts: **Entities/System Objects**, **Attributes** (each with a **Data Type** + a **Control Type**), **Categories** (grouping definitions), **Classes** (value-based groupings enabling conditional definitions) and **Values Based On / Generators** (producing values conditionally or dynamically).

## 2. Core concept

Three layers:
1. **Attribute definition** (`TOAVAttributes`): logical name, data type, UI control, picture, validation, prompt.
2. **Allowed values** (`TOAVAttributeValues`): the "domain" of a combo/radio/choice attribute.
3. **Per-object assignment** (`TSystemObjectOAVAttributes` + `TSystemObjectOAVClasses`): which attributes hang off which system object, with what order/required/category/formula/read-only/default.

The **per-instance values** do NOT live here: PXOAV generates them in the consumer.

## 3. Transactions (the definition layer)

| Transaction | PK | Role |
|---|---|---|
| **TOAVAttributes** | `OAVAttributeId` (autonumber) | Master catalogue: `Code` (logical name), `DataType`, `DataLength/Decimals`, `ControlType`, `Picture`, `ReferentialIntegrity`, `Prompt`/`Validation` (dynamic references). |
| **TOAVAttributeValues** | `OAVAttributeId, OAVAttributeValueCode` | Allowed values (subordinate level): `Description`, `Order`, `Default`. |
| **TSystemObjectOAVClasses** | `SystemObjectName, ClassCode` | Groups attributes by "class"/set within an object. |
| **TSystemObjectOAVAttributes** | `SystemObjectName, ClassCode, OAVAttributeId` | **The assignment**: `Order`, `Category`, `Required`, `ReadOnly` (+ dynamic), `Validation`, `Formula`+`OrderFormula`, `Default`. |

**Relationship with `TSystemObjects` (@System):** `SystemObjectName` is an FK to `TSystemObjects`. An object is marked as an OAV declaration through the `SystemObjectOAVDeclaration` flag (the UI combos filter on `Where SystemObjectOAVDeclaration`).

**Diagram (definition):**
```
TSystemObjects (@System)  ──1:N──►  TSystemObjectOAVClasses ──1:N──► TSystemObjectOAVAttributes
   SystemObjectName                    (Name, ClassCode)                (Name, ClassCode, OAVAttributeId) ──► TOAVAttributes
                                                                                                              OAVAttributeId ──1:N──► TOAVAttributeValues
```

## 4. Module domains

> **Note (legacy domains in root):** every @OAV domain lives in the KB's **root `#Domains/`**, not in a module-level `#Domains/`. That is a historical consequence: the module was created in GeneXus versions (Evo1/2/3) predating module-scoped domains, so they were defined in the root. Conceptually they **belong to @OAV** — only @OAV objects reference them (through `DataType = '<domain>'`).

**Enumerated:**

| Domain | Type | Values | Role |
|---|---|---|---|
| **DataType** | Char(3) | Date=`DTE`, Character=`CHR`, Numeric=`NUM`, Memo=`MEM`, Blob=`BLO` | The attribute's data type (it picks which `OAVValue*` column is used). |
| **ControlType** | Char(3) | EditBox=`EDT`, CheckBox=`CHK`, DynamicComboBox=`DCB`, DynamicRadioButton=`DRB`, DynamicCheckBox=`DCH` | The UI control the attribute is rendered with. |
| **OAVValuesOrdered** | Char(3) | Key=`KEY`, User=`USR` | Ordering of the allowed-value list (by key or user-defined). |
| **Positions** | Char(20) | Up, Down, Left, Right | Position of the label/control in the editor. |

**Value domains (no enum):**

| Domain | Type | Role |
|---|---|---|
| **DataLength** | Numeric(3.0) | Declared length of the attribute's data. |
| **DataDecimals** | Numeric(2.0) | Declared decimals of the data. |
| **OAVAttributeType** | Char(20) | Descriptor of the attribute's type. |
| **OAVAttributeCategory** | Char(30) | Category/grouping of the attribute. |
| **OAVAttributePicture** | Char(20) | Picture / edit mask. |
| **OAVAttributeFormula** | LongVarChar(1024) | Formula expression (computed attribute). |
| **OAVAttributeOrder** | VarLen (Numeric(5.0)) | Display/processing order of the attribute. |
| **OAVValueString** | Char(500) | Value stored when `DataType = Character`. |
| **OAVValueNumeric** | Numeric(18.5) | Value stored when `DataType = Numeric`. |
| **OAVValueMemo** | LongVarChar(2048) | Value stored when `DataType = Memo`. |
| **OAVValueBlob** | Blob | Value stored when `DataType = Blob`. |

**Domains used from other modules:** `DynamicCallReferenceCode` (from [@DynamicCallReferences](dynamiccallreferences.md), for the Prompt/Validation/DynamicReadOnly references) and base types from `@APIs` (`Boolean`, `MaxStr`, `MaxMem`, `Links`, `Objetos`, `ObjectDescription`, `WindowType`, `IdFirstLevel`, …).

> ⚠️ **Do not confuse:** `OAVShow` (All/Pending) does **not** belong to @OAV despite the name — `@APIs` declares and uses it (FormState). `OAVAttributeSistemaId` is a **project** enum, not a framework one.

## 5. The Object-Attribute-Value mechanism

**Defining an attribute:** create it in `TOAVAttributes` (type through `DataType`; control through `ControlType`; for combo/radio, load the allowed values into `TOAVAttributeValues`). Then **assign** it to a system object in `TSystemObjectOAVAttributes` (order, required, category, formula, validation, dynamic read-only, default). The `OAVAttributeReferentialIntegrity` flag decides whether the value is stored **WRI** (with a code referencing the value list) or **WORI** (free string/memo/blob).

**Storing/reading a value per (object, attribute, instance):** this does NOT happen in `@OAV`. The **PXOAV** pattern generates the per-instance value tables on the consuming transaction:
- **WRI** (`<Entity>OAVWRIValues`): PK = the entity key + `OAVAttributeId`, storing `OAVAttributeValueCode` (FK to `TOAVAttributeValues`).
- **WORI** (`<Entity>OAVWORIValues`): stores free `ValueString`/`ValueMemo`/`ValueBlob`.

PXOAV also generates subtypes, `Add/Update/Delete AttributeValue` procedures and the editor. You read and write through those generated procedures, using `OAVEditorValues`/`OAVAttributeValue` as the transport and `RetOAVAttributeIdFromCode` to resolve an attribute by its code.

**Difference from @SystemParameters:** SystemParameters is **global/singleton** configuration (one value per parameter for the whole system). OAV is **per object instance** (N values, one per entity record).

## 6. APIs vs Personalized

- **`APIs/`** (core, not to be touched): the 4 transactions, the metadata/formula/validation helpers, `SaveOAVSystemObjects`, `RetOAVAttributeIdFromCode`, and the exchange SDTs.
- **`Personalized/`**: a single object — **`RetMenusOAV`** (DataProvider), which builds the "OAV" menu node (Attributes / Object Attributes). Everything else lives in `APIs/`.

## 7. Pattern instances

- **PXWorkWithTOAVAttributes** — WW for the attribute catalogue (Definition/Control/Advanced tabs).
- **PXWorkWithTOAVAttributeValues** — CRUD for the allowed-value list.
- **PXWorkWithTSystemObjectOAVAttributes** — WW for the object↔attribute assignment (insert/delete/reorder through procedures).
- **PXParameterRequestSaveOAVSystemObjects** — exposes `SaveOAVSystemObjects` as an endpoint.

## 8. Key procedures / APIs

| Object | `Parm()` | Purpose |
|---|---|---|
| `RetOAVAttributeIdFromCode` | `in: &Code; out: &OAVAttributeId` | Resolves the numeric Id from the logical code (the typical entry point). |
| `RetOAVAttributeControlTypeForSD` [WS] | `in: &Id; out: &IsEditBox, &IsDynamicComboBox, …` | Control flags for dynamic rendering. |
| `RetOAVAttributeTypeDataForSD` [WS] | `in: &Id; out: &IsCharacter, &IsNumeric…, &IsDate, &IsMemo` | Data-type flags. |
| `SaveOAVSystemObjects` | — | Registers/removes the OAV SystemObjects against @System. |

**Exchange SDTs** (`APIs/`): `OAVAttributeValue` (a unified typed value: Blob/Character/Date/Numeric/Memo), `OAVEditorValues` (a collection of Id+ValueCode/Memo/Blob — the "envelope" an OAV editor reads and writes), `OAVValidationResult` (Ok+ErrorMessage), `OAVFormulaOrderCollection`.

> **Key point:** `@OAV/APIs` provides only the **definition** (catalogue, allowed values, per-object assignment) and the metadata/validation helpers. Reading and writing the **per-instance value** is done through the procedures the **PXOAV** pattern generates on the consuming transaction.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [system.md](system.md) — `TSystemObjects` (OAV attributes hang off a SystemObject).
- [systemparameters.md](systemparameters.md) — global configuration (in contrast with per-instance OAV).
- [dynamiccallreferences.md](dynamiccallreferences.md) — Prompt/Validation/DynamicReadOnly are resolved by dynamic reference.
- The `PXOAV` pattern (it generates the WRI/WORI tables + the editor in the consumer) — documented separately with the other patterns.
