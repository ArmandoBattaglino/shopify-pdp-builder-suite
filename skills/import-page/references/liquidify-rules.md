# Liquidify rules — HTML chunk → Liquid + schema

Leggi all'inizio della **Phase 5** della skill `import-page`. Spiega come
trasformare un `html_chunk` (estratto in Phase 4) in un file Liquid valido
con `{% schema %}` block, rispettando la **Editability Hard Rule**.

## Editability Hard Rule (riassunto)

Ogni testo, immagine, link, colore visibile nella sezione DEVE essere un
`setting` (top-level o block) del `{% schema %}`. **No testi hardcoded nel
markup.** Pieno riferimento in
`../create-new-pdp/references/section-schema-patterns.md`.

## Strategie

Hai 2 modi per generare il Liquid:

### Modo A — Sezione library match (preferito)

Se in Phase 4 la sezione aveva `suggested_library_slug !== null` E quella
sezione esiste nel workspace (verifica via
`mcp__working_suite__list_section_definitions`), allora:

1. Recupera il `liquid_source` della library section (es. `hero-basic@1.0.0`)
2. NON modificare il markup
3. Nel `{% schema %}` block, sostituisci i `default` dei settings whose `id`
   matcha le chiavi di `extracted_fields`
4. Push del file finale sul tema

**Esempio**: library `hero-basic` ha:
```liquid
{% schema %}
{"settings":[{"type":"text","id":"headline","default":"Compra ora"}]}
{% endschema %}
```
Se `extracted_fields.headline === "Addio occhiaie"`, sostituisci → `"default":"Addio occhiaie"`.

Vantaggi: la sezione è già A/B-testable, ha già il tracking analytics standard,
e l'operatore può ulteriormente editarla nel composer.

### Modo B — Custom workspace section (fallback)

Se non c'è library match, genera Liquid da zero a partire dall'HTML chunk.
Algoritmo (descrittivo, Claude lo applica come logica interna):

1. **Wrapper**: avvolgi il chunk in:
   ```liquid
   <section class="ws-import-{kind}-{short_hash}" data-ws-section="import-{kind}" data-ws-version="1.0.0">
     ...
   </section>
   ```
   dove `{short_hash}` = primi 8 char di `sha256(html_chunk)` per evitare collision di class name.

2. **Sostituzione field**: per ogni `extracted_fields[X]`, trova nel chunk il
   nodo corrispondente e sostituisci con `{{ section.settings.X | escape }}` (per
   tag testuali) o `{{ section.settings.X }}` (per richtext/html).
   - Immagini: `<img src="...">` → `<img src="{{ section.settings.X | image_url: width: 1600 }}">`
   - Link: `<a href="...">` → `<a href="{{ section.settings.X }}">`
   - Color inline: `style="background: #123456"` → `style="background: {{ section.settings.X }}"`

3. **Schema block**: alla fine del file, aggiungi un `{% schema %}` block:
   ```liquid
   {% schema %}
   {
     "name": "Import — {kind} ({title_for_human})",
     "tag": "section",
     "class": "ws-section ws-import-{kind}",
     "settings": [
       { "type": "<derived>", "id": "<from extracted_fields key>", "label": "<human label>", "default": "<extracted value>" }
       ...
     ]
   }
   {% endschema %}
   ```

4. **Derivazione setting type** (regole):
   - Field key contiene `image` o value è URL CDN immagine → `image_picker`
   - Field key contiene `url` o value matcha `^https?://` → `url`
   - Field key contiene `color` o value matcha `^#[0-9a-f]{3,6}$` → `color`
   - Field value lungo (> 200 char) o contiene `<br>` o tag HTML → `richtext`
   - Field value lungo (50-200 char) senza HTML → `textarea`
   - Field value corto < 50 char → `text`

5. **CTA tracking** (se la sezione ha CTA):
   - Aggiungi setting `cta_id` (type `text`, default `import-{kind}-cta-{position}`,
     label "ID stabile CTA"), come da
     `../create-new-pdp/references/analytics-instrumentation.md`
   - Aggiungi attributo `data-wsa-cta-id="{{ section.settings.cta_id | escape }}"`
     all'`<a>` o `<button>` della CTA
   - Aggiungi inline script per `wsa.track('cta_click', ...)` come pattern v1

## Blocks (per sezioni ripetibili)

Se il `kind` è `usp_grid`, `faq`, `reviews`, `testimonials`, `features_list`,
`comparison_table`, le sotto-unità (colonne, FAQ, recensioni) devono diventare
`blocks` nello schema, non settings top-level.

Esempio per `usp_grid` con 3 colonne identificate in Phase 4:
```liquid
{% for block in section.blocks %}
  <article {{ block.shopify_attributes }}>
    <img src="{{ block.settings.icon | image_url: width: 128 }}" alt="">
    <h3>{{ block.settings.title | escape }}</h3>
    <p>{{ block.settings.description | escape }}</p>
  </article>
{% endfor %}

{% schema %}
{
  "name": "Import — USP Grid",
  "blocks": [{
    "type": "usp",
    "name": "USP",
    "settings": [
      {"type":"image_picker","id":"icon","label":"Icona"},
      {"type":"text","id":"title","label":"Titolo"},
      {"type":"textarea","id":"description","label":"Descrizione"}
    ]
  }],
  "presets": [{
    "name": "Import — USP Grid",
    "blocks": [
      {"type":"usp","settings":{"title":"<extracted col 1 title>","description":"<extracted col 1 desc>"}},
      {"type":"usp","settings":{"title":"<extracted col 2 title>","description":"<extracted col 2 desc>"}},
      ...
    ]
  }]
}
{% endschema %}
```

I `blocks` nel preset popolano la sezione con i dati estratti al primo render —
poi l'operatore può aggiungere/rimuovere/modificare dall'editor Shopify o dal
composer Working Suite.

## App blocks (Katching, ReCharge, Judge.me) — manuali in v1

NON aggiungere `@app` blocks automaticamente in questa skill v1. La detezione
avverrà in v2. Per ora la skill non rileva e non riproduce app blocks — segnala
solo nel riepilogo Phase 6 che ci sono potenzialmente app blocks da
reinstallare manualmente.

## Validazione finale

Prima del push (Phase 5.3), valida che il Liquid generato:
1. Ha il blocco `{% schema %}` ben formato (JSON.parse non lancia)
2. Ogni `setting.id` referenziato nel markup esiste nello schema
3. Niente testi hardcoded fuori dai default settings/blocks
4. Niente classi CSS che collidono con il tema esistente (usa sempre prefisso
   `ws-import-` o `ws-section-`)

Se la validazione fallisce, segnala in Phase 6 e proponi all'operatore di
escludere quella sezione (puoi mettere uno spazio vuoto come placeholder).
