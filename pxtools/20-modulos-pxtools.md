# Módulos @PXTools — Funcionalidad Transversal Reutilizable

## Qué son

Los módulos @PXTools son **paquetes de objetos GeneXus reutilizables** que proveen funcionalidad transversal a cualquier aplicación. Se instalan bajo `Knowledge Base/@PXTools/@<NombreModulo>/` y muchos de ellos incluyen sus propias instancias de patterns PXTools (PXWorkWith para ABMs, PXParameterRequest para formularios, PXComposer para vistas compuestas).

## Ubicación en la KB

```
Knowledge Base/
└── @PXTools/
    ├── @APIs/          ← APIs utilitarias del framework
    ├── @Alerts/        ← Sistema de alertas
    ├── @Audit/         ← Auditoría
    ├── @CloudTasks/    ← Tareas en la nube
    ├── ...
    └── @WebServicesLog/ ← Log de WebServices
```

## Catálogo de módulos

> Esta página es el **índice** de módulos. Los que tienen documentación de comportamiento en detalle enlazan a su archivo en la subcarpeta [`modulos/`](modulos/) (p. ej. [modulos/security.md](modulos/security.md)).

### @APIs — APIs Utilitarias del Framework

**Propósito**: Módulo base que contiene las APIs internas del framework PXTools. Provee utilidades transversales usadas por todos los demás módulos y por los patterns.

> 📄 **Documentación detallada del módulo:** [modulos/apis.md](modulos/apis.md)

**Submódulos y contenido**:
- **#Domains**: Dominios base (CallType, GridHandlerColumnVisibility, GridHandlerConfigBoxAttachType, PasswordType)
- **APIs/Certificate**: Gestión de certificados
- **APIs/Classes**: 8 procedures para manipulación de clases CSS
- **APIs/Components**: Componentes UI reutilizables
- **APIs/Config**: SDTs y Procedures de configuración
- **APIs/Confirm**: WebPanels de confirmación (`PXParameterRequestConfirm`)
- **APIs/Context_APIs**: 3 procedures para manejo de contexto
- **APIs/Controller**: SDTs y Procedures para controladores
- **APIs/Date**: 8 procedures para operaciones de fecha
- **APIs/Download**: Utilidades de descarga
- **APIs/ErrorLog**: Log de errores
- **APIs/Escape**: 5 procedures para escaping de datos (`PXParameterRequestEncodeUrl`)
- **APIs/Excel**: SDTs y procedures para operaciones Excel
- **APIs/File**: Gestión de archivos
- **APIs/FormState**: Múltiples procedures para estado de formularios
- **APIs/Message**: Mensajería (`PXWorkWithMessages`)
- **APIs/StringTools**: Herramientas de texto (`PXParameterRequestNumberToDescription`)
- **Personalized/SecurityConnector**: Conector de seguridad (`PXParameterRequestLogin`)

**Instances PXTools incluidas**: PXParameterRequestConfirm, PXParameterRequestEncodeUrl, PXWorkWithMessages, PXParameterRequestNumberToDescription, PXParameterRequestLogin

---

### @Alerts — Sistema de Alertas y Notificaciones

**Propósito**: Sistema completo de alertas programables con soporte para múltiples canales de envío, idiomas, categorías y suscripciones/desuscripciones.

> 📄 **Documentación detallada del módulo:** [modulos/alerts.md](modulos/alerts.md)

**Contenido**:
- Transactions: SystemAlert, SystemAlertLanguages, SystemAlertCategories, SystemAlertUnsubscriptions
- Procedures: PrcSystemAlert (procesamiento de alertas)
- Scheduler para envío programado

**Instances PXTools incluidas**:
- `PXWorkWithSystemAlert` — ABM de alertas
- `PXWorkWithSystemAlertLanguages` — ABM de idiomas de alertas
- `PXWorkWithSystemAlertCategories` — ABM de categorías
- `PXWorkWithSystemAlertUnsubscriptions` — ABM de desuscripciones
- `PXComposerSystemAlertMessage` — Vista compuesta de mensajes
- `PXComposerSystemAlertSchedulerView` — Vista del scheduler
- `PXParameterRequestSystemAlertMessage` — Formulario de mensaje
- `PXParameterRequestUnsubscriptionConfirmation` — Confirmación de desuscripción
- `PXParameterRequestPPEXE_PrcSystemAlert` — Procesamiento de alerta

---

### @Audit — Auditoría

