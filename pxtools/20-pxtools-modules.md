# @PXTools Modules — Reusable Cross-Cutting Functionality

## What they are

The @PXTools modules are **reusable packages of GeneXus objects** providing functionality that cuts across any application. They install under `Knowledge Base/@PXTools/@<ModuleName>/` and many of them include their own PXTools pattern instances (PXWorkWith for CRUD screens, PXParameterRequest for forms, PXComposer for composed views).

## Location in the KB

```
Knowledge Base/
└── @PXTools/
    ├── @APIs/          ← the framework's utility APIs
    ├── @Alerts/        ← alert system
    ├── @Audit/         ← auditing
    ├── @CloudTasks/    ← cloud tasks
    ├── ...
    └── @WebServicesLog/ ← web service log
```

## Module catalogue

> This page is the module **index**. The ones with detailed behaviour documentation link to their file in the [`modules/`](modules/) subfolder (e.g. [modules/security.md](modules/security.md)).

### @APIs — The Framework's Utility APIs

**Purpose**: the base module holding the PXTools framework's internal APIs. It provides the cross-cutting utilities every other module and every pattern uses.

> 📄 **Detailed module documentation:** [modules/apis.md](modules/apis.md)

**Submodules and contents**:
- **#Domains**: base domains (CallType, GridHandlerColumnVisibility, GridHandlerConfigBoxAttachType, PasswordType)
- **APIs/Certificate**: certificate management
- **APIs/Classes**: 8 procedures for manipulating CSS classes
- **APIs/Components**: reusable UI components
- **APIs/Config**: configuration SDTs and Procedures
- **APIs/Confirm**: confirmation WebPanels (`PXParameterRequestConfirm`)
- **APIs/Context_APIs**: 3 procedures for context handling
- **APIs/Controller**: controller SDTs and Procedures
- **APIs/Date**: 8 procedures for date operations
- **APIs/Download**: download utilities
- **APIs/ErrorLog**: error logging
- **APIs/Escape**: 5 procedures for data escaping (`PXParameterRequestEncodeUrl`)
- **APIs/Excel**: SDTs and procedures for Excel operations
- **APIs/File**: file management
- **APIs/FormState**: several procedures for form state
- **APIs/Message**: messaging (`PXWorkWithMessages`)
- **APIs/StringTools**: text tools (`PXParameterRequestNumberToDescription`)
- **Personalized/SecurityConnector**: the security connector (`PXParameterRequestLogin`)

**PXTools instances included**: PXParameterRequestConfirm, PXParameterRequestEncodeUrl, PXWorkWithMessages, PXParameterRequestNumberToDescription, PXParameterRequestLogin

---

### @Alerts — Alert and Notification System

**Purpose**: a complete system of schedulable alerts supporting several delivery channels, languages, categories and subscriptions/unsubscriptions.

> 📄 **Detailed module documentation:** [modules/alerts.md](modules/alerts.md)

**Contents**:
- Transactions: SystemAlert, SystemAlertLanguages, SystemAlertCategories, SystemAlertUnsubscriptions
- Procedures: PrcSystemAlert (alert processing)
- A scheduler for programmed delivery

**PXTools instances included**:
- `PXWorkWithSystemAlert` — alerts CRUD
- `PXWorkWithSystemAlertLanguages` — alert languages CRUD
- `PXWorkWithSystemAlertCategories` — categories CRUD
- `PXWorkWithSystemAlertUnsubscriptions` — unsubscriptions CRUD
- `PXComposerSystemAlertMessage` — composed message view
- `PXComposerSystemAlertSchedulerView` — scheduler view
- `PXParameterRequestSystemAlertMessage` — message form
- `PXParameterRequestUnsubscriptionConfirmation` — unsubscription confirmation
- `PXParameterRequestPPEXE_PrcSystemAlert` — alert processing

---

### @Audit — Auditing

**Purpose**: an auditing system recording changes to entities (the PXAudit pattern).

> ⚠️ **Not present in this KB.** There is no `@PXTools/@Audit/` folder. Auditing, when used, is applied with the **PXAudit** pattern directly over the project's transactions (not as an installed module). This entry is kept only as a reference to the PXTools catalogue.

---

