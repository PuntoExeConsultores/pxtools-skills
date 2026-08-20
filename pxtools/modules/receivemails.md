# @ReceiveMails Module — Incoming Mail

> Behaviour of the `@PXTools/@ReceiveMails` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@ReceiveMails` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.ReceiveMails`.
- **Depends on:** `@APIs` (base), `@System`, `@MailAccounts` (the POP3 account), `@FileStorage` (attachments), `@SendMails` (the `MailMessage` domain). Infrastructure: `@Menus`, `@SystemParameters`, `@DynamicCallReferences`.

## 1. What it provides

**Receiving incoming mail over POP3**: it downloads mail from the configured accounts (@MailAccounts), stores each message in `ReceivedMails` (with attachments in @FileStorage) and leaves them available for the application to process. The model is **pull + status flag**: the module runs no business logic; the app queries the unprocessed messages, acts on them, and marks them processed.

## 2. Core concept

- **`ReceivedMails`** = the inbox: a header (sender, subject, text/HTML, attachments) + a **TO** level (one record per recipient).
- **Status**: `ReceivedMailProcesed` (header) and `ReceivedMailToProcesed` (per recipient, for the Alias mode where several accounts share a mailbox).
- The app calls `GetMails` (unprocessed) → processes each `MailItem` → `Upd…Processed`.

## 3. The `ReceivedMails` transaction

Two levels; `ReceivedMailId` = `IdFirstLevel` (autonumbered).

- **Header**: `ReceivedMailMailAccountId` (FK to the source account), `DateSent`/`DateReceived`, `Name`/`Address` (sender), `Subject`, `Text` (`MailMessage`) and `HTMLText` (with embedded images), `ReceivedMailFileStorageId` (attachments, nullable), **`ReceivedMailProcesed`** (Boolean).
- **`TO` level**: `ReceivedMailTOId` (sub PK), `ReceivedMailToAccountId` (FK to the destination account when recognised, nullable), `ToAddress`/`ToName`, **`ReceivedMailToProcesed`** (Boolean).
- **`ReceivedMailsBC`**: a Business Component over the same structure.

FKs through Groups: `ReceivedMailMailAccountId` / `ReceivedMailToAccountId : MailAccountId` (@MailAccounts); `ReceivedMailFileStorageId` → @FileStorage.

## 4. Module domains

The module **defines no domains of its own** (there is no `ReceiveMail*` domain). It reuses these from other modules:

| Domain | Owner | Use |
|---|---|---|
| `MailAccountType` | @MailAccounts | Main/Alias — decides the reading flow |
| `MailMessage` | @SendMails | Mail body (`Text`) |
| `FileStorageCategory` | @FileStorage | Attachment category |
| `SystemParameterCode` | @SystemParameters | POP3 parameters |
| `DynamicCallReferenceCode` | @DynamicCallReferences | Registration of the receive task |
| `TaskManagerExecutionResponse` | @TaskManager | Return contract of the task |
| `PXToolsEmail`, `PswEnc`, primitives | @APIs | Base types |

## 5. How it works

### 5.1 POP3 download — `LoadMails`
`Parm(in: &MailAccountId, out: &Error, out: &ErrorDescription)` — the heart of the module, using the `POP3Session` External Object:
- It resolves host/port/secure/attachdir/timeout: `POP3Default` → System Parameters; otherwise the account's own POP3 attributes. Credentials come from the account (password via `PPEXE_DePsw02`).
- `CheckandEmptyDirectoryAttachs` → `POP3Session.Login()` → a `Receive(&MailMessage)` loop up to `Count` or the account's limit.
- For each message (its own are ignored): `New` in `ReceivedMails` (sender, dates, subject, Text/HTMLText), attachments → `AddReceivedMailAttach` (creates a FileStorage entry), one `TO` record per To/CC/BCC recipient; inline images through `RetReceivedMailWithImagesEmbedded`.
- On success: `Commit` + `POP3Session.Delete()` (removes it from the server). On failure: depending on `ReceiveMailsDeleteLoadMailsWithFails`, it either deletes or calls `Skip()`. It updates `MailAccountPOP3LastExecution`.

