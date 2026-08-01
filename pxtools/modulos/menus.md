# Módulo @Menus — Menús de Navegación Web

> Comportamiento del módulo `@PXTools/@Menus`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@Menus/` (`APIs/` core + `Personalized/`).
- Cualificador: `PXTools.Menus`.
- **Depende de:** `@APIs` (base). Para el estilo SmartMenu usa el módulo **de GeneXus** `@SmartMenus` (no es un módulo PXTools — solo aporta el soporte del User Control). En el grafo canónico también `@System`.

## 1. Qué provee

El **árbol de menús de navegación web**: nodos jerárquicos que apuntan a objetos GeneXus, con imágenes/íconos, override por plataforma, y **seguridad por nodo**. Cada módulo declara su porción del menú; el framework la persiste, resuelve qué nodos ve cada usuario (según autorización) y los renderiza (top / left / dropdown / tree).

## 2. Concepto central

Tres piezas:
1. **`TMnuWeb`** — la tabla donde se persiste el árbol (un registro = un nodo).
2. **`SDTMenus`** — el **modelo declarativo** recursivo que cada módulo llena para describir su menú (se siembra en `TMnuWeb`).
3. **`MenuContext`** — el resultado, por usuario, de resolver qué nodos están **habilitados/visibles**; se calcula una vez y se guarda en `WebSession`; el render solo lee ese set.

## 3. Transacción `TMnuWeb`

- **PK compuesta**: `MnWOri` (`Origin`, Char(1)) + `MnWSec` (Numeric(10.0), autonumerado). El origen separa espacios de menú (Development/Customer/Favorites).
- **Jerarquía (self-FK)**: `MnWPadOri` + `MnWPad` → nodo padre (`MnWPad=0`/null = raíz). `MnWOrd` ordena hermanos. `MnWCat` agrupa árboles por zona (top/left).
- **UI del nodo**: `MnWName` (ObjectName destino), `MnWDsc`, `MnWAbr`, `MnWImg`/`MnWImgSel`/`MnWImageType`, `MnWIconClass`/`MnWIconValue`.
- **Destino**: `MnWPgm` (programa a invocar), `MnWTgt` (target).
- **Seguridad por nodo**: `MnWCodSeg` (código de seguridad) + `MnWSystemModuleName`/`…Description` (módulo del objeto — califica el nombre del programa al chequear autorización; Group `MnuWebSystemModule`).
- **Sublevel `ApplicationPlatform`** (PK += `MnWApplicationPlatformPlatform`): programa/target **por plataforma** (WebDesktop / WebResponsive / SmartDevices), + `MnWApplicationPlatformChildsAction` (cómo se renderizan los hijos: DropDown vs LeftMenus).

## 4. Dominios del módulo

**Propios** de @Menus (nombre `Menu*`/`Menus*`/`*MenuType`/`*MenuDropDownType` + `PXToolsMenus*`), todos **root-legacy** (viven en el `#Domains/` raíz):

| Dominio | Valores / Rol |
|---|---|
| **MenuLocation** | `Left, Top, Toolbar, Main, Category` — dónde se renderiza |
| **MenuOrder** | Alphabetic=`A`, Order=`O` |
| **MenuImageDisplayType** | Horizontal=`H`, Vertical=`V` |
| **MenuCategory** / **MenuSearch** / **MenusEnabled** / **MenusCollection** | Categoría (p.ej. Favorites), texto de búsqueda, flag de habilitados, SDT-colección de ítems |
| **PXToolsMenusChildsAction** | DropDown=`DPD`, LeftMenus=`LEF` — cómo se renderizan los hijos |
| **{Left,DesktopLeft,ResponsiveLeft}MenuType** | `Standard, TreeView, PXToolsSmartMenus` — estilo del menú lateral por plataforma |
| **{Top,DesktopTop,ResponsiveTop}MenuDropDownType** | `GXUIToolbar, PXToolsSmartMenus*, HorizontalFloat/Push` — estilo del menú superior |

**Usa de @APIs base:** `Origin` (Development=`D`, Customer=`C`, Favorites=`F` — separa espacios de menú), `NodeType` (tipo de objeto destino), `ApplicationPlatform` (`WebDesktop, WebResponsive, SmartDevices`).

## 5. Mecanismo