### @CloudTasks — Scheduled Cloud Tasks

**Purpose**: management and execution of scheduled tasks. It supports certificates, process monitoring, and temporary file cleanup.

> 📄 **Detailed module documentation:** [modules/cloudtasks.md](modules/cloudtasks.md)

**Contents**:
- Transactions: CloudTasks, CloudTasksCertificates, KeyStores, CloudTaskDeleteFiles, CloudTasksProcessMonitor
- Debug and simulation utilities

**PXTools instances included**:
- `PXWorkWithCloudTasks` — tasks CRUD
- `PXWorkWithCloudTasksCertificates` — certificates CRUD
- `PXWorkWithKeyStores` — keystores CRUD
- `PXWorkWithCloudTaskDeleteFiles` — deletion tasks CRUD
- `PXWorkWithSimulateDeleteFiles` — simulator
- `PXWorkWithCloudTasksProcessMonitor` — process monitor
- `PXParameterRequestCloudTaskStatusMessage` — status message
- `DebugCloudTaskOnDemand` — on-demand debug
- `DebugCloudTasksUtils` — debug utilities

---

### @ControlPreferences — UI Control Preferences

**Purpose**: storing and retrieving user control preferences (column widths, visibility, order, and so on).

> 📄 **Detailed module documentation:** [modules/controlpreferences.md](modules/controlpreferences.md)

---

### @DynamicCallReferences — Dynamic Call References

**Purpose**: management of dynamic references to GeneXus objects. It allows invoking objects by name/reference at runtime without hard-coding the calls.

> 📄 **Detailed module documentation:** [modules/dynamiccallreferences.md](modules/dynamiccallreferences.md)

**PXTools instances included**:
- `PXWorkWithTDynamicCallReferences` — references CRUD

---

### @ExportImport — Data Export and Import

**Purpose**: infrastructure for exporting and importing data in different formats.

> 📄 **Detailed module documentation:** [modules/exportimport.md](modules/exportimport.md)

**PXTools instances included**:
- `PXParameterRequestImport` — import form
- `PXReportTemplateRepImportResult` — results report

---

### @FileStorage — File Storage

**Purpose**: a complete file management system supporting several storage servers, predefined categories, and file CRUD operations.

> 📄 **Detailed module documentation:** [modules/filestorage.md](modules/filestorage.md)

**PXTools instances included**:
- `PXWorkWithFileStorage` — files CRUD
- `PXWorkWithFileStorageServers` — servers CRUD
- `PXWorkWithFileStorageStorage` — storage CRUD
- `PXWorkWithPredefinedCategories` — categories CRUD
- `PXComposerFileStorage` — composed file view

---

### @MailAccounts — Mail Accounts

**Purpose**: management of the email accounts used for sending and receiving.

> 📄 **Detailed module documentation:** [modules/mailaccounts.md](modules/mailaccounts.md)

**PXTools instances included**:
- `PXWorkWithMailAccounts` — mail accounts CRUD

---

### @Menus — Menu System

**Purpose**: management of web navigation menus (a basic menu with hierarchical items).

> 📄 **Detailed module documentation:** [modules/menus.md](modules/menus.md)

**PXTools instances included**:
- `PXWorkWithTMnuWeb` — web menus CRUD

---

### @OAV — Object Attribute Values (Base)

**Purpose**: the base infrastructure for the PXOAV pattern. It provides the transactions and CRUD screens for managing dynamic attribute definitions and their values.

> 📄 **Detailed module documentation:** [modules/oav.md](modules/oav.md)

**PXTools instances included**:
- `PXWorkWithTOAVAttributes` — OAV attributes CRUD
- `PXWorkWithTOAVAttributeValues` — values CRUD
- `PXWorkWithTSystemObjectOAVAttributes` — CRUD for system objects with OAV
- `PXParameterRequestSaveOAVSystemObjects` — save OAV objects

---

### @ProcessMonitor — Process Monitor

**Purpose**: monitoring running processes with statuses, messages and process servers.

> 📄 **Detailed module documentation:** [modules/processmonitor.md](modules/processmonitor.md)

