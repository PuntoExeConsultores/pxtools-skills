# @Alerts Module — Alerts and Notifications

> Behaviour of the `@PXTools/@Alerts` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@Alerts/` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.Alerts`.
- **Depends on:** `@APIs` (base), `@SendMails` (the mail channel), `@SystemParameters`, `@DynamicCallReferences`, `@Menus`. In the canonical graph also `@FileStorage` (attachments), `@MailAccounts` and `@TaskManager` (optional).

## 1. What it provides

A framework of **multi-channel alerts/notifications** (**mail** + **in-app notification**), **multi-language**, with **categories**, **priorities**, **subscribe/unsubscribe**, **retries**, **time-outs** with an action process, and **periodicity cycles**. An alert is created suspended with a trigger date; an engine processes it and sends it through the enabled channels to its recipients (users or roles).

## 2. Core concept

- **An alert (`SystemAlert`)** = a header (trigger date, status, priority, category, retry/time-out configuration) plus **recipients** (`Parties` — each one a **user or a role**, domain `SecurityPartyType`).
- **Channels (`SystemAlertTypes`)**: Mail and/or System (in-app), each with its own state machine.
- **Per-language content (`SystemAlertLanguages`)**: subject + message (HTML/Text) + attachments, with a language fallback.
- **In-app inbox (`SystemAlertUserStates`)**: per-user status (Active/Viewed/Discarded).
- **Categories (`SystemAlertCategories`)**: a catalogue with priority, periodicity and enabled channels; users can **unsubscribe** per category.

## 3. Module transactions

`SystemAlertId` = `IdFirstLevel` (autonumbered). The main ones:

| Transaction | PK | Role |
|---|---|---|
| **SystemAlert** | `SystemAlertId` | The alert's header. Status (`SystemAlertState`), origin, priority, category, trigger date/time; retry (`ActionRetry…`) and time-out (`ActionTimeOut…` + `ActionTimeOutProcess`) configuration. Nested **Parties** level (`SystemAlertSecurityPartyId` + `…Type` User/Role). |
| **SystemAlertTypes** | `SystemAlertId, SystemAlertType` (Mail/System) | Enabled channel + `SystemAlertTypeState`. |
| **SystemAlertLanguages** | `SystemAlertId, SystemAlertLanguage` | `Subject`, `Message` (MaxMem), `MessageType` (HTML/Text). An **Attachs** level (Internal/External/FileStorage attachments). |
| **SystemAlertCategories** | `SystemAlertCategoryCode` | Catalogue: description, priority, `Periodicity` (every N days), `MaxRange`, channels (formulas). A `SystemAlertCategoryTypes` level (channel enabled/disabled per category). |
| **SystemAlertUnsubscriptions** | `CategoryCode, SecurityUserId` | User↔category unsubscription. |
| **SystemAlertUserStates** | `SystemAlertId, SecurityUserId` | Per-user in-app status. |
| **SystemAlertReferences / …References2** | `SystemAlertId (+ RefId)` | Key/value references; **v2** stores a `QueryString` uniquely identifying the alert, for **deduplication and cycle computation**. |
| **SystemAlertRetries** | `SystemAlertId, RetryDateTime` | Retry log (Alert/Error). |

## 4. Module domains

All `SystemAlert*` domains belong to @Alerts (by naming), all **root-legacy** (they live in the root `#Domains/`):

| Domain | Values |
|---|---|
| **SystemAlertState** | Suspended=`SUS`, Active=`ACT`, Discarded=`DIS`, Fail=`FAI`, Created=`CRE`, TimeOut=`TMO` |
| **SystemAlertTypeState** | + `ErrorRetry=TRY`, `ActionRetry=ART` (per channel) |
| **SystemAlertUserState** | Active=`ACT`, Viewed=`VIE`, Discarded=`DIS` |
| **SystemAlertType** | Mail=`M`, System=`S` |
| **SystemAlertOrigin** | Manual=`M`, System=`S` |
| **SystemAlertPriority** | High=`HIG`, Medium=`MED`, Low=`LOW` |
| **SystemAlertRetryType** | Alert=`A`, Error=`E` |
| **SystemAlertSubjectFilterTo** | Description, LanguageSubject |
| **SystemAlertUserDiscardType** | ToMe, ToAll, SameDayAndCategoryToMe, SameDayAndCategoryToAll |
| **SystemAlertCategoryCode** | Char(50), an **extensible enum**: a typed catalogue of categories; each application adds its own values. |

