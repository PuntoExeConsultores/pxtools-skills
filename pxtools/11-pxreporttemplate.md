# PXReportTemplate — Pattern de Templates para Reportes

## Qué es

PXReportTemplate define **templates de configuración para reportes**. A diferencia de los demás patterns, PXReportTemplate **no genera objetos GeneXus directamente** (su sección `<Objects />` está vacía). En su lugar, define un template/schema de configuración que otros patrones y objetos pueden usar para estandarizar la generación de reportes.

## Parent Objects

- `Procedure` — se asocia a Procedures de reporte

## Objetos que genera

```xml
<Objects />  <!-- No genera objetos directamente -->
```

PXReportTemplate actúa como un **metadata pattern**: almacena configuración que es leída en runtime por los objetos de reporte.

## Propósito

El pattern sirve para:
1. Estandarizar la configuración de reportes (formato, layout, encabezados)
2. Permitir que los desarrolladores definan templates reutilizables
3. Proveer un schema de configuración declarativo para reportes

## Archivos del pattern

| Archivo | Descripción |
|---------|-------------|
| `PXReportTemplate.Pattern` | Definición del pattern (Objects vacío) |
| `PXReportTemplateInstance.xml` | Schema de la instancia |
| `PXReportTemplateSettings.xml` | Preferencias globales |
| `PXReportTemplateCustomTypes.xml` | Tipos custom |
| `Resources/PXReportTemplateDefaultSettings.xml` | Settings por defecto |

## Uso real

- 8 instancias en pFacturas KB
- El módulo **@ExportImport** tiene una instancia: `PXReportTemplateRepImportResult` (reporte de resultados de importación)
- Se usa para definir la configuración de reportes PDF/Excel

## Integración

PXReportTemplate se integra con:
- **PXWorkWith** (modos de exportación)
- **@ExportImport** (reportes de importación)
- Procedures de reporte custom que leen la configuración del template

## Settings

El pattern tiene un SettingsSpecification (`PXReportTemplateSettings.xml`) con un DefaultSettings (`PXReportTemplateDefaultSettings.xml`), lo que indica que tiene configuración global aplicable a todas las instancias.