**Propósito**: Sistema de auditoría para registrar cambios en entidades (pattern PXAudit).

> ⚠️ **No presente en esta KB.** No existe la carpeta `@PXTools/@Audit/`. La auditoría, cuando se usa, se aplica con el pattern **PXAudit** directamente sobre las transacciones del proyecto (no como módulo instalado). Entrada conservada solo como referencia del catálogo PXTools.

---

### @CloudTasks — Tareas Programadas en la Nube

**Propósito**: Gestión y ejecución de tareas programadas. Soporta certificados, monitoreo de procesos, y limpieza de archivos temporales.

> 📄 **Documentación detallada del módulo:** [modulos/cloudtasks.md](modulos/cloudtasks.md)

**Contenido**:
- Transactions: CloudTasks, CloudTasksCertificates, KeyStores, CloudTaskDeleteFiles, CloudTasksProcessMonitor
- Utilidades de debug y simulación

**Instances PXTools incluidas**:
- `PXWorkWithCloudTasks` — ABM de tareas
- `PXWorkWithCloudTasksCertificates` — ABM de certificados
- `PXWorkWithKeyStores` — ABM de keystores
- `PXWorkWithCloudTaskDeleteFiles` — ABM de tareas de eliminación
- `PXWorkWithSimulateDeleteFiles` — Simulador
- `PXWorkWithCloudTasksProcessMonitor` — Monitor de procesos
- `PXParameterRequestCloudTaskStatusMessage` — Mensaje de estado
- `DebugCloudTaskOnDemand` — Debug bajo demanda
- `DebugCloudTasksUtils` — Utilidades de debug

---

### @ControlPreferences — Preferencias de Controles UI

**Propósito**: Almacenamiento y recuperación de preferencias de controles de usuario (anchos de columna, visibilidad, orden, etc.).

> 📄 **Documentación detallada del módulo:** [modulos/controlpreferences.md](modulos/controlpreferences.md)

---

### @DynamicCallReferences — Referencias de Llamadas Dinámicas

**Propósito**: Gestión de referencias dinámicas a objetos GeneXus. Permite invocar objetos por nombre/referencia en runtime sin hardcodear las llamadas.

> 📄 **Documentación detallada del módulo:** [modulos/dynamiccallreferences.md](modulos/dynamiccallreferences.md)

**Instances PXTools incluidas**:
- `PXWorkWithTDynamicCallReferences` — ABM de referencias

---

### @ExportImport — Exportación e Importación de Datos

**Propósito**: Infraestructura para exportar e importar datos en diferentes formatos.

> 📄 **Documentación detallada del módulo:** [modulos/exportimport.md](modulos/exportimport.md)

**Instances PXTools incluidas**:
- `PXParameterRequestImport` — Formulario de importación
- `PXReportTemplateRepImportResult` — Reporte de resultados

---

### @FileStorage — Almacenamiento de Archivos

**Propósito**: Sistema completo de gestión de archivos con soporte para múltiples servidores de almacenamiento, categorías predefinidas, y operaciones CRUD de archivos.

> 📄 **Documentación detallada del módulo:** [modulos/filestorage.md](modulos/filestorage.md)

**Instances PXTools incluidas**:
- `PXWorkWithFileStorage` — ABM de archivos
- `PXWorkWithFileStorageServers` — ABM de servidores
- `PXWorkWithFileStorageStorage` — ABM de almacenamiento
- `PXWorkWithPredefinedCategories` — ABM de categorías
- `PXComposerFileStorage` — Vista compuesta de archivos

---

### @MailAccounts — Cuentas de Correo

**Propósito**: Gestión de cuentas de correo electrónico para envío y recepción.

> 📄 **Documentación detallada del módulo:** [modulos/mailaccounts.md](modulos/mailaccounts.md)

**Instances PXTools incluidas**:
- `PXWorkWithMailAccounts` — ABM de cuentas de mail

---

### @Menus — Sistema de Menús

**Propósito**: Gestión de menús de navegación web (menú básico con items jerárquicos).

> 📄 **Documentación detallada del módulo:** [modulos/menus.md](modulos/menus.md)

**Instances PXTools incluidas**:
- `PXWorkWithTMnuWeb` — ABM de menús web

---

### @OAV — Object Attribute Values (Base)

**Propósito**: Infraestructura base para el patrón PXOAV. Provee las transacciones y ABMs para gestionar definiciones de atributos dinámicos y sus valores.

> 📄 **Documentación detallada del módulo:** [modulos/oav.md](modulos/oav.md)

