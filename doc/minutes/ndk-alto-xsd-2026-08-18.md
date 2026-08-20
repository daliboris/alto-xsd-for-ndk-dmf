# Schůzka 2026-08-17

Přítomní: Jan Bilwachs (JB), Boris Lehečka (BL), Filip Pavčík (FP); on-line

## Stav úprav ALTO XSD

- modifikace a upřesnění definovaná v DMF jsou zanesená do XSD ALTO (4.4 a 2.0)
  - je potřeba zkontrolovat, jestli jsou úpravy všechny a správně zanesené
  - vznikla přehledová tabulka, která se snaží kopírovat tabulku z DMF
    - přehledová tabulka zatím nefunguje pro verzi 4.4, která používá jiný způsob strukturování XSD
- výsledek úprav XSD ALTO, včetně zdrojových kódů transformací a pomocných dat, se postupně objeví na [GitHubu](https://github.com/daliboris/alto-xsd-for-ndk-dmf)

## K dořešení

- při posuzování kardinality jednotlivých elementů je potřeba vycházet z definice DMF, kardinalita uvedená v tabulce pochází z ALTO a je pouze orientační; např.
  - v tabulce je pro `<Page>` uvedena kardinalita `1-n` a Povinnost = `M`, jinde se píše, že `ALTO XML i OCR TXT budou vznikat pro úroveň stránky`, z čehož vyplývá, že kardinalita pro DMF musí být `1-1`
- z formulací DMF není vždy zřejmé, jak má vypadat výsledné ALTO, např.
  - > ALTO XML soubor pro zcela prázdné stránky bude obsahovat element /alto/Layout/Page/PrintSpace, ten ovšem nebude obsahovat tyto podelementy: TextBlock/…
  - v dostupných ALTO prázdných stránek se používá kombinace
    - [PrintSpace/Illustration](https://kramerius5.nkp.cz/search/api/v5.0/item/uuid:31daf42a0ca8b109e45ba21dcd80bfa8/streams/ALTO)
    - [Page bez PrintSpace](https://kramerius5.nkp.cz/search/api/v5.0/item/uuid:35a49e00-8f3c-11e7-9b05-5ef3fc9ae867/streams/ALTO)
    - [Page/BottomMargin](https://kramerius5.nkp.cz/search/api/v5.0/item/uuid:19de3398-91b0-11f1-9d92-0aa6f682e2ae/streams/ALTO)
    - první dva případy odpovídají specifikaci DMF (neobsahují `TextBlock`), ale věcně nedávají smysl (ilustrace nebo spodní okraj mají stejné rozměry jako celá stránka)
      - případy s prázdnou stránkou (`<PrintSpace>` bez podřízených elementů) a stránkou s (ne)textovým obsahem (`<PrintSpace>` s podřízenými elementy) bude možné kontrolovat pomocí pravidel Schematronu (dají se zakomponovat do XSD ALTO)
- upravené ALTO pro NDK formalizuje podobu atributů `@ID`, např. `P\d{4}_ST\d{4}` pro element `<String>`, v existujících ALTO bude ale podoba odlišná, protože DMF definuje podobu identifikátoru pomocí příkladu:
  - > ID – obsahuje unikátní sekvenční číslo řetězce na stránce, např. "P3_ST0001" (strana 3, řetězec 1)
  - v další verzi DMF by se měly definice pomocí příkladů nahradit přesnou specifikací podoby identifikátoru
- u `<TextBlock/@ID>` je potřeba povolit sufix `_SUB`, který se používá pro komponovaný blok
- DMF u elementu `<MeasurementUnit>` povoluje hodnotu `dpi`, která není definovaná v ALTO 2.0 ani 4.4
  - **FP zkontroluje** výskyt `dpi` v datech
- v digilince NK se využívá ALTO 4.2
  - není problém vygenerovat úpravy pro jakoukoli verzi ALTO

## Úkoly a výhledy

- **BL:** další úpravy NDK ALTO XSD na začátku září
- **BL:** bylo by možné připravit XSLT šablonu, která upraví existující ALTO tak, aby vyhovovalo novým požadavkům, např. přespáním identifikátorů do požadované podoby, doplnění chybějících atributů apod.
- **JB:** validátor bude muset zvolit způsob validace na základě jmenného prostoru vstupního dokumentu
