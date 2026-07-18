# Módulo @Alerts — Alertas y Notificaciones

> Comportamiento del módulo `@PXTools/@Alerts`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@Alerts/` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.Alerts`.
- **Depende de:** `@APIs` (base), `@SendMails` (canal correo), `@SystemParameters`, `@DynamicCallReferences`, `@Menus`. En el grafo canónico también `@FileStorage` (adjuntos), `@MailAccounts` y `@TaskManager` (opcional).

## 1. Qué provee

Un framework de **alertas/notificaciones multicanal** (**correo** + **notificación in-app**), **multi-idioma**, con **categorías**, **prioridades**, **suscripción/desuscripción**, **reintentos**, **time-out** con proceso de acción, y **ciclos de periodicidad**. Una alerta se crea suspendida con fecha de disparo; un motor la procesa y la envía por los canales habilitados a sus destinatarios (usuarios o roles).

## 2. Concepto central

- **Alerta (`SystemAlert`)** = cabezal (fecha de disparo, estado, prioridad, categoría, config de reintento/time-out) con **destinatarios** (`Parties` — cada uno es un **usuario o rol**, dominio `SecurityPartyType`).
- **Canales (`SystemAlertTypes`)**: Mail y/o System (in-app), cada uno con su propia máquina de estados.
- **Contenido por idioma (`SystemAlertLanguages`)**: subject + message (HTML/Text) + adjuntos, con fallback de idioma.
- **Bandeja in-app (`SystemAlertUserStates`)**: estado por usuario (Active/Viewed/Discarded).
- **Categorías (`SystemAlertCategories`)**: catálogo con prioridad, periodicidad y canales habilitados; los usuarios pueden **desuscribirse** por categoría.

## 3. Transacciones del módulo

`SystemAlertId` = `IdFirstLevel` (autonum). Las principales:

| Transacción | PK | Rol |
|---|---|---|
| **SystemAlert** | `SystemAlertId` | Cabezal de la alerta. Estado (`SystemAlertState`), origen, prioridad, categoría, fecha/hora de disparo; config de reintento (`ActionRetry…`) y time-out (`ActionTimeOut…` + `ActionTimeOutProcess`). Nivel anidado **Parties** (`SystemAlertSecurityPartyId` + `…Type` User/Role). |
| **SystemAlertTypes** | `SystemAlertId, SystemAlertType` (Mail/System) | Canal habilitado + `SystemAlertTypeState`. |
| **SystemAlertLanguages** | `SystemAlertId, SystemAlertLanguage` | `Subject`, `Message` (MaxMem), `MessageType` (HTML/Text). Nivel **Attachs** (adjuntos Internal/External/FileStorage). |
| **SystemAlertCategories** | `SystemAlertCategoryCode` | Catálogo: descripción, prioridad, `Periodicity` (cada N días), `MaxRange`, canales (fórmulas). Nivel `SystemAlertCategoryTypes` (canal habilitado/deshabilitado por categoría). |
| **SystemAlertUnsubscriptions** | `CategoryCode, SecurityUserId` | Desuscripción usuario↔categoría. |
| **SystemAlertUserStates** | `SystemAlertId, SecurityUserId` | Estado in-app por usuario. |
| **SystemAlertReferences / …References2** | `SystemAlertId (+ RefId)` | Referencias clave/valor; **v2** guarda un `QueryString` que identifica unívocamente la alerta para **dedup y cálculo de ciclo**. |
| **SystemAlertRetries** | `SystemAlertId, RetryDateTime` | Bitácora de reintentos (Alert/Error). |

## 4. Dominios del módulo

Todos `SystemAlert*` → @Alerts (nomenclatura), **root-legacy** (viven en el `#Domains/` raíz):

| Dominio | Valores |
|---|---|
| **SystemAlertState** | Suspended=`SUS`, Active=`ACT`, Discarded=`DIS`, Fail=`FAI`, Created=`CRE`, TimeOut=`TMO` |
| **SystemAlertTypeState** | + `ErrorRetry=TRY`, `ActionRetry=ART` (por canal) |
| **SystemAlertUserState** | Active=`ACT`, Viewed=`VIE`, Discarded=`DIS` |
| **SystemAlertType** | Mail=`M`, System=`S` |
| **SystemAlertOrigin** | Manual=`M`, System=`S` |
| **SystemAlertPriority** | High=`HIG`, Medium=`MED`, Low=`LOW` |
| **SystemAlertRetryType** | Alert=`A`, Error=`E` |
| **SystemAlertSubjectFilterTo** | Description, LanguageSubject |
| **SystemAlertUserDiscardType** | ToMe, ToAll, SameDayAndCategoryToMe, SameDayAndCategoryToAll |
| **SystemAlertCategoryCode** | Char(50) **enumerado extensible**: catálogo tipado de categorías; cada aplicación agrega sus valores. |