**PXTools instances included**:
- `PXWorkWithProcessStatus` — process statuses CRUD
- `PXWorkWithProcessStatusMessages` — messages CRUD
- `PXWorkWithProcessServers` — servers CRUD
- `PXWorkWithProcessStatusAdvanced` — advanced view

---

### @Projects — Project Management

**Purpose**: managing projects with types, members and roles.

> 📄 **Detailed module documentation:** [modules/projects.md](modules/projects.md)

**PXTools instances included**:
- `PXWorkWithProjects` — projects CRUD
- `PXWorkWithProjectTypes` — types CRUD
- `PXWorkWithProjectsMembers` — members CRUD

---

### @ReceiveMails — Incoming Mail

**Purpose**: processing incoming email.

> 📄 **Detailed module documentation:** [modules/receivemails.md](modules/receivemails.md)

**PXTools instances included**:
- `PXWorkWithReceivedMails` — received mail CRUD

---

### @ResponsiveLayout — *(a GeneXus module, not a PXTools one)*

**Purpose**: responsive positioning of sections. It is a **GeneXus platform module** (it uses the native responsive component), **not a PXTools module**; it appears under `@PXTools/` only as glue.

> ℹ️ **Not a PXTools module.** Under `@PXTools/@ResponsiveLayout/` there are only **layout parameter SDTs** (`Parameters`, `Panel{Top,Bottom,Left,Right,Center,Container}Properties`) and an example DataProvider (`RetLayoutParametersExample`) supporting the PXTools User Controls. No transactions, procedures or patterns. It does not count as a PXTools module.

---

### @Security — Security

**Purpose**: a complete security system with users, roles, domains, object and record access permissions, user registration, and silent single sign-on.

> 📄 **Detailed module documentation:** [modules/security.md](modules/security.md)

**PXTools instances included**:
- `PXWorkWithSecurityUsers` — users CRUD
- `PXWorkWithSecurityUsersDomains` — user domains CRUD
- `PXWorkWithSecurityRoles` — roles CRUD
- `PXWorkWithSecurityDomains` — domains CRUD
- `PXWorkWithSecurityObjectAccess` — object access CRUD
- `PXWorkWithSecurityObjectRecordsAccess` — record access CRUD
- `PXWorkWithSystemObjects` — system objects CRUD
- `PXWorkWithRegistrations` — user registrations CRUD
- `PXWorkWithSilentSignOnRequests` — SSO CRUD
- `PXComposerSecurityObjectAccess` — composed object access view
- `PXComposerSecurityObjectRecordAccess` — composed record access view
- `PXParameterRequestChangePassword` — password change
- `PXParameterRequestRegistrationBasic` — basic registration
- `PXParameterRequestRegistrationConfirmed` — registration with confirmation

---

### @SecurityProjects — Per-Project Security

**Purpose**: an extension of @Security for per-project access control.

> ℹ️ **No dedicated doc.** The module contains only **subtype groups** (`ProjectMemberSecurityUserId`, `ProjectMemberRolSecurityUserId`, `ProjectMemberRolSecurityRoleId`) connecting [@Projects](modules/projects.md) members with [@Security](modules/security.md) users/roles. It contributes no transactions or patterns of its own; its mechanics are described in [modules/projects.md](modules/projects.md).

---

### @SendMails — Outgoing Mail

**Purpose**: managing outgoing mail with an outbox, delivery series and tracking.

> 📄 **Detailed module documentation:** [modules/sendmails.md](modules/sendmails.md)

**PXTools instances included**:
- `PXWorkWithSendMailOutbox` — outbox CRUD
- `PXWorkWithSendMailOutboxSeries` — series CRUD
- `PXParameterRequestChange` — change form

---

### @SmartMenus — *(a GeneXus module, not a PXTools one)*

**Purpose**: an advanced menu User Control (search, favorites, recents). It is a **GeneXus platform module/User Control**, **not a PXTools module**; it appears under `@PXTools/` only as glue.

> ℹ️ **Not a PXTools module.** Under `@PXTools/@SmartMenus/` there are only the **contract SDTs** (`PXToolsSmartMenu`, `PXToolsSmartMenuSelectedItem`) and an example DataProvider (`RetSmartMenuExample`) supporting the PXTools User Control that wraps the GeneXus SmartMenu. No transactions or patterns. The PXTools menu actually in use is in [modules/menus.md](modules/menus.md).

