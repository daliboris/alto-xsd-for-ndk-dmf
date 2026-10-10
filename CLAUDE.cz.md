<!-- source-version: 2026-10-11T00:26:39 -->
# CLAUDE.md

Tento soubor obsahuje pokyny pro Claude Code (claude.ai/code) při práci s kódem v tomto repozitáři.

## Účel

Cílem je schéma **XSD 1.0** pro dokumenty ALTO, které odpovídá standardu české Národní digitální knihovny (NDK) popsanému v **NDK DMF** (definice metadatových formátů). DMF je dokument PDF, popř. DOCX, který vyjmenovává elementy a atributy ALTO používané v NDK a jejich povinnost (`M` povinné, `MA` povinné, je-li aplikovatelné, `R` doporučené, `RA` doporučené, je-li aplikovatelné, `O` nepovinné); u atributů uvádí také doporučené hodnoty, případně doporučenou podobu hodnoty jen formou příkladu (např. `P3_ST0001`), kterou musí schéma formalizovat.

Schéma NDK se nepíše ručně: originální ALTO XSD se transformuje pomocí formálních pravidel v XML, která se extrahují z dokumentu DMF (pipeline pracuje s jeho verzí DOCX v `src/input/docx/`). Výstupem jsou XSD s omezeními NDK (`alto-<ver>-ndk.xsd`), doplněná o anotace požadavků DMF a přehledovou tabulku HTML.

Transformace musí fungovat pro různé verze ALTO – minimálně 2.0, nejnovější 4.4 a připravovanou 5.0 (generuje se i 4.2, protože ji používá digitalizační linka NK). Každá celočíselná verze (2.x, 3.x, 4.x, …) má vlastní jmenný prostor (`http://www.loc.gov/standards/alto/ns-v2#`, `…/ns-v4#`, …) a XSD jsou strukturovaná odlišně (např. 4.x je postavená na pojmenovaných komplexních typech), takže logika transformace nesmí předpokládat jediný jmenný prostor ani jedinou strukturu schématu.

Repozitář obsahuje lokální kopie všech verzí ALTO XSD stažených z loc.gov (`src/schemas/alto/`) a také schémata dalších standardů používaných v NDK (`src/schemas/xml/`; seznam zdrojů v `src/data/standards.xml`).

Zápisy ze schůzek (česky) v `docs/minutes/` zaznamenávají otevřené otázky a rozhodnutí o tom, jak formulace DMF převést na omezení ve schématu (kardinalita, podoba `@ID`, sufix `_SUB` u `TextBlock/@ID`, `dpi` v `MeasurementUnit`, prázdné stránky, možná pravidla Schematronu). Před změnou logiky omezení si je přečti.

## Nástroje

Vše je XProc 3.x + XSLT 3.0 (Saxon). Projekt nemá build systém, linter ani testy. Pipeliny se spouštějí pomocí MorganaXProc-IIIse nebo XML Calabash (umístění nástrojů a syntaxi voleb viz uživatelský CLAUDE.md). Projekt je připravený i pro Oxygen XML (`alto-xsd-for-ndk-dmf.xpr`); kořenový `catalog.xml` řetězí `src/schemas/{alto,ndk}/catalog.xml` a mapuje jmenné prostory ALTO a `xml.xsd` / `xlink.xsd` na lokální kopie.

Submoduly v `src/includes/` (naklonovat pomocí `git submodule update --init`): `log-xpc-lib` (poskytuje `xlog:store`, používaný všude pro ladicí výstupy), `docx-xpc-lib` (DOCX → XML), `xdoc-xpc-lib` (dokumentace XProc).

## Spouštění pipelinů

Všechny vstupní body jsou v `src/run/xproc/`; vstupy a výchozí hodnoty voleb jsou relativní cesty, takže nejsou potřeba žádné argumenty:

```powershell
Morgana src/run/xproc/alto-for-ndk.xpl          # hlavní: generuje XSD pro NDK + přehledy HTML
xmlcalabash src/run/xproc/alto-for-ndk.xpl      # totéž pomocí Calabashe
```

