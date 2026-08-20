# PXWSData — Web Service Data Read Pattern

## What it is

PXWSData generates **data-read Procedures** with customizable code hooks. Unlike PXWSQuery (which generates DataProviders for paged lists), PXWSData generates Procedures that read one specific record with custom logic. It is used for "Get" or "Read" operations on individual entities.

## Parent Objects

- `Transaction` — read based on the transaction's keys
- `(None)` — standalone read

## Objects it generates

For each `version/method`:

| Object | GeneXus type | Naming | Description |
|--------|--------------|--------|-------------|
| SDTData | SDT | `WSTransaction{Trn}V{ver}Structure` | Shared data structure |
| MethodData | Procedure | `WSData{Trn}V{ver}Method` | The read Procedure |
| SDTDataIn | SDT | `WSData{Trn}V{ver}DataIn` | Input SDT |
| SDTDataOut | SDT | `WSData{Trn}V{ver}DataOut` | Output SDT |

Total: **4 objects** per method per version.

## XML structure of the instance

```xml
<instance parentTransaction="Customer"
          privateName="Customer"
          category="BackOffice">

  <version version="1">
    <method name="&lt;default&gt;">

      <data>
        <!-- Input keys -->
        <key name="CustomerId"
             publicName="id"
             type="Public" />           <!-- Public: visible to the consumer -->
        <key name="CompanyId"
             publicName=""
             type="Multitenant" />      <!-- Multitenant: resolved from the connection -->

        <!-- Output attributes -->
        <attribute name="CustomerName"
                   publicName="name"
                   description="Customer Name" />
        <attribute name="CustomerEmail"
                   publicName="email"
                   description="Email" />
        <attribute name="CustomerAddress"
                   publicName="address"
                   description="Address" />

        <!-- Computed variables -->
        <variable name="TotalInvoices"
                  publicName="totalInvoices"
                  description="Total invoices count"
                  dataType="Numeric" length="10"
                  loadPreviousCode="// code to run first"
                  loadCode="&amp;TotalInvoices = Count(InvoiceId)" />
      </data>

      <!-- Code hooks -->
      <codes>
        <code type="Start" data="// initialization" />
        <code type="Subroutine" name="ValidateAccess"
              data="// check the user's permissions" />
      </codes>

      <!-- Extra Procedure variables -->
      <variables>
        <variable name="HasAccess"
                  dataType="Boolean"
                  description="Access flag" />
      </variables>
    </method>
  </version>
</instance>
```

## The Data node — data structure

The `data` node defines what gets read. It accepts three kinds of children, mixed together (ChildrenType="Mixed"):

### Keys

```xml
<key name="CustomerId" publicName="id" type="Public" />
```

| type | Description |
|------|-------------|
| `Public` | Visible in the input SDT, supplied by the consumer |
| `Multitenant` | Not visible, obtained automatically from the connection/session |
| `Private` | Internal, not exposed to the consumer |

### Attributes

Transaction attributes included in the response:

```xml
<attribute name="CustomerName" publicName="name" description="Name" />
```

Only attributes of the transaction's **base table** should be included.

### Computed variables

Variables with custom load code:

```xml
<variable name="FullName"
          publicName="fullName"
          loadPreviousCode="// code run before loading"
          loadCode="&amp;FullName = CustomerFirstName + ' ' + CustomerLastName" />
```

## Difference from PXWSQuery

| Aspect | PXWSQuery | PXWSData |
|--------|-----------|----------|
| **Purpose** | Paged lists with filters | Reading one record |
| **Generates** | DataProvider + Procedure | Procedure only |
| **Filters** | Search, AdvancedSearch, Conditions | Keys only |
| **Ordering** | Yes (orders, sort) | No |
| **Paging** | Yes (maxRows) | No |
| **Computed fields** | Yes (loadCode on fields) | Yes (loadCode on data) |
| **Multi-tenant** | Yes (separate node) | Yes (key type=Multitenant) |

## Code hooks

| type | When it runs |
|------|--------------|
| `Start` | At the start of the Procedure |
| `Subroutine` | Subroutine callable by name from the code |

## Relationship with PXWSLayer

PXWSData is normally invoked from a PXWSLayer method:

```
PXWSLayer (method callType="PXInstance")
    │
    └──► PXWSData (Procedure + SDTs)
```

## Relationship with PXWSTransaction

PXWSData and PXWSTransaction share the same SDT structure (`WSTransaction{Trn}V{ver}Structure`), which means the same data format serves both reading and writing.