---

### @Statistics — Statistics

**Purpose**: defining and recording system statistics.

> 📄 **Detailed module documentation:** [modules/statistics.md](modules/statistics.md)

**PXTools instances included**:
- `PXWorkWithStatisticDefinition` — definitions CRUD
- `PXWorkWithStatisticLog` — statistics log CRUD

---

### @System — System

**Purpose**: core PXTools system functionality. It includes procedures for saving system modules and objects.

> 📄 **Detailed module documentation:** [modules/system.md](modules/system.md)

**PXTools instances included**:
- `PXParameterRequestSaveSystemModules` — save modules
- `PXParameterRequestSaveSystemObjects` — save objects

---

### @SystemParameters — System Parameters

**Purpose**: managing global system parameters (not per entity, unlike PXEntityParameters).

> 📄 **Detailed module documentation:** [modules/systemparameters.md](modules/systemparameters.md)

**PXTools instances included**:
- `PXWorkWithSystemParametersPreferences` — parameters CRUD

---

### @TableCleaner — Table Cleanup

**Purpose**: configuring and running scheduled table cleanup (purging old data).

> 📄 **Detailed module documentation:** [modules/tablecleaner.md](modules/tablecleaner.md)

**PXTools instances included**:
- `PXWorkWithTableCleanerConfiguration` — configuration CRUD

---

### @TaskManager — Task Manager

**Purpose**: a task management system with queues, execution and monitoring.

> 📄 **Detailed module documentation:** [modules/taskmanager.md](modules/taskmanager.md)

**PXTools instances included**:
- `PXWorkWithTaskManager` — tasks CRUD
- `PXWorkWithTaskManagerQueues` — queues CRUD
- `PXParameterRequestExecutionMessage` — execution message
- `PXParameterRequestUpdTaskManagerParameters` — update parameters

---

### @WSLayer — Web Services Layer

**Purpose**: infrastructure for the web services layer, including the access whitelist.

> 📄 **Detailed module documentation:** [modules/wslayer.md](modules/wslayer.md)

**PXTools instances included**:
- `PXWorkWithWSWhiteList` — whitelist CRUD

---

### @WebServicesLog — Web Service Log

**Purpose**: logging, statistics and monitoring of web service invocations.

> 📄 **Detailed module documentation:** [modules/webserviceslog.md](modules/webserviceslog.md)

**PXTools instances included**:
- `PXWorkWithWebServicesLog` — log CRUD
- `PXWorkWithWebServicesStatistics` — statistics CRUD
- `PXWorkWithWebServicesRAStatistics` — per-Remote-Address statistics
- `PXComposerWebServicesLog` — composed log view
- `PXComposerWebServicesStatisticCounters` — counters view
- `PXParameterRequestWebServiceLogView` — detailed log view

---

## Summary of PXTools instances across the modules

| Module | PXWorkWith | PXParameterRequest | PXComposer | PXReportTemplate | Total |
|--------|-----------|--------------------|------------|------------------|-------|
| @APIs | 1 | 4 | 0 | 0 | 5 |
| @Alerts | 4 | 3 | 2 | 0 | 9 |
| @CloudTasks | 6 | 3 | 0 | 0 | 9 |
| @DynamicCallReferences | 1 | 0 | 0 | 0 | 1 |
| @ExportImport | 0 | 1 | 0 | 1 | 2 |
| @FileStorage | 4 | 0 | 1 | 0 | 5 |
| @MailAccounts | 1 | 0 | 0 | 0 | 1 |
| @Menus | 1 | 0 | 0 | 0 | 1 |
| @OAV | 3 | 1 | 0 | 0 | 4 |
| @ProcessMonitor | 4 | 0 | 0 | 0 | 4 |
| @Projects | 3 | 0 | 0 | 0 | 3 |
| @ReceiveMails | 1 | 0 | 0 | 0 | 1 |
| @Security | 9 | 3 | 2 | 0 | 14 |
| @SendMails | 2 | 1 | 0 | 0 | 3 |
| @Statistics | 2 | 0 | 0 | 0 | 2 |
| @System | 0 | 2 | 0 | 0 | 2 |
| @SystemParameters | 1 | 0 | 0 | 0 | 1 |
| @TableCleaner | 1 | 0 | 0 | 0 | 1 |
| @TaskManager | 2 | 2 | 0 | 0 | 4 |
| @WSLayer | 1 | 0 | 0 | 0 | 1 |
| @WebServicesLog | 3 | 1 | 2 | 0 | 6 |

