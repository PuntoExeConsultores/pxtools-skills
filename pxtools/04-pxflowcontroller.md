# PXFlowController — Pattern de Orquestación de Flujos

## Qué es

PXFlowController genera **WebPanels de flujo de trabajo** que orquestan secuencias de acciones, confirmaciones e iteraciones. Transforma lógica de flujo compleja (que normalmente requiere mucho código manual) en una definición declarativa XML.

## Parent Objects

Se puede asociar a:
- `Procedure` — flujo posterior a un proceso
- `Transaction` — flujo posterior a una transacción
- `(None)` — flujo independiente

## Objetos que genera

De **una sola instancia**, genera:

| Objeto | Tipo GeneXus | Naming | Condición |
|--------|-------------|--------|-----------|
| FlowController (Desktop) | WebPanel | `Ct{Element.name}` | Siempre |
| FlowControllerResponsive | WebPanel | `Ct{Element.name}` | Siempre |

Cada `level` en la instancia genera un par de WebPanels (Desktop + Responsive).

## Estructura XML de la instancia

```xml
<instance>
  <!-- Atributos raíz -->
  <!-- templatesGroup: grupo de templates a aplicar -->

  <level name="MiFlujo" description="Flujo de facturación">
    <!-- Propiedades del level -->
    <!-- name: nombre del flujo -->
    <!-- description: descripción -->
    <!-- masterPage: MasterPage a usar (<default> o referencia) -->
    <!-- theme: Theme a usar (<default> o referencia) -->
    <!-- isGXFlowTask: bool — integración con GXFlow -->
    <!-- generateWeb: enum{<default>;True;False} -->
    <!-- generateWebResponsive: enum{<default>;True;False} -->
    <!-- generateSD: enum{<default>;True;False} -->
    <!-- templateObject: WebPanel template (Desktop) -->
    <!-- templateResponsiveObject: WebPanel template (Responsive) -->
    <!-- templateSDObject: SDPanel template (Smart Devices) -->

    <parameters>
      <parameter name="FacturaId" />
      <parameter name="ClienteId" />
    </parameters>

    <blocksLevels>
      <blocksLevel name="Main">
        <!-- Bloque de líneas: cada línea es un paso del flujo -->
        <linesBlock lineFrom="1">
          <mainCode>
            <![CDATA[
              // Código GeneXus que se ejecuta en la línea 1
              &FacturaTotal = GetFacturaTotal(&FacturaId)
            ]]>
          </mainCode>

          <actions>
            <action name="Confirmar"
                    callType="Link"
                    linkType="PXInstance"
                    instanceObject="PXParameterRequestConfirmarFactura"
                    instanceLevel="Level1"
                    instanceLevelNode="Level"
                    target="New"
                    popupWidth="400"
                    popupHeight="300"
                    actionLine="1"
                    nextLine="2">
              <parameters>
                <parameter name="FacturaId" />
              </parameters>
            </action>

            <action name="Cancelar"
                    callType="Subroutine"
                    subroutine="CancelarFlujo"
                    actionLine="1"
                    nextLine="99">
            </action>
          </actions>

          <confirms>
            <confirm name="ConfirmarEnvio"
                     question="¿Desea enviar la factura?"
                     questionType="Constant"
                     popupWidth="350"
                     popupHeight="200"
                     confirmLine="1"
                     nextLine="3">
              <responses>
                <response responseValue="Si" responseLine="2" />
                <response responseValue="No" responseLine="1" />
              </responses>
            </confirm>
          </confirms>
        </linesBlock>

        <linesBlock lineFrom="2">
          <mainCode><![CDATA[
            // Línea 2: enviar factura
            EnviarFactura(&FacturaId)
          ]]></mainCode>
        </linesBlock>
      </blocksLevel>
    </blocksLevels>

    <codes>
      <code type="Start" data="...código de inicio..." />
      <code type="Refresh" data="...código de refresh..." />
      <code type="Subroutine" name="CancelarFlujo" data="...código..." />
    </codes>

    <variables>
      <variable name="FacturaTotal" basedOn="FacturaTotal" />
    </variables>
  </level>
</instance>
```

## Concepto de Bloques de Líneas

El flujo se organiza en **líneas numeradas**. Cada bloque de líneas (`linesBlock`) define:

```
┌─────────────────────────────────────────────┐
│ Bloque de Líneas (lineFrom=1)               │
│                                             │
│  1. mainCode: código GeneXus a ejecutar     │
│  2. iterationCode: código de iteración      │
│     (opcional, para loops)                  │
│  3. actions: acciones disponibles           │
│     └─ Cada acción tiene nextLine           │
│  4. confirms: confirmaciones                │
│     └─ Cada respuesta tiene responseLine    │
│                                             │
│  El flujo salta a nextLine/responseLine     │
│  según la acción o respuesta elegida        │
└─────────────────────────────────────────────┘
```

### Flujo de ejecución