**Used from other modules:** `MailMessageType` / `AttachmentType` (@SendMails), `SecurityPartyType` / `SecurityPartyIdCollection` (@Security — even though only @Alerts uses `SecurityPartyIdCollection`, it belongs to @Security by naming), `Language` (@APIs base). Note: `NotificationMethod` and `SMSParameterDeliverStatus` do **not** belong to @Alerts (only @APIs uses them).

## 5. How it works

### 5.1 Creating an alert — `AddSystemAlertSDT`
`parm(in: &SDTSystemAlert, out: &SystemAlertId)` (`ExecuteInNewLUW=True`). `SDTSystemAlert` bundles everything: `DateTime, Description, Priority, CategoryCode`, the retry/time-out configuration, `Types[]` (channels), `Languages[]` (subject/message/type + `Attachments[]`), `SecurityPartyId[]` (recipients), `References[]`. It filters the channels against `SystemAlertCategoryTypes` (honouring the disabled ones), inserts the header (State=Suspended, Origin=System) and its children, and if there are references it creates a `SystemAlertReferences2` row with the query string for deduplication/cycles. Simple wrappers: `AddSystemAlertParty` (one recipient/channel), `AddSystemAlertParties` (a collection, System channel).

### 5.2 Sending — `TskAlerts` (the engine)
`Parm(in: &TaskManagerId, out: &Error, out: &ErrorMessage)`:
1. It walks the alerts that are **Suspended with a date ≤ now** and those that are **Active**.
2. For each channel in `Suspended`/`ActionRetry`:
   - **Time-out**: if it has expired, set `…TimeOut` and dynamically invoke `ActionTimeOutProcess`.
   - **Cycle**: for categories with `MaxRange>0`, compute the periodicity (`SystemAlertReferences2` + `RetSystemAlertFromReferences2FirstAlertDate`) and send only on the matching days.
   - **Mail**: build `MailData` (`PXTools.SendMails`), inject the alert master page, replace `[!BodyContent!]`, add the `[!Unsubscription!]` link (Base64), expand the recipients (User / Role / Domain-Role), apply the user's language (`RetSystemAlertLanguage`), check the subscription, attach files and send through `SendMailOutboxWithErrorRetry` (To for users, BCC for roles).
   - **System**: create/update `SystemAlertUserStates` (Active) → it shows up in the user's inbox.
   - When finished: the channel goes to `ActionRetry` or `Discarded`.
3. `UpdSystemAlertState` recomputes the global status; the `'Clean User Alerts'` subroutine purges anything older than `AlertBackDaysToDepure`.

