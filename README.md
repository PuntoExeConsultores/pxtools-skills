# PXTools — Documentación para IAs

Documentación técnica del framework **PXTools** (conjunto de *patterns* para GeneXus, desarrollado por **PuntoExe Consultores**), pensada para **entregarla como contexto a asistentes de IA** que ayuden a programar con PXTools sobre fuentes de GeneXus externalizadas por KBbridge.

## Qué contiene

- **[`pxtools-start-here.md`](pxtools-start-here.md)** — punto de entrada / índice general.
- **[`pxtools/`](pxtools/)** — documentación detallada:
  - `00-overview.md` … `13-*.md` — arquitectura del framework y patterns: **PXWorkWith**, **PXParameterRequest**, **PXComposer**, **PXFlowController**, **PXOAV**, **PXEntityParameters**, **PXReportTemplate**, **PXWSLayer/Query/Data/Transaction**, semántica de grids/webpanels y acciones/UI.
  - `20-modulos-pxtools.md` + **[`pxtools/modulos/`](pxtools/modulos/)** — un documento por módulo PXTools (@Security, @TaskManager, @Alerts, @SendMails, @OAV, …): qué provee, transacciones, mecanismos, **APIs vs Personalized**, dominios y dependencias entre módulos.
  - `30-33-*.md` — guía de reconocimiento de patterns, capacidades dual-platform, limitaciones y estrategia de migración.

## Cómo usarla con una IA

Cargá estos archivos como contexto o *skill* de tu asistente (Claude Code, Cursor, etc.) al trabajar sobre una KB de GeneXus con PXTools. Todos los ejemplos son **genéricos**: no contienen objetos de KBs de clientes.

## Cambios

Ver [CHANGELOG.md](CHANGELOG.md) para el registro de qué se agregó o corrigió en cada publicación.

## Recursos

- Sitio: https://pxtools.puntoexe.com.uy/
- Manual: https://sites.google.com/puntoexe.com.uy/pxtools-manual/

## Licencia

Ver el archivo [LICENSE](LICENSE).

---

© PuntoExe Consultores.
