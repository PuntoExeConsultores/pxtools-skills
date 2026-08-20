# @MailAccounts Module — Mail Accounts

> Behaviour of the `@PXTools/@MailAccounts` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@MailAccounts` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.MailAccounts`.
- **Depends on:** `@APIs` (base), `@System`. Generation infrastructure: `@SystemParameters`, `@Menus`, `@DynamicCallReferences`; real couplings: `@SendMails` (the FullTest action) and `@ProcessMonitor` (`UpdMailAccountNotVerified`).

## 1. What it provides

The **configuration of mail accounts** (SMTP for sending, POP3 for receiving) consumed by @SendMails and @ReceiveMails. It models **Main** and **Alias** accounts (an Alias points at its Main), with encrypted credentials, error counters and auto-disabling.

## 2. Core concept: Main / Alias and "default = global"

- A **Main** account has its own SMTP/POP3 configuration. An **Alias** reuses the Main's (and adds its own address as `ReplyTo`); it is only allowed if its domain (`@…`) matches the Main's.
- There is no single "default account". The `SMTPDefault`/`POP3Default` flags mean **"inherit the global configuration"** (System Parameters SMTP*/POP3*); when False, the row's own host/port/secure/… are used.

## 3. The `MailAccounts` transaction

A single level; `MailAccountId` = `IdFirstLevel` (autonumbered).

- **PK** `MailAccountId`; candidate key `MailAccountName`.
- **Self-FK**: `MailAccountMainAccountId → MailAccountId` (Alias → Main; it infers the Main's name/verified/disabled).
- **Identity**: `Name`, `Type` (`MailAccountType`), `Disabled`, `Address` (`PXToolsEmail`, stored lowercase), `User`, `Password` (`PswEnc`), `Suggest` (a formula: "Name (Address)").
- **POP3**: `POP3Default`, `POP3Host/Port/Secure/AttachsDirectory`, `POP3AlternativeUser`/`POP3User`/`POP3Password`, `POP3Verified`, `POP3Disabled`, `POP3Counter`, `POP3LastExecution`, `POP3Message`.
- **SMTP**: `SMTPDefault`, `SMTPHost/Port/Authentication/Timeout/Secure/Log`, `SMTPVerified`, `SMTPAlternativeUser`/`SMTPUser`/`SMTPPassword`, `SMTPDisabled`, `SMTPCounter`, `SMTPMessage`, `SMTPLastExecution`.
- **Limits**: `ReceiveMailsLimit`, `SendMailsLimit`.

**Encryption**: on save the password is encrypted with `PPEXE_DePsw01` (from the `PswIng` variable into the `PswEnc` attribute); on read it is decrypted with `PPEXE_DePsw02`. Rules: `Default(Type, Main)`; SMTP/POP3 are visible/editable only for `Type=Main`; VerifyPOP3/VerifySMTP/FullTest buttons.

## 4. Module domains

**Its own** (root-legacy, in the root `#Domains/`): **`MailAccountType`** — Main=`MAI`, Alias=`ALI`. It is the only `MailAccount*` domain.

**Used from other modules:** `PXToolsConnectionType` (SMTP/POP3), `PXToolsEmail`, `PswEnc` (@APIs base); `SystemParameterCode.*` (@SystemParameters).

> ⚠️ **Do not confuse:** `StatisticMailAccount` (BothEnabled=`0`, POP3Disabled=`1`, SMTPDisabled=`2`, BothDisabled=`3`), despite describing an account's state, is a **[@Statistics](statistics.md)** domain (`Statistic*` naming) and is used only by @Statistics.

## 5. How it works

- **Global configuration** (System Parameters, SMTP/POP3/MailAccounts categories): generic host/port/user/pass/secure/timeout, read with `RetSystemParameterPreference…(SystemParameterCode.SMTPHost | …)`. The codes are declared by the Personalized DataProviders (§6).
- **Account resolution in @SendMails** (`SendMail`/`SendMailSessionLogin`): with neither `MailAccountId` nor email → the global configuration; with an explicit email → ad-hoc; with a `MailAccountId` → it reads the account (Alias → resolves the Main + ReplyTo), and if `not SMTPDefault` it overrides host/port/etc. On send it updates the counters (`UpdMailAccountSMTPCounter` / `…Reset…`; `ChkMailAccountErrorToRetry` decides retry vs disable).
- **Consumption in @ReceiveMails** (`LoadMails`): the same, over POP3 (`POP3Default` → global; `RetMailAccountByAddress` resolves recipients to an account; it honours `ReceiveMailsLimit`; it updates `POP3LastExecution`/`POP3Counter`).
- **Connection test**: `ChkConnectionSMTP(...)` (`SMTPSession.Login()`) and `ChkConnectionPOP3(...)` (`POP3Session.Login()`), starting from the global configuration and overriding it with the account's.

## 6. APIs vs Personalized

- **`APIs/`** (core): the transaction, the `Ret*/Chk*/Upd*/Add*` procedures and the batch task.
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | `RetSystemParametersSMTP` / `…POP3` / `…MailAccounts` (DataProviders) | Declare the `SystemParameterCode` values of the global configuration (host/user/pass/port/secure/timeout/retries/attachDir…). |
  | `RetMenuMailAccounts` (DataProvider) | The module's menu entry. |
  | `RetDynamicCallReferenceMailAccounts` (DataProvider) | Registers `TskMailAccountAutomaticEnabled` in @TaskManager. |
  | `PrcMailAccountDisabledNotificationByType` (Procedure) | **Hook**: when an account becomes disabled, it resolves the owner and sends them an alert. This is where the project connects "account disabled" to its own notification logic. |

## 7. Pattern instance

**PXWorkWithMailAccounts** — the accounts WW. Selection with Verified/Disabled and "Main with Alias" filters; VerifyAccounts / EnablePOP3SMTP / EnableAccount actions; View with a grid of the open Main's Alias accounts; a form with Test POP3/SMTP/Full Test and Combo/Suggest selection of the Main.

## 8. Key procedures / APIs

**Reading/resolution**: `RetMailAccountData(...)` (full load, passwords decrypted), `RetMailAccountByAddress(in: &Address, out: &MailAccountId)`, `RetMailAccountName`, `RetMailAccountIdType`, `RetMailAccountSuggest`, `RetMailAccountMains(out: &Ids)`, `RetMailAccountSendMailsLimit`/`…ReceiveMailsLimit` (with a global fallback), `RetMailAccountMessagesToRetry`, `RetValidationMainAccountFQDN` (validates the Alias↔Main domain).

**Checks**: `ChkConnectionSMTP`/`ChkConnectionPOP3` (test), `ChkMailAccountErrorToRetry`, `ChkMailAccountMainWithAlias`.

**Creation/state**: `AddMailAccountAlias(in: &MainAccountId, &Name, &Address, inout: &MailAccountId)`, `UpdMailAccountDisabled`, `UpdMailAccountEnableAccount`, `UpdMailAccountEnablePOP3SMTP`, `UpdMailAccountSMTPCounter`/`…Reset…`, `UpdMailAccountPOP3Counter`/`…LastExecution`.

**Batch**: `TskMailAccountAutomaticEnabled(in: &TaskManagerId, out…)` — re-enables disabled SMTP accounts whose error was a temporary provider limit (throttling).

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [sendmails.md](sendmails.md) — consumes the SMTP account (`SendMailOutBoxFromMailAccountId`).
- The **@ReceiveMails** module — consumes the POP3 account.
- The **@SystemParameters** (global SMTP/POP3 configuration), **@Alerts** (disabled-account notification) and **@TaskManager** (`TskMailAccountAutomaticEnabled`) modules.
