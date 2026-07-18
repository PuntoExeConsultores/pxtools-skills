# Módulo @SystemParameters — Parámetros Globales del Sistema

> Comportamiento del módulo `@PXTools/@SystemParameters`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@SystemParameters/` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.SystemParameters`.
- **Depende de:** `@APIs` (base), `@Menus` (`RetMenusSystemParameters`).

## 1. Qué provee

Un **almacén clave→valor global y plano**: un valor por código de parámetro (opcionalmente desglosado por idioma) para toda la aplicación. El **código es un enum**, el **valor se guarda como texto** y se tipa al leer/escribir según el tipo declarado del parámetro. Es la config global del sistema (a diferencia de @OAV / PXEntityParameters, que es **por entidad/registro**).

## 2. Concepto central: metadata vs valor

Dos tablas separadas:
- **`SystemParameters`** — la **metadata**: qué parámetros existen y de qué tipo son. **Se regenera desde código** (DataProviders); no guarda el valor.
- **`SystemParametersPreferences`** — el **valor** de cada parámetro (la "preferencia"). Es la tabla real clave→valor.

Leer/escribir siempre pasa por el `SystemParameterType` del parámetro para tipar el valor textual.

> **vs @OAV:** @SystemParameters es **global** (un valor por código para toda la app); @OAV guarda un valor por cada (objeto-instancia, atributo) — configuración distinta por entidad.

## 3. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **SystemParameters** (BC) | `SystemParameterCode` (`SystemParameterCode`, Char(50)) | **Catálogo/metadata**: `SystemParameterDescription`, `SystemParameterType` (dominio), `SystemParameterCategory` (dominio), `SystemParameterMultiLanguage` (Bool). |
| **SystemParametersPreferences** (BC) | `SystemParameterPreferenceCode` + `SystemParameterPreferenceLanguage` (`Language`, default `None`) | **Valor**: `SystemParameterPreferenceValue` (`MaxStr`, valores cortos) y `SystemParameterPreferenceMemoValue` (`MaxMem`, para Memo/HTML/Chosen-JSON). Según el tipo usa uno u otro. Instancia PXWorkWith. |
| **SystemParametersPreferencesComponent** (BC) | (igual) | Variante Component usada como pestaña por idioma dentro del View. |

## 4. Dominios del módulo

Enum = valor almacenado. **Propios** de @SystemParameters (nombre `SystemParameter*`), todos **root-legacy** (viven en el `#Domains/` raíz — creados antes de los dominios por módulo):

| Dominio | Rol |
|---|---|
| **SystemParameterCode** (Char(50)) | **Catálogo de TODOS los códigos** de parámetro. Cada módulo/feature agrega aquí sus enum-values. Fuente de verdad de los códigos. (Muchos valores son de módulos/proyecto.) |
| **SystemParameterType** (Char(50)) | Tipo del valor: `String, Integer, Date, Boolean, Password, Memo, HTML, Combo, Chosen`. Determina qué columna se usa y qué proc de lectura/escritura aplica. |
| **SystemParameterCategory** (Char(50)) | Agrupador para la UI (General, Security, SMTP, POP3, SendMail, ReceiveMails, FileStorage, CloudTasks, TaskManager, Logs, MailAccounts, TableCleaner, WebServicesLog…). El grid filtra por esta categoría. |

## 5. Mecanismo: definir, sembrar, leer

1. **Definir** — un DataProvider `RetSystemParameters<X>` que devuelve `SDTSystemParameters` (colección con Code/Description/Type/Category/MultiLanguage + sub-colección `SystemParameterComboValues{Value, Description}` para tipo Combo/Chosen). Cada item declara un parámetro. **Cada módulo aporta su propio `RetSystemParameters<Modulo>`** y lo enchufa en la siembra y en el agregador de lectura — así declara sus parámetros sin tocar el núcleo.
2. **Sembrar** (idempotente) — `AddSystemParameters(in: &ShowMessages)` invoca cada `RetSystemParameters<Modulo>()`, acumula todo y hace **upsert** vía `AddSystemParameter` (`New … When Duplicate … EndNew`), y **borra los huérfanos** (los que ya no declara ningún DP). Es el "Upgrade System Parameters". Se dispara solo desde el Start del grid (`CheckSystemParametersExistence` → si no hay nada, `AddSystemParameters.Call(False)`) o por botón.
3. **Leer** — desde cualquier objeto: `PXTools.SystemParameters.RetSystemParameterPreference<Tipo>.Udp(SystemParameterCode.<Codigo>)`.

