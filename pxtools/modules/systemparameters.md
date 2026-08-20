# @SystemParameters Module — Global System Parameters

> Behaviour of the `@PXTools/@SystemParameters` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@SystemParameters/` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.SystemParameters`.
- **Depends on:** `@APIs` (base), `@Menus` (`RetMenusSystemParameters`).

## 1. What it provides

A **global, flat key→value store**: one value per parameter code (optionally broken down by language) for the whole application. The **code is an enum**, the **value is stored as text** and is typed on read/write according to the parameter's declared type. This is the global system configuration (unlike @OAV / PXEntityParameters, which are **per entity/record**).

## 2. Core concept: metadata vs value

Two separate tables:
- **`SystemParameters`** — the **metadata**: which parameters exist and of what type. It is **regenerated from code** (DataProviders); it does not hold the value.
- **`SystemParametersPreferences`** — the **value** of each parameter (the "preference"). This is the real key→value table.

Reading and writing always goes through the parameter's `SystemParameterType` to type the textual value.

> **vs @OAV:** @SystemParameters is **global** (one value per code for the whole app); @OAV stores one value per (object instance, attribute) — different configuration per entity.

## 3. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **SystemParameters** (BC) | `SystemParameterCode` (`SystemParameterCode`, Char(50)) | **Catalogue/metadata**: `SystemParameterDescription`, `SystemParameterType` (domain), `SystemParameterCategory` (domain), `SystemParameterMultiLanguage` (Bool). |
| **SystemParametersPreferences** (BC) | `SystemParameterPreferenceCode` + `SystemParameterPreferenceLanguage` (`Language`, default `None`) | **The value**: `SystemParameterPreferenceValue` (`MaxStr`, short values) and `SystemParameterPreferenceMemoValue` (`MaxMem`, for Memo/HTML/Chosen-JSON). It uses one or the other depending on the type. Has a PXWorkWith instance. |
| **SystemParametersPreferencesComponent** (BC) | (same) | Component variant used as a per-language tab inside the View. |

## 4. Module domains

Enum = the stored value. **Owned** by @SystemParameters (named `SystemParameter*`), all **root-legacy** (they live in the root `#Domains/` — created before per-module domains existed):

| Domain | Role |
|---|---|
| **SystemParameterCode** (Char(50)) | **The catalogue of EVERY parameter code**. Each module/feature adds its enum values here. The source of truth for codes. (Many values belong to modules or to the project.) |
| **SystemParameterType** (Char(50)) | The value's type: `String, Integer, Date, Boolean, Password, Memo, HTML, Combo, Chosen`. It determines which column is used and which read/write procedure applies. |
| **SystemParameterCategory** (Char(50)) | Grouping for the UI (General, Security, SMTP, POP3, SendMail, ReceiveMails, FileStorage, CloudTasks, TaskManager, Logs, MailAccounts, TableCleaner, WebServicesLog…). The grid filters by this category. |

## 5. How it works: declare, seed, read

1. **Declare** — a `RetSystemParameters<X>` DataProvider returning `SDTSystemParameters` (a collection with Code/Description/Type/Category/MultiLanguage + a `SystemParameterComboValues{Value, Description}` sub-collection for Combo/Chosen types). Each item declares one parameter. **Each module contributes its own `RetSystemParameters<Module>`** and plugs it into the seeding and the read aggregator — that way it declares its parameters without touching the core.
2. **Seed** (idempotent) — `AddSystemParameters(in: &ShowMessages)` invokes every `RetSystemParameters<Module>()`, accumulates all of them and **upserts** through `AddSystemParameter` (`New … When Duplicate … EndNew`), then **deletes the orphans** (those no DataProvider declares any more). This is the "Upgrade System Parameters" action. It fires by itself from the grid's Start (`CheckSystemParametersExistence` → if there is nothing, `AddSystemParameters.Call(False)`) or from a button.
3. **Read** — from any object: `PXTools.SystemParameters.RetSystemParameterPreference<Type>.Udp(SystemParameterCode.<Code>)`.

## 6. APIs vs Personalized

- **`APIs/`** (core): the transactions, the `SDTSystemParameters` SDT, the generic seeding (`AddSystemParameter`), and the whole `Ret*/Upd*/Chk*` read/write family.
- **`Personalized/`** (the project's hook points):
  | Object | What gets customized |
  |---|---|
  | `AddSystemParameters` | The **list of `RetSystemParameters<Module>()` to seed** (each module's DataProvider is added/removed here). |
  | `RetSystemParameters` | The **read aggregator**: concatenates every `RetSystemParameters<Module>()` into a single SDT. |
  | `RetSystemParametersGeneral` / `…HTMLToPDF` / `…SynchronizationWS` (DataProviders) | Per-feature parameter declarations (a template file to copy when declaring your own parameters). |
  | `RetMenusSystemParameters` | The "System Preferences" menu entry (where it shows up). |

## 7. Pattern instance

**PXWorkWithSystemParametersPreferences** — the editing WorkWith:
- **Selection**: a grid filtered by `SystemParameterCategory` + description; in Load it resolves the displayed value according to the type. The "Upgrade System Parameters" action → `AddSystemParameters`.
- **View with per-language tabs** (`RetSystemLanguages`): each tab embeds `SystemParametersPreferencesComponent` with `&Language` → per-language editing when `MultiLanguage=True`.
- **Transaction level**: one row with typed variables (`&String, &Integer, &Date, &Boolean, &Password, &Memo, &HTML, &Combo, &Chosen`); in Start only the one matching the type is shown; in After(Confirm) it is serialized into `…Value`/`…MemoValue`.

## 8. Key procedures / APIs

Consumption pattern: `PXTools.SystemParameters.Ret…<Type>.Udp(SystemParameterCode.<Code>)`.

**Typed reads** (language-neutral, filtering `Language.None`): `RetSystemParameterPreferenceString / Integer / IntegerSigned / Date / Boolean / Password` (decrypts) `/ Memo`. Raw variant: `RetSystemParameterPreferenceValue(in: &Code, in: &Language, out: &Value)` and `…MemoValue`.

**Per-language reads** (multi-language): the `RetSystemParameterPreferenceLanguage{String, Integer, Date, Boolean, Password, Memo}(in: &Code, in: &Language, out: …)` family.

**Metadata**: `RetSystemParameter01`→type, `RetSystemParameter02`→description, `RetSystemParameterComboValues`→options, `ChkSystemParameterExistence`→exists, `CheckSystemParametersExistence`→is there any? (triggers auto-seeding).

**Writes** (upsert, always `Language.None`): `UpdSystemParameterPreferenceInteger / Date / Chosen` (stores the collection as JSON). String/Boolean are written through the Preferences BC.

**Real example** (`APIs/ServerNow`): `&HoursAdjustment = …RetSystemParameterPreferenceIntegerSigned.Udp(SystemParameterCode.ServerNowHoursAdjustment)`.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- The **@OAV** module — **per-entity** parameters/attributes (in contrast with this one, which is global).
- `modules/apis.md` — `PXToolsParametersSDT` is a different framework configuration (startup flags loaded per platform); do not confuse it with these UI-editable parameters.
