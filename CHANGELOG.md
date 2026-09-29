# Changelog

Change log of the PXTools skills (documentation for AIs).

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This repository does not version software but documentation, so entries are grouped by the date they
were published to `master`.

## [Unreleased]

Nothing pending.

## 2026-09-29

### Added
- **`pxtools/01-pxworkwith.md`** — section 9.6 documented `descriptionAttribute` as the way to link a
  Selection column to its View, and said nothing about the other route: an eye icon written as an in-grid
  `<action>` pointing at `instanceLevelNode="View"`. Both are valid and they can coexist on the same grid,
  so choosing between them is style. What the section now states is the part that is not optional — the
  action's `<parameters>` node. An action passes **exactly what it declares**, and pointing it at a View
  does not make the row's key travel on its own. Omit the node and the build is green while the icon does
  nothing: the generated `Link()` is positional, so the missing key leaves no hole and every remaining
  argument shifts up one place, landing the window type in the id. The entry shows the action written
  correctly, then both generated forms side by side, and names the only signal — a `spc0023` on the
  Selection about a `Character` parameter linked where a `Numeric` is expected, which reads like noise
  from a generated object and on a Selection means the link is losing the key.
- **`pxtools/01-pxworkwith.md`** — the same section gained a note on **which** column to give the link
  when using `descriptionAttribute`. It has to be one a person can see and aim at, and an identifier is
  the obvious choice and often the worst: a short number is a small target and nothing about it suggests
  it can be clicked. A date or a name reads as a label and gives the pointer something to land on.

## 2026-09-28

### Added
- **`pxtools/modules/systemparameters.md`** — the *declare, seed, read* section now states what happens
  when a module is registered in `AddSystemParameters` but not in `RetSystemParameters`. The two are not
  interchangeable: the first creates the parameter rows, the second serves the **combo options**, live, on
  every paint of the preferences screen (`RetSystemParameterComboValues` → `RetSystemParameters`). A module
  in the first only gets its parameters created and editable, and any `Combo` or `Chosen` parameter it
  declares comes out as an **empty dropdown** — nothing errors, and the screen looks built. The entry names
  the wrong turn the symptom invites, which is to read an empty combo as "the seeding has not run" or "there
  is no data yet" and go debug `AddSystemParameters` and the table behind the options, both of which are
  fine. It also explains why the omission survives so long: it only shows on `Combo` and `Chosen`
  parameters, so a module whose parameters are all strings, numbers or booleans behaves identically
  registered or not — the aggregator drifts module after module, and the first combo parameter added to any
  of them is the one that pays.

- **`pxtools/modules/taskmanager.md`** — the *how to integrate a batch task* recipe gained the step it was
  missing: registering the procedure in the application's `RetDynamicCallReference<App>` provider is not
  enough if that provider is not itself invoked from `AddDynamicCallReferences`, the single place where
  every module's provider is gathered into `TDynamicCallReferences`. A provider nobody calls feeds nothing
  — the code never reaches the table, the combo does not offer it, the runner cannot resolve it — so the
  task compiles and simply never runs, with nothing reporting the omission. A brand-new module's provider
  has to be added to that list; an existing module's is already there.
- **`pxtools/modules/webserviceslog.md`** — new section on what the `WebServiceLogStatus` values mean,
  because the natural reading of `Failed` is wrong. The status records **how the call went, not whether the
  business answer was the one somebody hoped for**: `WithoutResponse` is what `AddWebServiceLog` writes when
  it opens the row, so a row left in that state is a call that never reached its end; `Success` covers a
  service that ran and returned what it was supposed to, **including returning nothing** when empty is the
  correct answer; `Failed` is only for something breaking internally. Using `Failed` for a correct negative
  answer fills the log with problems that do not exist and buries the calls that really did break.

## 2026-09-28

