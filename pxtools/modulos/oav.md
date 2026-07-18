# Módulo @OAV — Object-Attribute-Value (Atributos Dinámicos)

> Comportamiento del módulo `@PXTools/@OAV`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@OAV` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.OAV`.
- **Depende de:** `@APIs` (base), `@DynamicCallReferences` (dominio `DynamicCallReferenceCode` de Prompt/Validation/DynamicReadOnly), `@Menus` (`RetMenusOAV`). En el grafo canónico también `@System` (por `TSystemObjects`), aunque en esta KB no aparece como referencia calificada.

## 1. Qué provee

Un patrón **EAV (Entity-Attribute-Value)**: permite agregar **atributos dinámicos a cualquier entidad en runtime, sin alterar el esquema** de la transacción base. El módulo `@OAV` aporta la **infraestructura de definición** (catálogo de atributos, valores permitidos, asignación por objeto); el pattern **PXOAV** — aplicado sobre una transacción consumidora — genera el **almacenamiento de valores por instancia** + el editor.

**Intención de diseño** (terminología del manual PXTools): busca *asociar dinámicamente atributos a entidades del sistema* para (1) **minimizar el impacto estructural** en la base al agregar atributos, (2) **independizar al usuario del desarrollador** — que puedan definirse atributos nuevos sin tocar código — y (3) construir sistemas **flexibles y adaptables** (enfoque *metadata-driven*: configuración sobre código). Conceptos del pattern: **Entidades/System Objects**, **Atributos** (cada uno con **Data Type** + **Control Type**), **Categorías** (agrupan definiciones), **Clases** (agrupaciones por valor que habilitan definiciones condicionales) y **Values Based On / Generadores** (producen valores de forma condicional/dinámica).

## 2. Concepto central

Tres capas:
1. **Definición del atributo** (`TOAVAttributes`): nombre lógico, tipo de dato, control UI, picture, validación, prompt.
2. **Valores permitidos** (`TOAVAttributeValues`): el "dominio" de un atributo tipo combo/radio/choice.
3. **Asignación por objeto** (`TSystemObjectOAVAttributes` + `TSystemObjectOAVClasses`): qué atributos cuelgan de qué objeto del sistema, con qué orden/requerido/categoría/fórmula/read-only/default.

Los **valores por instancia** NO viven acá: los genera PXOAV en el consumidor.

## 3. Transacciones (capa de definición)

| Transacción | PK | Rol |
|---|---|---|
| **TOAVAttributes** | `OAVAttributeId` (autonumber) | Catálogo maestro: `Code` (nombre lógico), `DataType`, `DataLength/Decimals`, `ControlType`, `Picture`, `ReferentialIntegrity`, `Prompt`/`Validation` (referencias dinámicas). |
| **TOAVAttributeValues** | `OAVAttributeId, OAVAttributeValueCode` | Valores permitidos (subordinada): `Description`, `Order`, `Default`. |
| **TSystemObjectOAVClasses** | `SystemObjectName, ClassCode` | Agrupa atributos por "clase"/juego dentro de un objeto. |
| **TSystemObjectOAVAttributes** | `SystemObjectName, ClassCode, OAVAttributeId` | **Asignación**: `Order`, `Category`, `Required`, `ReadOnly` (+ dinámico), `Validation`, `Formula`+`OrderFormula`, `Default`. |

**Relación con `TSystemObjects` (@System):** `SystemObjectName` es FK a `TSystemObjects`. Un objeto se marca como declaración-OAV con el flag `SystemObjectOAVDeclaration` (los combos de la UI filtran `Where SystemObjectOAVDeclaration`).

**Diagrama (definición):**
```
TSystemObjects (@System)  ──1:N──►  TSystemObjectOAVClasses ──1:N──► TSystemObjectOAVAttributes
   SystemObjectName                    (Name, ClassCode)                (Name, ClassCode, OAVAttributeId) ──► TOAVAttributes
                                                                                                              OAVAttributeId ──1:N──► TOAVAttributeValues
```

## 4. Dominios del módulo

> **Nota (dominios legacy en root):** todos los dominios de @OAV viven en el **`#Domains/` root** de la KB, no en un `#Domains/` del módulo. Es una consecuencia histórica: el módulo se creó en versiones GeneXus (Evo1/2/3) previas a los dominios asociados a módulo, así que se definieron en el root. Conceptualmente **pertenecen a @OAV** — solo objetos de @OAV los referencian (vía `DataType = '<dominio>'`).

**Enumerados:**

| Dominio | Tipo | Valores | Rol |
|---|---|---|---|
| **DataType** | Char(3) | Date=`DTE`, Character=`CHR`, Numeric=`NUM`, Memo=`MEM`, Blob=`BLO` | Tipo de dato del atributo (elige qué columna `OAVValue*` se usa). |
| **ControlType** | Char(3) | EditBox=`EDT`, CheckBox=`CHK`, DynamicComboBox=`DCB`, DynamicRadioButton=`DRB`, DynamicCheckBox=`DCH` | Control UI con que se renderiza el atributo. |
| **OAVValuesOrdered** | Char(3) | Key=`KEY`, User=`USR` | Orden de la lista de valores permitidos (por clave o definido por usuario). |
| **Positions** | Char(20) | Up, Down, Left, Right | Posición del label/control en el editor. |

**De valor (sin enum):**

