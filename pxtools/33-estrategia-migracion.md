# Estrategia de Migración — De KB sin Patterns a PXTools

## Visión general

Este documento describe el proceso para migrar una **Knowledge Base (KB) de GeneXus desarrollada sin patterns** a una KB que usa PXTools. La migración no es "todo o nada": se puede hacer progresivamente, WebPanel por WebPanel.

## Prerequisitos

1. **PXTools instalado** en la KB destino
2. **Módulos @PXTools necesarios** importados (mínimo: @APIs, @System, @Security)
3. **Patterns habilitados** en GeneXus (extensión de patterns cargada)
4. **Transacciones definidas** (PXWorkWith requiere transacciones)

## Proceso de migración en 6 fases

### Fase 1: Inventario y clasificación

```
┌─────────────────────────────────────────────┐
│         INVENTARIO DE WEBPANELS              │
│                                             │
│  Para cada WebPanel de la KB:               │
│                                             │
│  1. Identificar tipo (ver guía de           │
│     reconocimiento de patterns)             │
│  2. Clasificar:                             │
│     ● 100% migrable a pattern              │
│     ● Parcialmente migrable (>70%)         │
│     ● No migrable                          │
│  3. Asignar pattern destino                │
│  4. Identificar dependencias               │
└─────────────────────────────────────────────┘
```

**Resultado**: Lista clasificada de WebPanels con pattern destino y prioridad.

### Fase 2: Migración de infraestructura

Antes de migrar WebPanels individuales, instalar los módulos @PXTools que proveen infraestructura:

| Funcionalidad actual | Módulo @PXTools | Acción |
|---------------------|-----------------|--------|
| Seguridad custom | @Security | Migrar usuarios/roles/permisos |
| Menús manuales | @Menus o @SmartMenus | Migrar estructura de menú |
| Parámetros del sistema | @SystemParameters | Migrar parámetros |
| Envío de mails | @SendMails + @MailAccounts | Migrar configuración |
| Gestión de archivos | @FileStorage | Migrar almacenamiento |
| Logs | @WebServicesLog | Integrar logging |

### Fase 3: Migración de ABMs simples (PXWorkWith)

Empezar por los WebPanels más simples: **ABMs (Alta-Baja-Modificación)** de tablas maestras.

#### Proceso por cada WebPanel:

**Paso 1 — Crear instancia PXWorkWith**

Asociar a la transacción correspondiente y configurar:

```xml
<instance>
  <level name="MiEntidad">
    <selection>
      <grid>
        <!-- Mapear columnas de la grilla actual -->
        <attributes>
          <attribute name="AtributoId" />
          <attribute name="AtributoNombre" />
          <attribute name="AtributoEstado" />
        </attributes>
      </grid>
      <filter>
        <!-- Mapear filtros existentes -->
        <search>
          <attribute name="AtributoNombre" />
        </search>
      </filter>
      <orders>
        <!-- Mapear ordenamientos existentes -->
        <order name="PorNombre">
          <orderAttribute name="AtributoNombre" ascending="true" />
        </order>
      </orders>
      <actions>
        <!-- Mapear acciones/botones existentes -->
      </actions>
    </selection>
  </level>
</instance>
```

**Paso 2 — Migrar lógica custom a hooks**

La lógica que no es declarativa se migra a los hooks de código:

```xml
<codes>
  <code type="Start">
    <![CDATA[
      // Código de inicialización del WebPanel original
    ]]>
  </code>
  <code type="Refresh">
    <![CDATA[
      // Código del Refresh original
    ]]>
  </code>
  <code type="Load">
    <![CDATA[
      // Código del Load original (cálculos por fila)
    ]]>
  </code>
</codes>
```

**Paso 3 — Generar y comparar**

1. Generar los objetos desde la instancia
2. Comparar visualmente el WebPanel generado vs. el original
3. Ajustar la instancia hasta que coincidan

**Paso 4 — Reemplazar**

1. Actualizar las referencias al WebPanel original para apuntar al generado
2. Desactivar/eliminar el WebPanel manual original

### Fase 4: Migración de formularios (PXParameterRequest)

Migrar popups, diálogos de confirmación y formularios de captura de parámetros.

#### Criterios de mapeo:

