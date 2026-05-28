# PXOAV — Pattern de Object Attribute Values (Atributos Dinámicos)

## Qué es

PXOAV implementa el patrón **EAV (Entity-Attribute-Value)** en GeneXus. Permite agregar **atributos dinámicos** a cualquier entidad sin modificar la estructura de la transacción base. Los atributos se definen en runtime (no en design-time) y pueden tener diferentes tipos de datos, validaciones, fórmulas y controles.

## Parent Objects

- `Transaction` — la transacción a la que se agregan atributos dinámicos

## Objetos que genera

PXOAV es el pattern que más objetos genera. De una sola instancia:

### Atributos GeneXus (24+)

| Categoría | Atributos generados |
|-----------|-------------------|
| **Valores WRI** (With Referential Integrity) | AttOAVAttributeValuesWRIId, ValuesWRIOAVAttributeId, ValuesWRIOAVAttributeValueCode, OAVAttributeValuesWRIAttributeName (×N), OAVAttributeValuesWRIAttributeDescription (×N) |
| **Valores WORI** (Without Referential Integrity) | AttOAVAttributeValuesWORIId, ValuesWORIOAVAttributeId, OAVAttributeValuesWORIString, OAVAttributeValuesWORIMemo, OAVAttributeValuesWORIBlob, OAVAttributeValuesWORIAttributeName (×N), OAVAttributeValuesWORIAttributeDescription (×N) |
| **Definición** | OAVAttributeDefinitionId, OAVAttributeDefinitionParentId, OAVAttributeDefinitionParentValue, OAVAttributeDefinitionOrder, OAVAttributeDefinitionFormula, OAVAttributeDefinitionFormulaOrder, OAVAttributeDefinitionDefaultId, OAVAttributeDefinitionCategory, OAVAttributeDefinitionRequired, OAVAttributeDefinitionReadOnly, OAVAttributeDefinitionDynamicReadOnlyId/URL, OAVAttributeDefinitionValidationId/URL, OAVAttributeDefinitionAttributeName/Description (×N) |
| **Subtipo OAV** | OAVAttributeSubtypeId, OAVAttributeSubtypeCode, OAVAttributeSubtypeDescription, OAVAttributeSubtypeLargeDescription, OAVAttributeSubtypeDataType, OAVAttributeSubtypeDataLength, OAVAttributeSubtypeDataDecimals, OAVAttributeSubtypeDataDescription, OAVAttributeSubtypeDataFrom/To, OAVAttributeSubtypeControlType, OAVAttributeSubtypeControlShowDescriptions, OAVAttributeSubtypeControlNullDescription, OAVAttributeSubtypeControlCheckBoxSelected/Unselected, OAVAttributeSubtypeOrderedValues, OAVAttributeSubtypeDefaultValues, OAVAttributeSubtypePicture, OAVAttributeSubtypeReferentialIntegrity, OAVAttributeSubtypeValidationId/URL, OAVAttributeSubtypePromptId/URL |

### Transacciones (3)

| Objeto | Naming | Descripción |
|--------|--------|-------------|
| WRITransaction | `WRITransaction` | Transacción de valores con integridad referencial |
| WORITransaction | `WORITransaction` | Transacción de valores sin integridad referencial |
| DefinitionTransaction | `DefinitionTransaction` | Transacción de definición de atributos OAV |

### Groups GeneXus (9)

| Objeto | Descripción |
|--------|-------------|
| DefinitionOAVAttributeGroup | Grupo de definición de subtipos OAV |
| DefinitionValidationGroup | Grupo de validación |
| DefinitionDynamicReadOnlyGroup | Grupo de solo lectura dinámica |
| DefinitionDefaultGroup | Grupo de valores por defecto |
| DefinitionAttributeGroup (×N) | Grupo por cada atributo definido |
| ValuesWRIAttributeGroup (×N) | Grupo de atributos WRI |
| ValuesWORIAttributeGroup (×N) | Grupo de atributos WORI |
| ValuesWRIOAVAttributeGroup | Grupo OAV de atributos WRI |
| ValuesWORIOAVAttributeGroup | Grupo OAV de atributos WORI |

### Procedures (10+)