**Usa de otros módulos:** `MailMessageType` / `AttachmentType` (@SendMails), `SecurityPartyType` / `SecurityPartyIdCollection` (@Security — pese a que `SecurityPartyIdCollection` solo lo usa @Alerts, es de @Security por nomenclatura), `Language` (@APIs base). Nota: `NotificationMethod` y `SMSParameterDeliverStatus` **no** son de @Alerts (los usa solo @APIs).

## 5. Mecanismo

### 5.1 Crear una alerta — `AddSystemAlertSDT`
`parm(in: &SDTSystemAlert, out: &SystemAlertId)` (`ExecuteInNewLUW=True`). El `SDTSystemAlert` agrupa todo: `DateTime, Description, Priority, CategoryCode`, config de retry/time-out, `Types[]` (canales), `Languages[]` (subject/message/type + `Attachments[]`), `SecurityPartyId[]` (destinatarios), `References[]`. Filtra los canales contra `SystemAlertCategoryTypes` (respeta deshabilitados), inserta el cabezal (State=Suspended, Origin=System) y sus hijos, y si hay referencias crea un `SystemAlertReferences2` con el querystring para dedup/ciclo. Wrappers simples: `AddSystemAlertParty` (un destinatario/canal), `AddSystemAlertParties` (colección, canal System).

### 5.2 Enviar — `TskAlerts` (motor)
`Parm(in: &TaskManagerId, out: &Error, out: &ErrorMessage)`:
1. Recorre alertas **Suspended con fecha ≤ ahora** y **Active**.
2. Por cada canal en `Suspended`/`ActionRetry`:
   - **Time-out**: si venció, `…TimeOut` e invoca dinámicamente `ActionTimeOutProcess`.
   - **Ciclo**: para categorías con `MaxRange>0`, calcula periodicidad (`SystemAlertReferences2` + `RetSystemAlertFromReferences2FirstAlertDate`) y solo envía en los días que corresponden.
   - **Mail**: arma `MailData` (`PXTools.SendMails`), inyecta la master page de alerta, reemplaza `[!BodyContent!]`, agrega el link `[!Unsubscription!]` (Base64), expande destinatarios (User / Role / Domain-Role), aplica el idioma del usuario (`RetSystemAlertLanguage`), chequea suscripción, adjunta y envía con `SendMailOutboxWithErrorRetry` (To usuarios, BCC roles).
   - **System**: crea/actualiza `SystemAlertUserStates` (Active) → aparece en la bandeja del usuario.
   - Al terminar: canal → `ActionRetry` o `Discarded`.
3. `UpdSystemAlertState` recalcula el estado global; sub `'Clean User Alerts'` purga las más viejas que `AlertBackDaysToDepure`.

### 5.3 Cómo corre (scheduler)
- **`PrcSystemAlert`** (`IsMain=True, CallProtocol=CommandLine`): toma lock (`StartProcessStatus`), llama `TskAlerts`, libera. Variante manual (también botón "Process Alerts" del WW).
- La ejecución **programada** la hace **@TaskManager**: `Personalized/RetDynamicCallReferenceAlerts` registra `TskAlerts` como `ReferenceType.TaskManagerExecution` + dos `TableCleanerProcess`.

### 5.4 Suscripción / desuscripción
`AddSystemAlertUnsubscription` / `DelSystemAlertUnsubscription` (usuario+categoría). El link de baja del mail abre el panel externo `UnsubscriptionConfirmation` (valida email → `AddSystemAlertUnsubscription`). `TskAlerts` omite destinatarios desuscriptos.

## 6. APIs vs Personalized

