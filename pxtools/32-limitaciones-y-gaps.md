# Limitaciones y Gaps — Lo que los Patterns NO Soportan

## Propósito

Este documento lista las **características de UI y lógica que los patterns de PXTools no soportan actualmente**. Es fundamental para:
1. Identificar WebPanels que no pueden migrarse 100% a patterns
2. Determinar qué requeriría implementación a nivel del generador
3. Planificar qué funcionalidad implementar manualmente post-migración

---

## Limitaciones por pattern

### PXWorkWith

#### Layout y UI
| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| Layout custom de grilla | No se puede personalizar libremente el HTML/layout de cada fila de la grilla | WebPanels con grillas tipo "card" o layouts no tabulares no son migrables |
| Grillas anidadas en Selection | Selection solo soporta una grilla principal, no grillas dentro de grillas | Pantallas con master-detail inline en la misma grilla |
| Controles de terceros custom | Solo soporta controles predefinidos por PXTools (Chosen, DatePicker, etc.) | Controles custom como editores WYSIWYG, mapas, gráficos interactivos |
| Drag-and-drop | No soportado | Pantallas de reordenamiento, kanban boards |
| Múltiples grillas en Selection | Solo una grilla principal por Selection | Pantallas con varias grillas lado a lado |

#### Lógica
| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| Comunicación entre tabs | Los tabs (sections) en View son independientes: solo el tab activo está renderizado, por lo que no se comunican en tiempo real vía `GlobalEvents`. Sin embargo, pueden compartir información mediante **WebSession**: un tab escribe en WebSession y al activarse otro tab, este lee la información en su Start/Refresh | No es una limitación bloqueante, pero requiere código en hooks para implementar la sincronización vía WebSession |
| Workflows en grilla | No soporta workflows visuales dentro de la grilla | Tableros con estados y transiciones |

#### Exports
| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| Export a PDF | No genera export a PDF nativo | Se requiere implementación manual |

### PXParameterRequest

| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| Wizard visual con stepper | Se pueden implementar formularios multi-paso usando un PXWorkWith View con Tabs (cada tab = un paso, con visibilidad condicional y navegación libre entre pasos completados). Lo que **no existe aún** es: (1) un componente visual tipo "stepper" con números de paso y estado de avance, y (2) validaciones declarativas que impidan avanzar si faltan datos en pasos previos. Ambas funcionalidades se planean implementar | Para wizards con indicador visual de progreso se requiere implementación manual del stepper |
| Upload de archivos integrado | El upload de archivos requiere implementación con @FileStorage | Formularios con carga de archivos |
| Validación asíncrona | No soporta validación de campos en tiempo real (on-blur) | Formularios que validan mientras el usuario escribe |
| Layout completamente custom | El layout del formulario sigue una estructura predefinida | Formularios con diseño muy específico |

### PXComposer

| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| Layout dinámico | La disposición de componentes es fija en design-time | Dashboards configurables por el usuario |
| Responsive automático de componentes | Cada componente tiene su propio responsive, pero la composición puede no adaptarse bien | Dashboards complejos en móvil |

### PXFlowController

| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| UI de progreso visual | No genera una barra de progreso o indicador de paso actual | Flujos largos donde el usuario necesita ver en qué paso está |
| Persistencia de estado | El estado del flujo se pierde si se cierra el browser | Flujos que deben poder resumirse |
| Flujos paralelos | Solo soporta flujo secuencial (una línea activa) | Flujos con ramas paralelas |
| Undo/rollback | No hay mecanismo de undo de pasos completados | Flujos que necesitan retroceder con rollback de datos |

### Patterns WS (PXWSLayer, PXWSQuery, PXWSData, PXWSTransaction)

| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| GraphQL | Solo genera REST y SOAP, no GraphQL | APIs modernas que requieren GraphQL |
| WebSockets | No soporta comunicación bidireccional | APIs que necesitan push/streaming |
| Batch operations | No genera endpoints de operaciones batch (salvo PXWSTransaction) | APIs que necesitan crear/actualizar múltiples registros en una llamada |
| Custom response codes | Los códigos de respuesta HTTP son fijos | APIs que necesitan respuestas HTTP específicas |
| Rate limiting | No genera rate limiting a nivel de API | APIs públicas que necesitan control de tasa |

### PXOAV

| Limitación | Descripción | Impacto |
|-----------|-------------|---------|
| Atributos calculados complejos | Las fórmulas son limitadas; no soportan joins complejos | Atributos dinámicos que dependen de múltiples tablas |
| Performance con muchos atributos | El modelo EAV tiene overhead de performance vs. columnas nativas | Entidades con >100 atributos dinámicos |
| Indexación | Los valores EAV no se indexan eficientemente | Consultas frecuentes sobre atributos dinámicos |

---

## Limitaciones transversales

### Generación de código
| Limitación | Descripción |
|-----------|-------------|
| Generador compilado | El código del generador (DLL) no es modificable por el usuario. Esto es por diseño: protege la propiedad intelectual y el licenciamiento del producto |
| Extensión del generador | No se pueden agregar nuevos tipos de nodos o propiedades sin modificar la DLL del generador |
| Post-procesamiento | No hay hooks post-generación para modificar el código generado |

**Nota importante**: El diseño visual SÍ es completamente personalizable mediante **Templates de UI** (objetos GeneXus que el desarrollador crea con total libertad). Los Templates definen el layout completo de cada pantalla y solo dejan "Template Elements" como placeholders donde el generador inyecta el contenido específico. Ver [00-overview.md](00-overview.md#templates-de-ui--personalización-total-del-diseño-visual).

### Integración
| Limitación | Descripción |
|-----------|-------------|
| CI/CD | No hay integración nativa con pipelines de CI/CD |
| Testing | No genera tests unitarios automáticos |
| Versionado de instancias | Las instancias se versionan con la KB, no tienen versionado independiente |

---

## Qué se necesitaría implementar en el generador

### Alta prioridad (funcionalidad frecuentemente solicitada)
1. **Grillas tipo card/responsive** — Template de grilla con layout flexible
2. **Wizard multi-paso** — Extensión de PXParameterRequest o nuevo pattern
3. **Export a PDF** — Agregar modo de export PDF en PXWorkWith

### Media prioridad
5. **Indicador de progreso en flujos** — UI de stepper/progress bar en PXFlowController
6. **Validación asíncrona** — Validación on-blur en formularios
7. **Batch API operations** — Endpoint de operaciones múltiples en PXWSTransaction
8. **Dashboard configurable** — Layout dinámico en PXComposer

### Baja prioridad
9. **GraphQL** — Nuevo pattern o extensión de PXWSLayer
10. **Drag-and-drop** — Extensión de PXWorkWith
11. **Tests automáticos** — Generación de tests unitarios

---

## Criterios para determinar si un WebPanel es 100% migrable

Un WebPanel es **100% migrable** si cumple TODOS:

- [ ] Su estructura corresponde a un pattern conocido (ver [30-guia-reconocimiento-patterns.md](30-guia-reconocimiento-patterns.md))
- [ ] No usa controles de terceros no soportados por PXTools
- [ ] No tiene JavaScript custom integrado con la lógica del server
- [ ] La comunicación entre componentes (si la hay) se resuelve con `GlobalEvents`
- [ ] El layout no es altamente personalizado
- [ ] No implementa funcionalidad listada como "limitación" arriba

Un WebPanel **parcialmente migrable** (>70%) se puede migrar y complementar con:
- Código en hooks de eventos (Events, Codes)
- Variables y subrutinas custom
- WebComponents externos embebidos en tabs

Un WebPanel **no migrable** (<30%) requiere mantenerse como WebPanel manual.
