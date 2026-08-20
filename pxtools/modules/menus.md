# @Menus Module — Web Navigation Menus

> Behaviour of the `@PXTools/@Menus` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@Menus/` (`APIs/` core + `Personalized/`).
- Qualifier: `PXTools.Menus`.
- **Depends on:** `@APIs` (base). For the SmartMenu style it uses the **GeneXus** module `@SmartMenus` (not a PXTools module — it only provides the User Control's support). In the canonical graph also `@System`.

## 1. What it provides

The **web navigation menu tree**: hierarchical nodes pointing at GeneXus objects, with images/icons, per-platform overrides, and **per-node security**. Each module declares its slice of the menu; the framework persists it, works out which nodes each user can see (based on authorization) and renders them (top / left / dropdown / tree).

## 2. Core concept

Three pieces:
1. **`TMnuWeb`** — the table the tree is persisted in (one record = one node).
2. **`SDTMenus`** — the recursive **declarative model** each module fills in to describe its menu (it gets seeded into `TMnuWeb`).
3. **`MenuContext`** — the per-user result of working out which nodes are **enabled/visible**; computed once and stored in `WebSession`; rendering only reads that set.

## 3. The `TMnuWeb` transaction

- **Composite PK**: `MnWOri` (`Origin`, Char(1)) + `MnWSec` (Numeric(10.0), autonumbered). The origin separates menu spaces (Development/Customer/Favorites).
- **Hierarchy (self-FK)**: `MnWPadOri` + `MnWPad` → the parent node (`MnWPad=0`/null = root). `MnWOrd` orders siblings. `MnWCat` groups trees by zone (top/left).
- **Node UI**: `MnWName` (the target ObjectName), `MnWDsc`, `MnWAbr`, `MnWImg`/`MnWImgSel`/`MnWImageType`, `MnWIconClass`/`MnWIconValue`.
- **Target**: `MnWPgm` (the program to invoke), `MnWTgt` (target).
- **Per-node security**: `MnWCodSeg` (security code) + `MnWSystemModuleName`/`…Description` (the object's module — it qualifies the program name when checking authorization; the `MnuWebSystemModule` Group).
- **`ApplicationPlatform` sublevel** (PK += `MnWApplicationPlatformPlatform`): program/target **per platform** (WebDesktop / WebResponsive / SmartDevices), plus `MnWApplicationPlatformChildsAction` (how children are rendered: DropDown vs LeftMenus).

## 4. Module domains

**Owned** by @Menus (named `Menu*`/`Menus*`/`*MenuType`/`*MenuDropDownType` + `PXToolsMenus*`), all **root-legacy** (they live in the root `#Domains/`):

| Domain | Values / Role |
|---|---|
| **MenuLocation** | `Left, Top, Toolbar, Main, Category` — where it is rendered |
| **MenuOrder** | Alphabetic=`A`, Order=`O` |
| **MenuImageDisplayType** | Horizontal=`H`, Vertical=`V` |
| **MenuCategory** / **MenuSearch** / **MenusEnabled** / **MenusCollection** | Category (e.g. Favorites), search text, enabled flag, the SDT collection of items |
| **PXToolsMenusChildsAction** | DropDown=`DPD`, LeftMenus=`LEF` — how children are rendered |
| **{Left,DesktopLeft,ResponsiveLeft}MenuType** | `Standard, TreeView, PXToolsSmartMenus` — side menu style per platform |
| **{Top,DesktopTop,ResponsiveTop}MenuDropDownType** | `GXUIToolbar, PXToolsSmartMenus*, HorizontalFloat/Push` — top menu style |

**Used from @APIs base:** `Origin` (Development=`D`, Customer=`C`, Favorites=`F` — separates menu spaces), `NodeType` (type of the target object), `ApplicationPlatform` (`WebDesktop, WebResponsive, SmartDevices`).

## 5. How it works

### 5.1 Declaring and seeding

> ⚠️ **The menu is NOT data you load by hand.** `TMnuWeb` can be viewed and edited through its WorkWith, but the tree is **declared in code** and seeded from there. Adding an option by editing the table leaves it out of its module's `RetMenus<X>`: it does not travel to another installation and the next seeding will not reproduce it. The option goes into the DataProvider, not into the screen.

