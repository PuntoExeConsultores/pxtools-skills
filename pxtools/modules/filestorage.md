# @FileStorage Module — File Storage

> Behaviour of the `@PXTools/@FileStorage` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@FileStorage` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.FileStorage`.
- **Depends on:** `@APIs` (base), `@System`. Generation infrastructure: `@Menus`, `@SystemParameters`, `@DynamicCallReferences`, `@ControlPreferences`.

## 1. What it provides

A **file management** framework: it stores one or many files under a "header" (`FileStorage`), with support for **categories**, **grouping**, **several storage server types**, and a **reusable UI component** (`PXComposerFileStorage`) for uploading/viewing/downloading. The physical content lives in a **Blob**.

## 2. Core concept: header + files

- **`FileStorage`** = the header (description, category, usage type, server).
- **`FileStorageStorage`** = the **files** (1:N with the header); the content sits in the `FileStorageStorageBlob` Blob. They can be grouped by `FileStorageStorageGroupId`.
- **`FileStorageType`** defines how the header is used: `Unique` (one file), `Predefined` (files organised by categories from a catalogue), `Free` (free multi-file).

## 3. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **FileStorage** | `FileStorageId` | **Header**: `Description`, `FileStorageType`, `FileStorageStorageType`, `FileStorageCategory`, `FileStoragePredefinedCategoryId` (FK), `FileStorageServerId` (FK), `FileStorageKeepOriginalNames`. |
| **FileStorageStorage** | `FileStorageId, FileStorageStorageId` | **Files**: `Description`, `Blob` (content), `Name`, `Extension`, `DateTime`, `GroupId`. |
| **FileStorageServers** | `FileStorageServerId` | Storage servers: `Type`, `Name`, `Host`, `User`, `Password` (PswEnc), `Address`, `Port`. |
| **PredefinedCategories** | `PredefinedCategoryId` (+ a `Category` level) | Catalogue of predefined categories (2 levels). |

BCs: `FileStorageBC` (2 levels) and `FileStorageFirstLevelBC` (header only, for deletion).

## 4. Module domains

Its own (named `FileStorage*`), all **root-legacy** (they live in the root `#Domains/`):

| Domain | Values |
|---|---|
| **FileStorageType** | Unique, Predefined, Free |
| **FileStorageStorageType** | Blob, FTP, S3 |
| **FileStorageServerType** | S3, FTP, FileServer, GoogleDrive |
| **FileStorageCategory** (Char(3), **open enum**) | Classifies the functional origin; the framework's base values are General, IssueTracking, SendMail, SystemAlert (each application extends it with its own categories). |
| **FileStorageExtension** | XML, PFX, ZIP |

## 5. How it works

- **Physical content**: always in the `FileStorageStorageBlob` Blob (one row per file). The Blob's real backend (disk/database/cloud) is resolved by the GeneXus environment's Storage Service. `FileStorageStorageType`/`FileStorageServers` are the multi-server description layer; **today only `Blob` is implemented end to end** (`RetFileStorageStorageLink` resolves `Blob` through `PathToURL`); FTP/S3/cloud are extension points of the model.
- **Choosing a server**: `FileStorageStorageType` + `FileStorageServerId` (`ValFileStorage` requires `ServerId>0` when the type is not Blob).
- **SDTs**: `SDTFileStorage` (header + a `Storage.Item` collection with `Id/Description/Path/FileName/FileExtension`) — the main SDT for creation; `SDTFileStorageData` (a standalone file: Path/Name/Extension); `SDTFileStorageReference` (a pointer: StorageId + StorageStorageId).
- **Names**: `FileStorageKeepOriginalNames` (renames back to the original name on read) and the `FileStorageStorageNamePrefix` parameter (unique names `prefix+Id+StorageId`).

## 6. APIs vs Personalized

- **`APIs/`** (core): the transactions, the SDTs, and the whole `Add*/Ret*/Del*/Upd*/Val*/Chk*` family.
- **`Personalized/`**:
  | Object | What gets customized |
  |---|---|
  | `RetSystemParametersFileStorage` (DataProvider) | `FileStorageStorageNamePrefix`, `FileStorageTempDirectory`. |
  | `RetMenusFileStorage` (DataProvider) | Menu items (File Storage / Predefined Categories / Servers). |
  | `RetDynamicCallReferenceFileStorage` (DataProvider) | Registers the Table Cleaner. |
  | `PrcTableCleanerFileStorage` (Procedure) | Purge by date. |
  | `PXToolsParameterRequestTemplateWithUploadify` (WebPanel) | Multi-file upload template (Uploadify user control). |

## 7. Pattern instances

- **PXWorkWith**: `PXWorkWithFileStorage`, `PXWorkWithFileStorageServers`, `PXWorkWithPredefinedCategories`, `PXWorkWithFileStorageStorage` (generates the selection views the composer uses).
- **`PXComposerFileStorage`** — the **main reusable component**, 4 levels: `CmFileStorageComponent` (orchestrator), `WbFileStorageWebPanel`, `PrFileStoragePrompt` (a prompt returning the `FileStorageId`), `PuFileStorageDisplay` (popup). `CmFileStorageComponent` receives Mode + type/category/server, validates (`ValFileStorage`), creates the header (`AddFileStorage`) and instantiates the sub-UI according to `FileStorageType`.

## 8. Key procedures / APIs

**Creation**: `AddFileStorage(in: &SDTFileStorage, out: &FileStorageId)` (**the main entry point**), `AddFileStorageStorage(in: &FileStorageId, in: &Storage, in: &GroupId)`.

**Reading**: `RetFileStorage(in: &FileStorageId, in: &GroupId, in: &StorageId, out: &Storage)`, `RetFileStorageFiles(… out: &SDTFileStorageData)`, `RetFileStorageFirstFile`, `RetFileStorageReferenceFile`, `RetFileStorageData` (header metadata), `RetFileStorageStorageLink` (download URL), `RetFileStorageStorageBlobExist`, `RetFileStorageStorageCounter`, `RetLastFileStorageStorageId`.

**Deletion**: `DelFileStorage(in: &FileStorageId, out…)` (header + files), `DelFileStorageStorage(… StorageStorageId=0 → all)`, `…ByGroup`/`…ByName`/`…ByExtension`, bulk `DelFileStorageStorages`/`…TableCleaner`.

**Modification/utilities**: `UpdFileStorageStorage` (deletes and re-adds), `UpdFileStorageStorageDescription`, `UpdFileStorageCopy(from, to)` (copies between storages, using `FileStorageTempDirectory`), `ValFileStorage(…)` (validation), `ChkFileStorageStorageExtension`, `IsFileStorageInstalled`.

**Typical use**: fill `&SDTFileStorage` (Description, Category, FileStorageType; each `Storage.Item` with `Path`=content, FileName, FileExtension) → `AddFileStorage.Call(&SDTFileStorage, &FileStorageId)`; read with `RetFileStorageFirstFile`/`RetFileStorageFiles` and use `Item.Path`. For a reusable embedded UI, instantiate `CmFileStorageComponent`.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- Consumers: [sendmails.md](sendmails.md) / the @ReceiveMails module (attachments), [cloudtasks.md](cloudtasks.md) (certificate content), [processmonitor.md](processmonitor.md) (process output).
- The **@SystemParameters** (prefix/temp dir) and **@TableCleaner** (purge) modules.
