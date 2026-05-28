# PXWSLayer — Pattern Orquestador de API REST/SOAP

## Qué es

PXWSLayer es el **orquestador de servicios web** de PXTools. Genera una capa de API que expone métodos REST y/o SOAP, delegando la lógica a otros patterns de WS (PXWSQuery, PXWSData, PXWSTransaction) o a objetos GeneXus directos. Soporta generación de objetos **API con OpenAPI 3.0**.

## Parent Objects

- `Transaction` — genera servicios basados en una transacción
- `(None)` — servicios independientes

## Objetos que genera

| Objeto | Tipo GeneXus | Naming | Condición |
|--------|-------------|--------|-----------|
| SOAPWS | Procedure | `SOAPTrnV1` | `instance/version` (si generateSOAP=True) |
| RESTMethod | Procedure | `RESTTrnV1Method` | `instance/version/methods/method` (si generateREST=True) |
| APIWS | API | `APIWS` | `instance/version` (si generateRESTAPIObject=True) |
| SDTWSMethodIn | SDT | `WSMethod{Trn}V{ver}In` | Cuando `callType='GXObject'` |
| SDTWSMethodOut | SDT | `WSMethod{Trn}V{ver}Out` | Cuando `callType='GXObject'` |

## Estructura XML de la instancia

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
      <!-- Método que delega a PXWSQuery -->
      <method name="Query"
              callType="PXInstance"
              instanceObject="PXWSQueryCustomer"
              instanceVersion="1"
              instanceMethod="&lt;default&gt;"
              generateOpenAPIInterface="True">
      </method>

      <!-- Método que delega a PXWSTransaction -->
      <method name="Load"
              callType="PXInstance"
              instanceObject="PXWSTransactionCustomer"
              instanceVersion="1"
              instanceMethod="Load">
      </method>

      <!-- Método que delega a PXWSData -->
      <method name="GetData"
              callType="PXInstance"
              instanceObject="PXWSDataCustomer"
              instanceVersion="1"
              instanceMethod="&lt;default&gt;">
      </method>

      <!-- Método que llama a un objeto GeneXus directamente -->
      <method name="CustomMethod"
              callType="GXObject"
              gxObject="PrcCustomerCustomMethod"
              actionPreviousCode="// código previo">
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

      <!-- Método con código inline (Event) -->
      <method name="Ping"
              callType="Event"
              event="&amp;Result = 'OK'">
      </method>
    </methods>
  </version>
</instance>
```

## Propiedades de la instancia

### Instance (raíz)

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `parentTransaction` | reference(Transaction) | Transacción base para keys y multi-tenant |
| `publicName` | string | Nombre público en el servicio web |
| `category` | custom(WSCategory) | Categoría de seguridad del WS |

### Version

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `version` | int | Número de versión |
| `generateSOAP` | enum{default;True;False} | Generar servicio SOAP |
| `generateREST` | enum{default;True;False} | Generar servicio REST |
| `generateRESTAPIObject` | enum{default;True;False} | Generar objeto API (OpenAPI) |
| `generateRESTProcedureObject` | enum{default;True;False} | Generar Procedure REST |
| `generateOpenAPIInterface` | enum{default;True;False} | Generar interfaz OpenAPI |

### Method

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `name` | string | Nombre público del método |
| `callType` | enum{PXInstance;GXObject;Event} | Tipo de invocación |
| `instanceObject` | reference(PXWSTransaction;PXWSQuery;PXWSData) | Instancia PXTools a invocar |
| `instanceVersion` | string | Versión de la instancia |
| `instanceMethod` | string | Método de la instancia |
| `gxObject` | reference(Procedure;DataProvider) | Objeto GeneXus directo |
| `actionPreviousCode` | code | Código previo a la ejecución del GXObject |
| `event` | code | Código inline del método (callType=Event) |
| `generateOpenAPIInterface` | enum{default;True;False} | OpenAPI para este método |

## Relación con otros patterns WS

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
        │           │  │ Custom ─────────┼─┼──► Procedure GeneXus
        │           │  │ Ping ───────────┼─┼──► Event (código inline)
        │           │  └─────────────────┘ │
        │           └─────────────────────┘
        │
        ▼
  Genera: API Object + Procedures REST + Procedure SOAP + SDTs
```

## Variables en métodos

Cuando `callType="GXObject"`, se definen variables que se mapean a SDTs In/Out:

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

Tipos de variable soportados: Audio, Bitmap, Blob, Boolean, Character, Date, DateTime, Long Varchar, Numeric, VarChar, o basados en Domain, Attribute, SDT, ExternalObject, BusinessComponent.

## Versionamiento

PXWSLayer soporta **versionamiento de API** nativo. Cada `version` puede tener diferentes métodos, generadores y configuraciones. Esto permite evolucionar la API sin romper consumidores existentes:

```xml
<version version="1" generateREST="True" generateSOAP="True">
  <!-- Métodos v1 -->
</version>
<version version="2" generateREST="True" generateSOAP="False">
  <!-- Métodos v2 (solo REST, sin SOAP) -->
</version>
```

## Settings (PXWSLayerSettings.xml)

Configuración global para todas las instancias PXWSLayer:
- Generación por defecto (SOAP, REST, API Object, OpenAPI)
- Prefijos de nombrado
- Configuración de seguridad por categoría

## Integración con módulos @PXTools

- **@WSLayer**: Contiene WhiteList de IPs/dominios permitidos (PXWorkWithWSWhiteList)
- **@WebServicesLog**: Log y estadísticas de invocaciones de WS
- **@Security**: Control de acceso por categoría de WS