### 5.1 Declarar y sembrar
- **Declarar**: cada módulo aporta un DataProvider **`RetMenus<X>`** con `Output = SDTMenus` que devuelve su porción del árbol. `SDTMenus` es una colección recursiva (`Item` con `Name/Description/Program/Module/InstanceReference/Image…/SecurityCode/Category`, `Parent`, y `Childs : SDTMenus`; + colección `ApplicationPlatform` para overrides).
- **Sembrar** (idempotente): `PDefaultMenus` → `ChkMenusExistance` (si no hay `Origin=Development`, siembra) → **`AddDefaultMenus`** (`Personalized/`, la lista de `RetMenus<X>()` a agregar) → `AddMenus` → **`AddMenusRecursive`**: por cada item resuelve el padre (`RetParentMenu`), verifica el módulo, hace `New … When Duplicate` (upsert por `MnWName`) en `TMnuWeb` con `MnWOri=Development`, genera las filas de `ApplicationPlatform` (una por plataforma soportada) y desciende en `Childs`.

### 5.1.1 `Parent`: SOLO en el nodo raíz de cada `RetMenus<X>`

`Parent` sirve para **enganchar la porción de árbol que aporta el módulo debajo de un nodo que ya
existe** (típicamente `'Basic'`). Se usa **únicamente en el/los ítems del primer nivel** del
DataProvider. La jerarquía interna ya está dada por el anidamiento en `Childs`.

```
SDTMenus
{
	Item
	{
		Description	= 'OAuth Service'
		Module		= PXToolsModules.OAuthService
		Parent		= 'Basic'              // ✅ engancha el módulo bajo el menú 'Basic'
		Childs
		{
			Item
			{
				Description	= 'Clients'
				Module		= PXToolsModules.OAuthService
				// ❌ NO poner Parent aquí: el padre lo da el anidamiento
				InstanceReference
				{
					LevelName	= !'OAuthServiceClient'
					NodeType	= NodeType.Selection
				}
			}
		}
	}
}
```

**Por qué**: en `AddMenusRecursive`, `Parent` se lee **solo cuando no viene un padre por la
recursión**:

```genexus
For &Item in &SDTMenus
	If &MnWPad.IsEmpty()                                  // nivel raíz
		&ItemMnwPadSec = RetParentMenu.Udp(&Item.Parent)  // <- único lugar donde se usa Parent
		…
	Else                                                   // hijos: viene de la recursión
		&ItemMnWPadOri = &MnWPadOri
		&ItemMnwPadSec = &MnWPad
	EndIf
```

Un `Parent` en un hijo **se ignora**: no rompe nada, pero es engañoso — sugiere una referencia que
el motor nunca resuelve. Y si además apunta a un texto que no coincide con ningún `MnWName`
(p. ej. `Parent = !'OAuth Service'` cuando ese nodo se declaró con `Description` y sin `Name`),
alguien puede perder tiempo buscando ahí un problema inexistente.

> **`RetParentMenu` crea el padre si no existe**: busca por `MnWName = &MnwName` y, si no lo
> encuentra, hace `New` con ese nombre y `Origin.Development`. O sea que un `Parent` mal escrito
> en el nodo raíz **no falla**: crea silenciosamente un menú nuevo con ese texto. Si aparece un
> nodo huérfano inesperado en el árbol, revisar los `Parent` de los `RetMenus<X>`.

### 5.2 Seguridad por nodo y menú del usuario
- **`PPEXE_DeMnW05`** (motor) recorre el árbol por categoría/padre; para cada **hoja** llama `PCheckMenuSecurity` (hook, default True) y `PIsAuthorized.Udp(<Modulo>.<Program>)` (autorización real). **Propagación bottom-up**: si un hijo queda habilitado, el padre también, y su clave `MnWOri+MnWSec` se agrega a `MenuContext.Enabled`.
- El chequeo se hace **una vez** al armar el `MenuContext` (persistido en session con `PSetMenuContext`/`PLoadMenuContext`); en render, **`PPEXE_CtMnW01`** solo consulta `MenuContext.Enabled.IndexOf(<clave>)` para decidir si pintar cada nodo.

`MenuContext` (SDT): `Enabled`/`Visible` (sets de claves), `SelectedMain/TopTab/TopDropDown/Left`, `SearchValue`, `Variables` (pares Name/Value de contexto de menú).

## 6. APIs vs Personalized

