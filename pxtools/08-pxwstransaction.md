# PXWSTransaction — Pattern CRUD de Transacciones vía WebService

## Qué es

PXWSTransaction genera **Procedures de Load, Save y Delete** sobre transacciones GeneXus usando **Business Components**. Es el pattern para exponer operaciones CRUD (Create, Read, Update, Delete) de entidades como servicios web.

## Parent Objects

- `Transaction` — la transacción sobre la cual se generan las operaciones CRUD
- `(None)` — independiente

## Objetos que genera

Por cada `version`:

| Objeto | Tipo GeneXus | Naming | Condición |
|--------|-------------|--------|-----------|
| SDTStructure | SDT | `WSTransaction{Trn}V{ver}Structure` | Siempre |
| MethodLoad | Procedure | `WSTransaction{Trn}V{ver}Load` | Siempre |
| SDTLoadIn | SDT | `WSTransaction{Trn}V{ver}LoadIn` | Siempre |
| SDTLoadOut | SDT | `WSTransaction{Trn}V{ver}LoadOut` | Siempre |
| MethodSave | Procedure | `WSTransaction{Trn}V{ver}Save` | Solo si `modeInsert=True` o `modeUpdate=True` |
| SDTSaveIn | SDT | `WSTransaction{Trn}V{ver}SaveIn` | Siempre (estructura) |
| SDTSaveOut | SDT | `WSTransaction{Trn}V{ver}SaveOut` | Siempre (estructura) |
| MethodDelete | Procedure | `WSTransaction{Trn}V{ver}Delete` | Solo si `modeDelete=True` |
| SDTDeleteIn | SDT | `WSTransaction{Trn}V{ver}DeleteIn` | Siempre (estructura) |
| SDTDeleteOut | SDT | `WSTransaction{Trn}V{ver}DeleteOut` | Siempre (estructura) |

Total: hasta **10 objetos** por versión (3 Procedures + 7 SDTs).

## Estructura XML de la instancia

```xml
<instance parentTransaction="Customer"
          category="BackOffice">

  <version version="1"
           modeInsert="true"
           modeUpdate="true"
           modeDelete="true">

    <structure name="Customer">
      <!-- Keys -->
      <key name="CustomerId"
           publicName="id"
           description="Customer ID"
           type="Public" />
      <key name="CompanyId"
           publicName=""
           description=""
           type="Multitenant" />

      <!-- Atributos del nivel principal -->
      <attribute name="CustomerName"
                 publicName="name"
                 description="Customer Name"
                 updateAttribute="Always" />
      <attribute name="CustomerEmail"
                 publicName="email"
                 description="Email"
                 updateAttribute="Always" />
      <attribute name="CustomerStatus"
                 publicName="status"
                 description="Status"
                 updateAttribute="On Insert" />

      <!-- Sub-nivel (detalle de la transacción) -->
      <level name="CustomerAddresses">
        <key name="CustomerAddressId"
             publicName="addressId"
             description="Address ID"
             type="Public" />
        <attribute name="CustomerAddressStreet"
                   publicName="street"
                   description="Street"
                   updateAttribute="Always" />
        <attribute name="CustomerAddressCity"
                   publicName="city"
                   description="City"
                   updateAttribute="Always" />
      </level>
    </structure>
  </version>
</instance>
```

## Propiedades de la versión

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `version` | string | Número de versión |
| `modeInsert` | bool | Habilitar inserción (afecta generación de Save) |
| `modeUpdate` | bool | Habilitar actualización (afecta generación de Save) |
| `modeDelete` | bool | Habilitar eliminación (genera Delete Procedure) |

## Estructura multinivel

PXWSTransaction soporta transacciones con **niveles anidados** (master-detail). Cada `level` dentro de `structure` representa un nivel de la transacción:

```
Customer (nivel principal)
├── key: CustomerId
├── attribute: CustomerName
├── attribute: CustomerEmail
│
└── CustomerAddresses (sub-nivel)
    ├── key: CustomerAddressId
    ├── attribute: Street
    └── attribute: City
```

Los sub-niveles solo pueden agregarse dentro del nodo `structure` (raíz), controlado por `CanAddIf="self::structure"`.

## Keys — Tipos