## 6. APIs vs Personalized

- **`APIs/`** (core): las transacciones, el SDT `SDTSystemParameters`, la siembra genérica (`AddSystemParameter`), y toda la familia de lectura/escritura `Ret*/Upd*/Chk*`.
- **`Personalized/`** (puntos de enganche del proyecto):
  | Objeto | Qué se customiza |
  |---|---|
  | `AddSystemParameters` | La **lista de `RetSystemParameters<Modulo>()` a sembrar** (se agrega/quita el DP de cada módulo). |
  | `RetSystemParameters` | El **agregador de lectura**: concatena todos los `RetSystemParameters<Modulo>()` en un solo SDT. |
  | `RetSystemParametersGeneral` / `…HTMLToPDF` / `…SynchronizationWS` (DataProviders) | Declaración de parámetros por feature (archivo-plantilla a copiar para declarar parámetros propios). |
  | `RetMenusSystemParameters` | La entrada de menú "System Preferences" (dónde aparece). |

## 7. Instancia de pattern

**PXWorkWithSystemParametersPreferences** — el WorkWith de edición:
- **Selection**: grid filtrado por `SystemParameterCategory` + descripción; en Load resuelve el valor visible según tipo. Acción "Upgrade System Parameters" → `AddSystemParameters`.
- **View con tabs por idioma** (`RetSystemLanguages`): cada tab embebe `SystemParametersPreferencesComponent` con `&Language` → edición por idioma cuando `MultiLanguage=True`.
- **Transaction level**: una fila con variables tipadas (`&String, &Integer, &Date, &Boolean, &Password, &Memo, &HTML, &Combo, &Chosen`); en Start se muestra solo la que corresponde al tipo; en After(Confirm) se serializa a `…Value`/`…MemoValue`.

## 8. Procedimientos / APIs clave

Patrón de consumo: `PXTools.SystemParameters.Ret…<Tipo>.Udp(SystemParameterCode.<Codigo>)`.

**Lectura tipada** (idioma-neutro, filtran `Language.None`): `RetSystemParameterPreferenceString / Integer / IntegerSigned / Date / Boolean / Password` (desencripta) `/ Memo`. Variante cruda: `RetSystemParameterPreferenceValue(in: &Code, in: &Language, out: &Value)` y `…MemoValue`.

**Lectura por idioma** (multi-language): familia `RetSystemParameterPreferenceLanguage{String, Integer, Date, Boolean, Password, Memo}(in: &Code, in: &Language, out: …)`.

**Metadata**: `RetSystemParameter01`→tipo, `RetSystemParameter02`→descripción, `RetSystemParameterComboValues`→opciones, `ChkSystemParameterExistence`→existe, `CheckSystemParametersExistence`→¿hay alguno? (auto-siembra).

**Escritura** (upsert, siempre `Language.None`): `UpdSystemParameterPreferenceInteger / Date / Chosen` (guarda colección como JSON). Para String/Boolean se guarda vía la BC de Preferences.

**Ejemplo real** (`APIs/ServerNow`): `&HoursAdjustment = …RetSystemParameterPreferenceIntegerSigned.Udp(SystemParameterCode.ServerNowHoursAdjustment)`.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- Módulo **@OAV** — parámetros/atributos **por entidad** (contraste con este, que es global).
- `modulos/apis.md` — `PXToolsParametersSDT` es una config de framework distinta (flags de arranque cargados por plataforma), no confundir con estos parámetros editables por UI.
