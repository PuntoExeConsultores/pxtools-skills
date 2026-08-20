# Módulo @ReceiveMails — Recepción de Correos

> Comportamiento del módulo `@PXTools/@ReceiveMails`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@ReceiveMails` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.ReceiveMails`.
- **Depende de:** `@APIs` (base), `@System`, `@MailAccounts` (cuenta POP3), `@FileStorage` (adjuntos), `@SendMails` (dominio `MailMessage`). Infra: `@Menus`, `@SystemParameters`, `@DynamicCallReferences`.

## 1. Qué provee

**Recepción de correos entrantes por POP3**: baja los mails de las cuentas configuradas (@MailAccounts), guarda cada uno en `ReceivedMails` (con adjuntos en @FileStorage) y los deja disponibles para que la aplicación los procese. El modelo es **pull + marca de estado**: el módulo NO ejecuta lógica de negocio; la app consulta los no procesados, actúa, y los marca procesados.

## 2. Concepto central

- **`ReceivedMails`** = bandeja de entrada: cabecera (remitente, subject, texto/HTML, adjuntos) + nivel **TO** (un registro por destinatario).
- **Estado**: `ReceivedMailProcesed` (cabecera) y `ReceivedMailToProcesed` (por destinatario, para el modo Alias donde varias cuentas comparten buzón).
- La app hace `GetMails` (no procesados) → procesa cada `MailItem` → `Upd…Processed`.

## 3. Transacción `ReceivedMails`

Dos niveles; `ReceivedMailId` = `IdFirstLevel` (autonum).

- **Cabecera**: `ReceivedMailMailAccountId` (FK cuenta origen), `DateSent`/`DateReceived`, `Name`/`Address` (remitente), `Subject`, `Text` (`MailMessage`) y `HTMLText` (con imágenes embebidas), `ReceivedMailFileStorageId` (adjuntos, nullable), **`ReceivedMailProcesed`** (Boolean).
- **Nivel `TO`**: `ReceivedMailTOId` (PK sub), `ReceivedMailToAccountId` (FK cuenta destino si se reconoce, nullable), `ToAddress`/`ToName`, **`ReceivedMailToProcesed`** (Boolean).
- **`ReceivedMailsBC`**: Business Component sobre la misma estructura.

FKs por Group: `ReceivedMailMailAccountId` / `ReceivedMailToAccountId : MailAccountId` (@MailAccounts); `ReceivedMailFileStorageId` → @FileStorage.

## 4. Dominios del módulo

El módulo **no define dominios propios** (no existe ningún dominio `ReceiveMail*`). Reutiliza de otros módulos:

| Dominio | Dueño | Uso |
|---|---|---|
| `MailAccountType` | @MailAccounts | Main/Alias — decide el flujo de lectura |
| `MailMessage` | @SendMails | Cuerpo del mail (`Text`) |
| `FileStorageCategory` | @FileStorage | Categoría del adjunto |
| `SystemParameterCode` | @SystemParameters | Parámetros POP3 |
| `DynamicCallReferenceCode` | @DynamicCallReferences | Registro de la task de recepción |
| `TaskManagerExecutionResponse` | @TaskManager | Contrato de retorno de la task |
| `PXToolsEmail`, `PswEnc`, primitivos | @APIs | Tipos base |

## 5. Mecanismo

### 5.1 Bajada POP3 — `LoadMails`
`Parm(in: &MailAccountId, out: &Error, out: &ErrorDescription)` — el corazón, usa el External Object `POP3Session`:
- Resuelve host/puerto/secure/attachdir/timeout: `POP3Default` → System Parameters; si no, atributos POP3 de la cuenta. Credenciales de la cuenta (password `PPEXE_DePsw02`).
- `CheckandEmptyDirectoryAttachs` → `POP3Session.Login()` → bucle `Receive(&MailMessage)` hasta `Count` o el límite por cuenta.
- Por cada mail (ignora los propios): `New` en `ReceivedMails` (remitente, fechas, subject, Text/HTMLText), adjuntos → `AddReceivedMailAttach` (crea entrada FileStorage), un registro `TO` por destinatario To/CC/BCC; imágenes inline con `RetReceivedMailWithImagesEmbedded`.
- Éxito: `Commit` + `POP3Session.Delete()` (borra del servidor). Fallo: según `ReceiveMailsDeleteLoadMailsWithFails`, borra o `Skip()`. Actualiza `MailAccountPOP3LastExecution`.