| type | Descripción | En Load | En Save | En Delete |
|------|-------------|---------|---------|-----------|
| `Public` | Visible al consumidor | SDT In | SDT In | SDT In |
| `Multitenant` | Automático desde conexión | Interno | Interno | Interno |
| `Private` | Interno, no expuesto | Interno | Interno | Interno |

## Atributos — updateAttribute

| Valor | Descripción |
|-------|-------------|
| `Always` | Se actualiza en Insert y Update |
| `On Insert` | Solo se asigna en Insert, ignorado en Update |
| `On Update` | Solo se asigna en Update, ignorado en Insert |

Esto permite controlar qué campos son editables en cada modo sin código custom.

## Operaciones generadas

### Load
- Lee un registro por sus keys públicos
- Input: SDTLoadIn (keys públicos)
- Output: SDTLoadOut (estructura completa + mensajes)

### Save
- Inserta o actualiza según exista o no el registro
- Input: SDTSaveIn (estructura completa)
- Output: SDTSaveOut (resultado + mensajes de error)
- Usa **Business Component** de GeneXus internamente

#### El Save es un REEMPLAZO COMPLETO, no un patch

Es la característica del pattern que más fácil se malinterpreta, y hacerlo destruye datos sin dejar
rastro. El código generado hace `BC.Load(...)` y **después asigna todos los atributos** desde el SDT
de entrada:

- Un atributo que no venga en el SDT **se graba vacío**. No se conserva el valor que tenía.
- En cada nivel subordinado, después del insert-or-update de los ítems recibidos, hay un bloque
  `// Delete <Nivel>` que **elimina las filas que no vengan en la colección**.

Dicho de otra forma: **el SDT de entrada es el estado deseado completo, no un delta.**

Consecuencia para cualquier consumidor —una API REST, una pantalla, una tarea batch, una tool de
asistente—: la única forma segura de modificar es

```
Load  →  aplicar los cambios sobre lo que se cargó  →  Save
```

Un `Save` armado desde cero con los dos o tres campos que se querían cambiar **vacía el resto del
registro y borra todos sus niveles subordinados**. Y lo hace en silencio: no hay error, no hay
warning, y la operación devuelve `Succeed = True`.

Un caso concreto de cómo se pierde un dato sin que nadie lo pida: si el consumidor decide qué
atributos tocar preguntando si el valor recibido está vacío, confunde *"no me lo mandaron"* con *"lo
quieren vacío"*. Hay que distinguir la **ausencia** del campo de su valor vacío — mandar un campo
vacío es una forma legítima de borrarlo, no mandarlo no lo es.

### Delete
- Elimina un registro por sus keys
- Input: SDTDeleteIn (keys)
- Output: SDTDeleteOut (resultado + mensajes)
- Usa **Business Component** de GeneXus internamente

## SDT Structure compartido

El SDT `WSTransaction{Trn}V{ver}Structure` es compartido entre Load, Save y Delete. También es compartido con PXWSData si ambos patterns se aplican a la misma transacción y versión.

## Relación con PXWSLayer

PXWSTransaction es normalmente invocado desde métodos de PXWSLayer:

```
PXWSLayer
├── method "Load"  ──► PXWSTransaction (Load)
├── method "Save"  ──► PXWSTransaction (Save)
└── method "Delete" ──► PXWSTransaction (Delete)
```

## Relación entre los 4 patterns WS

```
┌────────────────────────────────────────────────────┐
│                  PXWSLayer                          │
│              (Orquestador API)                      │
│                                                    │
│  Método "Query"  ──► PXWSQuery (listas paginadas)  │
│  Método "Get"    ──► PXWSData (lectura individual) │
│  Método "Load"   ──► PXWSTransaction.Load          │
│  Método "Save"   ──► PXWSTransaction.Save          │
│  Método "Delete" ──► PXWSTransaction.Delete        │
│  Método custom   ──► Procedure/DataProvider GX     │
│  Método inline   ──► Event (código directo)        │
└────────────────────────────────────────────────────┘
```

## Categoría de seguridad

La propiedad `category` define un valor del dominio `WSCategory` que se usa para controlar el acceso a los servicios. Esto se integra con el módulo **@Security** de PXTools para verificar permisos antes de ejecutar operaciones.
