# PXWSQuery — Web Service Data Query Pattern

## What it is

PXWSQuery generates **DataProviders and Procedures** for data queries with filtering, ordering, paging and search. It is the pattern for exposing lists/queries as web services. It is normally invoked from PXWSLayer as a "Query" method.

## Parent Objects

- `Transaction` — query based on the transaction's extended table
- `(None)` — standalone query

## Objects it generates

For each `version/method`:

| Object | GeneXus type | Naming | Description |
|--------|--------------|--------|-------------|
| WSQueryOrder | Domain | `WSQuery{Trn}V{ver}` | Enumerated domain for the ordering options |
| SDTDPQueryIn | SDT | `DPQuery{Trn}V{ver}In` | DataProvider input SDT |
| SDTDPQueryOut | SDT | `DPQuery{Trn}V{ver}Out` | DataProvider output SDT |
| DataProvider | DataProvider | `DPQuery{Trn}V{ver}` | The DataProvider holding the query |
| SDTWSQueryIn | SDT | `WSQuery{Trn}V{ver}In` | WS Procedure input SDT |
| SDTWSQueryOut | SDT | `WSQuery{Trn}V{ver}Out` | WS Procedure output SDT |
| Procedure | Procedure | `WSQuery{Trn}V{ver}` | Procedure wrapping the DataProvider |

Total: **7 objects** per method per version.

## XML structure of the instance

```xml
<instance parentTransaction="Customer"
          privateName="Customer"
          category="BackOffice">

  <version version="1">
    <method name="&lt;default&gt;"
            maxRows="100"
            includeContext="false">

      <!-- Multi-tenant attributes (automatic filters) -->
      <multiTenantAttributes>
        <multiTenantAttribute
            attribute="CompanyId"
            returnConnectionProcedure="PrcReturnCompanyId" />
      </multiTenantAttributes>

      <!-- Output fields -->
      <fields>
        <attribute name="CustomerId" publicName="id" description="Customer ID" />
        <attribute name="CustomerName" publicName="name" description="Name" />
        <attribute name="CustomerEmail" publicName="email" description="Email" />
        <!-- Computed variable -->
        <variable name="FullAddress" publicName="fullAddress"
                  description="Full address"
                  dataType="VarChar" length="500"
                  loadPreviousCode="// code to run first"
                  loadCode="&amp;FullAddress = CustomerStreet + ', ' + CustomerCity" />
      </fields>

      <!-- Filters -->
      <filters>
        <!-- Simple search (free text) -->
        <search>
          <attribute name="CustomerName" description="Name" />
          <attribute name="CustomerEmail" description="Email" />
        </search>

        <!-- Advanced search (specific filters) -->
        <advancedSearch>
          <attribute name="CustomerName"
                     publicName="name"
                     description="Name"
                     type="Like"
                     precision="Flexible"
                     isRequired="false"
                     allowNullValue="false" />
          <attribute name="CustomerCity"
                     publicName="city"
                     description="City"
                     type="Equal"
                     isRequired="false"
                     collection="true" />    <!-- collection=true filters by several values -->
          <attribute name="CustomerCreatedDate"
                     publicName="createdDate"
                     description="Created Date"
                     type="Range"
                     precision="Strict"
                     isRequired="false" />
        </advancedSearch>

        <!-- Fixed conditions -->
        <conditions>
          <condition value="CustomerActive = true" />
        </conditions>
      </filters>

      <!-- Orders -->
      <orders generateFieldsOrder="True">
        <order name="ByName" condition="">
          <orderAttribute name="CustomerName" description="Name" ascending="true" />
        </order>
        <order name="ByDate" condition="">
          <orderAttribute name="CustomerCreatedDate" description="Created" ascending="false" />
        </order>
      </orders>

      <!-- Sort (ordering of the output fields) -->
      <sort>
        <sortField name="CustomerName" ascending="true" />
      </sort>

      <!-- Code hooks -->
      <codes>
        <code type="Start" data="// startup code" />
        <code type="Subroutine" name="ValidateAccess" data="// check access" />
      </codes>

      <!-- Extra variables -->
      <variables>
        <variable name="TotalRecords"
                  destination="Procedure"
                  dataType="Numeric" length="10" />
      </variables>
    </method>
  </version>
</instance>
```

## Filter types

