# Módulo @SendMails — Envío de Correos

> Comportamiento del módulo `@PXTools/@SendMails`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@SendMails` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.SendMails`.
- **Depende de:** `@APIs` (base), `@System`, `@MailAccounts` (cuenta SMTP), `@FileStorage` (adjuntos). Infra de generación: `@Menus`, `@SystemParameters`, `@DynamicCallReferences`, `@ControlPreferences`.

## 1. Qué provee

Envío de correos con **outbox persistente**, **reintentos**, envío por **lotes/series**, adjuntos (incl. desde @FileStorage) y resolución de la **cuenta SMTP** (@MailAccounts o parámetros del sistema). Soporta envío **diferido** (encola y una tarea batch envía/reintenta) o **inmediato**.

## 2. Concepto central: outbox + series

- **`SendMailOutbox`** = un mail encolado (cabecera: from, subject, message, destinatarios To/CC/BCC, adjuntos).
- **`SendMailOutboxSeries`** = los **intentos/lotes** de ese mail — **aquí vive la máquina de estados y los reintentos**. Un outbox se parte en varias series si supera el tope de destinatarios.
- El **`MailData`** (SDT) es la **API de intercambio**: se arma en el código y se pasa a los procs de envío.

## 3. Transacciones del módulo

`SendMailOutboxId` = `IdFirstLevel` (autonum).

| Transacción | PK | Rol |
|---|---|---|
| **SendMailOutbox** | `SendMailOutboxId` | **Cabecera**: `Status` (`SendMailOutboxStatus`), subject, `MessageType` (HTML/Text), message, remitente (email/description/user/password `PswEnc`, o FK `MailAccountId`), FKs FileStorage. Niveles **Attachments** (`AttachmentType`) y **Mails** (email + `MailType` To/CC/BCC). |
| **SendMailOutboxSeries** | `SendMailOutboxId, SeriesDateTime` | **Intento/lote** (hora programada). `Status` (`SendMailOutboxSeriesStatus`), `ErrorCode`/`Description`/`ErrorRetries`. Nivel **Mails** (destinatario efectivo). Regla: al editar una serie `Fail` → vuelve a `Suspended` y resetea reintentos; tras cada update recalcula el estado de la cabecera. |
| **SendMailOuboxBC** (BC) | `SendMailOutboxId` | Business Component sobre la misma tabla (uso programático). |

FKs por Group: `SendMailOutBoxFromMailAccountId : MailAccountId` (@MailAccounts), `…FileStorage` (@FileStorage).

## 4. Dominios del módulo

Propios (nombre `SendMail*`/`MailMessage*`/`AttachmentType`), todos **root-legacy** (viven en el `#Domains/` raíz):

| Dominio | Valores |
|---|---|
| **SendMailOutboxStatus** | Fail=`FAI`, Discarded=`DIS`, Active=`ACT` |
| **SendMailOutboxSeriesStatus** | Suspended=`SUS`, ErrorRetry=`TRY`, Fail=`FAI`, Discarded=`DIS` |
| **SendMailOutboxMailType** | To, CC, BCC |
| **MailMessageType** | HTML, Text |
| **MailMessage** | LongVarChar(512000) — cuerpo del mensaje (también lo consume @ReceiveMails) |
| **AttachmentType** | Internal=`INT` (Blob), External=`EXT` (path/URL), FileStorage=`FSG` |

**Usa de otros módulos:** `MailAccountId`/`MailAccountType` (@MailAccounts), `PswEnc`/`PXToolsEmail` (@APIs base). `MailMessageType` y `AttachmentType` los reutiliza además @Alerts.

## 5. Mecanismo

### 5.1 `MailData` (SDT — la API)
`From : MailSender` { eMail, Description, User, Password, **MailAccountId** }; `To`/`CC`/`BCC`/`ReplyTo` : colecciones `MailDestination`; `Subject`; `FileStorage`/`FileStorageGroup` (adjunta un grupo entero); `Message` { Type, Data }; `Attachments[]` { Type, Path, Name, Extension, FileStorage/Group/Storage }. Respuesta: `SDTSendMailResponse` { Error.Code, Error.Description, Id }.

### 5.2 Encolar
`AddSendMailOutbox(&MailData) → &SendMailOutboxId` inserta la cabecera (Active), cifra la password y crea Mails/Attachments. Dos estrategias encima:
- **`SendMailOutboxWithErrorRetry(in: &MailData)`** — **diferido**: crea una o varias `Series` en `Suspended`, programadas a `ServerNow + SendMailsTimePeriod`; parte en lotes si supera `SendMailsRecipientsTop`. El envío lo hace la tarea batch.
- **`SendMailOutboxWithoutErrorRetry(in: &MailData, out: &Response)`** — **inmediato**: envía con `SendMail`; crea la serie ya en `Discarded` (ok) o `Fail`.