| Objeto | Descripción |
|--------|-------------|
| ValuesDeleteAttributeValueProcedure | Eliminar un valor de atributo |
| ValuesAddAttributeValueDefaultProcedure | Agregar valor por defecto |
| ValuesUpdateAttributesValuesProcedure | Actualizar valores |
| ValuesDeleteAttributeValuesProcedure | Eliminar valores múltiples |
| ValuesAddAttributeValueProcedure | Agregar un valor |
| ValuesAddAttributeBlobProcedure | Agregar valor blob |
| ValuesAddAttributeStringProcedure | Agregar valor string |
| ValuesAddAttributeMemoProcedure | Agregar valor memo |
| + más Procedures para cada tipo de dato... | |

## Conceptos fundamentales

### Modelo EAV (Entity-Attribute-Value)

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Entidad    │     │   Definición     │     │    Valores       │
│  (Customer)  │────►│  de Atributos    │────►│   de Atributos   │
│              │     │  (OAV Attrs)     │     │  (OAV Values)    │
│  CustomerId  │     │  AttrId          │     │  EntityId        │
│  Name        │     │  Code            │     │  AttrId          │
│  ...         │     │  Description     │     │  Value           │
│              │     │  DataType        │     │  ValueCode (WRI) │
│              │     │  ControlType     │     │  String (WORI)   │
│              │     │  Required        │     │  Memo (WORI)     │
│              │     │  ReadOnly        │     │  Blob (WORI)     │
└──────────────┘     │  Validation      │     └──────────────────┘
                     │  Formula         │
                     │  DefaultValue    │
                     └──────────────────┘
```

### Dos tipos de valores

| Tipo | Tabla | Descripción |
|------|-------|-------------|
| **WRI** (With Referential Integrity) | ValuesWRI | Valores con código que referencia a la tabla de definición. Garantiza integridad referencial. |
| **WORI** (Without Referential Integrity) | ValuesWORI | Valores libres (string, memo, blob) sin validación de integridad. Más flexible. |

### Tipos de datos soportados

Los atributos dinámicos soportan múltiples tipos vía el atributo `DataType`:
- String, Memo, Blob (archivos)
- Numéricos (con length y decimals)
- Fecha, DateTime, Boolean
- Con picture personalizable

### Tipos de control

Cada atributo puede usar diferentes controles de UI vía `ControlType`:
- Input text
- ComboBox (con ShowDescriptions)
- CheckBox (con Selected/Unselected values)
- Prompt (con PromptId/URL)

### Características avanzadas

- **Fórmulas**: atributos calculados con orden de evaluación
- **Validaciones**: reglas de validación dinámicas (ValidationId/URL)
- **ReadOnly dinámico**: solo lectura condicional (DynamicReadOnlyId/URL)
- **Valores por defecto**: precarga de valores (DefaultId)
- **Categorías**: agrupación de atributos
- **Jerarquía**: atributos padre-hijo (ParentId/ParentValue)
- **Ordenamiento**: orden de visualización (Order)
- **Required**: campo obligatorio

## Instancia — Nodos principales

```
instance
├── definition
│   ├── DefinitionTransaction (config de la transacción de definición)
│   └── Attributes
│       └── Attribute (×N, cada atributo de la entidad que identifica al "objeto")
├── values
│   ├── ValuesWRITransaction (config de tabla WRI)
│   ├── ValuesWORITransaction (config de tabla WORI)
│   ├── DeleteAttributeValue (Procedure)
│   ├── AddAttributeValueDefault (Procedure)
│   ├── UpdateAttributesValues (Procedure)
│   ├── DeleteAttributeValues (Procedure)
│   ├── AddAttributeValue (Procedure)
│   ├── AddAttributeBlob (Procedure)
│   ├── AddAttributeString (Procedure)
│   └── AddAttributeMemo (Procedure)
```

## Uso real

- 2 instancias en pFacturas KB
- El módulo **@OAV** provee la infraestructura base: TOAVAttributes, TOAVAttributeValues, TSystemObjectOAVAttributes con sus respectivos PXWorkWith para ABM
- Se usa para agregar campos personalizables a entidades sin modificar la base de datos

## Integración con otros patterns

- **PXWorkWith** puede mostrar/editar atributos OAV en tabs de View
- **PXFlowController** puede invocar instancias PXOAV
- **@OAV module** proporciona los objetos base para gestionar definiciones y valores
