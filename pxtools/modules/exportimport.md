# Módulo @ExportImport — Exportación e Importación de Datos

> Comportamiento del módulo `@PXTools/@ExportImport`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@ExportImport` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.ExportImport`.
- **Depende de:** `@APIs` (base).

## 1. Qué provee

Infraestructura genérica para **importar/exportar entidades vía Excel**. El módulo aporta el **chasis común** (form de carga, helpers de lectura de celdas, reporte de errores); la lógica por entidad la implementa cada consumidor con procs `Imp<Entidad>`/`Exp<Entidad>` (generados por el pattern `PXExportImport`, fuera de este módulo). **No define transacciones** — opera sobre datos de terceros.

## 2. Concepto central: dispatcher + hooks por entidad

- Un **enum** (`PXToolsExportImportObject`) registra "qué se puede importar"; cada aplicación agrega el valor de su entidad.
- Dos **dispatchers** (`ChkObjects` / `ImpObjects`) hacen `Do Case` sobre ese enum y delegan al **hook** `Imp<Entidad>` correspondiente.
- El resultado es un `SDTImportResult` (colección de errores) que, si no está vacío, se muestra en el reporte `RepImportResult`.

## 3. Dominios del módulo

Ambos son de @ExportImport (nomenclatura: el nombre real del módulo va tras `PXTools`), **root-legacy** (viven en el `#Domains/` raíz):

| Dominio | Rol |
|---|---|
| **PXToolsExportImportObject** (Char(50)) | Registro de "qué se importa/exporta". **Punto de extensión**: cada consumidor agrega el valor de su entidad (código + descripción). |
| **PXToolsImportMode** (Char(20)) | `Check` (validación sin persistir) / `Update` (persistencia). |

## 4. Mecanismo (flujo de import)

1. **Form `PuImport`** (WebPanel generado por `PXParameterRequestImport`): `parm(in: &ExportImportObject, &WindowSelf)`. Presenta el blob `&File` (Excel) y los checks `&OnlyCheck`, `&UpdateWithoutErrors`, `&DeleteSubLevelRecordsNotFound`. El botón alterna "Check" vs "Import" según `&OnlyCheck`.
2. **Event Enter** llama `ChkObjects.Udp(...)` (siempre) y, si no es solo-check y hubo éxito o `UpdateWithoutErrors`, `ImpObjects.Udp(...)`.
3. `ChkObjects`/`ImpObjects` resuelven el blob a path (`RetBlobPath`), hacen `Do Case` sobre `&ExportImportObject` y delegan al `Imp<Entidad>.Udp(&FilePath, PXToolsImportMode.Check|Update, &UpdateWithoutErrors, &DeleteSubLevelRecordsNotFound)`, que retorna un `SDTImportResult`.
4. El SDT de errores se serializa a JSON y se guarda en **WebSession** (`PPEXE_AcSUV02`). `&Succeed = (&SDTImportResult.Count = 0)`.
5. Si hubo errores, se abre el reporte **`RepImportResult`** (PDF, HTTP) que recupera el JSON (`PPEXE_DeSUV02`), hace `FromJson()` e imprime cada error (Sheet / Row / ErrorMessage). Si todo OK, "Import succeed".

### Contrato del hook por entidad
Un consumidor implementa `Imp<Entidad>` con:
```
parm(in: &FilePath, in: &PXToolsImportMode, in: &UpdateWithoutErrors, in: &DeleteSubLevelRecordsNotFound, out: &SDTImportResult)
```
Dentro: abre el Excel (`&ExcelDocument.Open(&FilePath)`), itera filas leyendo celdas con los `RetExcelCell*`, valida (agregando ítems a `SDTImportResult` con Sheet/Row/ErrorMessage) y, en modo `Update` (si `Count=0` o `UpdateWithoutErrors`), persiste vía Business Component. Para registrarlo: agrega el valor al enum `PXToolsExportImportObject` y un `Case` en `ChkObjects`/`ImpObjects`.

## 5. APIs vs Personalized

- **`APIs/`** (core): `RepImportResult` (reporte), los helpers `RetExcelCell*`, el SDT `SDTImportResult`, y las instancias de patterns.
- **`Personalized/`** (el **dispatcher**, se edita para agregar cada entidad):
  | Objeto | Qué se customiza |
  |---|---|
  | `ChkObjects` (Procedure) | Modo `Check`: `Do Case` enum → `Imp<Entidad>`(Check). |
  | `ImpObjects` (Procedure) | Modo `Update`: `Do Case` enum → `Imp<Entidad>`(Update). |

## 6. Instancias de patterns

- **PXParameterRequestImport** — genera los popups `PuImport` / `PuImportSpanish` (blob + 3 checks + acción con las llamadas condicionales a `ChkObjects`/`ImpObjects`/`RepImportResult`).
- **PXReportTemplateRepImportResult** — aplica cabezal/pie estándar al reporte de resultados.

## 7. Procedimientos / APIs clave

| Objeto | `Parm()` | Propósito |
|---|---|---|
| `RepImportResult` (Main, HTTP) | — (lee `SDTImportResult` de session) | Reporte PDF de errores de import. |
| `RetExcelCellText` / `…Memo` / `…Short` / `…Long` / `…Date` | `(in: &ExcelDocument, &CellRow, &CellColumn; out: &Valor)` | Lee una celda normalizando fórmulas, `.0` final y notación científica. |
| `SDTImportResult` (SDT) | `{ Sheet, Row, ErrorCode, ErrorMessage }` | **Contrato de resultado** de todo hook de import. |

Dependencias externas (`PXTools.APIs`): `RetBlobPath` (blob→path), `PPEXE_AcSUV02`/`PPEXE_DeSUV02` (WebSession — trasladan el `SDTImportResult` al reporte).

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- `modulos/apis.md` — `RetBlobPath` y los helpers de WebSession usados por el flujo.
- El pattern `PXExportImport` (define los `Imp<Entidad>`/`Exp<Entidad>` en el módulo consumidor) — documentado aparte con los demás patterns.