| WebPanel actual | behaviour PXParameterRequest |
|----------------|------------------------------|
| Popup con "¿Está seguro?" + Si/No | `PopupParameterRequest` |
| Formulario modal que captura datos | `ParameterRequest` |
| Panel de filtros flotante | `FloatingParameterRequest` |
| Formulario embebido | `Panel` |

### Fase 5: Migración de pantallas compuestas (PXComposer)

Migrar dashboards y pantallas que combinan múltiples componentes.

#### Proceso:

1. Identificar los componentes individuales de la pantalla
2. Verificar que cada componente ya esté migrado a un pattern (PXWorkWith, PXParameterRequest)
3. Crear instancia PXComposer que los compose:

```xml
<instance>
  <level name="Dashboard">
    <forms>
      <form name="Main" platform="Any">
        <components>
          <component type="WebComponent"
                     callType="PXInstance"
                     instanceObject="PXWorkWithFacturas"
                     instanceLevel="Level1"
                     instanceLevelNode="Selection" />
          <component type="WebComponent"
                     callType="PXInstance"
                     instanceObject="PXWorkWithClientes"
                     instanceLevel="Level1"
                     instanceLevelNode="Selection" />
        </components>
      </form>
    </forms>
  </level>
</instance>
```

### Fase 6: Migración de flujos (PXFlowController)

Migrar WebPanels que implementan flujos de trabajo paso a paso.

#### Proceso:

1. Diagramar el flujo actual (pasos, decisiones, confirmaciones)
2. Mapear cada paso a una "línea" del PXFlowController
3. Mapear cada decisión a una acción con nextLine
4. Mapear cada confirmación a un nodo confirm con responses

---

## Migración dual-platform simultánea

Si se está migrando de Desktop a Responsive, aprovechar la migración a patterns para hacerlo en un solo paso:

```
WebPanel manual (Desktop)
    │
    ▼ Migrar a pattern
    │
    ├── generateWeb = True          ← Mantener Desktop
    └── generateWebResponsive = True ← Agregar Responsive
```

Esto es más eficiente que:
1. Migrar a pattern en Desktop
2. Luego habilitar Responsive

## Orden recomendado de migración

```
1. Módulos @PXTools (infraestructura)
   │
2. ABMs simples (tablas maestras)
   │    └── PXWorkWith solo con Selection
   │
3. ABMs con detalle
   │    └── PXWorkWith con Selection + View + Tabs
   │
4. Formularios y popups
   │    └── PXParameterRequest
   │
5. Pantallas compuestas
   │    └── PXComposer (requiere que los componentes ya estén migrados)
   │
6. Flujos de trabajo
   │    └── PXFlowController
   │
7. APIs/WebServices
        └── PXWSLayer + PXWSQuery + PXWSData + PXWSTransaction
```

## Métricas de progreso

| Métrica | Cómo medir |
|---------|-----------|
| % WebPanels migrados | (WP migrados / WP totales) × 100 |
| Cobertura de patterns | (WP con pattern / WP migrables) × 100 |
| Deuda de UI | Cantidad de WP marcados como "no migrables" |
| Dual-platform | % de instancias con generateWebResponsive = True |

## Riesgos y mitigación

| Riesgo | Mitigación |
|--------|-----------|
| Pérdida de funcionalidad | Comparar visualmente antes de reemplazar |
| Performance diferente | Probar con volumen de datos real |
| Lógica no capturada en hooks | Revisar exhaustivamente el código Events del WP original |
| Dependencias rotas | Mapear todas las references antes de eliminar el WP original |
| Resistencia del equipo | Migrar progresivamente, empezando por lo más simple |

## Checklist de migración por WebPanel

- [ ] WebPanel clasificado (100% / parcial / no migrable)
- [ ] Pattern destino identificado
- [ ] Transacción asociada existente
- [ ] Instancia de pattern creada
- [ ] Columnas/campos mapeados
- [ ] Filtros mapeados
- [ ] Ordenamientos mapeados
- [ ] Acciones mapeadas
- [ ] Código custom migrado a hooks
- [ ] Objetos generados
- [ ] Comparación visual OK
- [ ] Tests funcionales OK
- [ ] Referencias actualizadas
- [ ] WebPanel original desactivado
- [ ] generateWebResponsive habilitado (si aplica)
