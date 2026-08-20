# @SendMails Module — Outgoing Mail

> Behaviour of the `@PXTools/@SendMails` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@SendMails` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.SendMails`.
- **Depends on:** `@APIs` (base), `@System`, `@MailAccounts` (the SMTP account), `@FileStorage` (attachments). Generation infrastructure: `@Menus`, `@SystemParameters`, `@DynamicCallReferences`, `@ControlPreferences`.

## 1. What it provides

Sending mail with a **persistent outbox**, **retries**, **batch/series** delivery, attachments (including from @FileStorage) and resolution of the **SMTP account** (from @MailAccounts or the system parameters). It supports **deferred** sending (enqueue, and a batch task sends/retries) or **immediate** sending.

## 2. Core concept: outbox + series

- **`SendMailOutbox`** = one queued mail (header: from, subject, message, To/CC/BCC recipients, attachments).
- **`SendMailOutboxSeries`** = that mail's **attempts/batches** — **this is where the state machine and the retries live**. One outbox is split into several series if it exceeds the recipient cap.
- **`MailData`** (an SDT) is the **exchange API**: you build it in code and pass it to the sending procedures.

## 3. Module transactions

`SendMailOutboxId` = `IdFirstLevel` (autonumbered).

| Transaction | PK | Role |
|---|---|---|
| **SendMailOutbox** | `SendMailOutboxId` | **Header**: `Status` (`SendMailOutboxStatus`), subject, `MessageType` (HTML/Text), message, sender (email/description/user/password `PswEnc`, or the `MailAccountId` FK), FileStorage FKs. **Attachments** (`AttachmentType`) and **Mails** (email + `MailType` To/CC/BCC) levels. |
| **SendMailOutboxSeries** | `SendMailOutboxId, SeriesDateTime` | An **attempt/batch** (its scheduled time). `Status` (`SendMailOutboxSeriesStatus`), `ErrorCode`/`Description`/`ErrorRetries`. A **Mails** level (the effective recipient). Rule: editing a `Fail` series → it returns to `Suspended` and resets the retries; after each update the header's status is recomputed. |
| **SendMailOuboxBC** (BC) | `SendMailOutboxId` | Business Component over the same table (programmatic use). |

FKs through Groups: `SendMailOutBoxFromMailAccountId : MailAccountId` (@MailAccounts), `…FileStorage` (@FileStorage).

## 4. Module domains

Its own (named `SendMail*`/`MailMessage*`/`AttachmentType`), all **root-legacy** (they live in the root `#Domains/`):

| Domain | Values |
|---|---|
| **SendMailOutboxStatus** | Fail=`FAI`, Discarded=`DIS`, Active=`ACT` |
| **SendMailOutboxSeriesStatus** | Suspended=`SUS`, ErrorRetry=`TRY`, Fail=`FAI`, Discarded=`DIS` |
| **SendMailOutboxMailType** | To, CC, BCC |
| **MailMessageType** | HTML, Text |
| **MailMessage** | LongVarChar(512000) — the message body (also consumed by @ReceiveMails) |
| **AttachmentType** | Internal=`INT` (Blob), External=`EXT` (path/URL), FileStorage=`FSG` |

**Used from other modules:** `MailAccountId`/`MailAccountType` (@MailAccounts), `PswEnc`/`PXToolsEmail` (@APIs base). `MailMessageType` and `AttachmentType` are also reused by @Alerts.

## 5. How it works

### 5.1 `MailData` (the SDT — the API)
`From : MailSender` { eMail, Description, User, Password, **MailAccountId** }; `To`/`CC`/`BCC`/`ReplyTo` : `MailDestination` collections; `Subject`; `FileStorage`/`FileStorageGroup` (attaches a whole group); `Message` { Type, Data }; `Attachments[]` { Type, Path, Name, Extension, FileStorage/Group/Storage }. The response: `SDTSendMailResponse` { Error.Code, Error.Description, Id }.

### 5.2 Enqueueing
`AddSendMailOutbox(&MailData) → &SendMailOutboxId` inserts the header (Active), encrypts the password and creates the Mails/Attachments. Two strategies sit on top:
- **`SendMailOutboxWithErrorRetry(in: &MailData)`** — **deferred**: creates one or more `Series` in `Suspended`, scheduled at `ServerNow + SendMailsTimePeriod`; it splits into batches if `SendMailsRecipientsTop` is exceeded. The batch task does the sending.
- **`SendMailOutboxWithoutErrorRetry(in: &MailData, out: &Response)`** — **immediate**: sends through `SendMail`; creates the series already `Discarded` (success) or `Fail`.