### Added
- **`pxtools/modules/messaging.md`** — new module document for `@PXTools/@Messaging` (88 objects, 7
  transactions), written around the two ideas the module is unreadable without: a row of
  `MessagingAccounts` **is** one bot, so everything is scoped by account; and a row of `MessagingLink`
  is not a directory entry but **the credential** — it issues an internal authorization and hands out
  a live access token on every turn, which is why unlinking must revoke rather than flag, and why an
  expired authorization under an active link is reissued in silence. Covers the inbound path (the
  header secret that authenticates the origin *and* identifies the bot in one query), the
  conversation memory living in the message table itself, the two-level outbox with its
  classification of provider errors by code, pending actions, and the Mini App's two halves — a
  replayable signed payload plus an invitation that expires and is spent once, neither sufficient
  alone.
- **`pxtools/modules/ai.md`** — new module document for `@PXTools/@AI`: a row of `AIAccounts` is one
  configured way of calling a model, and its connector code resolves through @DynamicCallReferences
  to the object that speaks to a provider, so changing provider is changing a value on a screen. Two
  connector kinds because they are two contracts (chat takes messages and returns text; transcription
  takes audio). Documents the connector `Parm` and why the tool-surface access token **enters as a
  parameter** — the platform owns it, the engine does not.
- **`pxtools/modules/oauthservice.md`** — new module document for `@PXTools/@OAuthService`: an
  authorization server implemented inside the KB, not a client of one. Core idea: the authorization is
  the grant and tokens are disposable, with every identity field on a token resolved through the
  foreign key — which is what makes revocation instant and what makes issuing a new authorization
  silently kill the previous grant. Includes a **Known gaps** section with defects verified in the
  code: the authorization code is never consumed on exchange (replayable for its full ten-minute
  window), the purge deletes authorizations by an expiration that is always creation + 10 minutes even
  for grants whose tokens live for weeks, the authorize endpoint alone does not check client status,
  refresh is impossible under a never-expiring policy, and the module's own domains carry values
  belonging to one installation.
- **`pxtools/modules/mcpserver.md`** — new module document for `@PXTools/@MCPServer`: a stateless
  JSON-RPC-over-POST engine where one application can publish several servers, each an endpoint plus a
  registry. Documents the tool contract and the two things about it that are easy to miss — a business
  error comes back as a tool *result* flagged as an error and not as a protocol error, and the second
  output exists so a chart re-runs the tool server-side, because figures that pass through the model
  can come back subtly wrong and nobody would notice. Known gaps include a built-in tool hard-coded in
  one natural language inside the generic engine, rate limiting that cannot emit its own
  `Retry-After`, six dead External Object methods (so the protocol version header is never validated),
  and an internal procedure link that is serialized into the JSON handed to the protocol builder —
  whether it reaches the client cannot be determined from the KB.

### Changed
- **`pxtools/20-pxtools-modules.md`** — the three tables at the foot of the index were still describing
  the product as it was before the four new modules: the instance summary listed none of them, and the
  dependency graph was missing **@SecurityProjects** as well, a gap that predates this change. All
  three were then verified row by row against the Knowledge Base — every count in the instance table
  now matches what is on disk, every module has a row in the dependency graph or a stated reason not
  to, and the two tables carry a note naming what is deliberately absent and why (@MCPServer publishes
  no screens; the base modules are not "brought in for a need"). A table that says what it omits can be
  audited; one that is merely incomplete cannot.

- **`pxtools/20-pxtools-modules.md`** — the four modules above were entirely absent from the index:
  added their catalogue sections, their rows in the inter-module dependency table, and their entries
  in "when to bring each module in". Until now a reader of the index had no way to learn that the
  product had a chat channel, an authorization server, an MCP surface or an AI abstraction at all.

- **`pxtools-start-here.md`** and **`pxtools/21-oauth-service.md`** — the entry point still said "25+
  modules" and named only four of them, so a reader arriving there could not learn that the product had
  a chat channel, an MCP surface or an AI abstraction; it now points at the catalogue as the way in and
  says that every module has its own document. And `21-oauth-service.md` — the one module promoted to a
  top-level document, which is why it was easy to write `modules/oauthservice.md` without noticing it —
  now declares itself the **integration guide** (endpoints, host hooks, configuration, import) and links
  to the module reference, which links back. Two documents about one module stay honest only while each
  says what the other is for.

