# Library mapping — quando library vs custom

Leggi all'inizio di **Phase 5.2** della skill `import-page`. Decide se una
sezione identificata in Phase 4 va riusata dalla library Working Suite o
generata come custom workspace-private.

## Regola generale

Per ogni sezione in input (con `suggested_library_slug` set da Phase 4):

1. Se `suggested_library_slug !== null` → verifica esistenza nel workspace
   - Chiama `mcp__working_suite__list_section_definitions(workspace_id, slug=suggested_library_slug)`
   - Se ritorna ≥ 1 row → **usa library**
   - Se ritorna 0 row → procedi come `suggested_library_slug === null`
2. Se `suggested_library_slug === null` → **genera custom** workspace-private

## Lookup dettagliato

Una sezione library esiste se in `public.section_definitions` c'è una row con:
- `slug = <suggested_library_slug>`
- `status = 'published'`
- `visibility = 'marketplace'` OPPURE (`visibility = 'workspace'` AND `workspace_id = <workspace_id from context>`)

Se più versioni esistono (es. `hero-basic@1.0.0` e `hero-basic@1.1.0`), prendi
**l'ultima** per ordine semver. La selezione di una versione precedente è
post-import nel composer (gate 3-1+).

## Custom section generation

Se non c'è library match, la sezione diventa una NUOVA `section_definition`
privata del workspace. Le proprietà:
- `slug`: derivato come `import-<kind>-<short_hash>` dove `short_hash` =
  primi 8 char di `sha256(html_chunk)`. Garantisce idempotenza: due chunk
  identici producono la stessa sezione (evita duplicati su re-import).
- `version`: `1.0.0` per il primo. Se esiste già con `slug` collision MA
  `liquid_source` differente, bump a `1.1.0`, `1.2.0`, ecc.
- `visibility`: `workspace`
- `workspace_id`: dal contesto
- `status`: `published`
- `category`: derivata dal `kind`:

| kind | category |
|---|---|
| `hero` | `hero` |
| `usp_grid` | `benefit` |
| `product_main`, `product_gallery` | `feature` |
| `reviews`, `testimonials` | `testimonial` |
| `faq` | `faq` |
| `cta_band` | `cta` |
| `features_list` | `feature` |
| `comparison_table` | `comparison` |
| `image_text`, `rich_text` | `other` |
| `unknown` | `other` |
| `header` | `header` |
| `footer` | `footer` |

- `page_kinds`: `[<target_page_type from context>]` (singleton, l'operatore
  può estendere dopo dal builder UI)
- `liquid_source`: il Liquid generato da `liquidify-rules.md`
- `fields`: derivati dallo schema generato

Per CREARE la section_definition: chiama `mcp__working_suite__create_private_section`
con i campi sopra. Restituisce `{id, slug, version}`.

## Verifica idempotenza (re-import della stessa pagina)

Se l'operatore lancia l'import di nuovo sulla stessa URL, vogliamo che:
- Le sezioni custom NON vengano duplicate nel section_definitions (gli stessi
  html_chunk → stesso `short_hash` → matching slug `import-<kind>-<hash>`)
- I file Liquid sul tema vengano sovrascritti (selective push idempotente)
- Una NUOVA page_schema viene creata (no merge automatico — l'operatore può
  archiviare la vecchia manualmente)

Implementation: prima di creare una custom section_definition, controlla se
esiste già:
```
mcp__working_suite__list_section_definitions(workspace_id, slug=`import-{kind}-{hash}`, version=`1.0.0`)
```
Se sì → riusa quella (no nuova INSERT).

## Naming dei file Liquid

Pattern v1 (riusato): `<store_prefix>-<page_type_short>-<NN>-<section_slug>.liquid`

| Pezzo | Esempio | Note |
|---|---|---|
| `store_prefix` | `glow` (per glowria.shop) | Primi 4 char dello store_domain prefix |
| `page_type_short` | `pdp`, `adv`, `lst`, `qz`, `home`, `pg` | Vedi convention v1 |
| `NN` | `01`, `02`, …, `99` | Posizione zero-padded della sezione nella pagina |
| `section_slug` | `hero-basic` (library) o `import-hero-a1b2c3d4` (custom) | Dal section_definition.slug |

Esempio completo: `glow-pdp-03-import-reviews-7f3a9c2b.liquid`

## Pre-flight check before push

Prima di iniziare il push delle sezioni in Phase 5.3, verifica:
1. Lo store ha Custom App connessa (`mcp__working_suite_shopify_admin__check_connection`)
2. Il tema main esiste e non è in "Theme editor lock" (controlla via Admin API)
3. Il template JSON di destinazione (es. `templates/product.crema.json`) NON
   esiste già — se esiste, **STOP** e chiedi all'operatore via AskUserQuestion:
   - "Sovrascrivi (rimpiazza la pagina esistente sullo store)"
   - "Cambia slug (es. crema-v2)"
   - "Indietro"

Se l'operatore sceglie "Cambia slug" → cambia `target_slug` in `${target_slug}-v2`
(o successivo numero libero) e procedi.
