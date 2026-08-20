# PXReportTemplate — Report Template Pattern

## What it is

PXReportTemplate defines **configuration templates for reports**. Unlike the other patterns, PXReportTemplate **does not generate GeneXus objects directly** (its `<Objects />` section is empty). Instead it defines a configuration template/schema that other patterns and objects can use to standardise report generation.

## Parent Objects

- `Procedure` — it attaches to report Procedures

## Objects it generates

```xml
<Objects />  <!-- Generates no objects directly -->
```

PXReportTemplate acts as a **metadata pattern**: it stores configuration that report objects read at runtime.

## Purpose

The pattern exists to:
1. Standardise report configuration (format, layout, headers)
2. Let developers define reusable templates
3. Provide a declarative configuration schema for reports

## Pattern files

| File | Description |
|------|-------------|
| `PXReportTemplate.Pattern` | Pattern definition (empty Objects) |
| `PXReportTemplateInstance.xml` | Instance schema |
| `PXReportTemplateSettings.xml` | Global preferences |
| `PXReportTemplateCustomTypes.xml` | Custom types |
| `Resources/PXReportTemplateDefaultSettings.xml` | Default settings |

## Real-world use

- The **@ExportImport** module has one instance: `PXReportTemplateRepImportResult` (import result report)
- It is used to define the configuration of PDF/Excel reports

## Integration

PXReportTemplate integrates with:
- **PXWorkWith** (export modes)
- **@ExportImport** (import reports)
- Custom report Procedures that read the template configuration

## Settings

The pattern has a SettingsSpecification (`PXReportTemplateSettings.xml`) with DefaultSettings (`PXReportTemplateDefaultSettings.xml`), which means it carries global configuration applicable to every instance.
