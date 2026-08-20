# Módulo @MailAccounts — Cuentas de Correo

> Comportamiento del módulo `@PXTools/@MailAccounts`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@MailAccounts` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.MailAccounts`.
- **Depende de:** `@APIs` (base), `@System`. Infra de generación: `@SystemParameters`, `@Menus`, `@DynamicCallReferences`; acoplamientos reales: `@SendMails` (acción FullTest) y `@ProcessMonitor` (`UpdMailAccountNotVerified`).

## 1. Qué provee

La **configuración de las cuentas de correo** (SMTP para envío, POP3 para recepción) que consumen @SendMails y @ReceiveMails. Modela cuentas **Main** y **Alias** (una Alias apunta a su Main), con credenciales cifradas, contadores de error y auto-deshabilitado.

## 2. Concepto central: Main / Alias y "default = global"

- Una cuenta **Main** tiene su propia config SMTP/POP3. Una **Alias** reutiliza la Main (y agrega su dirección como `ReplyTo`); solo se permite si su dominio (`@…`) coincide con el de la Main.
- No hay "cuenta por defecto" única. Los flags `SMTPDefault`/`POP3Default` significan **"heredar la config global"** (System Parameters SMTP*/POP3*); en False, se usan los host/port/secure/… propios de la fila.

## 3. Transacción `MailAccounts`

Un solo nivel; `MailAccountId` = `IdFirstLevel` (autonum).

- **PK** `MailAccountId`; candidate key `MailAccountName`.
- **Auto-FK**: `MailAccountMainAccountId → MailAccountId` (Alias → Main; trae inferidos nombre/verified/disabled de la Main).
- **Identidad**: `Name`, `Type` (`MailAccountType`), `Disabled`, `Address` (`PXToolsEmail`, guardado en minúsculas), `User`, `Password` (`PswEnc`), `Suggest` (fórmula "Name (Address)").
- **POP3**: `POP3Default`, `POP3Host/Port/Secure/AttachsDirectory`, `POP3AlternativeUser`/`POP3User`/`POP3Password`, `POP3Verified`, `POP3Disabled`, `POP3Counter`, `POP3LastExecution`, `POP3Message`.
- **SMTP**: `SMTPDefault`, `SMTPHost/Port/Authentication/Timeout/Secure/Log`, `SMTPVerified`, `SMTPAlternativeUser`/`SMTPUser`/`SMTPPassword`, `SMTPDisabled`, `SMTPCounter`, `SMTPMessage`, `SMTPLastExecution`.
- **Límites**: `ReceiveMailsLimit`, `SendMailsLimit`.

**Cifrado**: al guardar, la password se cifra con `PPEXE_DePsw01` (de la var `PswIng` al atributo `PswEnc`); al leer se descifra con `PPEXE_DePsw02`. Reglas: `Default(Type, Main)`; SMTP/POP3 solo visibles/editables para `Type=Main`; botones VerifyPOP3/VerifySMTP/FullTest.

## 4. Dominios del módulo

**Propio** (root-legacy, en el `#Domains/` raíz): **`MailAccountType`** — Main=`MAI`, Alias=`ALI`. Es el único dominio `MailAccount*`.

**Usa de otros módulos:** `PXToolsConnectionType` (SMTP/POP3), `PXToolsEmail`, `PswEnc` (@APIs base); `SystemParameterCode.*` (@SystemParameters).

> ⚠️ **No confundir:** `StatisticMailAccount` (BothEnabled=`0`, POP3Disabled=`1`, SMTPDisabled=`2`, BothDisabled=`3`), pese a describir el estado de una cuenta, es un dominio de **[@Statistics](statistics.md)** (nomenclatura `Statistic*`) y solo lo usa @Statistics.

## 5. Mecanismo

