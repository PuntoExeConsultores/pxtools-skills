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

## Cómo declarar los `<orders>`

Cada `<order>` se traduce literalmente a una cláusula `Order` del DataProvider generado, así
que le aplican las mismas reglas de rendimiento que a cualquier `For Each`:

**1. Empezar siempre por los atributos filtrados por igual.** El primero es el (o los)
atributo multi-tenant del Settings del Layer, que el pattern filtra en todas las consultas
(`Where EmisorId = &EmisorId`). Después van los filtros `type="Equal"` de la propia instancia
—típicamente la clave del padre en una consulta sobre un nivel subordinado— y recién al final
el atributo por el que se quiere ordenar.

```xml
<order name="PorVigencia">
  <orderAttribute name="EmisorId" description="Emisor" />
  <orderAttribute name="EmisorCodigoProductosServiciosId" description="Código del producto" />
  <orderAttribute name="ProductosServiciosPrecioVigencia" description="Vigencia" ascending="False" />
</order>
```

**2. Verificar que exista un índice que respalde ese orden.** La secuencia de atributos del
`<order>` tiene que ser prefijo de algún índice de la tabla base. Se comprueba leyendo
`Knowledge Base/#Tables/<Tabla>.Table.gxSource`, sección `#Indexes`. Si no existe, hay que
sacar el orden o crear un índice de usuario — y crear un índice es una reorganización de la
base, o sea una decisión del dueño de la KB, no algo que se agregue al pasar. Ordenar por un
atributo de la **tabla extendida** (el nombre de una foránea) nunca queda respaldado: ordenar
por la clave foránea.

**3. Un mismo conjunto de atributos no puede usarse en dos órdenes.** El dominio enumerado que
genera el pattern usa como `Description` de cada valor **la lista de nombres de atributos** del
orden. Dos órdenes con los mismos atributos (por ejemplo el mismo atributo ascendente y
descendente) producen dos valores con la misma descripción y el apply falla con
`Failed processing Domain '<WSQuery…Order>' properties`. Si se necesita ida y vuelta sobre el
mismo atributo, hay que declarar un solo orden con el sentido más útil.

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
