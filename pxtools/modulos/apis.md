# Módulo @APIs — Base del Framework PXTools

> Comportamiento del módulo `@PXTools/@APIs`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@APIs/`
  - `APIs/` — **core del framework** (≈420 objetos). Intocable.
  - `Personalized/` — **capa de extensión del proyecto** (≈127 objetos, 11 subcarpetas). Es donde cada aplicación adapta el framework sin tocar el core.
  - `#Domains/` — dominios base del framework (4 module-scoped) + la mayoría de los dominios **base** del framework viven en el `#Domains/` **raíz** (legacy) y en `@PXTools/#Domains/` (paquete). Ver §8.
- **Depende de:** nada en el `APIs/` core — es la **base universal** del framework (todos los módulos dependen de @APIs). Solo la capa `Personalized/` (master pages / login) referencia `@Menus` (PXTools) y `@ResponsiveLayout` (módulo **de GeneXus**, no PXTools).

**Convención de nombres del framework**
- `PPEXE_*` — procedures helper (por convención `…Ac…` = action/update, `…De…` = return/query).
- `HPEXE_*` / `H*` — web panels / master pages.
- `Pu*` / `RPu*` — popups (desktop / responsive) generados por patterns.
- `Wb*` / `Tr*` / `Ve*` — web panels generados por patterns (form / selection / view).

## 1. Qué provee

`@APIs` es el **módulo base** del framework: provee la **infraestructura transversal** que consumen todos los demás módulos y todas las instancias de patterns. No modela una entidad de negocio; aporta:

- El **contexto de sesión** (`Context`) y su ciclo de vida (login → session → cada request).
- Las **MasterPages** (shells de UI) y el layout responsive.
- La **configuración global** del framework (`PXToolsParametersSDT`).
- Un gran **catálogo de APIs utilitarias** (strings, fechas, archivos, Excel, mensajes, links, escape, descargas, etc.).
- El **connector de seguridad** (autenticación/autorización — ver [security.md](security.md)).
- Los **templates** que usan los generadores de patterns.

Todos los módulos dependen de `@APIs`.

## 2. Concepto central: core vs Personalized

`@APIs` materializa el modelo de extensión de PXTools:

- **`APIs/`** = framework genérico. Se actualiza con el framework; **no se toca**.
- **`Personalized/`** = los **puntos de extensión** previstos. PXTools deja ahí objetos (SDT `Context`, `PXToolsParametersSDT`, SecurityConnector, MasterPages, Languages, Templates) **para que el proyecto los adapte**. Un proyecto nuevo customiza `Personalized/` y consume `APIs/`.

## 3. Contexto de sesión (`Context`) — el núcleo

El SDT **`Context`** (`Personalized/Context_Personalized/Context.StructuredDataType.gxSource`) es la "mochila" de sesión del usuario logueado. Es un SDT plano (sin colecciones), pensado para **serializarse a XML** y viajar en la `WebSession`. Campos del framework: `SecurityUserId`, `SecurityUserCode`, `SecurityUserName`, `SecurityUserQualifiedCode` (código cualificado `usuario@dominio`), `SecurityDomainId`/`SecurityDomainName` (tenant/dominio — soporte multi-tenant), `SecurityUserType` (BackOffice / FrontEnd / Invite). **Cada proyecto embebe además sus atributos de negocio** en este SDT (ese es el punto de personalización).

### 3.1 Ciclo de vida del contexto

| Proc | Ubicación | `Parm()` | Rol |
|---|---|---|---|
| `PLoadContext` | `APIs/Context_APIs/` | `out: &Context` | **Lee** el contexto de la `WebSession` (`&Context.FromXml(&Session.Get("context"))`). Es el **hidratador** que invocan todos los objetos generados y master pages. |
| `PSetContext` | `APIs/Context_APIs/` | `in: &Context` | **Guarda** el contexto en la `WebSession` (`&Session.Set("context", &Context.ToXml())`). Setter de bajo nivel. |
| `PGetUserCode` | `APIs/Context_APIs/` | `out: &UserCode` | Atajo: `PLoadContext` + devuelve el código de usuario. |
| `PSaveContext` | `Personalized/Context_Personalized/` | `in: &UserCode, in: &AmbienteId` | **Construye** el `&Context` desde cero: parsea `usuario@dominio`, resuelve el dominio, lee la tabla de usuarios y puebla los campos; termina con `PSetContext`. **Punto de personalización** de qué se carga en el contexto. |
| `PCheckContext` | `Personalized/Context_Personalized/` | `&Context, &Ok` | **Valida** la sesión: si `LoginSupport` está activo y existen usuarios, devuelve `&Ok=False` cuando no hay sesión válida (→ redirige a login). |
| `SaveLoginContext` | `Personalized/Context_Personalized/` | `in: &UserCode, in: &AmbienteId` | **Orquestador post-login**: arma el contexto (`PSaveContext`), genera los menús habilitados del usuario, elige el menú principal según `SecurityUserType` (si `SeparateMenusByUserType`), y registra el WebSessionId. Lo ejecuta la pantalla de login tras validar credenciales. |