| Type | Description | Generates |
|------|-------------|-----------|
| `Equal` | Exact equality | `WHERE Attr = &Value` |
| `Range` | From-to range | `WHERE Attr >= &From AND Attr <= &To` |
| `Like` | Partial search | `WHERE Attr LIKE '%' + &Value + '%'` |

### Filter precision

| Precision | On Range | On Like |
|-----------|----------|---------|
| `Strict` | Exact range | Exact pattern match |
| `Flexible` | Appends "Z" to the "To" value | Adds "%" before and after |
| `<default>` | Uses the Settings configuration | Uses the Settings configuration |

### Filter properties

| Property | Description |
|----------|-------------|
| `isRequired` | Whether it is mandatory to run the query |
| `allowNullValue` | For Numeric/Boolean, allows a null value meaning "no filter" |
| `collection` | Allows filtering by several values (IN clause) |
| `whenExtraCondition` | Extra condition attached to the filter |
| `generateCondition` | Whether the condition is generated automatically in the DataProvider |

## Multi-tenancy

PXWSQuery supports **multi-tenancy** natively. `multiTenantAttributes` are automatic filters added to every query without the consumer ever seeing them:

```xml
<multiTenantAttributes>
  <multiTenantAttribute
      attribute="CompanyId"
      returnConnectionProcedure="PrcReturnCompanyId" />
</multiTenantAttributes>
```

`returnConnectionProcedure` is a Procedure that obtains the multi-tenant attribute's value from the current connection/session.

## Computed variables in Fields

Variables inside `fields` are computed fields that do not exist in the database:

```xml
<variable name="TotalAmount"
          publicName="totalAmount"
          dataType="Numeric" length="12" decimals="2"
          loadPreviousCode="// code run before the load"
          loadCode="&amp;TotalAmount = Sum(InvoiceAmount)" />
```

- `loadPreviousCode`: code executed before assigning the value
- `loadCode`: code that computes and assigns the variable's value

## Variable destination

Variables declared in the `variables` node (not in `fields`) carry a `destination` property:

| destination | Created in |
|-------------|------------|
| `DataProvider` | Inside the generated DataProvider |
| `Procedure` | Inside the generated wrapper Procedure |

## Automatic ordering

When `generateFieldsOrder="True"`, PXWSQuery automatically generates one ordering option per output field, in addition to the manually declared orders. It produces an **enumerated Domain** holding every possible order.

## How to declare the `<orders>`

Each `<order>` translates literally into an `Order` clause of the generated DataProvider, so the same performance rules as any `For Each` apply:

**1. Always start with the attributes filtered by equality.** First comes the multi-tenant attribute (or attributes) from the Layer Settings, which the pattern filters on every query (`Where CompanyId = &CompanyId`). Then the instance's own `type="Equal"` filters — typically the parent key in a query over a subordinate level — and only at the end the attribute you actually want to sort by.

```xml
<order name="ByEffectiveDate">
  <orderAttribute name="CompanyId" description="Company" />
  <orderAttribute name="ProductId" description="Product code" />
  <orderAttribute name="ProductPriceEffectiveDate" description="Effective date" ascending="False" />
</order>
```

**2. Check that an index backs that order.** The `<order>`'s attribute sequence has to be a prefix of some index on the base table. Verify it by reading `Knowledge Base/#Tables/<Table>.Table.gxSource`, section `#Indexes`. If there is none, either drop the order or create a user index — and creating an index is a database reorganization, so it is the KB owner's decision, not something to add in passing. Ordering by an attribute of the **extended table** (the descriptor of a foreign key) is never backed: order by the foreign key instead.

**3. The same set of attributes cannot be used in two orders.** The enumerated domain the pattern generates uses **the list of attribute names** as each value's `Description`. Two orders over the same attributes (for instance the same attribute ascending and descending) produce two values with identical descriptions and the apply fails with `Failed processing Domain '<WSQuery…Order>' properties`. If you need both directions over the same attribute, declare a single order with the more useful direction.

## Hook codes

| type | Target | When |
|------|--------|------|
| `Start` | DataProvider/Procedure | At the start |
| `Subroutine` | DataProvider/Procedure | Callable by name |

## Relationship with PXWSLayer

PXWSQuery is normally invoked from a PXWSLayer method:

```
PXWSLayer (method callType="PXInstance")
    │
    └──► PXWSQuery (DataProvider + Procedure + SDTs)
```