- **`APIs/`** (core): todas las transacciones, el SDT `SDTSystemAlert`, el motor `TskAlerts`/`PrcSystemAlert`, y toda la familia `Add*/Ret*/Chk*/Val*/Upd*/Dlt*`.
- **`Personalized/`** (customización del proyecto):
  | Objeto | Qué se customiza |
  |---|---|
  | `RetSystemAlertCategoriesCodes` (DataProvider) | **Las categorías propias del proyecto** (código + descripción) a sembrar. Suele venir vacío (cascarón). |
  | `AddSystemAlertCategoriesCodes` (Procedure) | Orquestador de siembra: junta las categorías declaradas, hace upsert y borra las huérfanas. |
  | `RetMenusAlerts` (DataProvider) | La entrada de menú del módulo (Alerts / Categories / Unsubscriptions). |
  | `RetDynamicCallReferenceAlerts` (DataProvider) | Registro en @TaskManager de `TskAlerts` + los table-cleaners. |

Parametrización: `APIs/RetSystemParametersSystemAlerts` publica los System Parameters (categoría `SystemAlerts`) que gobiernan el módulo (`AlertBackDaysToDepure`, `AlertDefaultMailAccount`, `AlertMailMasterPage`, `AlertUnsubscriptionPanelURL`, etc.).

## 7. Instancias de patterns

| Instancia | Qué es |
|---|---|
| **PXWorkWithSystemAlert** | ABM principal + **bandeja in-app** (`SystemAlertMessages`/`…Home`). Vistas Calendar/Table; acciones Activate/Deactivate/Process Alerts; detalle con Languages/References/Parties/Retries. |
| **PXWorkWithSystemAlertLanguages** | ABM del contenido por idioma (subject/message HTML o texto). |
| **PXWorkWithSystemAlertCategories** | ABM de categorías (checks Mail/System, Periodicity/MaxRange). Acción "Upgrade …Codes" → `AddSystemAlertCategoriesCodes`. |
| **PXWorkWithSystemAlertUnsubscriptions** | Consulta/baja de desuscripciones. |
| **PXComposerSystemAlertMessage / …SchedulerView** | Popups (contenido de alerta / calendario). |
| **PXParameterRequestSystemAlertMessage** | "Alert Content View": muestra el mensaje y permite descartar (marca Viewed/Discarded). |
| **PXParameterRequestUnsubscriptionConfirmation** | Panel **externo** (master page `HMPExternal`) de confirmación de baja. |
| **PXParameterRequestPPEXE_PrcSystemAlert** | Request UI para lanzar `PrcSystemAlert`. |

## 8. Procedimientos / APIs clave

**Alta**: `AddSystemAlertSDT(in: &SDTSystemAlert, out: &SystemAlertId)` (principal), `AddSystemAlertParty/Parties` (wrappers), `AddSystemAlertCategoriesCode`, `AddSystemAlertUnsubscription`, `AddSystemAlertUserCategoriesExcluded`.

**Consulta**: `RetSystemAlertLanguage(id, lang, out sdt)` (con fallback), `RetSystemAlertTypes`, `RetSystemAlertIdFromReferences/…2` (buscar por referencias), `RetSystemAlertFromReferences2FirstAlertDate` (ciclo).

**Chequeo/validación**: `ChkSystemAlertTypeEnabled`, `ValSystemAlertParty` (¿es destinatario?), `ValSystemAlertLanguages` (¿tiene contenido? — requisito para activar), `ValSystemAlertUserState` (badge de bandeja).

**Estado**: `UpdSystemAlertActivate/Deactivate`, `UpdSystemAlertState`, `UpdSystemAlertTypeState[s]`, `UpdSystemAlertUserState` (0 = todos), `UpdActionTakenFromReferences` (marca "acción tomada" y responde el hilo).

**Baja/limpieza**: `DelSystemAlertUnsubscription`, `DltSystemAlertFromNotificactions` (descarte por `SystemAlertUserDiscardType`), `PrcTableCleanerAlerts[…UserState]`.

**Proceso**: `TskAlerts(in: &TaskManagerId, out…)` (motor), `PrcSystemAlert` (wrapper CommandLine).

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [taskmanager.md](taskmanager.md) — ejecuta `TskAlerts` de forma programada (`RetDynamicCallReferenceAlerts`).
- [security.md](security.md) — `Parties` son usuarios o roles (`SecurityParty`); los destinatarios se expanden por rol/dominio.
- Módulos **@SendMails** (envío del canal Mail), **@SystemParameters** (config), **@FileStorage** (adjuntos), **@ProcessMonitor** (lock del proceso).
