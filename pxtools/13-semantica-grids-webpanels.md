# Semantica de Grids en WebPanels — Para Reconocimiento de Patterns

Este documento describe **como interpretar la presencia de grids en un WebPanel** para determinar a que pattern de PXTools migrar. La existencia de un grid no siempre implica PXWorkWith — hay matices importantes que aprender.

## Aplica a

- Reconocimiento de patrones para migracion de WebPanels manuales (proceso PuntoExe)
- Decisiones de diseno cuando se modela una nueva pantalla con PXTools
- Analisis de codigo legacy de GeneXus Evolution 1 / 2 / 3

## Tipos de grids segun proposito

### 1. Grid de listado / seleccion (UI principal)

**Proposito**: mostrar al usuario una lista de registros para que vea, filtre, busque, ordene, seleccione o ejecute acciones sobre ellos.

**Senales**:
- Es el elemento visual dominante del form
- Tiene `MaxRows` (paginacion)
- Tiene boton/imagen de busqueda
- Carga registros con un `For Each` sobre BD o iteracion de SDT
- El usuario interactua con el (clic en filas, scroll, paginacion)

**Pattern destino**:
- Si tiene Parm `out:` + Event Enter que retorna -> **PXWorkWith Prompt**
- Si tiene acciones CRUD (Insert/Update/Delete) -> **PXWorkWith Selection**
- Si tiene grilla readonly como visor con navegacion por fila -> **PXWorkWith Selection (visor)**

### 2. Grid auxiliar dentro de un Data Entry

**Proposito**: complementar un formulario tabular con una lista de items (detalle de factura, items seleccionados, errores a mostrar, etc.).

**Senales**:
- El elemento principal del form ES un formulario tabular con campos editables
- El grid es una zona acotada dentro del form, no domina visualmente
- Suele estar acoplado al form: cambios en campos del form afectan el grid o viceversa
- La accion principal (Aceptar/Confirmar) procesa los datos del form Y los del grid

**Pattern destino**: **PXParameterRequest** (con grid auxiliar dentro del nodo `grid`)

Ejemplo: `WnNewFactura` — cabecera de factura como form + detalle de items como grid.

### 3. Grid "fantasma" para perdurar variables SDT (legacy GeneXus)

**Proposito originario**: en versiones antiguas de GeneXus (Evolution 1 y anteriores) la **unica forma de perdurar el estado de un SDT/coleccion entre eventos** dentro del mismo WebPanel era declarar las variables como columnas de un grid, normalmente oculto.

**Senales**:
- El grid esta **oculto al usuario** — usa `Class="Hidden"` o se setea `Grid.Visible = 0` / `Grid.Visible = False` en `Event Start`
- Las "columnas" del grid son variables de tipo SDT, coleccion, o tipos primitivos sin display real
- El usuario no interactua con el — no hay scroll, no hay boton de busqueda, no hay paginacion
- El codigo accede a esas variables como estado del panel (por ejemplo en For Each Line para iterar)

**Importante**: en versiones modernas de GeneXus las variables del panel ya conservan estado entre eventos sin necesidad de un grid. **Un grid fantasma de SDT-persistencia es codigo legacy que NO debe contarse como un grid funcional al clasificar el WebPanel**.

**Implicancia para clasificacion**:
- Un WebPanel con un solo grid fantasma + form de campos editables -> **PXParameterRequest** (NO PXWorkWith Prompt)
- Un WebPanel con varios grids donde algunos son fantasmas -> contar solo los reales para decidir Selection vs auxiliar

### 4. Grid solo para layout / contenedor

**Proposito**: usar la estructura tabular de un grid solo como contenedor visual de otros controles (mucho menos comun).

Estos casos tipicamente no tienen filas de datos. Se ignoran a efectos de clasificacion del pattern.

## Como detectar un grid fantasma

### Senales en el `.gxForm`

```xml
<grid name="GridSDT" class="Hidden" ...>
   <!-- columnas que son variables SDT -->
   <columns>
     <column ...>
       <variable name="MiSDT" ... />
     </column>
   </columns>
</grid>
```

O bien: `class` que en el theme corresponde a una clase con `display:none` / `visibility:hidden`.

### Senales en el `.gxSource`

```genexus
Event Start
    Grid1.Visible = False    // o Visible = 0
EndEvent
```

> **CRITICO — distinguir phantom de visibilidad condicional**: un grid `phantom` (puro SDT-persistence) tiene `.Visible = False` y **NUNCA** `.Visible = True` en el codigo. Si el grid tiene **ambos patrones** (`= False` en algunas condiciones y `= True` en otras), se trata de un grid con **visibilidad condicional** — el usuario lo ve a veces, NO es phantom. Debe contarse como real grid.
>
> **Ejemplo phantom (filtrar)**:
> ```genexus
> Event Start
>     GridSDT.Visible = False
> EndEvent
> ```
>
> **Ejemplo condicional (NO filtrar — es real grid)**:
> ```genexus
> Event Start
>     If &Modo = "Edicion"
>         Documentos.Visible = True
>     Else
>         Documentos.Visible = False
>     EndIf
> EndEvent
> ```
>
> El detector debe strippear comentarios (`//`, `/* */`) antes de buscar los patrones de Visible para no contar codigo comentado.