### 5.3 How it runs (the scheduler)
- **`PrcSystemAlert`** (`IsMain=True, CallProtocol=CommandLine`): takes the lock (`StartProcessStatus`), calls `TskAlerts`, releases it. The manual variant (also the WW's "Process Alerts" button).
- **Scheduled** execution is done by **@TaskManager**: `Personalized/RetDynamicCallReferenceAlerts` registers `TskAlerts` as a `ReferenceType.TaskManagerExecution` plus two `TableCleanerProcess` entries.

### 5.4 Subscribing / unsubscribing
`AddSystemAlertUnsubscription` / `DelSystemAlertUnsubscription` (user+category). The unsubscribe link in the mail opens the external `UnsubscriptionConfirmation` panel (it validates the email → `AddSystemAlertUnsubscription`). `TskAlerts` skips unsubscribed recipients.

## 6. APIs vs Personalized

- **`APIs/`** (core): every transaction, the `SDTSystemAlert` SDT, the `TskAlerts`/`PrcSystemAlert` engine, and the whole `Add*/Ret*/Chk*/Val*/Upd*/Dlt*` family.
- **`Personalized/`** (the project's customization):
  | Object | What gets customized |
  |---|---|
  | `RetSystemAlertCategoriesCodes` (DataProvider) | **The project's own categories** (code + description) to be seeded. Usually ships empty (a shell). |
  | `AddSystemAlertCategoriesCodes` (Procedure) | Seeding orchestrator: gathers the declared categories, upserts them and deletes the orphans. |
  | `RetMenusAlerts` (DataProvider) | The module's menu entry (Alerts / Categories / Unsubscriptions). |
  | `RetDynamicCallReferenceAlerts` (DataProvider) | Registration of `TskAlerts` + the table cleaners in @TaskManager. |

Parameterization: `APIs/RetSystemParametersSystemAlerts` publishes the System Parameters (category `SystemAlerts`) governing the module (`AlertBackDaysToDepure`, `AlertDefaultMailAccount`, `AlertMailMasterPage`, `AlertUnsubscriptionPanelURL`, and so on).

## 7. Pattern instances

| Instance | What it is |
|---|---|
| **PXWorkWithSystemAlert** | The main CRUD screen + the **in-app inbox** (`SystemAlertMessages`/`…Home`). Calendar/Table views; Activate/Deactivate/Process Alerts actions; detail with Languages/References/Parties/Retries. |
| **PXWorkWithSystemAlertLanguages** | CRUD for the per-language content (subject/message, HTML or text). |
| **PXWorkWithSystemAlertCategories** | CRUD for categories (Mail/System checkboxes, Periodicity/MaxRange). The "Upgrade …Codes" action → `AddSystemAlertCategoriesCodes`. |
| **PXWorkWithSystemAlertUnsubscriptions** | Query/removal of unsubscriptions. |
| **PXComposerSystemAlertMessage / …SchedulerView** | Popups (alert content / calendar). |
| **PXParameterRequestSystemAlertMessage** | "Alert Content View": shows the message and allows discarding it (marks Viewed/Discarded). |
| **PXParameterRequestUnsubscriptionConfirmation** | The **external** panel (master page `HMPExternal`) confirming an unsubscription. |
| **PXParameterRequestPPEXE_PrcSystemAlert** | Request UI for launching `PrcSystemAlert`. |

## 8. Key procedures / APIs

**Creation**: `AddSystemAlertSDT(in: &SDTSystemAlert, out: &SystemAlertId)` (the main one), `AddSystemAlertParty/Parties` (wrappers), `AddSystemAlertCategoriesCode`, `AddSystemAlertUnsubscription`, `AddSystemAlertUserCategoriesExcluded`.

**Queries**: `RetSystemAlertLanguage(id, lang, out sdt)` (with fallback), `RetSystemAlertTypes`, `RetSystemAlertIdFromReferences/…2` (lookup by references), `RetSystemAlertFromReferences2FirstAlertDate` (cycle).

**Checks/validation**: `ChkSystemAlertTypeEnabled`, `ValSystemAlertParty` (is this a recipient?), `ValSystemAlertLanguages` (does it have content? — a prerequisite for activation), `ValSystemAlertUserState` (inbox badge).

**Status**: `UpdSystemAlertActivate/Deactivate`, `UpdSystemAlertState`, `UpdSystemAlertTypeState[s]`, `UpdSystemAlertUserState` (0 = everyone), `UpdActionTakenFromReferences` (marks "action taken" and replies on the thread).

**Removal/cleanup**: `DelSystemAlertUnsubscription`, `DltSystemAlertFromNotificactions` (discarding by `SystemAlertUserDiscardType`), `PrcTableCleanerAlerts[…UserState]`.

**Processing**: `TskAlerts(in: &TaskManagerId, out…)` (the engine), `PrcSystemAlert` (the CommandLine wrapper).

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [taskmanager.md](taskmanager.md) — runs `TskAlerts` on a schedule (`RetDynamicCallReferenceAlerts`).
- [security.md](security.md) — `Parties` are users or roles (`SecurityParty`); recipients are expanded by role/domain.
- The **@SendMails** (sending the Mail channel), **@SystemParameters** (configuration), **@FileStorage** (attachments) and **@ProcessMonitor** (process lock) modules.
