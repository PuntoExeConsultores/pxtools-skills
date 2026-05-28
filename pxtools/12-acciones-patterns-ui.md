# Acciones en Patterns de UI — Referencia Transversal

## Aplica a

Este documento describe el **sistema de acciones** compartido por los tres patterns de UI de PXTools:

- **PXWorkWith** — acciones en Selection, View, Prompt, Tabs
- **PXParameterRequest** — acciones en el formulario (Aceptar, Cancelar, custom)
- **PXComposer** — acciones en el panel compuesto

La estructura del nodo `action` es **practicamente identica** en los tres patterns. Las diferencias especificas se documentan donde aplican.

---

## 1. Estructura XML del nodo Action

```xml
<action name="MiAccion"
        caption="Texto visible"
        tooltip="Tooltip opcional"
        image="ImagenBoton"
        type="Standard|Custom"
        position="Top|Bottom|Both|Grid"

        conditionPreviousCode="// Codigo previo para evaluar condicion"
        condition="&EsValido = true"
        evaluateCondition="Event|Load"

        actionPreviousCode="// Codigo ANTES de la invocacion principal"

        callType="Link|Call|Prompt|External Link|Client Text Print|Subroutine|Event|Submit|None|Return"
        linkType="GXObject|PXInstance"
        gxObject="NombreObjeto"
        target="Self|New"
        popupWidth="400"
        popupHeight="300"

        actionPostCode="// Codigo DESPUES de la invocacion principal"
        refreshAction="..."

        confirm="¿Esta seguro?"
        closeWindowControl="..."
        closeWindowControlCondition="..."
        hasPostCode="true|false">

  <!-- Parametros del objeto invocado -->
  <parameters>
    <parameter name="Param1" />
    <parameter name="Param2" />
  </parameters>

  <!-- Seguridad -->
  <security object="MiObjeto" operation="Execute" />

  <!-- Invocaciones condicionales (ver seccion ConditionalCalls) -->
  <conditionalCalls>
    <conditionalCall condition="..." gxObject="..." />
  </conditionalCalls>
</action>
```

---

## 2. Propiedades del Action

### Identificacion y presentacion

| Propiedad | Tipo | Descripcion |
|-----------|------|-------------|
| `name` | string | Identificador unico de la accion |
| `caption` | string | Texto visible en el boton/link |
| `tooltip` | string | Tooltip al pasar el mouse |
| `image` | reference | Imagen del boton |
| `type` | enum | `Standard` (Aceptar/Cancelar) o `Custom` |
| `position` | enum | `Top`, `Bottom`, `Both`, `Grid` (dentro de cada fila) |

### Condicion de visibilidad/ejecucion

| Propiedad | Tipo | Descripcion |
|-----------|------|-------------|
| `conditionPreviousCode` | code | Codigo procedural que se ejecuta para preparar la evaluacion de la condicion. Permite calcular variables auxiliares necesarias para la condicion |
| `condition` | expression | Expresion GeneXus que determina si la accion se muestra/ejecuta. Si evalua `false`, la accion no se dispara |
| `evaluateCondition` | enum | `Event`: evalua la condicion al hacer clic. `Load`: evalua por cada fila de la grilla (permite visibilidad condicional por fila) |

### Codigo pre-invocacion

| Propiedad | Tipo | Descripcion |
|-----------|------|-------------|
| `actionPreviousCode` | code | Codigo procedural que se ejecuta **antes** de la invocacion principal. Uso tipico: validaciones adicionales, preparacion de datos, calculos previos |

### Invocacion principal

| Propiedad | Tipo | Descripcion |
|-----------|------|-------------|
| `callType` | enum | Tipo de invocacion (ver tabla detallada abajo) |
| `linkType` | enum | Solo para `callType="Link"`: `GXObject` o `PXInstance` |
| `gxObject` | reference | Objeto GeneXus destino (para GXObject) |
| `target` | enum | `Self` = misma ventana, `New` = popup/ventana nueva |
| `popupWidth` | int | Ancho del popup (solo si target=New) |
| `popupHeight` | int | Alto del popup (solo si target=New) |

### Propiedades PXInstance (para callType Link + linkType PXInstance)