## 2026-08-27

### Added
- **`pxtools/00-overview.md`** — new section *A `visibleCondition` over an editable control needs
  something to trigger the round trip*. The section gives the three `visibleConditionEvaluation` values
  with the kind of condition each fits — `Rules` (the default) and `Start` for values fixed before the
  form is drawn, `Refresh` for a control the user can change while it is open — and then makes the
  point the table alone does not: all three run **on the server**, so picking `Refresh` is only half
  the job and switching back to `Rules` does not rescue the other half. A checkbox ticked in the
  browser changes the variable on the page; if nothing goes back to the server the condition is never
  re-evaluated under any setting and the dependent control just stays hidden, with no error and
  nothing to search for. The missing piece is the **round trip**, and what causes it is a
  `<code type="ControlEvent">` on the control that changed whose body ends in `refresh`. Includes the
  event to use per control kind — `.Click` for `Check Box`, `Combo Box`, `Dynamic Combo Box` and
  `Radio Button`, `.IsValid` for `Edit` and prompt-backed fields — the `name` convention
  (`&MyVariable.Click` for a variable, `MyAttribute.Click` for an attribute), an example that
  normalizes a dependent value before refreshing, and a note on `invisibleProgrammingStyle="PXTools"`
  as the usual companion on the dependent control. It closes with the symptom worth recognising on
  sight: a field that never appears when its checkbox is ticked is almost always a missing refresh
  rather than a wrong expression — a condition that is correct and never re-evaluated looks exactly
  like a condition that is wrong
- **`pxtools/00-overview.md`** — new section *`<row>` puts controls side by side — it is not a
  wrapper*. Every tabular section already lays its children out one per line, so `<row>` has exactly
  one purpose: putting two or more controls on the same line. A row around a single control renders
  identically to the control declared on its own, which makes it a level of nesting that groups
  nothing — the same noise problem the *default values* section describes, in structural form: the
  markup announces a decision (*these belong together on one line*) where none was made, and whoever
  reads it has to open the row to find one thing inside. The section is deliberately placed in the
  overview rather than under a single pattern because the node is not a form feature: it is accepted
  by every tabular container in the UI patterns — attribute lists, tabs, columns, rectangles,
  fixed-data sections and filter areas — and the entry ships the one-liner over
  `Patterns/<Pattern>/<Pattern>Instance.xml` that lists which containers take it, so the claim can be
  checked per pattern instead of trusted. It also records what a row may contain, which is wider than
  input fields (`attribute`, `variable`, `variableReference`, `image`, `label`, `action`,
  `actionReference`, mixable in one row), and closes with the exception that keeps the rule from
  becoming dogma: a single-control row is a real decision when the row's own properties are the point
  — `responsiveSizes`, `firstElementAligned`, `columnsDependant`, `tableType`, or a
  `name`/`description` acting as a section title. What should never be written is the *bare* row that
  sets nothing

## 2026-08-21

### Added
- **`pxtools/00-overview.md`** — new section *Do not declare a property that already carries its
  default*. An instance should hold decisions and nothing else: a property written with the value it
  already has changes nothing either way, and the longer the instance grows the harder it is to see
  what was actually decided — when every filter carries `type="Equal" isRequired="False"
  allowNullValue="False"`, the one filter that is genuinely required disappears into the noise, and so
  does the real content of any diff. The section says where the defaults live — each pattern's own
  `Patterns/<Pattern>/<Pattern>Instance.xml`, where every property declares its `DefaultValue` — and
  gives a one-liner that lists them for an element before writing anything. The reason it is worth a
  section rather than a style note is the case that prompted it: `<orderAttribute ascending="True" />`,
  a line that says nothing because `true` is already the default, makes the PXWSQuery apply fail with
  an `InvalidCastException` on the order domain, while `ascending="False"` — the value that does change
  the behaviour — works and is used in production. It closes with the rule read the other way round:
  when an existing instance declares something, check the default before removing it, because a value
  that differs from the default is the author saying something