| Dominio | Tipo | Rol |
|---|---|---|
| **DataLength** | Numeric(3.0) | Largo declarado del dato del atributo. |
| **DataDecimals** | Numeric(2.0) | Decimales declarados del dato. |
| **OAVAttributeType** | Char(20) | Descriptor del tipo de atributo. |
| **OAVAttributeCategory** | Char(30) | Categoría/agrupación del atributo. |
| **OAVAttributePicture** | Char(20) | Picture / máscara de edición. |
| **OAVAttributeFormula** | LongVarChar(1024) | Expresión de fórmula (atributo calculado). |
| **OAVAttributeOrder** | VarLen (Numeric(5.0)) | Orden de display/proceso del atributo. |
| **OAVValueString** | Char(500) | Valor almacenado cuando `DataType = Character`. |
| **OAVValueNumeric** | Numeric(18.5) | Valor almacenado cuando `DataType = Numeric`. |
| **OAVValueMemo** | LongVarChar(2048) | Valor almacenado cuando `DataType = Memo`. |
| **OAVValueBlob** | Blob | Valor almacenado cuando `DataType = Blob`. |

**Usa dominios de otros módulos:** `DynamicCallReferenceCode` (de [@DynamicCallReferences](dynamiccallreferences.md), para las referencias de Prompt/Validation/DynamicReadOnly) y tipos base de `@APIs` (`Boolean`, `MaxStr`, `MaxMem`, `Links`, `Objetos`, `ObjectDescription`, `WindowType`, `IdFirstLevel`, …).

> ⚠️ **No confundir:** `OAVShow` (All/Pending) **no** es de @OAV pese al nombre — lo declara y usa `@APIs` (FormState). `OAVAttributeSistemaId` es un **enum del proyecto**, no del framework.

## 5. Mecanismo Object-Attribute-Value

**Definir un atributo:** alta en `TOAVAttributes` (tipo vía `DataType`; control vía `ControlType`; si es combo/radio se cargan los valores permitidos en `TOAVAttributeValues`). Luego se **asigna** a un objeto del sistema en `TSystemObjectOAVAttributes` (order, required, category, fórmula, validación, read-only dinámico, default). El flag `OAVAttributeReferentialIntegrity` decide si el valor se guarda **WRI** (con code que referencia la lista de valores) o **WORI** (string/memo/blob libre).

**Guardar/leer un valor por (objeto, atributo, instancia):** NO ocurre en `@OAV`. El pattern **PXOAV** genera, sobre la transacción consumidora, las tablas de valores por instancia:
- **WRI** (`<Entidad>OAVWRIValues`): PK = clave-entidad + `OAVAttributeId`, guarda `OAVAttributeValueCode` (FK a `TOAVAttributeValues`).
- **WORI** (`<Entidad>OAVWORIValues`): guarda `ValueString`/`ValueMemo`/`ValueBlob` libres.

Además PXOAV genera subtipos, procs `Add/Update/Delete AttributeValue` y el editor. Se lee/escribe con esos procs generados, usando `OAVEditorValues`/`OAVAttributeValue` como transporte y `RetOAVAttributeIdFromCode` para resolver el atributo por code.

**Diferencia con @SystemParameters:** SystemParameters es configuración **global/singleton** (un valor por parámetro para todo el sistema). OAV es **por instancia de objeto** (N valores, uno por cada registro de la entidad).

## 6. APIs vs Personalized

- **`APIs/`** (core, intocable): las 4 transacciones, los helpers de metadatos/fórmulas/validación, `SaveOAVSystemObjects`, `RetOAVAttributeIdFromCode`, los SDTs de intercambio.
- **`Personalized/`**: un solo objeto — **`RetMenusOAV`** (DataProvider) que construye el nodo de menú "OAV" (Attributes / Object Attributes). El resto vive en `APIs/`.

## 7. Instancias de patterns

- **PXWorkWithTOAVAttributes** — WW del catálogo de atributos (tabs Definition/Control/Advanced).
- **PXWorkWithTOAVAttributeValues** — ABM de la lista de valores permitidos.
- **PXWorkWithTSystemObjectOAVAttributes** — WW de asignación objeto↔atributo (insert/delete/mover-orden vía procs).
- **PXParameterRequestSaveOAVSystemObjects** — expone `SaveOAVSystemObjects` como endpoint.

## 8. Procedimientos / APIs clave

| Objeto | `Parm()` | Propósito |
|---|---|---|
| `RetOAVAttributeIdFromCode` | `in: &Code; out: &OAVAttributeId` | Resuelve el Id numérico desde el code lógico (entrada típica). |
| `RetOAVAttributeControlTypeForSD` [WS] | `in: &Id; out: &IsEditBox, &IsDynamicComboBox, …` | Flags de control para render dinámico. |
| `RetOAVAttributeTypeDataForSD` [WS] | `in: &Id; out: &IsCharacter, &IsNumeric…, &IsDate, &IsMemo` | Flags de tipo de dato. |
| `SaveOAVSystemObjects` | — | Registra/borra los SystemObjects OAV contra @System. |

**SDTs de intercambio** (`APIs/`): `OAVAttributeValue` (valor tipado unificado: Blob/Character/Date/Numeric/Memo), `OAVEditorValues` (colección Id+ValueCode/Memo/Blob — el "sobre" que un editor OAV lee/escribe), `OAVValidationResult` (Ok+ErrorMessage), `OAVFormulaOrderCollection`.

> **Punto clave:** `@OAV/APIs` provee únicamente la **definición** (catálogo, valores permitidos, asignación por objeto) y los helpers de metadatos/validación. La lectura/escritura del **valor por instancia** se hace con los procs que el pattern **PXOAV** genera en la transacción consumidora.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [system.md](system.md) — `TSystemObjects` (los atributos OAV cuelgan de un SystemObject).
- [systemparameters.md](systemparameters.md) — configuración global (contraste con OAV por instancia).
- [dynamiccallreferences.md](dynamiccallreferences.md) — Prompt/Validation/DynamicReadOnly se resuelven por referencia dinámica.
- El pattern `PXOAV` (genera las tablas WRI/WORI + editor en el consumidor) — documentado aparte con los demás patterns.