| Propiedad | Tipo | Descripcion |
|-----------|------|-------------|
| `instanceObject` | reference | Nombre de la instancia de pattern destino |
| `instanceLevel` | string | Nombre del level dentro de la instancia |
| `instanceLevelNode` | enum | Nodo a invocar: `Selection`, `View`, `Prompt`, `Transaction`, `Level` |
| `instanceLevelViewSection` | string | (Solo View) Tab/seccion especifica. Si se omite, va al tab principal |

### Codigo post-invocacion

| Propiedad | Tipo | Descripcion |
|-----------|------|-------------|
| `actionPostCode` | code | Codigo procedural que se ejecuta **despues** de la invocacion principal. Uso tipico: actualizar estados, guardar en WebSession, mensajes de confirmacion |
| `refreshAction` | enum | Como refrescar la pantalla/grilla despues de la accion |

### Otras propiedades

| Propiedad | Tipo | Descripcion |
|-----------|------|-------------|
| `confirm` | string/bool | Dialogo de confirmacion antes de ejecutar |
| `closeWindowControl` | string | Control para cierre del popup |
| `closeWindowControlCondition` | expression | Condicion para aceptar el cierre |
| `hasPostCode` | bool | Habilita codigo post-cierre del popup |
| `security` | subnodo | Control de acceso: `object` + `operation` |

---

## 3. Tipos de callType

| callType | Descripcion | Uso tipico |
|----------|-------------|------------|
| `Link` | Navega a otro WebPanel/Transaction | Abrir pantalla de detalle, edicion, otro pattern |
| `Call` | Invoca un Procedure (sin interfaz) | Contabilizar, procesar, actualizar registros |
| `Prompt` | Abre dialogo modal (popup) | Abrir un PXParameterRequest o lookup |
| `External Link` | Abre WebPanel/URL no generado por PXTools | Abrir WebPanel manual o URL externa |
| `Client Text Print` | Impresion desde el cliente | Imprimir comprobante |
| `Subroutine` | Ejecuta subrutina local | Logica interna reutilizable definida en `codes/Subroutine` |
| `Event` | Ejecuta codigo inline (sin objeto principal) | Cuando toda la logica es codigo procedural sin invocacion externa |
| `Submit` | Somete un proceso batch | Ejecutar tarea asincrona |
| `None` | No ejecuta nada | Solo importa el codigo pre/post |
| `Return` | Retorna/cierra popup | Boton Cancelar, Salir |

---

## 4. Regla critica: linkType y migracion Responsive

**Solo las acciones con `callType="Link"` necesitan `linkType="PXInstance"`** para ser independientes de la plataforma.

Cuando se genera para Responsive, los nombres de objetos generados por patterns cambian de prefijo:

```
Desktop:    WWFactura,  ViewFactura,  PrFactura,  TrFactura
Responsive: RWWFactura, RViewFactura, RPrFactura, RFactura
```

Si una accion usa `linkType="GXObject"` con `gxObject="TrFactura"`, al habilitar Responsive el nombre cambia y la referencia se rompe.

Con `linkType="PXInstance"`, el generador resuelve automaticamente el nombre correcto segun la plataforma:

```xml
<!-- INCORRECTO (se rompe al cambiar de plataforma) -->
<action callType="Link" linkType="GXObject" gxObject="TrFactura" />

<!-- CORRECTO (independiente de plataforma) -->
<action callType="Link" linkType="PXInstance"
        instanceObject="PXWorkWithFactura"
        instanceLevel="Factura"
        instanceLevelNode="Transaction" />
```

**Los demas callTypes NO necesitan PXInstance** porque invocan Procedures, subrutinas o codigo inline cuyos nombres no cambian entre plataformas:

```xml
<!-- OK: Call invoca un Procedure (nombre no cambia) -->
<action callType="Call" gxObject="Contabilizar" />

<!-- OK: Event ejecuta codigo inline (no depende de nombres) -->
<action callType="Event" />

<!-- OK: Subroutine es local al objeto (no depende de nombres) -->
<action callType="Subroutine" subroutine="ValidarDatos" />
```

---

## 5. ConditionalCalls — Invocacion condicional

