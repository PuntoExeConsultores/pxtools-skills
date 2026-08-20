# Capacidades Dual-Platform — Migración Desktop → Responsiva

## Visión general

PXTools permite generar **simultáneamente** dos versiones de cada WebPanel:
1. **Web Desktop** — Layout HTML clásico (tablas, divs, posicionamiento absoluto)
2. **Web Responsive** — Layout Abstracto de GeneXus (responsive, mobile-first)

Y opcionalmente:
3. **Smart Devices** — Layout para aplicaciones móviles nativas

Esto se logra desde **una única instancia de pattern** (.gxPattern), sin duplicar la definición.

## Mecanismo de generación dual

### En el .Pattern

Cada objeto generado tiene dos versiones:

```xml
<!-- Versión Desktop: usa WebForm template (HTML layout) -->
<Object Type="WebPanel" Id="Selection" Name="WW{Instance.Name}"
        Element="instance/level/selection">
  <Part Type="WebForm" Template="Templates\SelectionWebForm.dll" />
  <!-- ... mismas Variables, Events, Rules, Conditions -->
</Object>

<!-- Versión Responsive: usa AbstractForm template (layout abstracto) -->
<Object Type="WebPanel" Id="SelectionResponsive" Name="RWW{Instance.Name}"
        Element="instance/level/selection">
  <Part Type="WebForm" Template="Templates\SelectionAbstractForm.dll" />
  <!-- ... mismas Variables, Events, Rules, Conditions -->
</Object>
```

**Punto clave**: Ambas versiones comparten exactamente la misma lógica (Variables, Events, Rules, Conditions). Solo el **generador de Form** (DLL) es diferente: uno produce layout HTML (Desktop) y otro layout abstracto (Responsive). Además, cada versión puede usar un **Template de UI** distinto (`templateObject` para Desktop, `templateResponsiveObject` para Responsive) permitiendo diseños visuales diferentes por plataforma.

### Prefijos de nomenclatura

| Pattern | Desktop | Responsive |
|---------|---------|------------|
| PXWorkWith Selection | `WW{Name}` | `RWW{Name}` |
| PXWorkWith View | `View{Name}` | `RView{Name}` |
| PXWorkWith Prompt | `Pr{Name}` | `RPr{Name}` |
| PXWorkWith Controller | `Ct{Name}` | `RCt{Name}` |
| PXWorkWith Transaction | `D{Name}` | `R{Name}` |
| PXComposer | `Wb{Name}` | `RWb{Name}` |
| PXParameterRequest | `Wb{Name}` | `Wb{Name}` * |
| PXFlowController | `Ct{Name}` | `Ct{Name}` * |

\* PXParameterRequest y PXFlowController usan el **mismo nombre** para ambas versiones.

## Propiedades de control

Cada level/nodo de la instancia tiene 3 propiedades de generación:

```xml
<level name="Factura"
       generateWeb="True"              <!-- Generar versión Desktop -->
       generateWebResponsive="True"    <!-- Generar versión Responsive -->
       generateSD="False">             <!-- Generar versión Smart Devices -->
```

Valores posibles:
- `<default>` — usa el valor definido en Settings
- `True` — genera esta versión
- `False` — no genera esta versión

### Platform (en PXParameterRequest)

PXParameterRequest tiene una propiedad adicional `platform`:

```xml
<level platform="Any">          <!-- Genera para todas las plataformas -->
<level platform="Web Desktop">  <!-- Solo genera para Desktop -->
```

### Platform en forms (en PXComposer)

PXComposer permite definir **forms diferentes por plataforma**:

```xml
<forms>
  <form name="Desktop" platform="Web Desktop">
    <components>
      <!-- Layout específico Desktop -->
    </components>
  </form>
  <form name="Responsive" platform="Web Responsive">
    <components>
      <!-- Layout diferente para Responsive -->
    </components>
  </form>
</forms>
```

## Estrategia de migración progresiva

### Fase 1: Convivencia
```
Instancia PXWorkWith "Factura"
├── generateWeb = True           ← Desktop sigue funcionando
├── generateWebResponsive = True ← Se genera la versión Responsive en paralelo
│
├── WW Factura (Desktop)         ← Usuarios actuales siguen usando esto
└── RWWFactura (Responsive)      ← Se puede probar y validar
```

### Fase 2: Transición
- Redirigir usuarios a las versiones Responsive
- Validar que toda la funcionalidad está cubierta
- Ajustar templates/themes responsive si es necesario

### Fase 3: Apagado Desktop
```
Instancia PXWorkWith "Factura"
├── generateWeb = False          ← Se desactiva Desktop
├── generateWebResponsive = True ← Solo Responsive
│
└── RWWFactura (Responsive)      ← Única versión
```

## Ventajas de la generación dual

1. **Cero duplicación de lógica**: Variables, Events, Rules, Conditions son los mismos
2. **Migración sin riesgo**: ambas versiones coexisten simultáneamente
3. **Rollback instantáneo**: si algo falla en Responsive, Desktop sigue disponible
4. **Misma fuente de verdad**: un solo .gxPattern controla ambas versiones
5. **Validación A/B**: se puede comparar el comportamiento de ambas versiones

## Qué cambia entre versiones

| Aspecto | Desktop | Responsive |
|---------|---------|------------|
| Layout | HTML (tablas, posicionamiento absoluto) | Abstracto (responsive, flexbox) |
| MasterPage | MasterPage Desktop | MasterPage Responsive |
| Theme | Theme Desktop | Theme Responsive |
| Controles | Controles web estándar | Controles responsive |
| Tamaño | Fijo | Adaptable a pantalla |
| Touch | No | Sí |

## Qué NO cambia entre versiones

- Reglas de negocio (Rules)
- Eventos (Events code)
- Variables
- Condiciones de filtro
- Órdenes
- Acciones (structure)
- Parámetros
- Seguridad

## ResponsiveLayout — Control del posicionamiento responsive

Para el generador Responsive, PXTools incorpora la propiedad **ResponsiveLayout** que utiliza el mismo componente nativo de GeneXus para definir cómo se posicionan los subelementos de una sección según el tamaño de pantalla. Esto permite:

- Definir **breakpoints** (puntos de quiebre) por tamaño de pantalla
- Configurar la **disposición de elementos** para cada breakpoint (horizontal, vertical, columnas)
- Adaptar la visualización de filtros, campos de formulario, acciones y cualquier subelemento
- Todo declarativamente desde la instancia del pattern

Combinado con los **Templates de UI** (que dan libertad total para el diseño visual), el layout responsive es completamente personalizable sin necesidad de código CSS manual.

## Consideraciones de la generación dual

- Controles custom de terceros pueden no tener equivalente responsive
- UserControls específicos de Desktop no funcionan en layout abstracto
- El MasterPage y Theme deben existir en ambas versiones
