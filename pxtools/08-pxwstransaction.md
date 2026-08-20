# PXWSTransaction — Web Service Transaction CRUD Pattern

## What it is

PXWSTransaction generates **Load, Save and Delete Procedures** over GeneXus transactions using **Business Components**. It is the pattern for exposing CRUD (Create, Read, Update, Delete) operations on entities as web services.

## Parent Objects

- `Transaction` — the transaction the CRUD operations are generated for
- `(None)` — standalone

## Objects it generates

For each `version`:

| Object | GeneXus type | Naming | Condition |
|--------|--------------|--------|-----------|
| SDTStructure | SDT | `WSTransaction{Trn}V{ver}Structure` | Always |
| MethodLoad | Procedure | `WSTransaction{Trn}V{ver}Load` | Always |
| SDTLoadIn | SDT | `WSTransaction{Trn}V{ver}LoadIn` | Always |
| SDTLoadOut | SDT | `WSTransaction{Trn}V{ver}LoadOut` | Always |
| MethodSave | Procedure | `WSTransaction{Trn}V{ver}Save` | Only if `modeInsert=True` or `modeUpdate=True` |
| SDTSaveIn | SDT | `WSTransaction{Trn}V{ver}SaveIn` | Always (structure) |
| SDTSaveOut | SDT | `WSTransaction{Trn}V{ver}SaveOut` | Always (structure) |
| MethodDelete | Procedure | `WSTransaction{Trn}V{ver}Delete` | Only if `modeDelete=True` |
| SDTDeleteIn | SDT | `WSTransaction{Trn}V{ver}DeleteIn` | Always (structure) |
| SDTDeleteOut | SDT | `WSTransaction{Trn}V{ver}DeleteOut` | Always (structure) |

Total: up to **10 objects** per version (3 Procedures + 7 SDTs).

## XML structure of the instance

```xml
<instance parentTransaction="Customer"
          category="BackOffice">

  <version version="1"
           modeInsert="true"
           modeUpdate="true"
           modeDelete="true">

    <structure name="Customer">
      <!-- Keys -->
      <key name="CustomerId"
           publicName="id"
           description="Customer ID"
           type="Public" />
      <key name="CompanyId"
           publicName=""
           description=""
           type="Multitenant" />

      <!-- Attributes of the main level -->
      <attribute name="CustomerName"
                 publicName="name"
                 description="Customer Name"
                 updateAttribute="Always" />
      <attribute name="CustomerEmail"
                 publicName="email"
                 description="Email"
                 updateAttribute="Always" />
      <attribute name="CustomerStatus"
                 publicName="status"
                 description="Status"
                 updateAttribute="On Insert" />

      <!-- Sub-level (transaction detail) -->
      <level name="CustomerAddresses">
        <key name="CustomerAddressId"
             publicName="addressId"
             description="Address ID"
             type="Public" />
        <attribute name="CustomerAddressStreet"
                   publicName="street"
                   description="Street"
                   updateAttribute="Always" />
        <attribute name="CustomerAddressCity"
                   publicName="city"
                   description="City"
                   updateAttribute="Always" />
      </level>
    </structure>
  </version>
</instance>
```

## Version properties

| Property | Type | Description |
|----------|------|-------------|
| `version` | string | Version number |
| `modeInsert` | bool | Enable insert (affects Save generation) |
| `modeUpdate` | bool | Enable update (affects Save generation) |
| `modeDelete` | bool | Enable delete (generates the Delete Procedure) |

## Multi-level structure

PXWSTransaction supports transactions with **nested levels** (master-detail). Each `level` inside `structure` represents one level of the transaction:

```
Customer (main level)
├── key: CustomerId
├── attribute: CustomerName
├── attribute: CustomerEmail
│
└── CustomerAddresses (sub-level)
    ├── key: CustomerAddressId
    ├── attribute: Street
    └── attribute: City
```

Sub-levels can only be added inside the `structure` node (root), enforced by `CanAddIf="self::structure"`.

## Keys — types

| type | Description | In Load | In Save | In Delete |
|------|-------------|---------|---------|-----------|
| `Public` | Visible to the consumer | SDT In | SDT In | SDT In |
| `Multitenant` | Automatic from the connection | Internal | Internal | Internal |
| `Private` | Internal, not exposed | Internal | Internal | Internal |

## Attributes — updateAttribute

| Value | Description |
|-------|-------------|
| `Always` | Updated on both Insert and Update |
| `On Insert` | Assigned only on Insert, ignored on Update |
| `On Update` | Assigned only on Update, ignored on Insert |

This controls which fields are editable in each mode without any custom code.

## Generated operations

### Load
- Reads one record by its public keys
- Input: SDTLoadIn (public keys)
- Output: SDTLoadOut (full structure + messages)

### Save
- Inserts or updates depending on whether the record exists
- Input: SDTSaveIn (full structure)
- Output: SDTSaveOut (result + error messages)
- Uses a GeneXus **Business Component** internally

#### The Save is a FULL REPLACE, not a patch

This is the pattern's most easily misread characteristic, and getting it wrong destroys data without leaving a trace. The generated code does `BC.Load(...)` and **then assigns every attribute** from the input SDT:

- An attribute that does not travel in the SDT **is written empty**. Its previous value is not kept.
- In each subordinate level, after the insert-or-update of the items received, a `// Delete <Level>` block **removes the rows that did not come in the collection**.

Put differently: **the input SDT is the complete desired state, not a delta.**

The consequence for any consumer — a REST API, a screen, a batch task, an assistant tool alike — is that the only safe way to modify is

```
Load  →  apply the changes over what was loaded  →  Save
```

A `Save` assembled from scratch with the two or three fields somebody wanted to change **empties the rest of the record and wipes out its subordinate levels**. And it does so silently: no error, no warning, and the operation returns `Succeed = True`.

One concrete way data gets lost without anyone asking: if the consumer decides which attributes to touch by checking whether the incoming value is empty, it confuses *"they did not send it to me"* with *"they want it empty"*. The **absence** of a field must be told apart from an empty value — sending an empty field is a legitimate way to clear it, not sending it is not.

### Delete
- Deletes a record by its keys
- Input: SDTDeleteIn (keys)
- Output: SDTDeleteOut (result + messages)
- Uses a GeneXus **Business Component** internally

## Shared Structure SDT

The `WSTransaction{Trn}V{ver}Structure` SDT is shared between Load, Save and Delete. It is also shared with PXWSData if both patterns are applied to the same transaction and version.

## Relationship with PXWSLayer

PXWSTransaction is normally invoked from PXWSLayer methods:

```
PXWSLayer
├── method "Load"   ──► PXWSTransaction (Load)
├── method "Save"   ──► PXWSTransaction (Save)
└── method "Delete" ──► PXWSTransaction (Delete)
```

## How the four WS patterns relate

```
┌────────────────────────────────────────────────────┐
│                  PXWSLayer                          │
│              (API orchestrator)                     │
│                                                    │
│  "Query"  method ──► PXWSQuery (paged lists)       │
│  "Get"    method ──► PXWSData (single read)        │
│  "Load"   method ──► PXWSTransaction.Load          │
│  "Save"   method ──► PXWSTransaction.Save          │
│  "Delete" method ──► PXWSTransaction.Delete        │
│  custom   method ──► GX Procedure/DataProvider     │
│  inline   method ──► Event (code written inline)   │
└────────────────────────────────────────────────────┘
```

## Security category

The `category` property holds a value of the `WSCategory` domain, used to control access to the services. It integrates with the PXTools **@Security** module to check permissions before running the operations.