### 5.2 Orchestration — through @TaskManager
Entry points (`Parm(in: &TaskManagerId, out…)`) that iterate verified `Main` accounts and call `LoadMails`:
- `TskReceiveMails` (all Mains), `TskReceiveMailsMainWithAlias`, `TskReceiveMailsMainWithoutAlias`.
- `PrcReceiveMail` (`IsMain=True, CommandLine`): a lock (`StartProcessStatus`) + `TskReceiveMails`.
- Registered in `Personalized/RetDynamicCallReferenceReceiveMails`.

### 5.3 Delivery to the application (pull + flag)
- `GetMails` / `LoadGetMails` return `SDTReceiveMails` (a `MailItem` collection with sender, dates, subject, Text, Html, FileStorageId and a `To` sub-collection) of the **unprocessed** messages. `GetMainMails` filters by `not …Procesed`; `GetAliasMails` filters by recipient.
- The app processes each `MailItem` however it sees fit and flags it with `UpdReceivedMailProcessed` / `UpdReceiveMailAliasProcessed`.

> **"What to do with each mail" does NOT live in this module**: the external consumer implements it (outside @ReceiveMails). The module stays business-agnostic.

## 6. APIs vs Personalized

- **`APIs/`** (core): the transaction, `LoadMails`, the `Tsk*`/`Prc*` procedures, the `Get*` getters, the `Upd*Processed` procedures, `DelReceivedMails`, and the SDTs.
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | `RetSystemParametersReceiveMails` (DataProvider) | System parameters (`ReceiveMailsAccountLimit`, `ReceiveMailsDeleteLoadMailsWithFails`). |
  | `RetDynamicCallReferenceReceiveMails` (DataProvider) | Registers the `TskReceiveMails*` tasks + the Table Cleaner in @TaskManager. |
  | `RetMenusReceivedMails` (DataProvider) | Menu entry. |
  | `PrcTableCleanerReceiveMails` (Procedure) | Purges old mail (`DelReceivedMails`). |

## 7. Pattern instance

**PXWorkWithReceivedMails** — the inbox WW. Selection with filters by account (suggest), sender, date range, Processed status and Attachment; views and detail (`TrReceivedMails`, `VeReceivedMails`, `CtReceivedMails`, with `…Alias` variants).

## 8. Key procedures / APIs

**Getters (delivery)**: `GetMails(in: &MailAccountId, out: &SDTReceiveMail)` (routes Main/Alias), `GetMainMails`, `GetAliasMails`, `LoadGetMails` (POP3 download + return in one call).

**Download/loading**: `LoadMails`, `AddReceivedMailAttach(in: &Sender, in: &SDTReceivedMailAttach, out: &FileStorageId)`, `RetReceivedMailWithImagesEmbedded`, `RetReceivedMailToId`.

**Status flagging** (by the app once it has processed): `UpdReceivedMailProcessed(in: &ReceivedMailId)` (the header + every TO), `UpdReceiveMailAliasProcessed(in: &ReceivedMailId, in: &ToAccountId)` (one recipient; if none are left pending, it flags the header), `UpdReceivedMailToggleProcessed` (UI).

**Deletion**: `DelReceivedMails(in: &ReceivedMailId, …, out: &SDTCounter)` (used by the TableCleaner).

**SDTs**: `SDTReceiveMails` (`MailItem` + `To`) — the delivery contract; `SDTReceivedMailAttach` — attachments on their way to FileStorage.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- The **@MailAccounts** module — the POP3 account (`ReceivedMailMailAccountId`).
- The **@FileStorage** module — attachments (`ReceivedMailFileStorageId`).
- [taskmanager.md](taskmanager.md) — runs the `TskReceiveMails*` tasks (the ReceiveMails queues).
