# PXTools — Documentation for Training AIs

## What PXTools is

PXTools is a **pattern framework for GeneXus** developed by PuntoExe Consultores (Uruguay). It automates GeneXus code generation by turning declarative definitions (`.gxPattern` XML files) into complete GeneXus objects: WebPanels, Procedures, DataProviders, SDTs, APIs, Transactions, and others.

## Key concept

A **pattern** in PXTools has three levels:

1. **Pattern definition** (`.Pattern`) — XML schema defining the available structure, nodes and properties, and which GeneXus objects each node generates.
2. **Pattern instance** (`.gxPattern`) — the concrete configuration that follows the schema, written by the developer for a specific entity or feature.
3. **Generated objects** (`.Childs/`) — the resulting GeneXus code: WebPanels, Procedures, DataProviders, SDTs, and so on.

## Documentation index

All the detailed documentation lives in the [`pxtools/`](pxtools/) subfolder:

### Overview
- [00-overview.md](pxtools/00-overview.md) — Framework architecture, how the generator works, kinds of patterns

### UI patterns (they generate WebPanels)
- [01-pxworkwith.md](pxtools/01-pxworkwith.md) — Master-detail CRUD with Selection, View, Edit, Prompt
- [02-pxparameterrequest.md](pxtools/02-pxparameterrequest.md) — Modal/popup forms for capturing parameters
- [03-pxcomposer.md](pxtools/03-pxcomposer.md) — Screen composition out of WebComponents

### Flow pattern
- [04-pxflowcontroller.md](pxtools/04-pxflowcontroller.md) — Workflow orchestration with confirmations and chained actions

### API / Web Service patterns
- [05-pxwslayer.md](pxtools/05-pxwslayer.md) — REST/SOAP API orchestrator with OpenAPI 3.0
- [06-pxwsquery.md](pxtools/06-pxwsquery.md) — DataProviders with filtering, ordering and paging
- [07-pxwsdata.md](pxtools/07-pxwsdata.md) — Data-read procedures with code hooks
- [08-pxwstransaction.md](pxtools/08-pxwstransaction.md) — CRUD (Load/Save/Delete) over transactions via Business Components

### Cross-cutting reference
- [12-pattern-ui-actions.md](pxtools/12-pattern-ui-actions.md) — The action system shared by PXWorkWith, PXParameterRequest and PXComposer (callTypes, ConditionalCalls, PXInstance, execution cycle, the `execute` property)
- [13-grid-webpanel-semantics.md](pxtools/13-grid-webpanel-semantics.md) — How to read the presence of grids in a WebPanel to recognise the pattern: real vs phantom grids (legacy SDT persistence), multiple grids, For Each Line

### Data / configuration patterns
- [09-pxoav.md](pxtools/09-pxoav.md) — Object Attribute Values (dynamic extensible attributes)
- [10-pxentityparameters.md](pxtools/10-pxentityparameters.md) — Per-entity configurable parameters
- [11-pxreporttemplate.md](pxtools/11-pxreporttemplate.md) — Templates for report generation

### @PXTools modules
- [20-pxtools-modules.md](pxtools/20-pxtools-modules.md) — 25+ reusable modules: Security, Alerts, CloudTasks, FileStorage, etc.
- [21-oauth-service.md](pxtools/21-oauth-service.md) — `@OAuthService`: OAuth 2.0 + OpenID Connect Authorization Server (token / introspect / revoke / userinfo / .well-known + PKCE + JWT id_token HS256 + TaskManager purge)

### Cross-cutting guides
- [30-pattern-recognition-guide.md](pxtools/30-pattern-recognition-guide.md) — How to analyse hand-written WebPanels and decide which pattern to migrate them to
- [31-dual-platform-capabilities.md](pxtools/31-dual-platform-capabilities.md) — Desktop → Responsive migration, dual-platform generation
- [32-limitations-and-gaps.md](pxtools/32-limitations-and-gaps.md) — What the patterns do NOT support today
- [33-migration-strategy.md](pxtools/33-migration-strategy.md) — Migrating a KB without patterns to PXTools

## How to use this documentation

It is written so that an AI can:

1. **Understand** what each pattern generates and how it is configured
2. **Recognise**, in existing GeneXus code, which patterns could be applied
3. **Generate** valid pattern instances (.gxPattern) for new features
4. **Migrate** hand-written WebPanels to PXTools patterns
5. **Diagnose** which WebPanels cannot be migrated, and why

## Sources used

- KB 1: 179 PXWorkWith, 93 PXParameterRequest, 14 PXComposer, 1 PXFlowController, 8 PXReportTemplate, 2 PXOAV, 2 PXEntityParameters
- KB 2: examples of PXWSLayer, PXWSQuery, PXWSData, PXWSTransaction
- KB 3: 19 @PXTools modules
- Website: https://pxtools.puntoexe.com.uy/
- Manual: https://sites.google.com/puntoexe.com.uy/pxtools-manual/
- Pattern definitions: `Patterns/*.Pattern` from the externalized KBs