### 5.2 Orquestación — vía @TaskManager
Entry points (`Parm(in: &TaskManagerId, out…)`) que iteran cuentas `Main` verificadas y llaman `LoadMails`:
- `TskReceiveMails` (todas las Main), `TskReceiveMailsMainWithAlias`, `TskReceiveMailsMainWithoutAlias`.
- `PrcReceiveMail` (`IsMain=True, CommandLine`): lock (`StartProcessStatus`) + `TskReceiveMails`.
- Registrados en `Personalized/RetDynamicCallReferenceReceiveMails`.

### 5.3 Entrega a la aplicación (pull + marca)
- `GetMails` / `LoadGetMails` devuelven `SDTReceiveMails` (colección `MailItem` con remitente, fechas, subject, Text, Html, FileStorageId y subcolección `To`) de los **no procesados**. `GetMainMails` filtra por `not …Procesed`; `GetAliasMails` por destinatario.
- La app procesa cada `MailItem` a su criterio y marca con `UpdReceivedMailProcessed` / `UpdReceiveMailAliasProcessed`.

> **El "qué hacer con cada mail" NO vive en este módulo**: lo implementa el consumidor externo (fuera de @ReceiveMails). El módulo permanece agnóstico del negocio.

## 6. APIs vs Personalized

- **`APIs/`** (core): la transacción, `LoadMails`, los `Tsk*`/`Prc*`, los getters `Get*`, los `Upd*Processed`, `DelReceivedMails`, los SDTs.
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | `RetSystemParametersReceiveMails` (DataProvider) | System-parameters (`ReceiveMailsAccountLimit`, `ReceiveMailsDeleteLoadMailsWithFails`). |
  | `RetDynamicCallReferenceReceiveMails` (DataProvider) | Registra los `TskReceiveMails*` + Table Cleaner en @TaskManager. |
  | `RetMenusReceivedMails` (DataProvider) | Entrada de menú. |
  | `PrcTableCleanerReceiveMails` (Procedure) | Purga correos viejos (`DelReceivedMails`). |

## 7. Instancia de pattern

**PXWorkWithReceivedMails** — WW de la bandeja de entrada. Selection con filtros por cuenta (suggest), remitente, rango de fechas, estado Processed y Attachment; vistas y detalle (`TrReceivedMails`, `VeReceivedMails`, `CtReceivedMails`, con variantes `…Alias`).

## 8. Procedimientos / APIs clave

**Getters (entrega)**: `GetMails(in: &MailAccountId, out: &SDTReceiveMail)` (enruta Main/Alias), `GetMainMails`, `GetAliasMails`, `LoadGetMails` (baja POP3 + devuelve en una llamada).

**Bajada/carga**: `LoadMails`, `AddReceivedMailAttach(in: &Sender, in: &SDTReceivedMailAttach, out: &FileStorageId)`, `RetReceivedMailWithImagesEmbedded`, `RetReceivedMailToId`.

**Marca de estado** (la app tras procesar): `UpdReceivedMailProcessed(in: &ReceivedMailId)` (cabecera + todos los TO), `UpdReceiveMailAliasProcessed(in: &ReceivedMailId, in: &ToAccountId)` (un destinatario; si no quedan pendientes, marca la cabecera), `UpdReceivedMailToggleProcessed` (UI).

**Borrado**: `DelReceivedMails(in: &ReceivedMailId, …, out: &SDTCounter)` (usado por el TableCleaner).

**SDTs**: `SDTReceiveMails` (`MailItem` + `To`) — contrato de entrega; `SDTReceivedMailAttach` — adjuntos hacia FileStorage.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- Módulo **@MailAccounts** — cuenta POP3 (`ReceivedMailMailAccountId`).
- Módulo **@FileStorage** — adjuntos (`ReceivedMailFileStorageId`).
- [taskmanager.md](taskmanager.md) — ejecuta los `TskReceiveMails*` (colas ReceiveMails).