ConditionalCalls permite que una misma accion invoque **distintos objetos/interfaces segun una condicion**, funcionando como un `Do Case` declarativo para navegacion:

```xml
<action name="VerDetalle"
        caption="Ver Detalle"
        callType="Link">
  <conditionalCalls>
    <conditionalCall condition="FacturaTipo = 'Nacional'"
                     gxObject="ViewFacturaNacional"
                     linkType="PXInstance"
                     instanceObject="PXWorkWithFacturaNacional"
                     instanceLevel="Factura"
                     instanceLevelNode="View" />
    <conditionalCall condition="FacturaTipo = 'Exportacion'"
                     gxObject="ViewFacturaExportacion"
                     linkType="PXInstance"
                     instanceObject="PXWorkWithFacturaExportacion"
                     instanceLevel="Factura"
                     instanceLevelNode="View" />
  </conditionalCalls>
</action>
```

### Comportamiento

1. Al hacer clic en la accion, se evaluan las condiciones de cada `conditionalCall` en orden
2. La primera condicion que sea `true` determina el objeto/instancia a invocar
3. Si ninguna condicion es `true`, se usa el `gxObject` del action padre (si existe)

### Casos de uso

| Escenario | Ejemplo |
|-----------|---------|
| Diferente pantalla de detalle segun tipo de registro | Ver factura nacional vs exportacion |
| Diferente formulario de edicion segun estado | Editar borrador vs editar aprobado |
| Diferente flujo segun rol del usuario | Flujo administrador vs flujo operador |
| Diferentes vistas segun configuracion de empresa | Vista con NIIF vs vista Local |

### ConditionalCalls y PXInstance

Cada `conditionalCall` soporta las mismas propiedades de `linkType`/`instanceObject` que el action padre. Para migracion Responsive, cada conditionalCall debe usar `linkType="PXInstance"` si apunta a un objeto generado por pattern.

---

## 6. Ciclo de ejecucion completo

```
┌─────────────────────────────────────────────────────────────┐
│                   CICLO DE UNA ACCION                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. conditionPreviousCode                                   │
│     └─ Codigo para preparar variables de condicion          │
│                                                             │
│  2. condition                                               │
│     └─ Si es false → la accion NO se ejecuta (fin)          │
│                                                             │
│  3. confirm (si esta definido)                              │
│     └─ Muestra dialogo "¿Esta seguro?" → Si/No             │
│        Si No → fin                                          │
│                                                             │
│  4. actionPreviousCode                                      │
│     └─ Validaciones, preparacion de datos                   │
│                                                             │
│  5. INVOCACION PRINCIPAL (segun callType)                    │
│     ├─ Link → Navega a objeto/instancia                     │
│     │   └─ ConditionalCalls: evalua condiciones, elige dest │
│     ├─ Call → Invoca Procedure                              │
│     ├─ Event → Ejecuta codigo inline                        │
│     ├─ Prompt → Abre popup modal                            │
│     ├─ Subroutine → Ejecuta sub local                       │
│     ├─ Submit → Somete batch                                │
│     ├─ Return → Cierra/retorna                              │
│     └─ None → No hace nada                                  │
│                                                             │
│  6. actionPostCode                                          │
│     └─ Logica posterior (estados, WebSession, mensajes)      │
│                                                             │
│  7. refreshAction                                           │
│     └─ Refresco de pantalla/grilla si corresponde           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Diferencias por pattern

### PXWorkWith

- Las acciones pueden estar en `selection/actions`, `view/actions`, `prompt/actions` y en cada `section/actions` (tabs)
- Cuando `position="Grid"`, la accion se muestra por cada fila y la condicion se evalua en el Load
- Soporta acciones estandar predefinidas: `Insert`, `Update`, `Delete`, `Display`, `ExportExcel`

### PXParameterRequest

- Las acciones estan en `level/actions`
- Tiene la propiedad adicional **`execute`** que controla que validaciones se aplican antes de ejecutar la accion (ver seccion 8)
- Los tipos estandar son `Confirm` (Aceptar) y `Cancel` (Cancelar)
- Las acciones interactuan con el nodo `calls` para encadenar invocaciones al confirmar

### PXComposer

- Las acciones estan en `level/actions`
- Funcionan igual que en PXWorkWith pero sin contexto de grilla (no hay evaluacion por fila)
- Tipicamente son acciones de navegacion global del panel compuesto

---

## 8. Propiedad Execute en PXParameterRequest

PXParameterRequest tiene una propiedad exclusiva **`execute`** en las acciones que controla **que validaciones se aplican** antes de ejecutar la accion. Esto existe porque PXParameterRequest tiene una funcion principal de **data entry** con validaciones que deben ejecutarse antes de procesar los datos.

### Contexto: nodo Conditions del PXParameterRequest

El nodo `conditions` del PXParameterRequest define **condiciones de validacion generales** que son comunes a multiples acciones:

```xml
<conditions>
  <condition>not &FechaDesde.IsEmpty()</condition>
  <condition>not &FechaHasta.IsEmpty()</condition>
  <condition>&FechaDesde <= &FechaHasta</condition>
