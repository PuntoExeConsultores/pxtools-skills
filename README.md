# PXTools — Documentation for AIs

Technical documentation for the **PXTools** framework (a set of GeneXus *patterns* developed by **PuntoExe Consultores**), written to be **handed to AI assistants as context** when they help write PXTools code over GeneXus sources externalized by KBbridge.

## What's inside

- **[`pxtools-start-here.md`](pxtools-start-here.md)** — entry point / general index.
- **[`pxtools/`](pxtools/)** — the detailed documentation:
  - `00-overview.md` … `13-*.md` — framework architecture and patterns: **PXWorkWith**, **PXParameterRequest**, **PXComposer**, **PXFlowController**, **PXOAV**, **PXEntityParameters**, **PXReportTemplate**, **PXWSLayer/Query/Data/Transaction**, grid and web panel semantics, and actions/UI.
  - `20-pxtools-modules.md` + **[`pxtools/modules/`](pxtools/modules/)** — one document per PXTools module (@Security, @TaskManager, @Alerts, @SendMails, @OAV, …): what it provides, its transactions, how it works, **APIs vs Personalized**, domains, and inter-module dependencies.
  - `30-33-*.md` — pattern recognition guide, dual-platform capabilities, limitations, and migration strategy.

## Using it with an AI

Load these files as context or as a *skill* for your assistant (Claude Code, Cursor, etc.) when working on a GeneXus KB that uses PXTools. Every example is **generic**: none of them contain objects from customer KBs.

## Changes

See [CHANGELOG.md](CHANGELOG.md) for the record of what was added or corrected in each release.

## Resources

- Site: https://pxtools.puntoexe.com.uy/
- Manual: https://sites.google.com/puntoexe.com.uy/pxtools-manual/

## License

See the [LICENSE](LICENSE) file.

---

© PuntoExe Consultores.