- `alto-for-ndk.xpl` – čte `src/schemas/alto/{4.4,4.2,2.0}/*.xsd` a `src/data/ndk-dmf-data.xml`, zapisuje `src/schemas/ndk/alto/<ver>/alto-<ver>-ndk.xsd` a `src/schemas/ndk/alto/<ver>.html`. **Přepisuje tyto commitnuté výstupy.**
- `download-schemas.xpl` – řízený souborem `src/data/standards.xml`. Krok stahování je vypnutý (`p:use-when="false()"`); v současnosti jen znovu vytváří soubory katalogů z výpisu adresáře `src/schemas` (`directories-to-catalog.xsl`).
- `local.ndk-dmf-to-xml.xpl` – převádí dokument DMF ve formátu Word (`src/input/docx/local.NDK_ALTO_XML.docx`) do `src/data/ndk-dmf-data.xml`. Krok DOCX→XML je vypnutý pomocí `use-when="false()"`; místo něj se načte uložený mezivýsledek `src/_/debug/ndk-dmf-to-xml/0001-local.NDK_ALTO_XML.xml` a spustí se řetězec `src/xslt/ndk-dmf-to-xml/*.xsl` (fix-errors → xml-to-data → identify-ID-attribute-in-details → combine-attributes-data → move-attributes-details → reorder-elements). Soubory `local.*` git ignoruje.

## Architektura

Tok dat: DMF `.docx` → `src/data/ndk-dmf-data.xml` (jmenný prostor `https://www.daliboris.cz/ns/xproc/ndk`: `element/name`, `description/details`, `obligation`, `cardinality`, `attributes/attribute`) → sloučení do každého ALTO XSD.

Hlavní krok `ndk:xsd-for-ndk` (`src/xproc/xsd-for-ndk.xpl`, `p:library`) pro každé vstupní XSD:
1. `src/xslt/ndk/alto-add-ndk-restrictions.xsl` – aplikuje vlastní omezení NDK. Jsou napevno v parametrech šablony: `$changes` (minOccurs/maxOccurs apod. u elementů/atributů, buď globální `item` pro styl ALTO 2.0, nebo `items[@type=<complexType>]` pro ALTO 4.x, které strukturuje XSD pomocí pojmenovaných typů), `$simpleTypes` a `$complexTypes` (nové/nahrazené typy, např. vzory ID). Omezení se mění úpravou těchto parametrů. Šablona váže prefix `alto` na jmenný prostor v4 – na to je třeba myslet při přidávání pravidel, která mají fungovat pro 2.0 nebo 5.0.
2. `src/xslt/ndk/merge-mdf-data.xsl` – přidává k elementům (párovaným podle jména, klíč na `ndk:name`) `xsd:annotation/xsd:appinfo[@source='https://www.daliboris.cz/ns/xproc/ndk']` s popisy z DMF.
3. Uloží XSD a potom `src/xslt/ndk/alto-xsd-ndk-overview.xsl` vygeneruje přehledovou tabulku HTML (napodobuje tabulku z DMF) procházením odkazů na elementy a typy.

Názvy výstupních souborů a adresáře verzí se odvozují z názvu vstupního souboru (`alto-4-4.xsd` → verze `4.4`).

`src/xproc/xsd-for-ndk_.xpl` je starší samostatná varianta téhož pipelinu; aktivní je `xsd-for-ndk.xpl`.

## Konvence

- Pipeliny sdílejí společnou kostru (oddíly pro vstupní porty / výstupní porty / volby / proměnné / tělo) a volby `debug-path` + `base-uri`; ladění je zapnuté, když `debug-path` není prázdná. Mezivýsledky se zapisují pomocí `xlog:store ... step="N"` do `src/_/debug/<pipeline>/NNNN-<name>.xml` (git je ignoruje).
- Relativní cesty se převádějí pomocí `p:urify($path, $base-uri)`.
- Prefixy jmenných prostorů projektu: `ndk` = `https://www.daliboris.cz/ns/xproc/ndk`, `rm` = `https://www.daliboris.cz/ns/xproc/resource-manager`, `xlog` = `https://www.daliboris.cz/ns/xproc/logging/1.0`.
- Komentáře TODO v XSLT jsou částečně česky.