- **`pxtools/00-overview.md`** — new section *`&WindowSelf`: which kind of window the screen is running
  in*. Every generated screen receives the parameter and it is routinely treated as bookkeeping to pass
  along; it is not. It declares the kind of window the screen runs in, and two things are decided from
  it: the **`RecentLink` history**, which is kept separately per window kind so a modal browsing its own
  history does not disturb the main area behind it, and the **JavaScript PXTools injects so the window
  can size itself**. The section gives the four values with their meaning — empty for the main area,
  `MD`/`MD1`/`MD2`… for a modal opened with `Popup()`, `PU`… for a browser popup window (the old way,
  which does not hold focus and can end up lost behind the main window), `EM`… for an Embedded Page that
  becomes an `iFrame` — including the counting rule that trips people up: the first one is `MD`, never
  `MD0`, and a window that opens another of its kind increments. The reason this belongs in writing is
  the failure mode: passing the wrong value **fails silently**. Calling `SomeScreen.Popup()` with an
  empty `&WindowSelf` tells a modal it is the main area, the auto-sizing scripts are never injected, and
  the window simply comes out the wrong size — with no error, no warning, and a cause that looks nothing
  like the symptom

## 2026-08-20

### Added
- **`pxtools/modules/menus.md`** — new subsection *Pointing at the screen: `InstanceReference`, not
  the object name*, plus two warnings at the head of *Declaring and seeding*. Between them they cover
  the three ways a menu entry gets added wrong. The first is treating the menu as data: `TMnuWeb` has
  a WorkWith and can be edited by hand, which makes it look like a table to load, but the tree is
  **declared in code** and seeded from there — an option typed into the screen is missing from its
  module's `RetMenus<X>`, so it never travels to another installation and the next seeding does not
  reproduce it. The second is where the DataProvider lives: `RetMenus<X>` belongs to the
  `Personalized/` **of the module that contributes the options** — `RetMenusSecurity` under
  `@Security/`, `RetMenusOAuthService` under `@OAuthService/` — and only the sections without a module
  of their own stay in `@Menus/Personalized/`, several of them empty shells. Looking in
  `@Menus/Personalized/` alone and concluding a module has no menu leads to writing a **second**
  DataProvider with the same name in another module, which compiles and seeds the tree twice;
  `AddDefaultMenus` is the reliable catalogue of what actually gets seeded. The third is how a leaf
  names its target: `InstanceReference` with the **entity** and the node type, letting the framework
  resolve the generated object per platform, rather than `Program = RetObjectName.Udp(<Object>.Type)`,
  which ties the menu to one platform's object name and loses the responsive override. The subsection
  also records that PXWorkWith names the listing screen `Tr<Entity>` (`Ct<Entity>` is the query
  screen) and not `WW<Entity>` — searching for "WW" finds nothing and makes an existing menu
  declaration look absent

### Changed
- **The whole corpus is now written in English** — the 45 `.md` files (`README.md`,
  `pxtools-start-here.md`, the 21 pattern references under `pxtools/` and the 22 module documents
  under `pxtools/modules/`) were translated from Spanish, and so were the file and directory names
  that carried Spanish: `pxtools/modulos/` → `pxtools/modules/`,
  `12-acciones-patterns-ui.md` → `12-pattern-ui-actions.md`,
  `13-semantica-grids-webpanels.md` → `13-grid-webpanel-semantics.md`,
  `20-modulos-pxtools.md` → `20-pxtools-modules.md`,
  `30-guia-reconocimiento-patterns.md` → `30-pattern-recognition-guide.md`,
  `31-capacidades-dual-platform.md` → `31-dual-platform-capabilities.md`,
  `32-limitaciones-y-gaps.md` → `32-limitations-and-gaps.md`,
  `33-estrategia-migracion.md` → `33-migration-strategy.md`, and the module files
  (`seguridad.md` → `security.md`, `tareas-nube.md` → `cloudtasks.md`, and the rest). The
  cross-references between documents were rewritten to the new names in the same pass. What did
  **not** change is the content that is quoted rather than prose: GeneXus object, attribute, domain
  and pattern-node names stay as they are in the KB, and the Spanish literals inside code examples
  stay Spanish — translating either would turn a copyable example into one that does not compile.
  Rationale: the skills are read by AIs whose instruction-following and technical vocabulary are
  strongest in English, and the corpus is published under CC BY for an audience that is not only
  Spanish-speaking

