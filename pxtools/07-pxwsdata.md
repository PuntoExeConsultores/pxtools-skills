# PXWSData — Pattern de Lectura de Datos vía WebService

## Qué es

PXWSData genera **Procedures de lectura de datos** con hooks de código personalizables. A diferencia de PXWSQuery (que genera DataProviders para listas paginadas), PXWSData genera Procedures para leer un registro específico con lógica custom. Se usa para operaciones de "Get" o "Read" de entidades individuales.

## Parent Objects

- `Transaction` — lectura basada en los keys de la transacción
- `(None)` — lectura independiente

## Objetos que genera

Por cada `version/method`:

| Objeto | Tipo GeneXus | Naming | Descripción |
|--------|-------------|--------|-------------|
| SDTData | SDT | `WSTransaction{Trn}V{ver}Structure` | Estructura de datos compartida |
| MethodData | Procedure | `WSData{Trn}V{ver}Method` | Procedure de lectura |
| SDTDataIn | SDT | `WSData{Trn}V{ver}DataIn` | SDT de entrada |
| SDTDataOut | SDT | `WSData{Trn}V{ver}DataOut` | SDT de salida |

Total: **4 objetos** por cada método de cada versión.

## Estructura XML de la instancia

```xml
<instance parentTransaction="Customer"
          privateName="Customer"
          category="BackOffice">

  <version version="1">
    <method name="&lt;default&gt;">

      <data>
        <!-- Keys de entrada -->
        <key name="CustomerId"
             publicName="id"
             type="Public" />           <!-- Public: visible al consumidor -->
        <key name="CompanyId"
             publicName=""
             type="Multitenant" />      <!-- Multitenant: automático desde conexión -->

        <!-- Atributos de salida -->
        <attribute name="CustomerName"
                   publicName="name"
                   description="Customer Name" />
        <attribute name="CustomerEmail"
                   publicName="email"
                   description="Email" />
        <attribute name="CustomerAddress"
                   publicName="address"
                   description="Address" />

        <!-- Variables calculadas -->
        <variable name="TotalInvoices"
                  publicName="totalInvoices"
                  description="Total invoices count"
                  dataType="Numeric" length="10"
                  loadPreviousCode="// código previo"
                  loadCode="&amp;TotalInvoices = Count(InvoiceId)" />
      </data>

      <!-- Hooks de código -->
      <codes>
        <code type="Start" data="// inicialización" />
        <code type="Subroutine" name="ValidateAccess"
              data="// validar permisos del usuario" />
      </codes>

      <!-- Variables adicionales del Procedure -->
      <variables>
        <variable name="HasAccess"
                  dataType="Boolean"
                  description="Flag de acceso" />
      </variables>
    </method>
  </version>
</instance>
```

## Nodo Data — Estructura de datos

El nodo `data` define qué datos se leen. Acepta tres tipos de hijos mezclados (ChildrenType="Mixed"):

### Keys

```xml
<key name="CustomerId" publicName="id" type="Public" />
```

| type | Descripción |
|------|-------------|
| `Public` | Visible en el SDT de entrada, proporcionado por el consumidor |
| `Multitenant` | No visible, obtenido automáticamente de la conexión/sesión |
| `Private` | Interno, no expuesto al consumidor |

### Attributes

Atributos de la transacción que se incluyen en la respuesta:

```xml
<attribute name="CustomerName" publicName="name" description="Name" />
```

Solo se deben incluir atributos de la **tabla base** de la transacción.

### Variables calculadas

Variables con código de carga personalizado:

```xml
<variable name="FullName"
          publicName="fullName"
          loadPreviousCode="// código ejecutado antes de la carga"
          loadCode="&amp;FullName = CustomerFirstName + ' ' + CustomerLastName" />
```

## Diferencia con PXWSQuery

| Aspecto | PXWSQuery | PXWSData |
|---------|-----------|----------|
| **Propósito** | Listas paginadas con filtros | Lectura de un registro |
| **Genera** | DataProvider + Procedure | Solo Procedure |
| **Filtros** | Search, AdvancedSearch, Conditions | Solo Keys |
| **Ordenamiento** | Si (orders, sort) | No |
| **Paginación** | Si (maxRows) | No |
| **Campos calculados** | Si (loadCode en fields) | Si (loadCode en data) |
| **Multi-tenant** | Si (nodo separado) | Si (key type=Multitenant) |

## Hooks de código

| type | Cuándo se ejecuta |
|------|-------------------|
| `Start` | Al inicio del Procedure |
| `Subroutine` | Subrutina invocable por nombre desde el código |

## Relación con PXWSLayer

PXWSData es normalmente invocado desde un método de PXWSLayer:

```
PXWSLayer (method callType="PXInstance")
    │
    └──► PXWSData (Procedure + SDTs)
```

## Relación con PXWSTransaction

PXWSData y PXWSTransaction comparten la misma estructura SDT (`WSTransaction{Trn}V{ver}Structure`) lo que permite usar el mismo formato de datos para lectura y escritura.
