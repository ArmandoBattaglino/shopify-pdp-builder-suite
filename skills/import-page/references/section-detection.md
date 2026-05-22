# Section detection — DOM → sezioni semantiche

Leggi all'inizio della **Phase 4** della skill `import-page`. Output: array di
section objects per la Phase 5.

## Regola di splitting

Una "sezione" è un blocco visivo riconoscibile della pagina che assolve a UN
ruolo. Esempi:
- Hero (titolo principale + immagine sfondo + CTA)
- Griglia di USP (3-6 colonne con icona/titolo/desc)
- Galleria prodotto (immagini multiple)
- Recensioni (carousel o lista)
- FAQ accordion
- CTA band (fascia con bottone + sfondo colorato)
- Trust badges (loghi pagamento/certificazioni)
- Header / Footer (skippare in v1 — l'utente li ha già nel tema)

Non spezzare in modo troppo fine: una "sezione USP" che ha 3 colonne con icona
+ titolo + descrizione è UNA sezione, non 3 (le 3 colonne diventano `blocks`
ripetibili dentro la sezione).

Non includere mai più di **un kind** per sezione. Se nel DOM trovi qualcosa di
ambiguo (es. una hero che dentro ha già una griglia USP), separa in 2.

## Strategia per piattaforma

### Quando `detected_platform === "shopify_native"`

Il DOM ha già `<section class="shopify-section" id="shopify-section-XXX">`.
- Ogni `<section class="shopify-section">` top-level = 1 sezione nostra.
- L'`id` `shopify-section-XXX` contiene spesso il section_id Shopify originale
  (es. `shopify-section-hero` → kind `hero`).

### Quando `detected_platform === "pagefly"` o `"gempages"`

Il DOM ha "rows" / "containers" col markup tool-specifico:
- PageFly: `<div data-pf-type="container">` o `<div class="pf-X-container">`
- GemPages: `<div data-gp-component="container">`

Usa quelle root come confini. Dentro ogni root, leggi il pattern di contenuto
per assegnare il `kind`.

### Quando `detected_platform === "funnelish"`, `"heyflow"`, o `"unknown"`

Niente markup tool-specifico. Strategie:
1. **Landmark roles**: `<header>`, `<main>`, `<section>`, `<footer>`.
2. **Headings**: ogni `<h1>` o `<h2>` di solito apre una nuova sezione.
3. **Background color/image changes**: visuale → split. Cerca `<div>` con
   `style="background: ..."` o classi che indicano stacchi visivi.
4. **Container divs con id semantico**: `id="reviews"`, `id="faq"`, ecc.

Se la pagina è completamente piatta senza segnali (raro), produci una sezione
unica `kind: "rich_text"` con tutto il body.

## Output schema per ogni sezione

Ogni sezione è un oggetto JSON conforme a:

```json
{
  "kind": "<one of: hero, usp_grid, product_main, product_gallery, reviews, faq, cta_band, testimonials, features_list, comparison_table, rich_text, image_text, footer, header, unknown>",
  "title_for_human": "Hero con immagine sfondo e CTA",
  "html_chunk": "<section>...</section>",
  "extracted_fields": {
    "<field_id>": "<value>",
    ...
  },
  "suggested_library_slug": "hero-basic" | null,
  "confidence": 0.85
}
```

### `kind` — vocabolario chiuso

| kind | quando |
|---|---|
| `hero` | Sezione iniziale grande con headline + CTA + (di solito) immagine sfondo |
| `usp_grid` | Griglia di 2-6 colonne con icona + titolo + descrizione breve |
| `product_main` | Solo per page_type='pdp': info prodotto Shopify (titolo, prezzo, ATC) |
| `product_gallery` | Carousel/grid immagini del prodotto |
| `reviews` | Recensioni clienti (testimonials con stelle/rating) |
| `testimonials` | Citazioni "umane" (senza stelle, più narrativi) |
| `faq` | Lista domande/risposte (accordion o non) |
| `cta_band` | Fascia con CTA prominente, di solito a metà o fondo pagina |
| `features_list` | Lista lunga di feature/benefit (non grid) |
| `comparison_table` | Tabella di confronto (vs competitors o tier pricing) |
| `image_text` | Sezione classica "immagine a sinistra/destra + testo" |
| `rich_text` | Qualsiasi blocco testuale lungo non meglio classificabile |
| `header` | Top nav (di solito skippare — l'ha già il tema) |
| `footer` | Bottom (di solito skippare) |
| `unknown` | Non riesco a classificare con confidence > 0.5 |

### `extracted_fields` — cosa estrarre per kind

| kind | fields tipici |
|---|---|
| `hero` | `headline`, `subheadline`, `background_image`, `cta_label`, `cta_url`, `cta_id`, `text_color` |
| `usp_grid` | `section_title`, `columns_per_row` + blocks: `icon`, `title`, `description` |
| `faq` | `section_title` + blocks: `question`, `answer` (richtext) |
| `reviews` | `section_title` + blocks: `author_name`, `rating`, `text` |
| `cta_band` | `headline`, `cta_label`, `cta_url`, `cta_id`, `background_color` |
| `image_text` | `headline`, `body`, `image`, `image_position` (left/right) |
| `rich_text` | `content` (richtext, l'intero blocco testuale) |

Per kind più rari (`comparison_table`, `features_list`, `unknown`), estrai TUTTI i
testi visibili come campo `content` (richtext) — si può rifinire nel composer.

### `suggested_library_slug`

Mappare il `kind` con il library slug più probabile (la library Working Suite
contiene sezioni curate, vedi `library-mapping.md` per il lookup completo).

In v1 la library contiene solo 3 sezioni base:
- `hero-basic` → mappa da kind `hero`
- `usp-grid` → mappa da kind `usp_grid`
- `faq-accordion` → mappa da kind `faq`

Per gli altri kind in v1: `suggested_library_slug: null` (= genera custom workspace).

### `confidence` (0.0-1.0)

Stima soggettiva di quanto sei sicuro della classificazione `kind` + dei `fields`
estratti. Threshold consigliata:
- ≥ 0.8 = sicuro
- 0.5-0.8 = incerto, l'operatore probabilmente vuole rivedere nel composer
- < 0.5 = molto incerto, usa kind `unknown` invece

In Phase 4 la skill mostra le sezioni con confidence bassa all'operatore evidenziandole.

## Cosa NON fare

- ❌ Non spezzare in più sezioni quello che visivamente è UNA (es. la USP-grid
  va come 1 sezione, le colonne come blocks)
- ❌ Non includere `<script>` o `<style>` tag negli html_chunk (puliscili prima)
- ❌ Non estrarre testi placeholder come "Lorem ipsum" come field default — lascia
  vuoto e segnala in `confidence`
- ❌ Non inventare campi che non esistono nel DOM. Se l'HTML ha solo `headline` e
  niente `subheadline`, omettilo dall'oggetto invece di metterlo con stringa vuota