- **`APIs/`** (core): la transacción `TMnuWeb`, los SDTs (`SDTMenus`, `MenuContext`, `MenusStructure`…), el motor de siembra (`AddMenus*`, `RetParentMenu`), el motor de seguridad/consulta (`PPEXE_DeMnW0x`, `PPEXE_CtMnW01`), y los WebComponents de render (top/left/tree/dropdown).
- **`Personalized/`** (customización del proyecto):
  | Objeto | Qué se customiza |
  |---|---|
  | `AddDefaultMenus` | La **lista de `RetMenus<X>()` a sembrar** (una línea por sección/módulo). |
  | `RetMenus<X>` (DataProviders) | **Un DP por sección de menú**: el proyecto declara ahí el árbol de esa sección. Pueden quedar vacíos (cascarón `Output=SDTMenus`) si la sección no aplica. |
  | `PCheckMenuSecurity` | Hook de seguridad por hoja (default `True`; lógica extra opcional). |
  | `PLoadOrigin` | El `Origin` activo al insertar nodos (default `Development`). |
  | `PGetNotGenericOrigins` | Orígenes a excluir del armado estándar (default agrega `Favorites`). |
  | `PGetMenuObject` | Extrae el nombre de objeto limpio desde `MnWPgm`. |
  | `RetNodeTypePlatformPrefix` | Mapea (plataforma × `NodeType`) → prefijo de nombre del objeto generado (WebDesktop/Prompt=`Pr`, Selection=`Tr`, WebPanel=`Wb`; Responsive antepone `R`). |
  | `PSaveMenuContext` | Rellena `MenuContext.Variables` con datos del usuario (hook de contexto). |
  | `MenuSearch` / `MenuSearchResponsive` (WebComponent) | UI de búsqueda dentro del menú. |

## 7. Instancia de pattern

**PXWorkWithTMnuWeb** — WorkWith de administración del menú sobre `TMnuWeb` (edición de nodos: nombre/descripción/imágenes, combo de padre, combo dinámico de módulo, grid del sublevel `ApplicationPlatform` con combo `ChildsAction`). Genera `TrMnW02`, `CtMnW02`, etc.

## 8. Procedimientos / APIs clave

**Contexto**: `PLoadMenuContext(out: &MenuContext)`, `PSetMenuContext(in: &MenuContext)`, `PDefaultMenus` (asegura siembra + contexto).

**Motor de habilitación/consulta**:
- `PPEXE_DeMnW05(&MenuContext, in: &MnWCat, in: &MnWOriPar, in: &MnWSecPar, in: &NotIncludeOrigins, in: &UsrCod)` — calcula nodos habilitados.
- `PPEXE_CtMnW01(in: &MenuContext, in: &MnuOriSec, in: &IncludeVisible, out: &lEnabled)` — ¿nodo habilitado/visible?
- `PPEXE_DeMnW02(...)` — hijos ordenados (filtrando habilitados); `PPEXE_DeMnW01` — nodos-padre para combos.
- `ProgramNameWithModule(in: &MnWOri, in: &MnWSec, in: &MnWPgm, out: &ProgramName)` — nombre calificado por módulo (para autorización).

**Siembra**: `AddMenus(in: &SDTMenus, inout: &ColMissingModules)`, `AddMenusRecursive(...)`, `RetParentMenu(in: &MnwName, out: &MnWSec)`, `ChkMenusExistance`.

**Favoritos / búsqueda**: `PGetFavorites`, `PPEXE_AddToFavorites(...)`, `PGetMenuSearch`/`PSaveMenuSearch`.

**Render** (WebComponents en `APIs/Top/` y `APIs/Left/`): `HPEXE_TopMenus`, `HPEXE_TabsMenus`, `HPEXE_LeftMenus`, `HPEXE_TreeViewMenus`, `TreeViewMenusResponsive`, carga GXUI (`PLoadMenusToGXUIToolBar`), selección de nodo activo `PSetMainMenu`/`PSetTopMenu`/`PSetLeftMenu`. Todos leen `MenuContext` vía `PPEXE_CtMnW01`.

> **Flujo end-to-end:** módulo declara su árbol en `RetMenus<X>` (→ `SDTMenus`) → `AddDefaultMenus`/`AddMenusRecursive` lo persisten en `TMnuWeb` (idempotente) → en runtime `PPEXE_DeMnW05` chequea `PCheckMenuSecurity` + `PIsAuthorized` por hoja y propaga al padre, guardando el set en `MenuContext.Enabled` (session) → los WebComponents de render pintan solo lo habilitado.

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [security.md](security.md) — `PIsAuthorized` (autorización por objeto) que usa el motor de menú por hoja.
- `modulos/apis.md` — el menú se arma sobre el `Context` de sesión (§ contexto) y se muestra en las MasterPages.
- `modulos/system.md` — `SystemModuleName` (catálogo de módulos) usado para calificar el programa de cada nodo.