### 3.2 Cómo se inyecta `&Context` en los objetos generados

Dos mecanismos combinados:

1. **En cada objeto generado por un pattern**: se declara la variable `&Context` (tipo `Context, PXTools.APIs`) y, al inicio del `Event Start`, el bloque `// Context Variables Initialization.` + `PLoadContext.Call(&Context)`. Es **ubicuo** (cientos de objetos).
2. **En la MasterPage**: cada objeto declara `MasterPage = 'PXTools.APIs.HMPWkWnn'`; la master page, en su `Start`/`Refresh`, también hace `PLoadContext` y **resuelve seguridad** (`PCheckContext` → si falla redirige a login; si OK, `PIsAuthorized`).

**Flujo completo:** login → `SaveLoginContext` → `PSaveContext` (arma desde BD) → `PSetContext` (a session); luego **cada request** → el objeto y su master page hacen `PLoadContext` (leen de session).

## 4. MasterPages (`Personalized/MasterPages/`)

Cada `HMPWkWnn` es un **nivel de shell** según el rol del objeto; hay variante desktop (Theme `PXToolsDesktop`) y responsive (`PXToolsResponsive`).

| Master page | Rol |
|---|---|
| `HMPWkW01` | **Main** desktop (menú izquierdo, header, top-menu, recent-links). |
| `HMPWkW02` | Objetos **Embedded**. |
| `HMPWkW03` | Main para **Embedded Data Area** (listas/grillas embebidas). |
| `HMPWkW04` | Main para **Transactions** (editor). |
| `HMPWkW05` | Main para **Embedded Data Area de Transactions**. |
| `HMPWkW06` | Main para **Prompts** (selección/lookup). |
| `HMPWkW07` / `HMPWkW08` | Embedded Data Area (en Embedded / de Transactions) con Recent Link. |
| `HMPWkW09` / `HMPWkW10` | **Responsive** (main / prompts). |
| `HMPLogin` | Shell de **login** (mínima: sin carga de contexto ni seguridad — es pre-autenticación). |
| `HMPExternal` | Objetos **externos/públicos** (fuera del área privada autenticada). |

Complementos: `HWCHdrDesktop`/`HWCHdrResponsive` (headers), `Publico`/`Privado` (marcos), y **`RetLayoutParameters`** (DataProvider que define las **regiones del layout responsive**: Container/Top/Left/Center/Right/Bottom por plataforma y por dimensión Bootstrap, con modos Push/Float/Smart y theme classes).

> **Cómo elige un objeto su master page:** no es runtime — el **pattern** escribe la propiedad `MasterPage = 'PXTools.APIs.HMPWkWnn'` en la cabecera del objeto generado, según su rol (lista / view / editor de transacción / prompt / embebido / responsive).

## 5. Configuración global — `PXToolsParametersSDT`

`LoadPXToolsParameters` (`Personalized/Parameters/`) carga, por plataforma, el SDT **`PXToolsParametersSDT`** (`APIs/SDTs/`), la **configuración central del framework**. Lo consultan los objetos generados y las master pages (accessor `PPXToolsParameters` por nombre). Flags destacados:

- **Login/seguridad:** `LoginSupport`, `AuthenticationMode` (local / windows).
- **Menús:** `SeparateMenusByUserType` + `MainBackOffice/FrontEnd/InviteMenuName`, `LeftMenuType`, `SupportTopMenusDropDown/Tab`, `MenuIncludeSearch`, `RecentLinkLevels`.
- **UI/popups:** `PageRows`, `PopupBehaviour` + `Default/Max/Min Popup Width/Height`, `Modal…`, colección `SpecialPopupWindows`.
- **Plataformas/paths:** `SupportedPlatforms`, `ImagesPath`, `DocumentsPath`, `ExcelExtension`/`ExcelTemplateDirectory`, `ProjectNameSpace`, `SupportLogicalDelete`.

