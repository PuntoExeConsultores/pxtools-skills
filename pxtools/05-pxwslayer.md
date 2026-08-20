# PXWSLayer — REST/SOAP API Orchestrator Pattern

## What it is

PXWSLayer is the **web service orchestrator** of PXTools. It generates an API layer exposing REST and/or SOAP methods, delegating the logic to the other WS patterns (PXWSQuery, PXWSData, PXWSTransaction) or to plain GeneXus objects. It supports generating **API objects with OpenAPI 3.0**.

## Parent Objects

- `Transaction` — generates services based on a transaction
- `(None)` — standalone services

## Objects it generates

| Object | GeneXus type | Naming | Condition |
|--------|--------------|--------|-----------|
| SOAPWS | Procedure | `SOAPTrnV1` | `instance/version` (if generateSOAP=True) |
| RESTMethod | Procedure | `RESTTrnV1Method` | `instance/version/methods/method` (if generateREST=True) |
| APIWS | API | `APIWS` | `instance/version` (if generateRESTAPIObject=True) |
| SDTWSMethodIn | SDT | `WSMethod{Trn}V{ver}In` | When `callType='GXObject'` |
| SDTWSMethodOut | SDT | `WSMethod{Trn}V{ver}Out` | When `callType='GXObject'` |

## XML structure of the instance

```xml
<instance parentTransaction="Customer"
          publicName="Customer"
          category="BackOffice">

  <version version="1"
           generateSOAP="True"
           generateREST="True"
           generateRESTAPIObject="True"
           generateRESTProcedureObject="True"
           generateOpenAPIInterface="True">

    <methods>
      <!-- Method delegating to PXWSQuery -->
      <method name="Query"
              callType="PXInstance"
              instanceObject="PXWSQueryCustomer"
              instanceVersion="1"
              instanceMethod="&lt;default&gt;"
              generateOpenAPIInterface="True">
      </method>

      <!-- Method delegating to PXWSTransaction -->
      <method name="Load"
              callType="PXInstance"
              instanceObject="PXWSTransactionCustomer"
              instanceVersion="1"
              instanceMethod="Load">
      </method>

      <!-- Method delegating to PXWSData -->
      <method name="GetData"
              callType="PXInstance"
              instanceObject="PXWSDataCustomer"
              instanceVersion="1"
              instanceMethod="&lt;default&gt;">
      </method>

      <!-- Method calling a GeneXus object directly -->
      <method name="CustomMethod"
              callType="GXObject"
              gxObject="PrcCustomerCustomMethod"
              actionPreviousCode="// code to run first">
        <objectParameters>
          <parameter name="CustomerId" />
        </objectParameters>
        <methodParameters>
          <in>
            <parameter name="RequestData" />
          </in>
          <out>
            <parameter name="ResponseData" />
          </out>
        </methodParameters>
        <variables>
          <variable name="RequestData" SDT="SDTRequest" publicName="request" />
          <variable name="ResponseData" SDT="SDTResponse" publicName="response" />
        </variables>
      </method>

      <!-- Method with inline code (Event) -->
      <method name="Ping"
              callType="Event"
              event="&amp;Result = 'OK'">
      </method>
    </methods>
  </version>
</instance>
```

## Instance properties

### Instance (root)

| Property | Type | Description |
|----------|------|-------------|
| `parentTransaction` | reference(Transaction) | Base transaction for keys and multi-tenancy |
| `publicName` | string | Public name in the web service |
| `category` | custom(WSCategory) | Security category of the WS |

### Version

| Property | Type | Description |
|----------|------|-------------|
| `version` | int | Version number |
| `generateSOAP` | enum{default;True;False} | Generate the SOAP service |
| `generateREST` | enum{default;True;False} | Generate the REST service |
| `generateRESTAPIObject` | enum{default;True;False} | Generate the API object (OpenAPI) |
| `generateRESTProcedureObject` | enum{default;True;False} | Generate the REST Procedure |
| `generateOpenAPIInterface` | enum{default;True;False} | Generate the OpenAPI interface |

### Method

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Public name of the method |
| `callType` | enum{PXInstance;GXObject;Event} | Kind of invocation |
| `instanceObject` | reference(PXWSTransaction;PXWSQuery;PXWSData) | PXTools instance to invoke |
| `instanceVersion` | string | Version of that instance |
| `instanceMethod` | string | Method of that instance |
| `gxObject` | reference(Procedure;DataProvider) | Plain GeneXus object |
| `actionPreviousCode` | code | Code run before executing the GXObject |
| `event` | code | Inline code of the method (callType=Event) |
| `generateOpenAPIInterface` | enum{default;True;False} | OpenAPI for this method |

## Relationship with the other WS patterns

```
                    PXWSLayer (API Object)
                    ┌─────────────────────┐
                    │  version 1           │
                    │  ┌─────────────────┐ │
                    │  │ methods          │ │
                    │  │                 │ │
        ┌───────────┤  │ Query ──────────┼─┼──► PXWSQuery
        │           │  │ Load ───────────┼─┼──► PXWSTransaction (Load)
        │           │  │ Save ───────────┼─┼──► PXWSTransaction (Save)
        │           │  │ Delete ─────────┼─┼──► PXWSTransaction (Delete)
        │           │  │ GetData ────────┼─┼──► PXWSData
        │           │  │ Custom ─────────┼─┼──► GeneXus Procedure
        │           │  │ Ping ───────────┼─┼──► Event (inline code)
        │           │  └─────────────────┘ │
        │           └─────────────────────┘
        │
        ▼
  Generates: API Object + REST Procedures + SOAP Procedure + SDTs
```

## Variables in methods

When `callType="GXObject"`, you declare variables that map to the In/Out SDTs:

```xml
<variables>
  <variable name="CustomerData"
            publicName="data"
            description="Customer information"
            SDT="SDTCustomer"
            SDTStructure="&lt;root&gt;"
            collection="false" />
</variables>
```

Supported variable types: Audio, Bitmap, Blob, Boolean, Character, Date, DateTime, Long Varchar, Numeric, VarChar, or types based on a Domain, Attribute, SDT, ExternalObject or BusinessComponent.

## Versioning

PXWSLayer supports **API versioning** natively. Each `version` can have different methods, generators and settings, which lets the API evolve without breaking existing consumers:

```xml
<version version="1" generateREST="True" generateSOAP="True">
  <!-- v1 methods -->
</version>
<version version="2" generateREST="True" generateSOAP="False">
  <!-- v2 methods (REST only, no SOAP) -->
</version>
```

## Settings (PXWSLayerSettings.xml)

Global configuration for every PXWSLayer instance:
- Default generation (SOAP, REST, API Object, OpenAPI)
- Naming prefixes
- Per-category security configuration

## Integration with the @PXTools modules

- **@WSLayer**: holds the white list of allowed IPs/domains (PXWorkWithWSWhiteList)
- **@WebServicesLog**: log and statistics of WS invocations
- **@Security**: access control by WS category