## 2026-08-19

### Added
- **`pxtools/08-pxwstransaction.md`** — new section *El Save es un REEMPLAZO COMPLETO, no un patch*,
  under the generated `Save` operation. The generated code loads the Business Component and then
  assigns **every** attribute from the input SDT, so an attribute that does not travel in it is
  written empty rather than left alone; and in each subordinate level, after the insert-or-update of
  the items received, a `Delete <Level>` block removes the rows that did not come in the collection.
  The input SDT is the complete desired state, not a delta. The section states the consequence for
  every consumer — REST API, screen, batch task, assistant tool alike — which is that the only safe
  way to modify is Load, apply the changes over what was loaded, then Save: a `Save` assembled from
  scratch with the two or three fields somebody wanted to change empties the rest of the record and
  wipes its subordinate levels, in silence, with no error, no warning and `Succeed = True`. It closes
  with the way the data is lost without anyone asking for it — deciding which attributes to touch by
  asking whether the received value is empty confuses *"it was not sent to me"* with *"they want it
  empty"*, so absence of a field has to be distinguished from its empty value.

## 2026-08-16

### Added
- **`pxtools/06-pxwsquery.md`** — new section *How to declare the `<orders>`*. Each `<order>` becomes a
  literal `Order` clause of the generated DataProvider, so the performance rules of any `For Each`
  apply to it, and none of them were written down. Three of them, in the order they bite:
  every order must **lead with the equality-filtered attributes** — first the Layer's multi-tenant
  attribute, which the pattern filters on every query, then the instance's own `type="Equal"` filters
  (typically the parent key when the query runs over a subordinate level), and only then the attribute
  one actually wants to sort by. Second, the resulting attribute sequence must be a **prefix of an
  existing index**, verifiable in `#Tables/<Table>.Table.gxSource`; ordering by an attribute of the
  extended table — the descriptor of a foreign key — is never index-backed, so order by the foreign key
  instead, and when no index supports the order the choice is to drop it or to create a user index,
  which is a database reorganization and therefore the KB owner's call. Third, and this one only
  shows up as a failed apply: the generated enumerated domain uses **the list of attribute names as the
  `Description`** of each value, so two orders over the same attribute set — the same attribute
  ascending and descending, for instance — produce two values with the same description and the apply
  dies with `Failed processing Domain '<WSQuery…Order>' properties`.

## 2026-08-14

### Changed
- **`pxtools/01-pxworkwith.md`** — section *4.6 Where a ControlType is defined* now lists the **domain**
  as a fourth possible place, states the project convention (edit controls live in the instance, or in
  the form when the object has no instance — never on attributes or domains), and adds a subsection on
  the empty item. The rule it settles: an `(All)`, `(None)` or `(Select…)` is a need of **one screen** —
  a filter saying "do not filter" — and never a property of the data, so it belongs to the control:
  `<controlInfo controlType="Combo Box" emptyItem="True" emptyItemText="(All)" />`, in lower case,
  inside the instance. It also documents the false friend that costs the time: the domain *does* accept
  `EmptyItem` / `EmptyItemText` and the import takes it without complaint, but the combo is still
  generated with `AddEmptyItem="False"`, so nothing appears on screen and the symptom is "I set the
  property and nothing happened".
- **`pxtools/02-pxparameterrequest.md`** — new subsection on the `Subroutine` code hook. Unlike
  `Start`, `Refresh` and `Load`, which are single points in the life cycle, `Subroutine` is declared
  **once per subroutine**, the routine's name goes in the node's `name` attribute, and the CDATA
  carries only the body — the pattern writes the `Sub` and `EndSub` itself. The mistake this prevents
  is putting every routine in one node with hand-written `Sub … EndSub` inside: the pattern does not
  split them, the node ends up with an empty name (visible as an unnamed *Code Subroutine* in the
  visual editor) and none of the routines is generated.