## Dependencies between modules

A graph derived from the KB (qualified `, PXTools.<Module>` references and shared domains) and cross-checked against the canonical matrix in the PXTools manual. **Layers:** `@APIs` is the **universal base** (everything depends on it, it depends on nothing); `@System` is the second layer; the feature modules sit on top of both.

| Module | Depends on (functionally) |
|---|---|
| **@APIs** | — (universal base) |
| **@System** | @APIs |
| **@SystemParameters** | @APIs |
| **@DynamicCallReferences** | @APIs |
| **@Menus** | @APIs (it uses the GeneXus @SmartMenus module for the SmartMenu style) |
| **@ControlPreferences** | @APIs |
| **@WSLayer** | @APIs |
| **@ExportImport** | @APIs |
| **@Statistics** | @APIs, @DynamicCallReferences · triggered by @TaskManager |
| **@TableCleaner** | @APIs, @DynamicCallReferences, @SystemParameters · triggered by @TaskManager |
| **@WebServicesLog** | @APIs, @TableCleaner, @SystemParameters |
| **@OAV** | @APIs, @DynamicCallReferences (+@System) |
| **@MailAccounts** | @APIs, @System |
| **@FileStorage** | @APIs, @System |
| **@Projects** | @APIs, @Security |
| **@Security** | @APIs, @System, @SystemParameters, @ControlPreferences, @SendMails |
| **@SendMails** | @APIs, @System, @FileStorage, @MailAccounts (optionally @TaskManager) |
| **@ReceiveMails** | @APIs, @System, @MailAccounts, @FileStorage |
| **@ProcessMonitor** | @APIs, @System, @SystemParameters, @SendMails, @FileStorage, @TaskManager |
| **@TaskManager** | @APIs, @System, @DynamicCallReferences, @ProcessMonitor, @SystemParameters |
| **@Alerts** | @APIs, @SendMails, @SystemParameters (+@FileStorage, @MailAccounts, optionally @TaskManager) |
| **@CloudTasks** | @APIs, @ProcessMonitor, @FileStorage, @Alerts, @SystemParameters · triggered by @TaskManager |

> **Note:** nearly every module also references `@Menus` and `@DynamicCallReferences` through their `Personalized/RetMenus*` and `RetDynamicCallReference*` DataProviders (generation scaffolding), not through their functional logic. Those are omitted above except where the dependency is functional (e.g. @DynamicCallReferences in @TaskManager/@TableCleaner/@Statistics/@OAV).
>
> **`@ResponsiveLayout` and `@SmartMenus` are NOT PXTools modules** — they are **GeneXus** platform modules; under `@PXTools/` they only contribute supporting SDTs/APIs to the PXTools User Controls. They take no part in the PXTools dependency graph as modules.

This graph is the basis for **domain attribution**: a domain used by more than one module belongs to the common module they all depend on, or to `@APIs` (see the Domains section of each doc and the naming rule).

## When to bring each module in

| Need | Recommended module |
|------|--------------------|
| Authentication and authorization | @Security |
| Sending email | @SendMails + @MailAccounts |
| Receiving email | @ReceiveMails + @MailAccounts |
| Scheduled tasks (cron) | @CloudTasks |
| Task queue | @TaskManager |
| File storage | @FileStorage |
| Alerts/notifications | @Alerts |
| Navigation menus | @Menus (with the GeneXus SmartMenus User Control) |
| Configurable parameters | @SystemParameters |
| Dynamic attributes | @OAV |
| Change auditing | the PXAudit pattern (not a module — see @Audit) |
| REST/SOAP APIs | @WSLayer + @WebServicesLog |
| Export/import | @ExportImport |
| Process monitoring | @ProcessMonitor |
| Statistics | @Statistics |
| Data cleanup | @TableCleaner |
| Project management | @Projects |
