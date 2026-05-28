# PXEntityParameters — Pattern de Parámetros Configurables por Entidad

## Qué es

PXEntityParameters genera una **infraestructura completa de parámetros configurables** por entidad. Permite definir parámetros tipados (string, numeric, date, boolean, memo, password) que se almacenan como pares clave-valor, con soporte multi-idioma y controles UI especializados (Combo, Chosen).

## Parent Objects

- `Transaction` — la transacción que contiene los parámetros

## Objetos que genera

De una sola instancia genera un conjunto muy extenso de objetos:

### Domains (2)

| Objeto | Naming | Descripción |
|--------|--------|-------------|
| DomainEntityParameterCode | `EntityParameterCode` | Dominio para códigos de parámetro |
| DomainEntityParameterCategory | `EntityParameterCategory` | Dominio para categorías |

### Attributes (7)

| Objeto | Naming | Descripción |
|--------|--------|-------------|
| AttributeEntityParameterCode | `EntityParameterCode` | Código del parámetro |
| AttributeEntityParameterDescription | `EntityParameterDescription` | Descripción |
| AttributeEntityParameterType | `EntityParameterType` | Tipo de dato |
| AttributeEntityParameterCategory | `EntityParameterCategory` | Categoría |
| AttributeEntityParameterMultiLanguage | `EntityParameterMultiLanguage` | Flag multi-idioma |
| AttributeEntityParameterPreferenceCode | `EntityParameterPreferenceCode` | Código de preferencia |
| + 3 más para preferencias | | Language, Value, MemoValue |

### Transactions (3)

| Objeto | Naming | Descripción |
|--------|--------|-------------|
| TransactionEntityParameters | `EntityParameters` | Definición de parámetros |
| TransactionEntityParametersPreferences | `EntityParametersPreferences` | Valores de preferencias |
| TransactionEntityParametersPreferencesComponent | `EntityParametersPreferencesComponent` | Componente de preferencias |

### Groups (1)

| Objeto | Naming | Descripción |
|--------|--------|-------------|
| GroupEntityParametersPreferences | `EntityParametersPreferences` | Grupo de estructura |

### SDTs (1)

| Objeto | Naming | Descripción |
|--------|--------|-------------|
| SDTEntityParameters | `SDTEntityParameters` | SDT para manejo programático |

### Procedures (14+)

| Objeto | Descripción |
|--------|-------------|
| ProcedureAddEntityParameters | Agregar múltiples parámetros |
| ProcedureReturnEntityParameters | Obtener parámetros de una entidad |
| ProcedureAddEntityParameter | Agregar un parámetro individual |
| ProcedureCheckEntityParametersExistence | Verificar existencia |
| ProcedureReturnEntityParameterType | Obtener tipo de un parámetro |
| ProcedureReturnEntityParameterDescription | Obtener descripción |
| ProcedureReturnEntityParameterPreferenceBoolean | Obtener valor boolean |
| ProcedureReturnEntityParameterPreferenceDate | Obtener valor date |
| ProcedureReturnEntityParameterPreferenceInteger | Obtener valor integer |
| ProcedureReturnEntityParameterPreferenceMemo | Obtener valor memo |
| ProcedureReturnEntityParameterPreferencePassword | Obtener valor password |
| ProcedureReturnEntityParameterPreferenceString | Obtener valor string |
| ProcedureReturnEntityParameterPreferenceLanguage* | Versiones multi-idioma (×6) |
| ProcedureUpdateEntityParameterPreferenceChosen | Actualizar valor Chosen |
| ProcedureReturnEntityParameterComboValues | Obtener valores de combo |

### DataProviders (2)

| Objeto | Descripción |
|--------|-------------|
| DataProviderReturnEntityParametersGeneral | DP general de parámetros |
| DataProviderReturnEntityParameterChosenValues | DP de valores Chosen |

Total: **30+ objetos** de una sola instancia.

## Modelo de datos

```
┌──────────────────────┐
│  EntityParameters    │  ← Definición de parámetros
│  ────────────────    │
│  ParameterCode (PK)  │
│  Description         │
│  Type                │  ← String|Numeric|Date|Boolean|Memo|Password
│  Category            │
│  MultiLanguage       │  ← Soporte multi-idioma
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────┐
│  EntityParametersPreferences │  ← Valores de cada parámetro
│  ────────────────────────    │
│  EntityId (PK)               │  ← La entidad a la que pertenece
│  ParameterCode (PK, FK)      │
│  Language (PK)               │  ← Para multi-idioma
│  Value                       │  ← Valor string
│  MemoValue                   │  ← Valor memo (textos largos)
└──────────────────────────────┘
```

## Tipos de parámetro

| Tipo | Procedure de lectura | Almacenamiento |
|------|---------------------|---------------|
| `String` | ReturnEntityParameterPreferenceString | Campo Value |
| `Integer` | ReturnEntityParameterPreferenceInteger | Campo Value (parseado) |
| `Date` | ReturnEntityParameterPreferenceDate | Campo Value (parseado) |
| `Boolean` | ReturnEntityParameterPreferenceBoolean | Campo Value ("True"/"False") |
| `Memo` | ReturnEntityParameterPreferenceMemo | Campo MemoValue |
| `Password` | ReturnEntityParameterPreferencePassword | Campo Value (encriptado) |

## Soporte Multi-idioma

Cuando `MultiLanguage=true`, cada parámetro puede tener un valor diferente por idioma. Los Procedures `*Language*` manejan la lectura/escritura con el parámetro `Language` adicional.

## Controles UI

- **Combo**: valores predefinidos seleccionables (ReturnEntityParameterComboValues)
- **Chosen**: selector múltiple avanzado (ReturnEntityParameterChosenValues, UpdateEntityParameterPreferenceChosen)

## Uso real

- 2 instancias en pFacturas KB
- El módulo **@SystemParameters** usa este pattern para parámetros globales del sistema (PXWorkWithSystemParametersPreferences)

## Diferencia con @SystemParameters

| Aspecto | PXEntityParameters (pattern) | @SystemParameters (módulo) |
|---------|------------------------------|---------------------------|
| Alcance | Parámetros por entidad | Parámetros globales del sistema |
| Asociación | Vinculado a una Transaction | Módulo standalone |
| Personalización | Genera estructura custom | Estructura predefinida |