- **`pxtools/02-pxparameterrequest.md`** — the behaviour section described a `<behaviour>` node with
  long type names (`PopupParameterRequest`, `FloatingParameterRequest`, …). Those belong to the
  pattern **definition**; an externalized instance writes the behaviour as the `type` attribute of
  `<level>` instead, and no instance in the reference knowledge base contains a `<behaviour>` node at
  all. The section now leads with the real form and the values actually observed (`Popup`,
  `Component`, `Web Panel`, `Prompt`, `Process`), because the consequence is easy to hit and hard to
  read: a level that omits `type="Popup"` still generates and still opens, it just renders without the
  frame, the title and the action bar — a form pasted onto the page rather than a dialog. Also states
  that the popup **size** is not declared there: `popupWidth`, `popupHeight` and `popupBehaviour`
  belong to the caller's action, so one form can open at different sizes from different places.

### Added
- **`pxtools/modulos/menus.md`** — section *5.1.2 The module has to be in the `SystemModules`
  catalog*. Writing the `RetMenus<X>` and adding its line to `AddDefaultMenus` is **not enough**: a
  new module must also be registered in `@System/Personalized/SaveSystemModules`, because
  `AddMenusRecursive` looks each item's `Module` up in the catalog and drops the node when it is not
  there. The consequence that makes this hard to diagnose is that `AddDefaultMenus` then issues a
  **`RollBack` of the whole run** — one undeclared module leaves *every* module of that execution
  without menus, not just its own, so the symptom is "no new menu appeared" rather than an error about
  the module. Documents that the string compared is the `Value` of the `PXToolsModules` domain (the
  qualified module name) and must match the argument of `AddSystemModule` character for character,
  plus the four-step checklist for adding a module with menus and the order the two seeds must run in.

### Changed
- **`pxtools/modulos/system.md`** — the `SaveSystemModules` entry now states that registering the
  module there is a **precondition for its menus to be seeded**, with a cross-reference to the section
  above; previously it only described the procedure as rebuilding the module catalog, which did not
  suggest that skipping it breaks something elsewhere.

## 2026-08-11

### Added
- **`pxtools/12-acciones-patterns-ui.md`** — section *5.bis Confirms*. The `confirm` **property** of
  an action only takes fixed text; naming the record in the question ("Delete client X?") requires
  the **`confirms` node**, which was undocumented. Covers `questionType="Variable"` —without it the
  expression is rendered literally— the standard `HPEXE_Confirm` dialog, and the fact that the
  confirm is invoked as a subroutine, so **two** actions are needed: the one the user sees and the
  one that does the work. Includes the recipe for replacing the built-in delete when referential
  integrity blocks it, whose three parts all have to be present or the change has no effect.

### Changed
- **`pxtools/01-pxworkwith.md`** — the `modes` subnode was described as *"special modes: Export
  Excel, Charts, Update Grid Rows"*, which is **incomplete**: it also governs Insert, Update, Delete
  and Display. It is configured with **attributes, not children**, and its full table is now
  documented. The consequence that was missing: **disabling a mode is what frees the action name**,
  so overriding `Delete` requires `Delete="false"` *and* declaring the action — with the mode enabled
  the pattern still generates its own action against the transaction. Also adds the **required order
  of the subnodes** of `selection`, which the pattern validates as a sequence, and the `confirms`
  node to the table.

## 2026-08-10

### Added
- **`pxtools/01-pxworkwith.md`** — sections *4.5 Transaction (edit form)* and *4.6 Where a
  ControlType is defined*. The `transaction/layouts/layout` node was undocumented: it defines the
  transaction form, and its `<attributes>` **replaces the whole form**, so an omitted attribute
  disappears from the screen. Section 4.6 settles where a `ControlType` belongs — pattern instance,
  WebForm, or the attribute — and why the `.gxAttribute` is the worst option: it propagates to
  **every** appearance of the attribute, and a Dynamic Combo Box in a grid column loads the entire
  foreign table **for each row**. Rule for grids: always `Edit`; to show the foreign value, define a
  **subtype of the description attribute**, leave the `Id` as `visible="False"` (still available for
  actions) and display the name. In short: combo in the edit form, name subtype in the grid.