**Instances PXTools incluidas**:
- `PXWorkWithTOAVAttributes` — ABM de atributos OAV
- `PXWorkWithTOAVAttributeValues` — ABM de valores
- `PXWorkWithTSystemObjectOAVAttributes` — ABM de objetos del sistema con OAV
- `PXParameterRequestSaveOAVSystemObjects` — Guardar objetos OAV

---

### @ProcessMonitor — Monitor de Procesos

**Propósito**: Monitoreo de procesos en ejecución con estados, mensajes y servidores de proceso.

> 📄 **Documentación detallada del módulo:** [modulos/processmonitor.md](modulos/processmonitor.md)

**Instances PXTools incluidas**:
- `PXWorkWithProcessStatus` — ABM de estados de proceso
- `PXWorkWithProcessStatusMessages` — ABM de mensajes
- `PXWorkWithProcessServers` — ABM de servidores
- `PXWorkWithProcessStatusAdvanced` — Vista avanzada

---

### @Projects — Gestión de Proyectos

**Propósito**: Gestión de proyectos con tipos, miembros y roles.

> 📄 **Documentación detallada del módulo:** [modulos/projects.md](modulos/projects.md)

**Instances PXTools incluidas**:
- `PXWorkWithProjects` — ABM de proyectos
- `PXWorkWithProjectTypes` — ABM de tipos
- `PXWorkWithProjectsMembers` — ABM de miembros

---

### @ReceiveMails — Recepción de Correos

**Propósito**: Procesamiento de correos entrantes.

> 📄 **Documentación detallada del módulo:** [modulos/receivemails.md](modulos/receivemails.md)

**Instances PXTools incluidas**:
- `PXWorkWithReceivedMails` — ABM de correos recibidos

---

### @ResponsiveLayout — *(módulo de GeneXus, no PXTools)*

**Propósito**: Posicionamiento responsive de secciones. Es un **módulo de la plataforma GeneXus** (usa el componente responsive nativo), **no un módulo PXTools**; aparece bajo `@PXTools/` solo como glue.

> ℹ️ **No es un módulo PXTools.** Bajo `@PXTools/@ResponsiveLayout/` solo hay **SDTs de parámetros de layout** (`Parameters`, `Panel{Top,Bottom,Left,Right,Center,Container}Properties`) y un DataProvider de ejemplo (`RetLayoutParametersExample`) que dan **soporte a los User Controls de PXTools**. Sin transacciones, procs ni patterns. No se lo cuenta como módulo PXTools.

---

### @Security — Seguridad

**Propósito**: Sistema completo de seguridad con usuarios, roles, dominios, permisos de acceso a objetos y registros, registro de usuarios, single sign-on silencioso.

> 📄 **Documentación detallada del módulo:** [modulos/security.md](modulos/security.md)

**Instances PXTools incluidas**:
- `PXWorkWithSecurityUsers` — ABM de usuarios
- `PXWorkWithSecurityUsersDomains` — ABM de dominios de usuarios
- `PXWorkWithSecurityRoles` — ABM de roles
- `PXWorkWithSecurityDomains` — ABM de dominios
- `PXWorkWithSecurityObjectAccess` — ABM de acceso a objetos
- `PXWorkWithSecurityObjectRecordsAccess` — ABM de acceso a registros
- `PXWorkWithSystemObjects` — ABM de objetos del sistema
- `PXWorkWithRegistrations` — ABM de registros de usuario
- `PXWorkWithSilentSignOnRequests` — ABM de SSO
- `PXComposerSecurityObjectAccess` — Vista compuesta de acceso a objetos
- `PXComposerSecurityObjectRecordAccess` — Vista compuesta de acceso a registros
- `PXParameterRequestChangePassword` — Cambio de password
- `PXParameterRequestRegistrationBasic` — Registro básico
- `PXParameterRequestRegistrationConfirmed` — Registro con confirmación

---

### @SecurityProjects — Seguridad por Proyectos

**Propósito**: Extensión de @Security para control de acceso por proyecto.

> ℹ️ **Sin doc dedicada.** El módulo contiene únicamente **grupos de subtipos** (`ProjectMemberSecurityUserId`, `ProjectMemberRolSecurityUserId`, `ProjectMemberRolSecurityRoleId`) que conectan los miembros de [@Projects](modulos/projects.md) con los usuarios/roles de [@Security](modulos/security.md). No aporta transacciones ni patterns propios; su mecánica se describe en [modulos/projects.md](modulos/projects.md).

