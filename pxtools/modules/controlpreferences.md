# Módulo @ControlPreferences — Preferencias de UI por Control y Usuario

> Comportamiento del módulo `@PXTools/@ControlPreferences`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@ControlPreferences` (`APIs/Basic/`, `APIs/UserControlState/`, `Personalized/`).
- Cualificador: `PXTools.ControlPreferences`.
- **Depende de:** `@APIs` (base). A nivel de dominio reutiliza `SecurityUserCode` de @Security, pero no hay llamadas `PXTools.Security` (no es dependencia de código).

## 1. Qué provee

**Persistencia del estado de UI de un control por usuario**: anchos/orden/visibilidad de columnas y demás configuración de una grilla, para que cada usuario recupere su vista al volver a abrir el objeto. Es la infraestructura que respalda la personalización de grillas de los patterns.

## 2. Concepto central

Una tabla de pares **nombre/valor**, deduplicados por una PK de 5 campos = **(alcance, usuario, objeto, control, preferencia)**:
- El **usuario** se obtiene del contexto (`PGetUserCode`) y se guarda como texto — no hay FK a una tabla de usuarios.
- El **objeto** (`SystemObjectName`) comparte el dominio `ObjectName, GeneXus` con `TSystemObjects` (@System), pero **sin integridad referencial** — es solo la convención que enlaza la preferencia con el objeto GX cuya grilla se personaliza.
- El **valor** es un blob serializado (config/JSON).

## 3. Transacción `ControlPreferences`

| Atributo | Tipo | Rol |
|---|---|---|
| `ControlPreferenceTargetType`* | `ControlPreferenceTargetType` | Discriminador de alcance (hoy solo `None`). |
| `ControlPreferenceTargetId`* | `ObjectName, GeneXus` | **Identidad del usuario** (del contexto). |
| `ControlPreferenceSystemObjectName`* | `ObjectName, GeneXus` | Objeto GX (panel/WW) dueño de la grilla. |
| `ControlPreferenceControlName`* | `ObjectName, GeneXus` | Nombre del control/grilla. |
| `ControlPreferenceName`* | `Character(200)` | Clave de la preferencia individual. |
| `ControlPreferenceValue` | `MaxMem` | Valor serializado (config/JSON). |
| `ControlPreferenceFullName`! | fórmula | `SystemObjectName-ControlName-Name`. |

## 4. Dominios del módulo

- **`ControlPreferenceTargetType`** (Char(20), **root-legacy** — vive en el `#Domains/` raíz): único dominio propio; un solo valor `None`. Diseñado para extenderse (rol, global); hoy solo alcance por usuario.
- **Usa de otros módulos:** `SecurityUserCode` (@Security), `ObjectName`/`MaxMem` (@APIs/GeneXus base).

## 5. Mecanismo

Hay **dos generaciones** de la API, con la misma tabla de respaldo:

### A) `APIs/Basic/` — estado nativo gxui (SaveState/InitState de grilla)
- `ControlPreferencesSaveState` (Main, HTTP, `gxui.SaveState`): recibe el JSON del cliente, lo parsea a la colección `ControlPreferencesState`, y por ítem hace `Update` / `Delete` (si value=`undefined`) / `New`.
- `ControlPreferencesInitState` (DataProvider → `gxuiStates`): lee la tabla por TargetType/TargetId/SystemObjectName y devuelve los pares para restaurar la grilla.
- `CmControlPreferences` (WebComponent): en `Event Start` cablea `gxui_Settings.SaveURL` e `InitState`.
- `ClearControlPreferences` (proc): borra todo por `TargetId` + `ObjectName`.

### B) `APIs/UserControlState/` — user control GridHandler (el actual)
- SDT de intercambio **`ControlPreferencesUCState`**: `Object`, `Control`, colección `Properties{ Name, Value }`.
- Endpoints HTTP (parsean el JSON y delegan en el proc de negocio):
  | Endpoint | Delega en |
  |---|---|
  | `SaveControlPreferencesUCState(in &data)` | `UpdControlPreferencesUCState` |
  | `LoadControlPreferencesUCState(in &data)` | `RetControlPreferencesUCState` |
  | `ClearControlPreferencesUCState(in &data)` | (borra las filas que matchean) |
- `UpdControlPreferencesUCState(in &UCState, out &SaveResult)`: usuario vía `PGetUserCode`; por cada Property hace Update / Delete (valor vacío) / New.
- `RetControlPreferencesUCState(in &UCState, out &LoadResult)`: si `Properties.Count=1` devuelve un valor puntual; si no, todas las properties como JSON.
- `RetControlPreferencesURL(in &Operation('save'|'load'|'clear'), out &URL)`: devuelve el `.Link()` del Main correspondiente.

### Integración con grillas configurables
- `IsGridConfigurable(in &ObjectName, out &Configurable)` (hook en `@APIs/Personalized/`) decide por objeto si se ofrece la UI de configuración.
- `RetGridHandlerConfig` emite en su salida el nodo `ControlPreferences { SaveURL/LoadURL/ClearURL = RetControlPreferencesURL.Udp(...) }`, de modo que el user control GridHandler sabe a qué endpoints hacer POST del estado. Cada grid handler generado por PXWorkWith repite ese nodo.

## 6. APIs vs Personalized

- **`APIs/`** (core): la transacción, ambas generaciones de endpoints/procs, los SDTs.
- **`Personalized/`**: un solo objeto — **`RetControlPreferenceSystemObjectNameLength`** (`out: &Length`, retorna `50`). Punto customizable: cuántos caracteres del prefijo del nombre de objeto se conservan como `ControlPreferenceSystemObjectName`. Lo consumen los save/init de ambas generaciones.

## 7. Instancias de patterns

**Ninguna** — la transacción `ControlPreferences` no tiene PXWorkWith (es infraestructura de respaldo, no un ABM de usuario).

## 8. Procedimientos / APIs clave

| Objeto | `Parm()` | Propósito |
|---|---|---|
| `ControlPreferencesSaveState` (Main) | `in: &TargetType, &TargetId, &SystemObjectName, &SavePreferences` | Persistir estado gxui de grilla. |
| `ControlPreferencesInitState` (DP) | `in: &TargetType, &TargetId, &SystemObjectName` | Recuperar estado gxui. |
| `ClearControlPreferences` | `in: &TargetId, &ObjectName` | Borrar preferencias de un usuario+objeto. |
| `UpdControlPreferencesUCState` | `in: &UCState; out: &SaveResult` | Persistencia real (GridHandler). |
| `RetControlPreferencesUCState` | `in: &UCState; out: &LoadResult` | Lectura real (valor único o JSON). |
| `RetControlPreferencesURL` | `in: &Operation; out: &URL` | Resuelve URL save/load/clear para el user control. |
| `RetControlPreferenceSystemObjectNameLength` (Personalized) | `out: &Length` | Longitud del prefijo del nombre de objeto (=50). |

> **Aviso — homónimo de otro módulo:** `RetControlPreferencesChosen` (en `@SystemParameters`) **no** pertenece a este módulo; arma la lista de valores elegidos de un *system parameter*. Coincidencia de nombre solamente.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [system.md](system.md) — `TSystemObjects` comparte dominio con `SystemObjectName` (sin FK).
- `modulos/apis.md` — `PGetUserCode` (contexto/usuario) y el hook `IsGridConfigurable`.