### 5.3 State machine
- **Series**: `Suspended` → on success `Discarded` / on error `ErrorRetry`; it retries up to `SMTPErrorRetries`, and once exhausted → `Fail`. A `Fail` can be reactivated (`UpdRetrySeries`).
- **Header** (`UpdSendMailOutboxStatus`): any series still pending → `Active`; all `Discarded` → `Discarded`; any `Fail` → `Fail`.

### 5.4 Batch sending — through @TaskManager
- **`TskSendMails(in: &TaskManagerId, out…)`** is the worker: it walks the accounts (Main → Alias → no account), logs into SMTP once per account (`SendMailSessionLogin`) and sends the due series reusing the session (`SendMailSessionSend`); it honours the per-account limit.
- **`PrcSendMail`** (`IsMain=True, CommandLine`) is the scheduler's entry point: a lock (`StartProcessStatus`) + `TskSendMails`.
- `Personalized/RetDynamicCallReferenceSendMails` registers `TskSendMails` in @TaskManager plus `PrcTableCleanerSendMails`.

### 5.5 Account resolution
If `From.MailAccountId` is set → it uses that `MailAccount` (if it is an **Alias**, it resolves the Main and adds the alias as `ReplyTo`); it takes host/port/secure/auth from the account (unless `SMTPDefault` → the SMTP* System Parameters). Without a `MailAccountId` → it uses `From.eMail`/User/Password, or the SMTP System Parameters if those are empty.

## 6. APIs vs Personalized

- **`APIs/`** (core): the transactions, `MailData` and the SDTs, the enqueue/send/status procedures, the `TskSendMails`/`PrcSendMail` worker.
- **`Personalized/`** (3 integration DataProviders):
  | Object | What gets customized |
  |---|---|
  | `RetMenusSendMails` | Menu items (Outbox / Series). |
  | `RetSystemParametersSendMails` | The module's system parameters (`SendMailsRecipientsTop`, `SendMailsTimePeriod`, `SendMailsFailWhithoutMailAccount`, `SendMailsAccountLimit`). |
  | `RetDynamicCallReferenceSendMails` | Registration in @TaskManager (`TskSendMails`) + the Table Cleaner. |

> The SMTP* parameters (host/port/user/pass/secure/errorretries) are consumed as System Parameters but declared in another framework module.

## 7. Pattern instances

| Instance | What it is |
|---|---|
| **PXWorkWithSendMailOutbox** | The outbox WW; Retry / "Retry Failed Series" and Change (bulk change) actions. |
| **PXWorkWithSendMailOutboxSeries** | WW for the series/attempts. |
| **PXParameterRequestChange** | Bulk-change popup by filter (MailAccount/subject/dates/status/email) → `UpdSendMailsData`. |

## 8. Key procedures / APIs

| Proc | `Parm()` | Purpose |
|---|---|---|
| **`SendMailOutboxWithErrorRetry`** | `in: &MailData` | **The recommended API** (deferred, with retries). |
| **`SendMailOutboxWithoutErrorRetry`** | `in: &MailData, out: &SDTSendMailResponse` | Immediate send + outbox. |
| `SendMail` | `in: &MailData, out: &ErrorCode, out: &ErrorDescription` | Raw SMTP send (nothing persisted). |
| `AddSendMailOutbox` | `in: &MailData, out: &SendMailOutboxId` | Enqueues only (no series, no sending). |
| `SendMailSessionLogin` / `SendMailSessionSend` | (SMTP session + MailData) | Reusable login / send over an open session (batch). |
| `TskSendMails` | `in: &TaskManagerId, out…` | The batch worker (called by @TaskManager). |
| `RetrySendMailOutboxSerie` / `UpdRetrySeries` | (series PK / `&SendMailOutboxId`) | Retry one series / reactivate `Fail`→`ErrorRetry`. |
| `UpdSendMailOutboxStatus` | `in: &SendMailOutboxId` | Recomputes the header's status. |
| `UpdSendMailsData` | (filter + flags) | Bulk change of status/counters. |
| `RetMailDataFromSendMailOutboxSerie` | (series PK, out: &MailData) | Rebuilds `MailData` from the outbox. |

**Usage pattern**: build `&MailData` (From with either a `MailAccountId` or an email; To/CC/BCC; Subject; Message; Attachments) → `SendMailOutboxWithErrorRetry.Call(&MailData)` (deferred) or `SendMailOutboxWithoutErrorRetry.Call(&MailData, &Response)` (immediate).

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [taskmanager.md](taskmanager.md) — runs `TskSendMails` (the `SendMails` queue).
- The **@MailAccounts** (SMTP account/credentials), **@FileStorage** (attachments) and **@SystemParameters** (SMTP* + the module's parameters) modules.
- [alerts.md](alerts.md) — the alerts' Mail channel sends through `SendMailOutboxWithErrorRetry`.