---

### @SendMails — Envío de Correos

**Propósito**: Gestión de envío de correos con outbox, series de envío y seguimiento.

> 📄 **Documentación detallada del módulo:** [modulos/sendmails.md](modulos/sendmails.md)

**Instances PXTools incluidas**:
- `PXWorkWithSendMailOutbox` — ABM de outbox
- `PXWorkWithSendMailOutboxSeries` — ABM de series
- `PXParameterRequestChange` — Formulario de cambio

---

### @SmartMenus — *(módulo de GeneXus, no PXTools)*

**Propósito**: User Control de menú avanzado (búsqueda, favoritos, recientes). Es un **módulo/User Control de la plataforma GeneXus**, **no un módulo PXTools**; aparece bajo `@PXTools/` solo como glue.

> ℹ️ **No es un módulo PXTools.** Bajo `@PXTools/@SmartMenus/` solo hay los **SDTs de contrato** (`PXToolsSmartMenu`, `PXToolsSmartMenuSelectedItem`) y un DataProvider de ejemplo (`RetSmartMenuExample`) que dan **soporte al User Control de PXTools** que envuelve el SmartMenu de GeneXus. Sin transacciones ni patterns. El menú PXTools en uso está en [modulos/menus.md](modulos/menus.md).

---

### @Statistics — Estadísticas

**Propósito**: Definición y registro de estadísticas del sistema.

> 📄 **Documentación detallada del módulo:** [modulos/statistics.md](modulos/statistics.md)

**Instances PXTools incluidas**:
- `PXWorkWithStatisticDefinition` — ABM de definiciones
- `PXWorkWithStatisticLog` — ABM de log de estadísticas

---

### @System — Sistema

**Propósito**: Funcionalidades core del sistema PXTools. Incluye procedimientos para guardar módulos y objetos del sistema.

> 📄 **Documentación detallada del módulo:** [modulos/system.md](modulos/system.md)

**Instances PXTools incluidas**:
- `PXParameterRequestSaveSystemModules` — Guardar módulos
- `PXParameterRequestSaveSystemObjects` — Guardar objetos

---

### @SystemParameters — Parámetros del Sistema

**Propósito**: Gestión de parámetros globales del sistema (no por entidad, a diferencia de PXEntityParameters).

> 📄 **Documentación detallada del módulo:** [modulos/systemparameters.md](modulos/systemparameters.md)

**Instances PXTools incluidas**:
- `PXWorkWithSystemParametersPreferences` — ABM de parámetros

---

### @TableCleaner — Limpieza de Tablas

**Propósito**: Configuración y ejecución de limpieza programada de tablas (purge de datos antiguos).

> 📄 **Documentación detallada del módulo:** [modulos/tablecleaner.md](modulos/tablecleaner.md)

**Instances PXTools incluidas**:
- `PXWorkWithTableCleanerConfiguration` — ABM de configuración

---

### @TaskManager — Gestor de Tareas

**Propósito**: Sistema de gestión de tareas con colas, ejecución y monitoreo.

> 📄 **Documentación detallada del módulo:** [modulos/taskmanager.md](modulos/taskmanager.md)

**Instances PXTools incluidas**:
- `PXWorkWithTaskManager` — ABM de tareas
- `PXWorkWithTaskManagerQueues` — ABM de colas
- `PXParameterRequestExecutionMessage` — Mensaje de ejecución
- `PXParameterRequestUpdTaskManagerParameters` — Actualizar parámetros

---

### @WSLayer — Capa de WebServices

**Propósito**: Infraestructura para la capa de servicios web, incluyendo whitelist de acceso.

> 📄 **Documentación detallada del módulo:** [modulos/wslayer.md](modulos/wslayer.md)

**Instances PXTools incluidas**:
- `PXWorkWithWSWhiteList` — ABM de whitelist

---

### @WebServicesLog — Log de WebServices

**Propósito**: Registro, estadísticas y monitoreo de invocaciones de web services.

> 📄 **Documentación detallada del módulo:** [modulos/webserviceslog.md](modulos/webserviceslog.md)

**Instances PXTools incluidas**:
- `PXWorkWithWebServicesLog` — ABM de log
- `PXWorkWithWebServicesStatistics` — ABM de estadísticas
- `PXWorkWithWebServicesRAStatistics` — Estadísticas de rolling average
- `PXComposerWebServicesLog` — Vista compuesta de log
- `PXComposerWebServicesStatisticCounters` — Vista de contadores
- `PXParameterRequestWebServiceLogView` — Vista detallada de log

