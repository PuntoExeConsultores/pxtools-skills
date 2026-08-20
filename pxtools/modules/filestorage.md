# Módulo @FileStorage — Almacenamiento de Archivos

> Comportamiento del módulo `@PXTools/@FileStorage`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@FileStorage` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.FileStorage`.
- **Depende de:** `@APIs` (base), `@System`. Infra de generación: `@Menus`, `@SystemParameters`, `@DynamicCallReferences`, `@ControlPreferences`.

## 1. Qué provee

Un framework de **gestión de archivos**: guarda uno o varios archivos bajo una "cabecera" (`FileStorage`), con soporte para **categorías**, **agrupación**, **múltiples tipos de servidor** de almacenamiento, y un **componente de UI reutilizable** (`PXComposerFileStorage`) para subir/ver/descargar. El contenido físico vive en un **Blob**.

## 2. Concepto central: cabecera + archivos

- **`FileStorage`** = cabecera (descripción, categoría, tipo de uso, servidor).
- **`FileStorageStorage`** = los **archivos** (1:N con la cabecera); el contenido está en el Blob `FileStorageStorageBlob`. Se pueden agrupar por `FileStorageStorageGroupId`.
- **`FileStorageType`** define el modo de uso de la cabecera: `Unique` (1 archivo), `Predefined` (archivos organizados por categorías de un catálogo), `Free` (multi-archivo libre).

## 3. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **FileStorage** | `FileStorageId` | **Cabecera**: `Description`, `FileStorageType`, `FileStorageStorageType`, `FileStorageCategory`, `FileStoragePredefinedCategoryId` (FK), `FileStorageServerId` (FK), `FileStorageKeepOriginalNames`. |
| **FileStorageStorage** | `FileStorageId, FileStorageStorageId` | **Archivos**: `Description`, `Blob` (contenido), `Name`, `Extension`, `DateTime`, `GroupId`. |
| **FileStorageServers** | `FileStorageServerId` | Servidores de almacenamiento: `Type`, `Name`, `Host`, `User`, `Password` (PswEnc), `Address`, `Port`. |
| **PredefinedCategories** | `PredefinedCategoryId` (+ nivel `Category`) | Catálogo de categorías predefinidas (2 niveles). |

BCs: `FileStorageBC` (2 niveles) y `FileStorageFirstLevelBC` (solo cabecera, para borrado).

## 4. Dominios del módulo

Propios (nombre `FileStorage*`), todos **root-legacy** (viven en el `#Domains/` raíz):

| Dominio | Valores |
|---|---|
| **FileStorageType** | Unique, Predefined, Free |
| **FileStorageStorageType** | Blob, FTP, S3 |
| **FileStorageServerType** | S3, FTP, FileServer, GoogleDrive |
| **FileStorageCategory** (Char(3), **enum abierto**) | Clasifica el origen funcional; valores base del framework: General, IssueTracking, SendMail, SystemAlert (cada aplicación extiende con sus categorías). |
| **FileStorageExtension** | XML, PFX, ZIP |

## 5. Mecanismo

- **Contenido físico**: siempre en el Blob `FileStorageStorageBlob` (una fila por archivo). El backend real del Blob (disco/BD/nube) lo resuelve el Storage Service del entorno GeneXus. `FileStorageStorageType`/`FileStorageServers` son la capa de descripción multi-servidor; **hoy solo `Blob` está implementado end-to-end** (`RetFileStorageStorageLink` resuelve `Blob` vía `PathToURL`); FTP/S3/cloud son puntos de extensión del modelo.
- **Elección de servidor**: `FileStorageStorageType` + `FileStorageServerId` (`ValFileStorage` exige `ServerId>0` cuando el tipo ≠ Blob).
- **SDTs**: `SDTFileStorage` (cabecera + colección `Storage.Item` con `Id/Description/Path/FileName/FileExtension`) — el SDT principal de alta; `SDTFileStorageData` (archivo suelto: Path/Name/Extension); `SDTFileStorageReference` (puntero StorageId + StorageStorageId).
- **Nombres**: `FileStorageKeepOriginalNames` (renombra al nombre original en lectura) y el parámetro `FileStorageStorageNamePrefix` (nombres únicos `prefijo+Id+StorageId`).