## 6. Catálogo de grupos de APIs utilitarias (`APIs/`)

`APIs/` agrupa las utilidades en ~50 subcarpetas. Las principales:

| Grupo | Provee | Procs representativos |
|---|---|---|
| **Classes** | Manipular el string de clases CSS de un control. | `AddClass(inout, in)`, `HasClass`, `ToggleClass`. |
| **Date** | Semana ISO-8601 y tipo YearMonth con aritmética. | `DateOfWeekISO8601`, `YearMonthFromDate`, `YearMonthAddMonths`. |
| **Escape** | Encoding/escape XML-HTML y URL + extracción de nodos XML. | `Escape` (HtmlEncode), `EncodeURLParameters`, `GetXMLNodeContent`. |
| **StringTools** | Case, split/join, prefijos/sufijos, número→texto. | `Split`, `StartsWith`, `CapitalizedCase`, `NumberToDescription`. |
| **FormState** | Persistir/restaurar el estado de un web object en session, por nombre. | `SaveFormState`, `LoadFormState`, `ClearFormState`. |
| **File / Download** | Helpers de filesystem; descarga HTTP con MIME/Content-Disposition, retry y auto-delete. | `DelFile`, `PPEXE_SaveFile`; `DownloadFile`, SDT `DownloadSDT`. |
| **Excel** | Framework para armar Excel por template (celdas Row/Col/Value/format). | SDT `ExcelTemplateData`, `RetExcelElementFormat`. |
| **Message** | Encolar (session) y mostrar mensajes al usuario, con varios front-ends. | `PPEXE_Message`, `ShowMessages`, SDT `MessageCollection`. |
| **Config** | Cargar config (paths, classpath, WS) desde INI a un SDT `Config`. | `PLoadConfig`, `PPEXE_DeIni01`. |
| **Controller** | Navegación/flujo: call type, call stack, recent-links. | `PPEXE_DeCtl01`, SDT `ControllerCallStack`. |
| **Certificate** | X.509/keystore: PFX→JKS, public key de PEM. | `CreateKeyStoreFromPFX`, `GetPublicKeyFromPemFile`. |
| **Components / ErrorLog** | Naming interno de web components; log de errores deduplicado. | `RetComponentPrefix`; `AddErrorLog`, SDT `SDTErrorLog`. |
| **Confirm** | Diálogos reutilizables de confirmación/mensaje (desktop + responsive). | `HPEXE_Confirm`, `ResponsiveConfirm` (ver §9). |

*(Además: BlobTools, HTML, KB, Links, Network, Password, QRCode, Session, Tabs, Timezones, URL, Window, Zip, XMLUtil, Platform_APIs, …)*

## 7. APIs vs Personalized

### `APIs/` (core — no tocar)
El framework completo: contexto de bajo nivel (`Context_APIs`), el catálogo utilitario (§6), master pages base, SDTs (`PXToolsParametersSDT`, `MessageCollection`, `DownloadSDT`, …), y las instancias de patterns utilitarias (§9).

### `Personalized/` (puntos de extensión — se customiza por proyecto)
| Subcarpeta | Qué se customiza | Objeto(s) clave |
|---|---|---|
| **Context_Personalized** | El SDT `Context` y qué se carga/persiste por usuario (§3). | `Context`, `PSaveContext`, `PCheckContext`, `SaveLoginContext`. |
| **Parameters** | La **configuración central** del framework (§5). | `LoadPXToolsParameters`, `PXToolsParametersSDT`, `PPXToolsParameters`. |
| **SecurityConnector** | Implementación pluggable de autenticación/autorización (ver [security.md](security.md)). | `PCheckUser`, `PIsAuthorized`, `PIsAdministrator`, `HLogin`, `PXParameterRequestLogin`. |
| **MasterPages** | Los shells concretos + headers + regiones responsive (§4). | `HMPWkW01-10`, `HMPLogin`, `RetLayoutParameters`. |
| **Templates** | Los templates del generador que los patterns usan para scaffolding. | `PXToolsSelection/Transaction/View/ReportTemplate`, set `PXToolsSD*Template`. |
| **Languages** | Idiomas de UI soportados. | `RetSystemLanguages` → `SDTSystemLanguages`. |
| **Platform_Personalized** | Plataformas que targetea el proyecto. | `RetSuportedPlatforms` → `ApplicationPlatform` (WebDesktop, WebResponsive). |
| **System** | Shells del sistema: home, menú, startup, versión. | `HHome`, `HMenu`, `PXToolsWebStartUp`, `PDeVer01`. |
| **Files** | Diálogos de selección/upload de archivos. | `HPrFil01`, `HPrFil02`, `HPrFilEmbedded`. |
| **GridConfigurable** | Hook que decide (por nombre de objeto) si una grilla es configurable por el usuario. | `IsGridConfigurable`. |
| **Enums** | Anchor dummy que fuerza a incluir dominios enumerados en el build. | `PXToolsEnumsDummy`. |