### Heuristica final

Un grid es fantasma si **TODAS** las siguientes condiciones se cumplen:

1. Esta oculto via `class="Hidden"` (o similar) o `Visible = False`/`= 0` en Start
2. Sus columnas son **variables de panel** (no atributos de transaccion) y la mayoria son SDT/coleccion
3. NO tiene `MaxRows`, paginacion ni boton de busqueda asociado
4. NO hay `For Each` sobre BD que cargue el grid

Si alguna de estas condiciones no se cumple, probablemente sea un grid real.

## Patron For Each Line

`For Each Line` (o `For each line in <Grid>`) es una construccion de GeneXus que itera sobre las filas del grid en codigo. Su presencia tiene implicancias fuertes para la clasificacion.

### Que indica

`For Each Line` se usa cuando el desarrollador necesita **leer/escribir valores de las filas del grid programaticamente**. Tipicamente sucede con grids editables: el usuario edita valores en las filas y el codigo los recolecta en un evento (ej: Aceptar) recorriendo el grid.

### Implicancias para el pattern destino

| Contexto | Pattern recomendado |
|----------|--------------------|
| For Each Line + grid editable + form de campos editables fuera del grid | **PXParameterRequest** con grid auxiliar (data entry mixto) |
| For Each Line + grid editable como elemento principal + accion de guardado | **PXWorkWith Selection** con grid editable y `modes/updateGridRows` |
| For Each Line sobre grid fantasma SDT-persistencia | Ignorar — es legacy. Reclasificar como si el grid no existiera |
| For Each Line sin grid visual (grid fantasma) | El grid es fantasma -> determinar pattern por el resto del form |

### Como detectar For Each Line en el codigo

```genexus
For each line in Grid1
    &Total = &Total + GridColumn1
    If GridColumn2 > 0
        // ...
    EndIf
EndFor
```

O variantes:
- `For each line` (sin nombre de grid)
- `For each line in <NombreGrid>`

## Multiples grids en un WebPanel

### Casos posibles

1. **Multiples grids reales y visibles**: panel complejo con varias secciones de datos. Suele indicar un PXComposer (composicion) o un caso especial de migracion compleja.

2. **Un grid principal + grids auxiliares de datos asociados**: estructura mixta de Selection con grids embebidos. PXComposer puede ser la respuesta, o PXWorkWith con tabs en View.

3. **Un grid real + grids fantasma de SDT-persistencia**: contar SOLO los reales. Si solo queda 1 grid real, es PXWorkWith / PXParameterRequest segun caso.

4. **Todos los grids son fantasma**: es codigo legacy, el pattern se determina por el resto del form (no por los grids).

### Implicancia para complejidad

Multiples grids reales **incrementan el score de complejidad** porque:
- Mas codigo de carga (For Each multiples)
- Mas eventos (uno por grid tipicamente)
- Mas dependencias (cambios en uno pueden afectar otros)

Sin embargo, multiples grids fantasma **NO suben la complejidad real** — son simplemente legacy que se reescribe en hooks modernos.

## Resumen para clasificacion automatica

Al analizar un WebPanel con grids, seguir esta logica:

```
1. Contar todos los <grid> en el .gxForm
2. Para cada grid: determinar si es fantasma (oculto + variables SDT + sin MaxRows + sin search)
3. Calcular grids_reales = total - fantasmas
4. Si grids_reales == 0:
     -> Tratar como WebPanel sin grid (PXParameterRequest puro o Manual)
5. Si grids_reales == 1:
     -> Aplicar criterios estandar (Selection / Prompt / PR con grid aux)
6. Si grids_reales > 1:
     -> Marcar como "multi-grid" para revision
     -> Posibles destinos: PXComposer, PXWorkWith con tabs, o caso complejo
7. Adicional: si hay For Each Line:
     -> Indica grid editable
     -> Combinado con form externo -> PXParameterRequest con grid
     -> Solo grid -> PXWorkWith Selection con grid editable
```

## Indicadores numericos utiles para complejidad

Al analizar el codigo del `.gxSource`, estos contadores son utiles para el score de complejidad:

| Indicador | Significado |
|-----------|-------------|
| Lineas totales en eventos (Event ... EndEvent) | Volumen de logica del panel |
| Cantidad de eventos | Cantidad de puntos de extension |
| Cantidad de subrutinas | Granularidad de la logica |
| Cantidad de For Each (BD) | Loops sobre BD |
| Cantidad de For Each Line (grids) | Iteracion sobre UI |
| Cantidad de llamadas a Procedures | Acoplamiento con logica externa |
| Cantidad de IFs | Complejidad condicional |
| Cantidad de Do Case | Decisiones multi-rama |
| Variables tipo SDT | Manipulacion de estructuras |
| Variables tipo coleccion | Manejo de listas en memoria |
| Referencias a WebSession | Estado entre pantallas |
| Operaciones XML/JSON | Serializacion |
| Cantidad total de variables | Alcance del panel |
| Cantidad de grids (reales) | Complejidad UI |