### 5.3 Máquina de estados
- **Serie**: `Suspended` → éxito `Discarded` / error `ErrorRetry`; reintenta hasta `SMTPErrorRetries`, agotado → `Fail`. `Fail` puede reactivarse (`UpdRetrySeries`).
- **Cabecera** (`UpdSendMailOutboxStatus`): alguna serie pendiente → `Active`; todas `Discarded` → `Discarded`; alguna `Fail` → `Fail`.

### 5.4 Envío batch — vía @TaskManager
- **`TskSendMails(in: &TaskManagerId, out…)`** es el worker: recorre cuentas (Main → Alias → sin cuenta), hace login SMTP una vez por cuenta (`SendMailSessionLogin`) y envía las series vencidas reutilizando la sesión (`SendMailSessionSend`); respeta el límite por cuenta.
- **`PrcSendMail`** (`IsMain=True, CommandLine`) es el entry-point del scheduler: lock (`StartProcessStatus`) + `TskSendMails`.
- `Personalized/RetDynamicCallReferenceSendMails` registra `TskSendMails` en @TaskManager + `PrcTableCleanerSendMails`.

### 5.5 Resolución de cuenta
Si `From.MailAccountId` está seteado → usa esa `MailAccount` (si es **Alias**, resuelve la Main y agrega el alias como `ReplyTo`); toma host/puerto/secure/auth de la cuenta (salvo `SMTPDefault` → System Parameters SMTP*). Sin `MailAccountId` → usa `From.eMail`/User/Password, o los System Parameters SMTP si vacíos.

## 6. APIs vs Personalized

- **`APIs/`** (core): transacciones, `MailData` y SDTs, los procs de encolado/envío/estado, el worker `TskSendMails`/`PrcSendMail`.
- **`Personalized/`** (3 DataProviders de integración):
  | Objeto | Qué se customiza |
  |---|---|
  | `RetMenusSendMails` | Ítems de menú (Outbox / Series). |
  | `RetSystemParametersSendMails` | System-parameters del módulo (`SendMailsRecipientsTop`, `SendMailsTimePeriod`, `SendMailsFailWhithoutMailAccount`, `SendMailsAccountLimit`). |
  | `RetDynamicCallReferenceSendMails` | Registro en @TaskManager (`TskSendMails`) + Table Cleaner. |

> Los parámetros SMTP* (host/port/user/pass/secure/errorretries) se consumen como System Parameters pero se declaran en otro módulo del framework.

## 7. Instancias de patterns

| Instancia | Qué es |
|---|---|
| **PXWorkWithSendMailOutbox** | WW del outbox; acciones Retry/"Retry Failed Series", Change (cambio masivo). |
| **PXWorkWithSendMailOutboxSeries** | WW de las series/intentos. |
| **PXParameterRequestChange** | Popup de cambio masivo por filtro (MailAccount/subject/fechas/status/email) → `UpdSendMailsData`. |

## 8. Procedimientos / APIs clave

| Proc | `Parm()` | Propósito |
|---|---|---|
| **`SendMailOutboxWithErrorRetry`** | `in: &MailData` | **API recomendada** (diferido con reintentos). |
| **`SendMailOutboxWithoutErrorRetry`** | `in: &MailData, out: &SDTSendMailResponse` | Envío inmediato + outbox. |
| `SendMail` | `in: &MailData, out: &ErrorCode, out: &ErrorDescription` | Envío SMTP crudo (no persiste). |
| `AddSendMailOutbox` | `in: &MailData, out: &SendMailOutboxId` | Solo encola (sin serie ni envío). |
| `SendMailSessionLogin` / `SendMailSessionSend` | (sesión SMTP + MailData) | Login reutilizable / enviar sobre sesión abierta (batch). |
| `TskSendMails` | `in: &TaskManagerId, out…` | Worker batch (lo llama @TaskManager). |
| `RetrySendMailOutboxSerie` / `UpdRetrySeries` | (PK serie / `&SendMailOutboxId`) | Reintentar una serie / reactivar `Fail`→`ErrorRetry`. |
| `UpdSendMailOutboxStatus` | `in: &SendMailOutboxId` | Recalcula estado de cabecera. |
| `UpdSendMailsData` | (filtro + flags) | Cambio masivo de estado/contadores. |
| `RetMailDataFromSendMailOutboxSerie` | (PK serie, out: &MailData) | Reconstruye `MailData` desde el outbox. |

**Patrón de uso**: armar `&MailData` (From con `MailAccountId` o email; To/CC/BCC; Subject; Message; Attachments) → `SendMailOutboxWithErrorRetry.Call(&MailData)` (diferido) o `SendMailOutboxWithoutErrorRetry.Call(&MailData, &Response)` (inmediato).

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [taskmanager.md](taskmanager.md) — ejecuta `TskSendMails` (cola `SendMails`).
- Módulos **@MailAccounts** (cuenta/credenciales SMTP), **@FileStorage** (adjuntos), **@SystemParameters** (SMTP* + parámetros del módulo).
- [alerts.md](alerts.md) — el canal Mail de las alertas envía con `SendMailOutboxWithErrorRetry`.
