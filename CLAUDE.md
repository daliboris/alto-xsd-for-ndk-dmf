# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

The goal is an **XSD 1.0** schema for ALTO documents that conforms to the standard of the Czech National Digital Library (NDK), described in **NDK DMF** (metadata format definition). DMF is a PDF or DOCX document listing the ALTO elements and attributes used in NDK and their obligation (`M` mandatory, `MA` mandatory if applicable, `R` recommended, `RA` recommended if applicable, `O` optional); for attributes it also gives recommended values, or the recommended pattern of a value by example only (e.g. `P3_ST0001`), which the schema has to formalize.

The NDK schema is not written by hand: the original ALTO XSD is transformed using formal rules in XML that are extracted from the DMF document (the pipeline works with its DOCX version in `src/input/docx/`). Outputs are NDK-restricted XSDs (`alto-<ver>-ndk.xsd`) supplemented with annotations of the DMF requirements and an HTML overview table.

The transformation must work across ALTO versions – at minimum 2.0, the latest 4.4 and the upcoming 5.0 (4.2 is also generated because the NK digitization line uses it). Each major version (2.x, 3.x, 4.x, …) has its own namespace (`http://www.loc.gov/standards/alto/ns-v2#`, `…/ns-v4#`, …), and the XSDs are structured differently (e.g. 4.x is built from named complex types), so transformation logic must not assume one namespace or one schema structure.

The repository keeps local copies of all ALTO XSD versions downloaded from loc.gov (`src/schemas/alto/`), as well as schemas of other standards used in NDK (`src/schemas/xml/`; list of sources in `src/data/standards.xml`).

Meeting minutes (Czech) in `docs/minutes/` record open questions and decisions about how DMF wording maps to schema constraints (cardinality, `@ID` patterns, `_SUB` suffix for `TextBlock/@ID`, `dpi` in `MeasurementUnit`, empty pages, possible Schematron rules). Read them before changing restriction logic.

## Toolchain

Everything is XProc 3.x + XSLT 3.0 (Saxon). There is no build system, linter or test suite. Pipelines are run with MorganaXProc-IIIse or XML Calabash (see user-level CLAUDE.md for tool locations and option syntax). The project is also set up for Oxygen XML (`alto-xsd-for-ndk-dmf.xpr`); the root `catalog.xml` chains `src/schemas/{alto,ndk}/catalog.xml` and maps ALTO namespaces / `xml.xsd` / `xlink.xsd` to local copies.

Submodules in `src/includes/` (clone with `git submodule update --init`): `log-xpc-lib` (provides `xlog:store`, used for debug dumps everywhere), `docx-xpc-lib` (DOCX → XML), `xdoc-xpc-lib` (XProc documentation).

## Running the pipelines

All entry points are in `src/run/xproc/`; their inputs and option defaults are relative paths, so no arguments are needed:

```powershell
Morgana src/run/xproc/alto-for-ndk.xpl          # main: generate NDK XSDs + HTML overviews
xmlcalabash src/run/xproc/alto-for-ndk.xpl      # same with Calabash
```

- `alto-for-ndk.xpl` – reads `src/schemas/alto/{4.4,4.2,2.0}/*.xsd` and `src/data/ndk-dmf-data.xml`, writes `src/schemas/ndk/alto/<ver>/alto-<ver>-ndk.xsd` and `src/schemas/ndk/alto/<ver>.html`. **Overwrites those committed outputs.**
- `download-schemas.xpl` – driven by `src/data/standards.xml`. The download step is disabled (`p:use-when="false()"`); currently it only regenerates catalog files from the `src/schemas` directory listing (`directories-to-catalog.xsl`).
- `local.ndk-dmf-to-xml.xpl` – converts the DMF Word document (`src/input/docx/local.NDK_ALTO_XML.docx`) into `src/data/ndk-dmf-data.xml`. The DOCX→XML step is disabled with `use-when="false()"`; it instead loads the cached intermediate `src/_/debug/ndk-dmf-to-xml/0001-local.NDK_ALTO_XML.xml` and runs the `src/xslt/ndk-dmf-to-xml/*.xsl` chain (fix-errors → xml-to-data → identify-ID-attribute-in-details → combine-attributes-data → move-attributes-details → reorder-elements). `local.*` files are git-ignored.

## Architecture

Data flow: DMF `.docx` → `src/data/ndk-dmf-data.xml` (namespace `https://www.daliboris.cz/ns/xproc/ndk`: `element/name`, `description/details`, `obligation`, `cardinality`, `attributes/attribute`) → merged into each ALTO XSD.

Core step `ndk:xsd-for-ndk` (`src/xproc/xsd-for-ndk.xpl`, a `p:library`), per input XSD:
1. `src/xslt/ndk/alto-add-ndk-restrictions.xsl` – applies the actual NDK constraints. They are hard-coded in stylesheet params: `$changes` (minOccurs/maxOccurs etc. on elements/attributes, either global `item`s for ALTO 2.0 style or `items[@type=<complexType>]` for ALTO 4.x, which structures the XSD with named types), `$simpleTypes` and `$complexTypes` (new/replaced types, e.g. ID patterns). Edit these params to change restrictions. The stylesheet binds the `alto` prefix to the v4 namespace – keep this in mind when adding rules meant to work for 2.0 or 5.0.
2. `src/xslt/ndk/merge-mdf-data.xsl` – adds `xsd:annotation/xsd:appinfo[@source='https://www.daliboris.cz/ns/xproc/ndk']` with DMF descriptions to elements matched by name (key on `ndk:name`).
3. Stores the XSD, then `src/xslt/ndk/alto-xsd-ndk-overview.xsl` renders the HTML overview table (mimics the DMF table) by walking element/type references.

Output file names and version folders are derived from the input file name (`alto-4-4.xsd` → version `4.4`).

`src/xproc/xsd-for-ndk_.xpl` is an older standalone variant of the same pipeline; the active one is `xsd-for-ndk.xpl`.

## Conventions

- Pipelines share a common skeleton (section banners for input ports / output ports / options / variables / body) and the options `debug-path` + `base-uri`; debug is on when `debug-path` is non-empty. Intermediate results are written with `xlog:store ... step="N"` to `src/_/debug/<pipeline>/NNNN-<name>.xml` (git-ignored).
- Relative paths are resolved with `p:urify($path, $base-uri)`.
- Project namespace prefixes: `ndk` = `https://www.daliboris.cz/ns/xproc/ndk`, `rm` = `https://www.daliboris.cz/ns/xproc/resource-manager`, `xlog` = `https://www.daliboris.cz/ns/xproc/logging/1.0`.
- Inline TODO comments in XSLT are partly in Czech.
