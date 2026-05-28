# PXWSQuery — Pattern de Consulta de Datos vía WebService

## Qué es

PXWSQuery genera **DataProviders y Procedures** para consultas de datos con filtros, ordenamiento, paginación y búsqueda. Es el pattern para exponer listas/consultas como servicios web. Normalmente es invocado desde PXWSLayer como método de tipo "Query".

## Parent Objects

- `Transaction` — consulta basada en la tabla extendida de la transacción
- `(None)` — consulta independiente

## Objetos que genera

Por cada `version/method`:

| Objeto | Tipo GeneXus | Naming | Descripción |
|--------|-------------|--------|-------------|
| WSQueryOrder | Domain | `WSQuery{Trn}V{ver}` | Dominio enumerado para opciones de ordenamiento |
| SDTDPQueryIn | SDT | `DPQuery{Trn}V{ver}In` | SDT de entrada del DataProvider |
| SDTDPQueryOut | SDT | `DPQuery{Trn}V{ver}Out` | SDT de salida del DataProvider |
| DataProvider | DataProvider | `DPQuery{Trn}V{ver}` | DataProvider con la consulta |
| SDTWSQueryIn | SDT | `WSQuery{Trn}V{ver}In` | SDT de entrada del Procedure WS |
| SDTWSQueryOut | SDT | `WSQuery{Trn}V{ver}Out` | SDT de salida del Procedure WS |
| Procedure | Procedure | `WSQuery{Trn}V{ver}` | Procedure wrapper del DataProvider |

Total: **7 objetos** por cada método de cada versión.

## Estructura XML de la instancia

```xml
<instance parentTransaction="Customer"
          privateName="Customer"
          category="BackOffice">

  <version version="1">
    <method name="&lt;default&gt;"
            maxRows="100"
            includeContext="false">

      <!-- Atributos multi-tenant (filtros automáticos) -->
      <multiTenantAttributes>
        <multiTenantAttribute
            attribute="CompanyId"
            returnConnectionProcedure="PrcReturnCompanyId" />
      </multiTenantAttributes>

      <!-- Campos de salida -->
      <fields>
        <attribute name="CustomerId" publicName="id" description="Customer ID" />
        <attribute name="CustomerName" publicName="name" description="Name" />
        <attribute name="CustomerEmail" publicName="email" description="Email" />
        <!-- Variable calculada -->
        <variable name="FullAddress" publicName="fullAddress"
                  description="Full address"
                  dataType="VarChar" length="500"
                  loadPreviousCode="// código previo"
                  loadCode="&amp;FullAddress = CustomerStreet + ', ' + CustomerCity" />
      </fields>

      <!-- Filtros -->
      <filters>
        <!-- Búsqueda simple (texto libre) -->
        <search>
          <attribute name="CustomerName" description="Name" />
          <attribute name="CustomerEmail" description="Email" />
        </search>

        <!-- Búsqueda avanzada (filtros específicos) -->
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
                     collection="true" />    <!-- collection=true permite filtrar por múltiples valores -->
          <attribute name="CustomerCreatedDate"
                     publicName="createdDate"
                     description="Created Date"
                     type="Range"
                     precision="Strict"
                     isRequired="false" />
        </advancedSearch>

        <!-- Condiciones fijas -->
        <conditions>
          <condition value="CustomerActive = true" />
        </conditions>
      </filters>

      <!-- Órdenes -->
      <orders generateFieldsOrder="True">
        <order name="ByName" condition="">
          <orderAttribute name="CustomerName" description="Name" ascending="true" />
        </order>
        <order name="ByDate" condition="">
          <orderAttribute name="CustomerCreatedDate" description="Created" ascending="false" />
        </order>
      </orders>

      <!-- Sort (ordenamiento de campos en output) -->
      <sort>
        <sortField name="CustomerName" ascending="true" />
      </sort>

      <!-- Hooks de código -->
      <codes>
        <code type="Start" data="// código de inicio" />
        <code type="Subroutine" name="ValidateAccess" data="// validar acceso" />
      </codes>

      <!-- Variables adicionales -->
      <variables>
        <variable name="TotalRecords"
                  destination="Procedure"
                  dataType="Numeric" length="10" />
      </variables>
    </method>
  </version>
</instance>
```

## Tipos de filtro

| Tipo | Descripción | Genera |
|------|-------------|--------|
| `Equal` | Igualdad exacta | `WHERE Attr = &Value` |
| `Range` | Rango desde-hasta | `WHERE Attr >= &From AND Attr <= &To` |
| `Like` | Búsqueda parcial | `WHERE Attr LIKE '%' + &Value + '%'` |

### Precisión de filtros

| Precisión | En Range | En Like |
|-----------|----------|---------|
| `Strict` | Rango exacto | Match exacto del patrón |
| `Flexible` | Agrega "Z" al final del "To" | Agrega "%" antes y después |
| `<default>` | Usa configuración del Settings | Usa configuración del Settings |

### Propiedades de filtro

| Propiedad | Descripción |
|-----------|-------------|
| `isRequired` | Si es obligatorio para ejecutar la consulta |
| `allowNullValue` | Para Numeric/Boolean, permite valor nulo como "sin filtro" |
| `collection` | Permite filtrar por múltiples valores (IN clause) |
| `whenExtraCondition` | Condición adicional al filtro |
| `generateCondition` | Si genera la condición automáticamente en el DataProvider |

## Multi-tenant

PXWSQuery soporta **multi-tenancy** nativo. Los `multiTenantAttributes` son filtros automáticos que se agregan a todas las consultas sin que el consumidor los vea:

```xml
<multiTenantAttributes>
  <multiTenantAttribute
      attribute="CompanyId"
      returnConnectionProcedure="PrcReturnCompanyId" />
</multiTenantAttributes>
```

El `returnConnectionProcedure` es un Procedure que obtiene el valor del atributo multi-tenant desde la conexión/sesión actual.

## Variables calculadas en Fields

Las variables en `fields` son campos calculados que no existen en la base de datos:

```xml
<variable name="TotalAmount"
          publicName="totalAmount"
          dataType="Numeric" length="12" decimals="2"
          loadPreviousCode="// código que se ejecuta antes del load"
          loadCode="&amp;TotalAmount = Sum(InvoiceAmount)" />
```

- `loadPreviousCode`: código ejecutado antes de asignar el valor
- `loadCode`: código que calcula y asigna el valor de la variable

## Variables de destino

Las variables definidas en el nodo `variables` (no en `fields`) tienen una propiedad `destination`:

| destination | Se crea en |
|-------------|-----------|
| `DataProvider` | Dentro del DataProvider generado |
| `Procedure` | Dentro del Procedure wrapper generado |

## Ordenamiento automático

Cuando `generateFieldsOrder="True"`, PXWSQuery genera automáticamente una opción de ordenamiento por cada campo de salida, además de los órdenes manuales definidos. Genera un **Domain enumerado** con todos los órdenes posibles.

## Códigos de hook

| type | Destino | Cuándo |
|------|---------|--------|
| `Start` | DataProvider/Procedure | Al inicio |
| `Subroutine` | DataProvider/Procedure | Invocable por nombre |

## Relación con PXWSLayer

PXWSQuery es normalmente invocado desde un método de PXWSLayer:

```
PXWSLayer (method callType="PXInstance")
    │
    └──► PXWSQuery (DataProvider + Procedure + SDTs)
```