> 📍 **Where the `RetMenus<X>` lives: in the `Personalized/` OF THE MODULE contributing the options**, not in `@Menus/Personalized/`. `RetMenusSecurity` is in `@Security/Personalized/`, `RetMenusOAuthService` in `@OAuthService/Personalized/`, `RetMenusFileStorage` in `@FileStorage/Personalized/`, and so on. Only the sections with no module of their own stay in `@Menus/Personalized/`, several of them empty shells. **Before creating a `RetMenus<X>`, look for it across the whole tree** (`find . -name "RetMenus*.DataProvider.gxSource"`) or check the `AddDefaultMenus` list, which is the complete catalogue of what gets seeded: creating a second DataProvider with the same name in another module compiles and seeds the tree twice.

- **Declaring**: each module contributes a **`RetMenus<X>`** DataProvider with `Output = SDTMenus` returning its slice of the tree. `SDTMenus` is a recursive collection (`Item` with `Name/Description/Program/Module/InstanceReference/Image…/SecurityCode/Category`, `Parent`, and `Childs : SDTMenus`; plus an `ApplicationPlatform` collection for overrides).
- **Seeding** (idempotent): `PDefaultMenus` → `ChkMenusExistance` (if there is no `Origin=Development`, it seeds) → **`AddDefaultMenus`** (`Personalized/`, the list of `RetMenus<X>()` to add) → `AddMenus` → **`AddMenusRecursive`**: for each item it resolves the parent (`RetParentMenu`), verifies the module, does a `New … When Duplicate` (upsert by `MnWName`) into `TMnuWeb` with `MnWOri=Development`, generates the `ApplicationPlatform` rows (one per supported platform) and descends into `Childs`.

### 5.1.0 Pointing at the screen: `InstanceReference`, not the object's name

A leaf can declare its target in two ways, and **the good one is `InstanceReference`**: you name the **entity** and the node type, and the framework resolves the generated object per platform (`RetNodeTypePlatformPrefix`: Selection→`Tr`, Prompt→`Pr`, WebPanel→`Wb`, with `R` in front for Responsive).

```
Item
{
	Description	= 'Pending Actions'
	Module		= PXToolsModules.Messaging
	InstanceReference
	{
		LevelName	= !'MessagingPendingAction'      // ✅ the ENTITY, not the object
		NodeType	= NodeType.Selection
	}
}
```

`Program = RetObjectName.Udp(<Object>.Type)` is the old form and only makes sense for a standalone object that did not come out of a pattern. Using it for a generated screen ties the menu to **one** platform's object name and loses the responsive override.

While we are here: the object PXWorkWith generates for the listing is **`Tr<Entity>`** (`Ct<Entity>` is the query screen) — not `WW<Entity>`. Searching for "WW" finds nothing and makes an existing menu declaration look absent.

### 5.1.1 `Parent`: ONLY on the root node of each `RetMenus<X>`

`Parent` exists to **hook the slice of tree a module contributes underneath an already-existing node** (typically `'Basic'`). It is used **only on the first-level item(s)** of the DataProvider. The internal hierarchy already comes from the nesting in `Childs`.

```
SDTMenus
{
	Item
	{
		Description	= 'OAuth Service'
		Module		= PXToolsModules.OAuthService
		Parent		= 'Basic'              // ✅ hooks the module under the 'Basic' menu
		Childs
		{
			Item
			{
				Description	= 'Clients'
				Module		= PXToolsModules.OAuthService
				// ❌ do NOT put Parent here: the nesting provides the parent
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

**Why**: in `AddMenusRecursive`, `Parent` is read **only when no parent arrives through the recursion**:

```genexus
For &Item in &SDTMenus
	If &MnWPad.IsEmpty()                                  // root level
		&ItemMnwPadSec = RetParentMenu.Udp(&Item.Parent)  // <- the only place Parent is used
		…
	Else                                                   // children: it comes from the recursion
		&ItemMnWPadOri = &MnWPadOri
		&ItemMnwPadSec = &MnWPad
	EndIf