## 8. Dominios del módulo

@APIs es el **hogar de los dominios base** del framework. Además de los que dependen del uso, por la regla de nomenclatura **todo dominio con prefijo `PXTools*` (namespace del framework) o de nombre primitivo/genérico pertenece a @APIs**, sin importar cuántos módulos lo usen.

**Module-scoped (`@APIs/#Domains/`) — 4:**

| Dominio | Tipo | Valores |
|---|---|---|
| **CallType** | Character(1) | Call=`C`, Return=`R`, Refresh=`F` |
| **GridHandlerColumnVisibility** | Character(5) | True=`true`, False=`false`, UserDefined=`user` |
| **GridHandlerConfigBoxAttachType** | Character(20) | ContextualBox, CreateContainer |
| **PasswordType** | Character(3) | Character=`CHR`, Numeric=`NUM` |

**Package-shared (`@PXTools/#Domains/`) — 10 (base @APIs):** `ContextualBoxItemType`, `ContextualBoxReturnData`, `DetectedDevice` (Mobile/Desktop), `ImageType` (Image/Icon), `ParametersStyle` (Positional/Named), `PositionElementCollision`, `PositionElementHorizontal`, `PositionElementVertical`, `UserControlEventAction`. *(`WhiteListReason` vive acá pero lo usa solo @WSLayer → es de [wslayer.md](wslayer.md).)*

**Root-legacy base (`#Domains/` raíz) — familias principales:**
- **Fecha/hora:** `PXToolsDate`, `PXToolsDateTimeMinutes`, `PXToolsDateTimeSeconds`, `PXToolsDateTimeScenario`, `PXToolsYearMonth`.
- **Email/red:** `PXToolsEmail`, `PXToolsConnectionType` (SMTP/POP3), `PXToolsHttpClient`, `URLConnectionType`, `FQDN`.
- **UI/ventana:** `WindowType`, `WindowEnumType` (Normal/Popup/Embedded/Component/Modal), `RefreshType`, `MessageType`, `NodeType`, `SDCallType`, `TabType`, `TabCode`, `SessionVariable`, `Origin`, `ApplicationPlatform`.
- **Excel:** `ExcelDocumentValueType`, `ExcelDocumentElementType`, `ExcelTemplateType`, `ExcelDocumentElementFormatType`, `ExcelDocumentFilters*`.
- **Controller:** `ControllerCommand`, `ControllerTarget`, `ControllerExternalEvent`.
- **Descripción:** `DescriptionShort/Medium/Large/Maximmum/Path`, `DecimalDescriptionType`.
- **Primitivos base:** `Boolean`, `MaxStr`, `MaxMem`, `MaxXML`, `MaxLnk`, `Links`, `VarLen`, `IdFirstLevel`, `IdSubLevel`, `Name`, `Error`, `ErrorDescription`, `Path`, `Version`, `AutoInc`, `Flag`, `Logico`, `Origin`, `Target`, `Move`, `Page`, `SortOrder`, `OrderLarge`, `ObjectDescription`, `Objetos`, `Language`, `LanguagePlatform`, `AuthenticationMode`, `OperatingSystem`, `CPUType`.
- **Contratos compartidos base:** `TaskManagerExecutionResponse`… **no** — ese es de [@TaskManager](taskmanager.md); pero sí `SDTCounterType` (Failed/Succeed), `NotificationMethod`, `SMSParameterDeliverStatus`, `PXToolsCharLengths`, `PXToolsModules` (registro de namespaces).

> Un dominio con nombre de otro módulo (`SystemParameter*`, `Security*`, `DynamicCallReference*`, `TaskManager*`…) **no** es de @APIs aunque @APIs lo consuma: se documenta en su módulo dueño.

## 9. Instancias de patterns del módulo