- **`pxtools/modulos/wslayer.md`** — section *5.4 How a consumer is integrated*. The consumption
  pattern was missing: the category is **hardcoded from the domain** and the IP is obtained at the
  control point itself with `RetHTTPRemoteAddress.Udp()`, called as early as possible because it is a
  **network** control and must not depend on the request being parseable. Includes how to add a new
  category and the clarification that enabling the control requires no code change (a category with
  no rows allows everything). Adds the **`WSWhiteListId` foreign key antipattern**: a concept needs N
  rules (Accept and Deny coexisting) and a foreign key points to only one, plus the `Id` is
  autonumber and not hardcodable — the unit of the concept is the **category**, which is why the
  whole API takes it as a parameter.

### Changed
- **`pxtools/21-oauth-service.md`** — the endpoint section was **wrong** on three counts, all
  verified at runtime against a live server. (1) The real URLs were missing: a main proc with
  `CallProtocol='HTTP'` publishes as `<base>/<namespace>.<module>.a<object>` (the `a` prefix is added
  by the Java generator), while *Expose as Web Service* publishes as
  `<base>/rest/<Module>/<Object>` **respecting the module casing** — GeneXus does **not** lowercase
  REST paths, contrary to what the previous note claimed. (2) `Token` is no longer REST: it was
  migrated to main + `CallProtocol='HTTP'` to comply with RFC 6749. (3) The claim that the HTTP
  status cannot be set was false — `PXTools.APIs.SetHttpStatus` sets it via inline JAVA, and `Token`
  now returns real 400/401. Adds the exposure criterion: a contract defined by a third party (an RFC,
  a provider, a protocol) calls for main + HTTP; a contract you define yourself calls for an API
  object, the only one that provides `&RestCode`.

## 2026-08-01

### Added
- **`pxtools/modulos/menus.md`** — section *5.1.1 `Parent`: ONLY on the root node of each
  `RetMenus<X>`*. `Parent` hooks the portion of the tree contributed by the module underneath an
  existing node (typically `'Basic'`) and belongs only on the first-level items; the inner hierarchy
  comes from nesting in `Childs`. Includes the `AddMenusRecursive` fragment that proves it (`Parent`
  is read only when no parent arrives through the recursion) and the warning that `RetParentMenu`
  **creates** the menu when the name does not exist, so a misspelled `Parent` on the root node does
  not fail: it silently generates an orphan node.

## 2026-07-18

### Added
- `README.md`, `LICENSE` (CC BY 4.0) and `.gitignore`.

### Changed
- Documentation of modules, *root-legacy* domains, inter-module dependencies and the criteria for
  attributing a domain to its module.
- Genericized examples: customer object names were replaced with neutral ones.

## 2026-05-28

### Added
- Initial publication of the PXTools skills: framework overview, UI patterns (PXWorkWith,
  PXParameterRequest, PXComposer), flow (PXFlowController), API/WebServices (PXWSLayer, PXWSQuery,
  PXWSData, PXWSTransaction), data/configuration (PXOAV, PXEntityParameters, PXReportTemplate),
  per-module documentation of `@PXTools/*` and cross-cutting recognition and migration guides.

---

## How to write an entry

- Group by the date of publication to `master`, using whichever Keep a Changelog sections apply:
  **Added**, **Changed**, **Deprecated**, **Removed**, **Fixed**.
- Name the file that was touched and **what** was documented, not just that "something was
  documented".
- When an entry corrects previously incorrect documentation, say so explicitly: whoever read it
  before needs to know it changed.
- If the finding was verified against a real KB or against what the IDE emits, mention it — it
  separates what was verified from what was inferred.
