# PXTools — Documentación para Entrenamiento de IAs

## Qué es PXTools

PXTools es un **framework de patterns (patrones) para GeneXus** desarrollado por PuntoExe Consultores (Uruguay). Automatiza la generación de código GeneXus transformando definiciones declarativas (archivos XML `.gxPattern`) en objetos GeneXus completos: WebPanels, Procedures, DataProviders, SDTs, APIs, Transactions, entre otros.

## Concepto clave

Un **pattern** en PXTools tiene tres niveles:

1. **Definición del patrón** (`.Pattern`) — Schema XML que define la estructura, nodos y propiedades disponibles, y qué objetos GeneXus se generan de cada nodo.
2. **Instancia del patrón** (`.gxPattern`) — Configuración concreta que sigue el schema, definida por el desarrollador para una entidad o funcionalidad específica.
3. **Objetos generados** (`.Childs/`) — Código GeneXus resultante: WebPanels, Procedures, DataProviders, SDTs, etc.

## Índice de documentación

Toda la documentación detallada está en la subcarpeta [`pxtools/`](pxtools/):

### Visión general
- [00-overview.md](pxtools/00-overview.md) — Arquitectura del framework, cómo funciona el generador, tipos de patterns

### Patterns de UI (generan WebPanels)
- [01-pxworkwith.md](pxtools/01-pxworkwith.md) — CRUD maestro-detalle con Selection, View, Edit, Prompt
- [02-pxparameterrequest.md](pxtools/02-pxparameterrequest.md) — Formularios modales/popup para captura de parámetros
- [03-pxcomposer.md](pxtools/03-pxcomposer.md) — Composición de pantallas a partir de WebComponents

### Pattern de flujo
- [04-pxflowcontroller.md](pxtools/04-pxflowcontroller.md) — Orquestación de flujos de trabajo con confirmaciones y acciones encadenadas

### Patterns de API/WebServices
- [05-pxwslayer.md](pxtools/05-pxwslayer.md) — Orquestador de API REST/SOAP con OpenAPI 3.0
- [06-pxwsquery.md](pxtools/06-pxwsquery.md) — DataProviders con filtros, ordenamiento y paginación
- [07-pxwsdata.md](pxtools/07-pxwsdata.md) — Procedimientos de lectura de datos con hooks de código
- [08-pxwstransaction.md](pxtools/08-pxwstransaction.md) — CRUD (Load/Save/Delete) sobre transacciones vía Business Components

### Referencia transversal
- [12-acciones-patterns-ui.md](pxtools/12-acciones-patterns-ui.md) — Sistema de acciones compartido por PXWorkWith, PXParameterRequest y PXComposer (callTypes, ConditionalCalls, PXInstance, ciclo de ejecucion, propiedad execute)
- [13-semantica-grids-webpanels.md](pxtools/13-semantica-grids-webpanels.md) — Cómo interpretar la presencia de grids en un WebPanel para reconocer el pattern: grids reales vs fantasma (SDT-persistencia legacy), múltiples grids, For Each Line

### Patterns de datos/configuración
- [09-pxoav.md](pxtools/09-pxoav.md) — Object Attribute Values (atributos extensibles dinámicos)
- [10-pxentityparameters.md](pxtools/10-pxentityparameters.md) — Parámetros configurables por entidad
- [11-pxreporttemplate.md](pxtools/11-pxreporttemplate.md) — Templates para generación de reportes

### Módulos @PXTools
- [20-modulos-pxtools.md](pxtools/20-modulos-pxtools.md) — 25+ módulos reutilizables: Security, Alerts, CloudTasks, FileStorage, etc.

### Guías transversales
- [30-guia-reconocimiento-patterns.md](pxtools/30-guia-reconocimiento-patterns.md) — Cómo analizar WebPanels manuales y determinar a qué pattern migrar
- [31-capacidades-dual-platform.md](pxtools/31-capacidades-dual-platform.md) — Migración Desktop → Responsiva, generación dual-platform
- [32-limitaciones-y-gaps.md](pxtools/32-limitaciones-y-gaps.md) — Qué NO soportan los patterns hoy
- [33-estrategia-migracion.md](pxtools/33-estrategia-migracion.md) — Proceso de migración de KB sin patterns a PXTools

## Cómo usar esta documentación

Esta documentación está diseñada para que una IA pueda:

1. **Entender** qué genera cada pattern y cómo se configura
2. **Reconocer** en código GeneXus existente qué patterns podrían aplicarse
3. **Generar** instancias de patterns (.gxPattern) válidas para nuevas funcionalidades
4. **Migrar** WebPanels manuales a patterns de PXTools
5. **Diagnosticar** qué WebPanels no son migrables y por qué

## Fuentes de información utilizadas

- KB 1: 179 PXWorkWith, 93 PXParameterRequest, 14 PXComposer, 1 PXFlowController, 8 PXReportTemplate, 2 PXOAV, 2 PXEntityParameters
- KB 2: Ejemplos de PXWSLayer, PXWSQuery, PXWSData, PXWSTransaction
- KB 3: 19 módulos @PXTools
- Sitio web: https://pxtools.puntoexe.com.uy/
- Manual: https://sites.google.com/puntoexe.com.uy/pxtools-manual/
- Definiciones de patterns: `Patterns/*.Pattern` de las KBs externalizadas