</conditions>
```

Estas condiciones representan **reglas de validacion de datos obligatorios** que aplican independientemente de que accion se dispare (con excepcion del Cancel).

### Valores de execute

| Valor | Comportamiento |
|-------|---------------|
| **General Conditions** | Ejecuta las `conditions` generales del PXParameterRequest **y** las condiciones locales de la accion (`condition`). Uso: para acciones que requieren que todos los datos obligatorios esten completos (ej: Aceptar, Confirmar, Generar) |
| **Action Conditions Only** | Ejecuta **solo** las condiciones locales de la accion (`condition`, `conditionPreviousCode`), ignorando las `conditions` generales. Uso: para acciones que no requieren validacion de todos los campos (ej: Vista Previa parcial, Buscar, acciones auxiliares) |

### Ejemplo

```xml
<!-- Condiciones generales del formulario (datos obligatorios) -->
<conditions>
  <condition>not &FechaDesde.IsEmpty()</condition>
  <condition>not &ClienteId.IsEmpty()</condition>
</conditions>

<actions>
  <!-- Aceptar: valida condiciones generales + locales -->
  <action name="Confirm" caption="Aceptar"
          execute="General Conditions"
          callType="Call" gxObject="GenerarReporte" />

  <!-- Vista previa: solo valida su propia condicion -->
  <action name="Preview" caption="Vista previa"
          execute="Action Conditions Only"
          condition="&TipoReporte = 'PDF'"
          callType="Link" gxObject="PreviewReporte" />

  <!-- Cancelar: no necesita validar nada -->
  <action name="Cancel" caption="Cancelar"
          callType="Return" />
</actions>
```

En este ejemplo:
- **Aceptar** exige que FechaDesde y ClienteId esten llenos (condiciones generales)
- **Vista previa** solo verifica que el tipo de reporte sea PDF (condicion local), sin importar si los demas campos estan llenos
- **Cancelar** no ejecuta ninguna validacion

---

## 9. Mapeo de codigo manual a propiedades de Action

Guia para migrar eventos de WebPanels manuales a acciones de patterns:

| Codigo en WebPanel manual | Propiedad del Action |
|---------------------------|---------------------|
| Validaciones al inicio del Event (If campo.IsEmpty(), Msg, Return) | `conditionPreviousCode` + `condition`, o `execute="General Conditions"` con nodo `conditions` |
| `Do 'MiSubrutina'` antes de la invocacion | `actionPreviousCode` |
| `MiProcedimiento.Call(...)` | `callType="Call"`, `gxObject="MiProcedimiento"` |
| `MiWebPanel.Link(...)` o `Link(MiWebPanel, ...)` | `callType="Link"`, `linkType="PXInstance"` (si es pattern) o `linkType="GXObject"` |
| `Do Case` con diferentes Link segun condicion | `conditionalCalls` con multiples `conditionalCall` |
| Logica despues de la invocacion (If resultado, WebSession.Set, etc.) | `actionPostCode` |
| `Return` al final del evento | Implicito en el ciclo de la accion |
| Solo codigo sin invocacion a objeto | `callType="Event"` (todo va como codigo inline) |
| Solo retorno sin logica | `callType="Return"` |