```
Línea 1 ──► mainCode ──► Actions/Confirms
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
            Acción A   Acción B   Confirm
            nextLine=2 nextLine=3  │
                │          │    ┌──┴──┐
                ▼          ▼    ▼     ▼
             Línea 2   Línea 3 Si    No
                              resp=4 resp=1
                               │      │
                               ▼      ▼
                           Línea 4  Línea 1
                                   (vuelve)
```

## Tipos de acción (callType)

| callType | Descripción | Propiedades clave |
|----------|-------------|-------------------|
| `Link` | Invoca un objeto o instancia | linkType, gxObject/instanceObject, target |
| `External Link` | Link a URL/objeto externo | externalObject |
| `Client Text Print` | Impresión de texto en cliente | gxObject |
| `Subroutine` | Invoca una subrutina local | subroutine |

### linkType para acciones tipo Link

| linkType | Descripción | Propiedad |
|----------|-------------|-----------|
| `GXObject` | Llama directamente a un objeto GeneXus | gxObject (reference) |
| `PXInstance` | Llama a una instancia de PXTools | instanceObject + instanceLevel + instanceLevelNode |

### Patterns invocables vía PXInstance

Una acción con `linkType="PXInstance"` puede invocar:
- **PXWorkWith** (Selection, View, Edit, Prompt)
- **PXParameterRequest** (Level)
- **PXComposer** (Level)
- **PXFlowController** (otro flujo)
- **PXOAV** (gestión de OAV)

### Target de acciones

| target | Comportamiento |
|--------|---------------|
| `Self` | Navega en la misma ventana |
| `New` | Abre en popup/nueva ventana |

Cuando `target="New"`:
- `popupWidth` / `popupHeight` controlan el tamaño
- `closeWindowControl` permite controlar el cierre del popup
- `closeWindowControlCondition` evalúa si el cierre es aceptado
- `hasPostCode` habilita código post-cierre del popup

## Confirmaciones

Las confirmaciones (`confirm`) son diálogos modales con múltiples respuestas posibles:

```xml
<confirm name="ConfirmarAccion"
         question="¿Está seguro?"
         questionType="Constant"    <!-- o Variable -->
         gxObject="WbConfirm"       <!-- WebPanel custom de confirmación (opcional) -->
         responseDomain="MiDominio" <!-- Dominio para las respuestas -->
         confirmLine="1"
         nextLine="2">
  <responses>
    <response responseValue="Aceptar" responseLine="3" />
    <response responseValue="Cancelar" responseLine="1" />
  </responses>
</confirm>
```

- Pueden usar un WebPanel custom (`gxObject`) o el popup estándar de PXTools
- Cada respuesta dirige a una línea diferente del flujo
- `responseLine="1"` en los bloques de respuesta se reserva para validación

## Iteraciones

El nodo `iterationCode` permite crear loops dentro del flujo:

```xml
<linesBlock lineFrom="5">
  <iterationCode iterationLineNext="5">
    <![CDATA[
      // Este código se ejecuta en cada iteración
      // Si iterationLineNext=5 (la misma línea), itera
      For each Factura where FacturaEstado = "Pendiente"
    ]]>
  </iterationCode>
  <mainCode>
    <![CDATA[
      // Código principal de la iteración
      ProcesarFactura(FacturaId)
    ]]>
  </mainCode>
</linesBlock>
```

## Hooks de código (Codes)

| type | Cuándo se ejecuta |
|------|-------------------|
| `Start` | Al inicio del WebPanel (evento Start) |
| `Refresh` | En el evento Refresh |
| `Load` | En el evento Load |
| `Subroutine` | Subrutina invocable por nombre |

## Variables

Las variables del flujo se declaran con definición completa:

```xml
<variable name="Total"
          basedOn="FacturaTotal"    <!-- basado en atributo -->
          domain="Numeric10_2"      <!-- o basado en dominio -->
          SDT="MiSDT"              <!-- o basado en SDT -->
          dataType="Numeric"        <!-- o tipo primitivo -->
          length="10"
          decimals="2"
          collection="false"
          dimension="Scalar" />
```

## Capacidad dual-platform

Cada level tiene propiedades `generateWeb`, `generateWebResponsive` y `generateSD`. El .Pattern define dos Object entries para cada level:

- `FlowController` → genera WebPanel con template `FlowControllerWebForm.dll` (HTML)
- `FlowControllerResponsive` → genera WebPanel con template `FlowControllerAbstractForm.dll` (abstracto)

## Ejemplo: alta continua con confirmación (PXFlowControllerAltaContinua)

Esta instancia orquesta el **alta continua de registros**: tras crear uno, pregunta si se desea crear otro del mismo tipo y encadena la navegación según la respuesta.