---

## Resumen de instancias PXTools en módulos

| Módulo | PXWorkWith | PXParameterRequest | PXComposer | PXReportTemplate | Total |
|--------|-----------|-------------------|-----------|-----------------|-------|
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

## Dependencias entre módulos

Grafo derivado de la KB (referencias calificadas `, PXTools.<Módulo>` y dominios compartidos) y contrastado con la matriz canónica del manual PXTools. **Capas:** `@APIs` es la **base universal** (todos dependen de él y él de ninguno); `@System` es la segunda capa; sobre ellas se apoyan los módulos de feature.

| Módulo | Depende de (funcionales) |
|---|---|
| **@APIs** | — (base universal) |
| **@System** | @APIs |
| **@SystemParameters** | @APIs |
| **@DynamicCallReferences** | @APIs |
| **@Menus** | @APIs (usa el módulo GeneXus @SmartMenus para el estilo SmartMenu) |
| **@ControlPreferences** | @APIs |
| **@WSLayer** | @APIs |
| **@ExportImport** | @APIs |
| **@Statistics** | @APIs, @DynamicCallReferences · dispara @TaskManager |
| **@TableCleaner** | @APIs, @DynamicCallReferences, @SystemParameters · dispara @TaskManager |
| **@WebServicesLog** | @APIs, @TableCleaner, @SystemParameters |
| **@OAV** | @APIs, @DynamicCallReferences (+@System) |
| **@MailAccounts** | @APIs, @System |
| **@FileStorage** | @APIs, @System |
| **@Projects** | @APIs, @Security |
| **@Security** | @APIs, @System, @SystemParameters, @ControlPreferences, @SendMails |
| **@SendMails** | @APIs, @System, @FileStorage, @MailAccounts (opt @TaskManager) |
| **@ReceiveMails** | @APIs, @System, @MailAccounts, @FileStorage |
| **@ProcessMonitor** | @APIs, @System, @SystemParameters, @SendMails, @FileStorage, @TaskManager |
| **@TaskManager** | @APIs, @System, @DynamicCallReferences, @ProcessMonitor, @SystemParameters |
| **@Alerts** | @APIs, @SendMails, @SystemParameters (+@FileStorage, @MailAccounts, opt @TaskManager) |
| **@CloudTasks** | @APIs, @ProcessMonitor, @FileStorage, @Alerts, @SystemParameters · dispara @TaskManager |

> **Nota:** casi todos los módulos referencian además `@Menus` y `@DynamicCallReferences` a través de sus DataProviders `Personalized/RetMenus*` y `RetDynamicCallReference*` (andamiaje de generación), no de su lógica funcional. Se omiten arriba salvo cuando la dependencia es funcional (p.ej. @DynamicCallReferences en @TaskManager/@TableCleaner/@Statistics/@OAV).
>
> **`@ResponsiveLayout` y `@SmartMenus` NO son módulos PXTools** — son módulos de la plataforma **GeneXus**; bajo `@PXTools/` solo aportan SDTs/APIs de soporte a los User Controls de PXTools. No participan del grafo de dependencias PXTools como módulos.

Este grafo es la base de la **atribución de dominios**: un dominio usado por >1 módulo pertenece al módulo común del que todos dependen, o a `@APIs` (ver la §Dominios de cada doc y la regla de nomenclatura).

## Cuándo incorporar cada módulo

| Necesidad | Módulo recomendado |
|-----------|-------------------|
| Autenticación y autorización | @Security |
| Envío de emails | @SendMails + @MailAccounts |
| Recepción de emails | @ReceiveMails + @MailAccounts |
| Tareas programadas (cron) | @CloudTasks |
| Cola de tareas | @TaskManager |
| Almacenamiento de archivos | @FileStorage |
| Alertas/notificaciones | @Alerts |
| Menús de navegación | @Menus (con el User Control SmartMenus de GeneXus) |
| Parámetros configurables | @SystemParameters |
| Atributos dinámicos | @OAV |
| Auditoría de cambios | pattern PXAudit (no módulo — ver @Audit) |
| APIs REST/SOAP | @WSLayer + @WebServicesLog |
| Exportación/importación | @ExportImport |
| Monitoreo de procesos | @ProcessMonitor |
| Estadísticas | @Statistics |
| Limpieza de datos | @TableCleaner |
| Gestión de proyectos | @Projects |