## 6. APIs vs Personalized

- **`APIs/`** (core): las transacciones, los SDTs, y toda la familia `Add*/Ret*/Del*/Upd*/Val*/Chk*`.
- **`Personalized/`**:
  | Objeto | Qué se customiza |
  |---|---|
  | `RetSystemParametersFileStorage` (DataProvider) | `FileStorageStorageNamePrefix`, `FileStorageTempDirectory`. |
  | `RetMenusFileStorage` (DataProvider) | Ítems de menú (File Storage / Predefined Categories / Servers). |
  | `RetDynamicCallReferenceFileStorage` (DataProvider) | Registra el Table Cleaner. |
  | `PrcTableCleanerFileStorage` (Procedure) | Purga por fecha. |
  | `PXToolsParameterRequestTemplateWithUploadify` (WebPanel) | Plantilla de carga multi-archivo (user control Uploadify). |

## 7. Instancias de patterns

- **PXWorkWith**: `PXWorkWithFileStorage`, `PXWorkWithFileStorageServers`, `PXWorkWithPredefinedCategories`, `PXWorkWithFileStorageStorage` (genera las vistas de selección que usa el composer).
- **`PXComposerFileStorage`** — **componente reutilizable principal**, 4 niveles: `CmFileStorageComponent` (orquestador), `WbFileStorageWebPanel`, `PrFileStoragePrompt` (prompt que devuelve el `FileStorageId`), `PuFileStorageDisplay` (popup). `CmFileStorageComponent` recibe Mode + tipo/categoría/servidor, valida (`ValFileStorage`), crea la cabecera (`AddFileStorage`) e instancia la sub-UI según `FileStorageType`.

## 8. Procedimientos / APIs clave

**Alta**: `AddFileStorage(in: &SDTFileStorage, out: &FileStorageId)` (**entrada principal**), `AddFileStorageStorage(in: &FileStorageId, in: &Storage, in: &GroupId)`.

**Lectura**: `RetFileStorage(in: &FileStorageId, in: &GroupId, in: &StorageId, out: &Storage)`, `RetFileStorageFiles(… out: &SDTFileStorageData)`, `RetFileStorageFirstFile`, `RetFileStorageReferenceFile`, `RetFileStorageData` (metadata cabecera), `RetFileStorageStorageLink` (URL de descarga), `RetFileStorageStorageBlobExist`, `RetFileStorageStorageCounter`, `RetLastFileStorageStorageId`.

**Baja**: `DelFileStorage(in: &FileStorageId, out…)` (cabecera + archivos), `DelFileStorageStorage(… StorageStorageId=0 → todos)`, `…ByGroup`/`…ByName`/`…ByExtension`, bulk `DelFileStorageStorages`/`…TableCleaner`.

**Modificación/utilidad**: `UpdFileStorageStorage` (borra+re-agrega), `UpdFileStorageStorageDescription`, `UpdFileStorageCopy(from, to)` (copia entre storages, usa `FileStorageTempDirectory`), `ValFileStorage(…)` (validación), `ChkFileStorageStorageExtension`, `IsFileStorageInstalled`.

**Uso típico**: cargar `&SDTFileStorage` (Description, Category, FileStorageType; cada `Storage.Item` con `Path`=contenido, FileName, FileExtension) → `AddFileStorage.Call(&SDTFileStorage, &FileStorageId)`; leer con `RetFileStorageFirstFile`/`RetFileStorageFiles` y usar `Item.Path`. Para UI embebida reutilizable, instanciar `CmFileStorageComponent`.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- Consumidores: [sendmails.md](sendmails.md) / módulo @ReceiveMails (adjuntos), [cloudtasks.md](cloudtasks.md) (contenido de certificados), [processmonitor.md](processmonitor.md) (salida de procesos).
- Módulos **@SystemParameters** (prefijo/temp dir), **@TableCleaner** (purga).