```xml
<?xml version="1.0" encoding="utf-16"?>
<instance>
  <level name="AltaContinua" generateWebResponsive="True">
    <blocksLevels>
      <blocksLevel>
        <!-- Línea 1: ¿Crear otro registro del mismo tipo? -->
        <linesBlock lineFrom="1">
          <mainCode><![CDATA[
            &SDTAltaContinua = RetSDTAltaContinua.Udp()
            If not &SDTAltaContinua.Habilitada
              ControllerGotoLine 4          // Ir directamente al Selection
            Else
              ControllerConfirm Reingresar  // Preguntar al usuario
            EndIf
          ]]></mainCode>

          <confirms>
            <confirm name="Reingresar"
                     gxObject="PuConfirm, PXTools.APIs"
                     responseDomain="Boolean"
                     question="¿Desea crear un nuevo registro del mismo tipo?"
                     nextLine="1">
              <responses>
                <response responseValue="True" responseLine="2" />   <!-- Sí: crear nuevo -->
                <response responseValue="False" responseLine="3" />  <!-- No: ir al Selection -->
              </responses>

              <!-- Línea 2: Crear nuevo registro y navegar al View -->
              <linesBlock lineFrom="2">
                <mainCode><![CDATA[
                  AddPedido.Call(&Context.SecurityUserTenantId, 0,
                    &SDTAltaContinua.Tipo, ...)
                  For Each
                    Where TenantId = &Context.SecurityUserTenantId
                    Where TipoId = &SDTAltaContinua.Tipo
                    If TipoSaltaDatosCabecera
                      ControllerAction IrAlViewDetalle
                    Else
                      ControllerAction IrAlViewCabecera
                    EndIf
                  EndFor
                ]]></mainCode>
                <actions>
                  <!-- Acción que invoca PXWorkWith View, sección Cabecera -->
                  <action name="IrAlViewCabecera"
                          instanceObject="PXWorkWithPedidos, MiApp"
                          instanceLevel="Pedido"
                          instanceLevelNode="View"
                          instanceLevelViewSection="Cabecera"
                          nextLine="999">
                    <parameters>
                      <parameter name="TrnMode.Insert" />
                      <parameter name="0" />
                      <parameter name="&EstadoVacio" />
                    </parameters>
                  </action>
                  <!-- Acción que invoca PXWorkWith View, sección Detalle -->
                  <action name="IrAlViewDetalle"
                          instanceObject="PXWorkWithPedidos, MiApp"
                          instanceLevel="Pedido"
                          instanceLevelNode="View"
                          instanceLevelViewSection="Detalle"
                          nextLine="999">
                  </action>
                </actions>
              </linesBlock>

              <!-- Línea 3: No reingresar, ir al Selection -->
              <linesBlock lineFrom="3">
                <mainCode><![CDATA[
                  DelSDTAltaContinua.Call()
                  ControllerAction IrAlSelection
                ]]></mainCode>
                <actions>
                  <action name="IrAlSelection"
                          instanceObject="PXWorkWithPedidos, MiApp"
                          instanceLevel="Pedido"
                          instanceLevelNode="Selection"
                          nextLine="999" />
                </actions>
              </linesBlock>
            </confirm>
          </confirms>
        </linesBlock>

        <!-- Línea 4: Ir al Selection (cuando el alta continua no está habilitada) -->
        <linesBlock lineFrom="4">
          <mainCode><![CDATA[ControllerAction IrAlSelection]]></mainCode>
          <actions>
            <action name="IrAlSelection"
                    instanceObject="PXWorkWithPedidos, MiApp"
                    instanceLevel="Pedido"
                    instanceLevelNode="Selection"
                    nextLine="1" />
          </actions>
        </linesBlock>
      </blocksLevel>
    </blocksLevels>
    <variables>
      <variable name="SDTAltaContinua" SDT="SDTAltaContinua, MiApp" />
      <variable name="EstadoVacio" basedOn="PedidoEstado" />
      <variable name="TipoVacio" basedOn="PedidoTipo" />
    </variables>
  </level>
</instance>
```

### Diagrama del flujo

```
Línea 1: ¿Alta continua habilitada?
    │
    ├── NO ──► Línea 4 ──► IrAlSelection (PXWorkWithPedidos)
    │
    └── SÍ ──► Confirm "¿Crear un nuevo registro del mismo tipo?"
                 │
                 ├── True (Línea 2) ──► Crear registro ──►
                 │     ├── Si salta cabecera ──► View.Detalle
                 │     └── Si no ──► View.Cabecera
                 │
                 └── False (Línea 3) ──► Eliminar estado ──► IrAlSelection
```

**Puntos clave del ejemplo:**
- Usa `ControllerGotoLine` para saltar líneas sin acción del usuario
- Usa `ControllerConfirm` para mostrar diálogo de confirmación
- Usa `ControllerAction` para ejecutar una acción definida
- Las acciones referencian PXWorkWith con `instanceLevelViewSection` para navegar a un tab específico
- `generateWebResponsive="True"` genera versiones Desktop y Responsive

## Relación con otros patterns

```
PXFlowController
├── Puede invocar ──► PXWorkWith (Selection/View/Prompt)
├── Puede invocar ──► PXParameterRequest (confirmaciones, captura de datos)
├── Puede invocar ──► PXComposer (pantallas compuestas)
├── Puede invocar ──► PXOAV (gestión de atributos dinámicos)
├── Puede invocar ──► Otro PXFlowController (flujos anidados)
└── Puede invocar ──► Cualquier objeto GeneXus (Procedures, WebPanels)
```