| Instancia | Tipo | Qué es |
|---|---|---|
| **PXParameterRequestConfirm** | ParameterRequest (popup) | **Popup genérico de confirmación Sí/No**. Recibe `&ConfirmText`, lo muestra, y devuelve el booleano al caller vía la session var `ConfirmResponse`. Genera `PuConfirm` (desktop) y `RPuConfirm` (responsive). |
| **PXWorkWithMessages** | WorkWith (popup) | Ventana que carga los mensajes encolados (`PLoadMessages`), los muestra en grilla y limpia la cola. |
| **PXParameterRequestEncodeUrl** | ParameterRequest | Utilidad: toma una URL, llama `EncodeURLParameters` y muestra el resultado escapado. |
| **PXParameterRequestNumberToDescription** | ParameterRequest | Demo del proc `NumberToDescription` (número → texto en palabras). |
| **PXParameterRequestLogin** | ParameterRequest (página login) | **Pantalla de login** (usuario/password + ambiente): valida credenciales/acceso, resuelve plataforma, guarda contexto y redirige al menú. Genera `WbLogin`. Muy personalizada por proyecto. Ver [security.md](security.md). |

## 10. Helpers públicos más invocados (`PXTools.APIs`)

| Helper | `Parm()` | Propósito |
|---|---|---|
| `PPEXE_Message` | `in: &Message` | Encola un mensaje (session) para mostrar luego. Ubicuo, incluso en batch/WS. |
| `ShowMessages` | — (lee session) | Vacía la cola: carga, limpia y emite cada mensaje con `Msg()`. |
| `PSetMessages` / `PLoadMessages` / `PClearMessages` | `MessageCollection` | Persistir / leer / limpiar la cola de mensajes en session. |
| `PPEXE_Link` | `&Link` | Redirector **platform-aware**: navega el programa actual al link (desktop vs responsive). |
| `PPEXE_Return` | `in: &WindowType` | Script client-side de retorno/cierre para devolver un popup a su caller. |
| `PPEXE_Refresh` | `in: &WindowSelf` | Refresca la ventana caller client-side. |
| `PPEXE_Run` | `in: &PathAndFile, in: &Params, in: &WorkingDir` | Ejecuta un proceso/comando externo del OS (backends C#/Java). |
| `DownloadFile` | — (HTTP, lee `DownloadSDT` de session) | Streamea un archivo del server al browser con Content-Type/Disposition + delete-after-download. |
| `EncodeURLParameters` / `DecodeURLParameters` / `Escape` / `UnEscape` | `in, out` | Encoding URL / HTML. |
| `PuConfirm` (`APIs/Confirm/`) | `in: &ConfirmText, &WindowSelf` | Popup de confirmación (§9). Se **abre como ventana modal** y la respuesta se lee de la session var `ConfirmResponse` (no por `.Call` directo). |
| `NumberToDescription` | `in: &Language, &Number, &DecimalDescriptionType, &MainInvoque, out: &Description` | Número → texto en palabras. |

## 11. Integración: cómo lo consume la aplicación

- **Contexto:** todo objeto de UI hace `PLoadContext.Call(&Context)` en su Start (lo inyecta el pattern); del `&Context` salen usuario, dominio/tenant y los datos de negocio que el proyecto embebió en el SDT `Context` (§3).
- **Confirmaciones:** para un "¿Está seguro?" se abre `PuConfirm` (o vía `HPEXE_Confirm`) y se lee `ConfirmResponse`. Los patterns exponen esto como el atributo `confirm` de una acción.
- **Mensajes:** encolar con `PPEXE_Message` y mostrar con `ShowMessages`.
- **Navegación/descargas:** `PPEXE_Link`/`PPEXE_Return`/`PPEXE_Refresh` para el flujo entre pantallas y popups; `DownloadFile` + `DownloadSDT` para exportar archivos.
- **Configuración/plataforma:** consultar `PXToolsParametersSDT` (vía `PPXToolsParameters`) para flags globales; las master pages y el layout responsive salen de `Personalized/MasterPages/`.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [security.md](security.md) — el SecurityConnector vive en `@APIs/Personalized/SecurityConnector/` (autenticación, `PIsAuthorized`/`PIsEnabled`, login).
- `modulos/taskmanager.md` — gestor de tareas (usa el contexto y los helpers de este módulo).
- `00-overview.md`, `01-pxworkwith.md`, `02-pxparameterrequest.md` — los patterns que consumen estas APIs (contexto, master pages, confirm, mensajes).