- **Config global** (System Parameters, categorías SMTP/POP3/MailAccounts): host/port/user/pass/secure/timeout genéricos, leídos con `RetSystemParameterPreference…(SystemParameterCode.SMTPHost | …)`. Los códigos los declaran los DataProviders Personalized (§6).
- **Resolución de cuenta en @SendMails** (`SendMail`/`SendMailSessionLogin`): sin `MailAccountId` ni email → config global; con email explícito → ad-hoc; con `MailAccountId` → lee la cuenta (Alias → resuelve Main + ReplyTo), y si `not SMTPDefault` sobreescribe host/port/etc. Al enviar actualiza contadores (`UpdMailAccountSMTPCounter` / `…Reset…`; `ChkMailAccountErrorToRetry` decide si reintentar vs deshabilitar).
- **Consumo en @ReceiveMails** (`LoadMails`): análogo con POP3 (`POP3Default` → global; `RetMailAccountByAddress` resuelve destinatarios a cuenta; respeta `ReceiveMailsLimit`; actualiza `POP3LastExecution`/`POP3Counter`).
- **Test de conexión**: `ChkConnectionSMTP(...)` (`SMTPSession.Login()`) y `ChkConnectionPOP3(...)` (`POP3Session.Login()`), partiendo de la global y sobreescribiendo con la cuenta.

## 6. APIs vs Personalized

- **`APIs/`** (core): la transacción, los `Ret*/Chk*/Upd*/Add*` y la tarea batch.
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | `RetSystemParametersSMTP` / `…POP3` / `…MailAccounts` (DataProviders) | Declaran los `SystemParameterCode` de la config global (host/user/pass/port/secure/timeout/retries/attachDir…). |
  | `RetMenuMailAccounts` (DataProvider) | Entrada de menú del módulo. |
  | `RetDynamicCallReferenceMailAccounts` (DataProvider) | Registra `TskMailAccountAutomaticEnabled` en @TaskManager. |
  | `PrcMailAccountDisabledNotificationByType` (Procedure) | **Hook**: cuando una cuenta queda deshabilitada, resuelve el dueño y le envía una alerta. Punto donde el proyecto conecta "cuenta deshabilitada" con su lógica de notificación. |

## 7. Instancia de pattern

**PXWorkWithMailAccounts** — WW de cuentas. Selection con filtros por Verified/Disabled y "Main with Alias"; acciones VerifyAccounts / EnablePOP3SMTP / EnableAccount; View con grid de cuentas Alias de la Main abierta; form con Test POP3/SMTP/Full Test y selección Combo/Suggest de la Main.

## 8. Procedimientos / APIs clave

**Lectura/resolución**: `RetMailAccountData(...)` (carga completa, passwords descifradas), `RetMailAccountByAddress(in: &Address, out: &MailAccountId)`, `RetMailAccountName`, `RetMailAccountIdType`, `RetMailAccountSuggest`, `RetMailAccountMains(out: &Ids)`, `RetMailAccountSendMailsLimit`/`…ReceiveMailsLimit` (fallback global), `RetMailAccountMessagesToRetry`, `RetValidationMainAccountFQDN` (valida dominio Alias↔Main).

**Chequeos**: `ChkConnectionSMTP`/`ChkConnectionPOP3` (test), `ChkMailAccountErrorToRetry`, `ChkMailAccountMainWithAlias`.

**Alta/estado**: `AddMailAccountAlias(in: &MainAccountId, &Name, &Address, inout: &MailAccountId)`, `UpdMailAccountDisabled`, `UpdMailAccountEnableAccount`, `UpdMailAccountEnablePOP3SMTP`, `UpdMailAccountSMTPCounter`/`…Reset…`, `UpdMailAccountPOP3Counter`/`…LastExecution`.

**Batch**: `TskMailAccountAutomaticEnabled(in: &TaskManagerId, out…)` — re-habilita cuentas SMTP deshabilitadas cuyo error fue un límite temporal del proveedor (throttling).

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [sendmails.md](sendmails.md) — consume la cuenta SMTP (`SendMailOutBoxFromMailAccountId`).
- Módulo **@ReceiveMails** — consume la cuenta POP3.
- Módulos **@SystemParameters** (config global SMTP/POP3), **@Alerts** (notificación de cuenta deshabilitada), **@TaskManager** (`TskMailAccountAutomaticEnabled`).