```

A `Parent` on a child **is ignored**: it breaks nothing, but it is misleading — it suggests a reference the engine never resolves. And if it also points at text matching no `MnWName` (say, `Parent = !'OAuth Service'` when that node was declared with a `Description` and no `Name`), somebody can lose time hunting for a problem that does not exist there.

> **`RetParentMenu` creates the parent if it does not exist**: it looks up `MnWName = &MnwName` and, failing to find it, does a `New` with that name and `Origin.Development`. So a misspelled `Parent` on the root node **does not fail**: it silently creates a new menu with that text. If an unexpected orphan node shows up in the tree, check the `Parent` values of the `RetMenus<X>` DataProviders.

### 5.1.2 The module has to be in the `SystemModules` catalogue

⚠️ **Before seeding a new module's menus, register it in `@System/Personalized/SaveSystemModules`.** Writing the `RetMenus<X>` and adding it to `AddDefaultMenus` is not enough.

`AddMenusRecursive` validates each item against the catalogue before creating the node:

```genexus
If not &Item.Module.IsEmpty()
	&SystemModuleName = &Item.Module.Trim()
	For Each order SystemModuleName
		Where SystemModuleName = &SystemModuleName
		&OkModule = True
	When None
		&OkModule = False
	EndFor
Else
	&OkModule = True                                       // with no Module declared, it passes
EndIf

If &OkModule
	…creates the node…
Else
	AddMissingModules.Call(&MissingModules, &SystemModuleName)
EndIf
```

And `AddDefaultMenus` **discards the whole seeding** if any module went unresolved:

```genexus
If &ColMissingModules.Count > 0
	For &Item in &ColMissingModules
		Msg('Missing module  ' + &Item)
	EndFor
	Msg('Some menu options could not be saved! Execute SaveSystemModule extension, do F5 and retry!')
	RollBack                                               // <- EVERYTHING is lost, not just that module
Else
	Commit
EndIf
```

Two consequences worth keeping in mind:

- The `RollBack` is **global to the run**: a single undeclared module leaves *every* module of that execution without menus, not only its own. If after adding a module "no new menu shows up", this is the first thing to check.
- The value being compared is the **`Value` of the `PXToolsModules` domain**, which is the module's qualified name, and it has to match character for character the string passed to `AddSystemModule`:

```genexus
// #Domains/PXToolsModules.gxDomain
Messaging: { Description: "PXTools Messaging", Value: "PXTools.Messaging"}

