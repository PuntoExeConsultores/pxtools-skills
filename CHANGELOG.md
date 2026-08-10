# Changelog

Change log of the PXTools skills (documentation for AIs).

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This repository does not version software but documentation, so entries are grouped by the date they
were published to `master`.

## [Unreleased]

Nothing pending.

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