// @System/Personalized/SaveSystemModules
AddSystemModule.Call("PXTools.Messaging")     // the exact same string
```

**Checklist for registering a new module with menus**: (1) the value in the `PXToolsModules` domain; (2) `AddSystemModule.Call("<qualified name>")` in `SaveSystemModules`; (3) the `RetMenus<X>` DataProvider; (4) the line in `AddDefaultMenus`. Then run `SaveSystemModules` **before** `AddDefaultMenus` — if the catalogue was populated in the same run there is no problem, but the other way round there is.

### 5.2 Per-node security and the user's menu
- **`PPEXE_DeMnW05`** (the engine) walks the tree by category/parent; for each **leaf** it calls `PCheckMenuSecurity` (a hook, default True) and `PIsAuthorized.Udp(<Module>.<Program>)` (the real authorization). **Bottom-up propagation**: if a child ends up enabled, so does its parent, and its `MnWOri+MnWSec` key is added to `MenuContext.Enabled`.
- The check happens **once**, while building the `MenuContext` (persisted in the session with `PSetMenuContext`/`PLoadMenuContext`); at render time **`PPEXE_CtMnW01`** only consults `MenuContext.Enabled.IndexOf(<key>)` to decide whether to paint each node.

`MenuContext` (SDT): `Enabled`/`Visible` (key sets), `SelectedMain/TopTab/TopDropDown/Left`, `SearchValue`, `Variables` (Name/Value pairs of menu context).

## 6. APIs vs Personalized

- **`APIs/`** (core): the `TMnuWeb` transaction, the SDTs (`SDTMenus`, `MenuContext`, `MenusStructure`…), the seeding engine (`AddMenus*`, `RetParentMenu`), the security/query engine (`PPEXE_DeMnW0x`, `PPEXE_CtMnW01`), and the rendering WebComponents (top/left/tree/dropdown).
- **`Personalized/`** (the project's customization):
  | Object | What gets customized |
  |---|---|
  | `AddDefaultMenus` | The **list of `RetMenus<X>()` to seed** (one line per section/module). |
  | `RetMenus<X>` (DataProviders) | **One DataProvider per menu section**: the project declares that section's tree there. They can be left empty (a `Output=SDTMenus` shell) when the section does not apply. **The ones belonging to a named module do NOT live here but in that module's `Personalized/`** — see the warning in §5.1. |
  | `PCheckMenuSecurity` | Per-leaf security hook (default `True`; extra logic optional). |
  | `PLoadOrigin` | The active `Origin` when inserting nodes (default `Development`). |
  | `PGetNotGenericOrigins` | Origins excluded from the standard build (by default it adds `Favorites`). |
  | `PGetMenuObject` | Extracts the clean object name out of `MnWPgm`. |
  | `RetNodeTypePlatformPrefix` | Maps (platform × `NodeType`) → the generated object's name prefix (WebDesktop/Prompt=`Pr`, Selection=`Tr`, WebPanel=`Wb`; Responsive prepends `R`). |
  | `PSaveMenuContext` | Fills `MenuContext.Variables` with user data (context hook). |
  | `MenuSearch` / `MenuSearchResponsive` (WebComponent) | The search UI inside the menu. |

## 7. Pattern instance

**PXWorkWithTMnuWeb** — the menu administration WorkWith over `TMnuWeb` (editing nodes: name/description/images, parent combo, dynamic module combo, a grid for the `ApplicationPlatform` sublevel with a `ChildsAction` combo). It generates `TrMnW02`, `CtMnW02`, and so on.

## 8. Key procedures / APIs

**Context**: `PLoadMenuContext(out: &MenuContext)`, `PSetMenuContext(in: &MenuContext)`, `PDefaultMenus` (ensures seeding + context).

**Enablement/query engine**:
- `PPEXE_DeMnW05(&MenuContext, in: &MnWCat, in: &MnWOriPar, in: &MnWSecPar, in: &NotIncludeOrigins, in: &UsrCod)` — computes the enabled nodes.
- `PPEXE_CtMnW01(in: &MenuContext, in: &MnuOriSec, in: &IncludeVisible, out: &lEnabled)` — is this node enabled/visible?
- `PPEXE_DeMnW02(...)` — ordered children (filtering by enabled); `PPEXE_DeMnW01` — parent nodes for combos.
- `ProgramNameWithModule(in: &MnWOri, in: &MnWSec, in: &MnWPgm, out: &ProgramName)` — the module-qualified name (for authorization).

**Seeding**: `AddMenus(in: &SDTMenus, inout: &ColMissingModules)`, `AddMenusRecursive(...)`, `RetParentMenu(in: &MnwName, out: &MnWSec)`, `ChkMenusExistance`.

**Favorites / search**: `PGetFavorites`, `PPEXE_AddToFavorites(...)`, `PGetMenuSearch`/`PSaveMenuSearch`.

**Rendering** (WebComponents in `APIs/Top/` and `APIs/Left/`): `HPEXE_TopMenus`, `HPEXE_TabsMenus`, `HPEXE_LeftMenus`, `HPEXE_TreeViewMenus`, `TreeViewMenusResponsive`, the GXUI loader (`PLoadMenusToGXUIToolBar`), and the active-node selectors `PSetMainMenu`/`PSetTopMenu`/`PSetLeftMenu`. They all read `MenuContext` through `PPEXE_CtMnW01`.

> **End-to-end flow:** a module declares its tree in `RetMenus<X>` (→ `SDTMenus`) → `AddDefaultMenus`/`AddMenusRecursive` persist it into `TMnuWeb` (idempotently) → at runtime `PPEXE_DeMnW05` checks `PCheckMenuSecurity` + `PIsAuthorized` per leaf and propagates upwards, storing the set in `MenuContext.Enabled` (session) → the rendering WebComponents paint only what is enabled.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [security.md](security.md) — `PIsAuthorized` (per-object authorization), used by the menu engine on each leaf.
- `modules/apis.md` — the menu is built on the session `Context` (§ context) and displayed in the MasterPages.
- `modules/system.md` — `SystemModuleName` (the module catalogue) used to qualify each node's program.
