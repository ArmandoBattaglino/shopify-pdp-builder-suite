---
name: acquire-competitor-page
description: Acquisisce UNA SINGOLA pagina competitor (PDP, advertorial, listicle, quiz, home) e la salva come TEMPLATE editabile nella LIBRARY di Working Suite — page_schema + section_definitions Liquid riutilizzabili. È un'azione READ-ONLY sullo store Shopify dell'utente: NON tocca mai il tema (niente check_connection, niente push_theme_asset, niente theme list/pull/push), NON chiede "quali pagine", studia un solo source_url e si ferma. Da usare quando trovi una pagina competitor che funziona e vuoi catturarla come modello riutilizzabile da adattare/pubblicare in un secondo momento con un'altra azione.
---

# /acquire-competitor-page — Acquire single page → Library

Studi **UNA sola pagina competitor** (`source_url`) e la salvi come **template editabile in libreria** (`page_schema` + `section_definitions`). **NON pubblichi nulla, NON tocchi il tema dell'utente.** Finisci con un messaggio di "salvato in libreria" e ti fermi.

Questa skill è il gemello "acquire-only" di `/clone-competitor-store`: riusa la stessa logica di **studio competitor** + **editable-liquidify**, ma per **una pagina sola** e con destinazione **libreria** (non tema live). Riusa anche la logica **library-save** di `/import-page` (Phase 5.2 + 5.4-5.5), senza nessun push asset.

## Cosa NON fa MAI questa skill (HARD RULES — leggere per prime)

- 🚫 **MAI `check_connection`** (`mcp__working_suite_shopify_admin__check_connection`). Non serve: non scriviamo sul tema, quindi non ci interessa se la connessione Admin è valida. `store_id` qui è SOLO il *bucket di libreria* (a quale store appartiene il template salvato), non un puntatore per chiamare Shopify.
- 🚫 **MAI `push_theme_asset`** (`mcp__working_suite_shopify_admin__push_theme_asset`). Non scriviamo nessun file `.liquid`/`.json` sul tema. Tutto resta in libreria (DB Working Suite).
- 🚫 **MAI** `theme list` / `theme pull` / `theme push` / Shopify CLI / `curl` verso l'Admin API dello store / `.env` / Theme Access token / `SHOPIFY_CLI_THEME_TOKEN`. Zero contatto con lo store Shopify dell'utente.
- 🚫 **MAI** chiedere "quali pagine vuoi replicare?" (è la logica multi-pagina di `/clone-competitor-store`, qui vietata). Una sola pagina = `source_url`. Niente loop su più pagine, niente orchestrazione funnel→PDP→home.
- 🚫 **MAI** creare il prodotto Shopify, assegnare template, sovrascrivere `index.json`, creare Pages in Admin. Niente "caso speciale per tipo pagina" di clone (6.x.7): finisce in libreria, punto.
- ✅ **Editabilità è legge** (vedi Phase 3): ogni testo/immagine/link/colore visibile diventa un `setting` nel `{% schema %}`. Le `section_definition` arrivano in libreria SEMPRE editabili (`liquid_source` con `{% schema %}` → `parseLiquidSchema` deriva i campi). MAI `fields:[]`.
- ✅ **Anti-hallucination**: replica SOLO quello che si vede/legge nella pagina. Niente sezioni "tipiche", niente testi inventati. Se manca qualcosa, lo ometti, non lo inventi.
- ✅ **Solo `mcp__working_suite__*`** (tool di libreria) per scrivere. Se mancano → STOP (Phase 1). NESSUN fallback CLI/curl.

> **Convenzione nomi tool MCP.** I tool di libreria sono esposti al modello come `mcp__working_suite__*` (gli `name` interni sono `working_suite.section-definitions.list`, `working_suite.section-definitions.create-private`, `working_suite.page-schemas.create`, `working_suite.section-instances.add`). I tool che toccano lo store sono `mcp__working_suite_shopify_admin__*` (check_connection, push_theme_asset) — **questa skill non li usa MAI**. Se nella tua sessione i tool di libreria appaiono con un prefisso diverso, usa quel prefisso ma MAI i tool `*_shopify_admin__*`.

---

## Input (param-block di sessione)

Working Suite passa i parametri della sessione come righe `key: value` dopo la slash command (stessa convenzione delle altre skill). Leggi:

```
source_url:        <https://… — URL della SINGOLA pagina competitor>   (OBBLIGATORIO)
target_page_type:  "pdp" | "advertorial" | "listicle" | "quiz" | "home"  (OBBLIGATORIO)
store_id:          <uuid workspace_stores — SOLO bucket di libreria, NON per chiamare Shopify>
workspace_id:      <uuid workspace>
store_domain:      <store>.myshopify.com  (solo etichetta/contesto, non per fetch)
```

**Validazione input (Phase 0):**
- Se manca `source_url` → STOP: "Manca il `source_url` della pagina da acquisire. Rilancia l'azione indicando l'URL della pagina competitor."
- Se manca `target_page_type` o non è in {pdp, advertorial, listicle, quiz, home} → STOP e chiedilo.
- Se manca `workspace_id` → STOP: "Manca `workspace_id` dal contesto di sessione; riapri la chat builder."
- `store_id` serve come **bucket** (`page_schema.store_id`). Se manca → STOP: "Manca `store_id` dal contesto; riapri la chat builder dallo store giusto." (Lo usiamo SOLO per `create_page_schema`, MAI per chiamare lo store.)

---

## Reference da leggere (prima di iniziare le fasi)

Riusa la logica già scritta nelle skill esistenti — **non reinventarla**:

- **Studio competitor + editable-liquidify di UNA pagina** → leggi
  `../clone-competitor-store/SKILL.md` (+ la sua cartella `references/`, in particolare
  `competitor-discovery.md`, `visual-replication.md`, `css-scraping.md`,
  `editability-and-app-blocks.md`, `pdp-main-configuration.md`).
  Riusa **solo**: la mappatura sezioni (clone 6.x.2.1), il diagnostic PDP main-custom (clone 6.x.2.0),
  l'estrazione palette/font/logo (Fase 4), e la build sezione editabile con `{% schema %}` (clone 6.x.5.b,
  blocchi A/B/C). **Ignora** tutta l'orchestrazione multi-pagina, lo Step 1→Step 2 literal/IT, il
  "quali pagine", i push sul tema, e la creazione product/page/home assignment (clone 6.x.7).
- **Editability hard rule + liquidify HTML→Liquid** → `../create-new-pdp/references/section-schema-patterns.md`
  e (se accessibile sul box) `import-page/references/liquidify-rules.md`,
  `import-page/references/library-mapping.md`,
  `import-page/references/section-detection.md`.
- **Library-save via MCP** → leggi la Phase 5 di `import-page` (sul VPS):

  ```bash
  ssh -o ConnectTimeout=20 root@46.101.212.167 'cat /srv/shopify-pdp-builder/skills/import-page/SKILL.md'
  ```

  Riusa **Phase 5.2** (match library via `list_section_definitions`, altrimenti
  `create_private_section` con `liquid_source` che contiene il `{% schema %}` → i campi sono DERIVATI da
  `parseLiquidSchema`, quindi sempre editabili, MAI `fields:[]`) + **Phase 5.4-5.5**
  (`create_page_schema` poi `add_section_instance` per ogni sezione in ordine). **Ignora** la Phase 5.1
  (`check_connection`), la Phase 5.2.c (`push_theme_asset`), la 5.3 (template JSON sul tema) e la 5.5
  `finalize_import` (qui non c'è un `import_id`).

---

## Flow (5 fasi, in sequenza)

### Phase 1 — MCP guard

**Goal:** assicurarti di poter scrivere in libreria, e di NON aver bisogno di toccare lo store.

1. Verifica che i tool di libreria `mcp__working_suite__*` siano disponibili (in particolare
   `list_section_definitions`, `create_private_section`, `create_page_schema`, `add_section_instance`).
2. Se NON disponibili → **STOP** con il messaggio esatto:
   > "MCP non configurato; riapri la chat builder."
   NON fare fallback a CLI/curl. NON tentare nulla via `ssh`/`curl` verso lo store.
3. **Non chiamare** `check_connection`: questa skill non scrive sul tema, quindi non verifica la
   connessione Admin. Se ti viene la tentazione di "controllare prima la connessione store" → fermati,
   è esplicitamente vietato.

Conferma all'operatore in 1 riga: "Acquisirò la pagina come template di libreria. Non toccherò il tuo store Shopify." Poi vai a Phase 2.

---

### Phase 2 — Studio della pagina (READ-ONLY) → sezioni con campi estratti

**Goal:** capire la pagina `source_url` e produrre una lista ordinata di sezioni, ognuna con i suoi
`extracted_fields` (testo/immagine/link/colore che diventeranno `settings` editabili).

1. **Fetch/studio read-only del competitor.** Usa, in ordine di preferenza:
   - Il tool di studio MCP se presente nella tua sessione (`mcp__working_suite__study_schema` /
     `working_suite.clone.study-schema`): renderizza la pagina (post-hydration) + ritorna `Section[]`
     con `kind`, `extracted_fields`, `confidence`, `flags[]`, `suggested_library_slug` e uno screenshot.
     È il modo migliore per pagine JS-rendered.
   - Altrimenti `WebFetch` su `source_url` (prompt: "Restituisci l'HTML completo così com'è — non
     riassumere; mantieni tag, classi, `data-*`, `<style>`, `<script>`, `<meta>`"). Per il CSS pubblico
     vedi `clone-competitor-store/references/css-scraping.md` (i CSS sono asset pubblici serviti a
     qualsiasi browser).
   - Se l'utente preferisce, può salvare l'HTML della pagina (Chrome → Salva con nome → "Pagina web,
     completa") e passartelo: lo leggi con `Read`. Niente OCR sugli screenshot per i testi.

   ⚠️ Tutto qui è verso il **dominio competitor**, MAI verso lo store Shopify dell'utente. Nessuna
   chiamata all'Admin API dell'utente.

2. **Mappatura sezioni** (logica clone 6.x.2.1): ricostruisci la lista ordinata di sezioni visibili. Per
   ognuna salva `nn` (zero-padded), `role`/`kind` semantico, `notes` (cosa si vede). Anti-hallucination:
   se vedi 7 sezioni → 7 sezioni, niente di più.

3. **Diagnostic PDP main-custom** (SOLO se `target_page_type === "pdp"`, logica clone 6.x.2.0): cerca nei
   markup i signal `kaching-bundle`/`bundler`, `free-gift`/`mystery-gift`, `countdown`/`urgency`,
   `recharge`/`subscription`, `trust-strip`, accordion `<details>`/`INGREDIENTS`. Se ne trovi ≥1 → la
   sezione PDP `main` va modellata come **custom** (con slot `@app` per app esterne). Default in dubbio:
   custom.

4. **Estrazione campi per editabilità (REGOLA DURA):** per ogni sezione, ogni elemento visibile diventa
   un campo:
   - testo singola riga → `text`; multiriga → `textarea`; paragrafo con bold/link → `richtext`
   - immagine → `image_picker` (MAI `<img src>` hardcoded)
   - link/CTA → coppia `url` (link) + `text` (label)
   - colore (CTA/sfondo/heading) → `color`
   - liste ripetibili (FAQ, testimonial, bullet, card, gallery) → `blocks` con `type` dedicato + `max_blocks`
   Salva tutto in `sections[<NN>].extracted_fields`. I `default` dei campi = testo/colore **letterale**
   del competitor (così il template in libreria mostra subito il contenuto reale; l'adattamento al brand
   avviene dopo, con l'azione "Adatta al mio prodotto").

5. **Palette/font/logo** (clone Fase 4 + 6.x.5.0): estrai palette pagina-specifica (CSS `:root` vars,
   inline style, hex ricorrenti), font heading/body, eventuale logo URL del competitor. Servono come
   `default` dei `setting type:"color"` e per gli `@import` font nelle sezioni.

Mostra all'operatore la lista sezioni mappata + i campi principali. (Se vuoi confermare la struttura puoi
usare `AskUserQuestion`, ma NON chiedere "quali pagine" — la pagina è una sola, già data.)

---

### Phase 3 — Liquidify editabile di ogni sezione

**Goal:** per ogni sezione, produrre il `liquid_source` editabile (markup + `<style>` scoped +
`{% schema %}`) che andrà in libreria. **Nessun push sul tema** — questo è solo il sorgente che salveremo
come `section_definition`.

Per ogni sezione (logica clone 6.x.5.b, blocchi A/B/C + `editability-and-app-blocks.md` +
`section-schema-patterns.md`):

1. **Match library prima di creare custom** (logica import-page 5.2.a): chiama
   `list_section_definitions` (filtrando per `page_kind = target_page_type` e, se il prior ha dato un
   `suggested_library_slug`, per quello `slug`). Se esiste già una sezione di libreria adatta → riusala
   come base (riempi i suoi campi con gli `extracted_fields` come `default`) e segna la sezione come
   `library_match` (NON ricrei una private section per quella).

2. **Altrimenti, custom wrapped section** (import-page 5.2.b):
   - `short_hash` = primi 8 char di `sha256` del contenuto sorgente della sezione (es. dal markup o dal
     selettore competitor). Esempio: `sha256("<chunk>") → "9f3a1c0d…"` → `9f3a1c0d`.
   - Wrapper con classe scoped unica: `ws-acq-<kind>-<short_hash>` (tutte le classi CSS della sezione
     iniziano con questo prefisso → niente conflitti cross-section).
   - Markup mobile-first (375px → `@media min-width:750px` → `1200px`), `<style>` inline nel file
     (niente `assets/*.css` esterni, niente Tailwind/Bootstrap), font via `@import` Google Fonts.
   - **Tutti** i testi/immagini/link/colori resi via `{{ section.settings.X }}` / `{% for block in
     section.blocks %}` — MAI hardcoded nel markup.
   - **Se la sezione è la `main` di una PDP custom**: includi `{ "type": "@app" }` nei `blocks` dello
     schema (per Katching Bundles / subscription / recensioni), come da `pdp-main-configuration.md`.
   - `{% schema %}` completo con `settings`/`blocks`/`default` popolati dagli `extracted_fields`. Questo
     schema È la fonte dell'editabilità: i campi della libreria verranno **derivati** da qui via
     `parseLiquidSchema`.

Esempio scheletro di sezione (default = testi/colori letterali del competitor):

```liquid
<section class="ws-acq-{{ section_kind }}-{{ short_hash }}">
  <style>
    @import url('https://fonts.googleapis.com/css2?family=...');
    .ws-acq-{{ short_hash }}-cta {
      background: {{ section.settings.cta_bg | default: '#E36B6E' }};
      color: {{ section.settings.cta_text | default: '#ffffff' }};
    }
  </style>
  <h2>{{ section.settings.heading }}</h2>
  {{ section.settings.body }}
  {% if section.settings.hero_image %}
    <img src="{{ section.settings.hero_image | image_url: width: 1600 }}"
         alt="{{ section.settings.hero_image.alt | escape }}" loading="lazy">
  {% else %}
    <div class="img-placeholder">[Immagine — consigliato 1920×800]</div>
  {% endif %}
  <a class="ws-acq-{{ short_hash }}-cta"
     href="{{ section.settings.cta_url | default: '#' }}">{{ section.settings.cta_label }}</a>
</section>
{% schema %}
{
  "name": "<nome leggibile>",
  "tag": "section",
  "settings": [
    { "type": "text",         "id": "heading",   "label": "Titolo",     "default": "<testo competitor>" },
    { "type": "richtext",     "id": "body",      "label": "Paragrafo",  "default": "<p>…</p>" },
    { "type": "image_picker", "id": "hero_image","label": "Immagine" },
    { "type": "url",          "id": "cta_url",   "label": "Link CTA",   "default": "" },
    { "type": "text",         "id": "cta_label", "label": "Testo CTA",  "default": "<label competitor>" },
    { "type": "color",        "id": "cta_bg",    "label": "Colore CTA", "default": "<hex competitor>" }
  ],
  "blocks": [ { "type": "@app" } ],
  "presets": [ { "name": "<nome leggibile>" } ]
}
{% endschema %}
```

Salva per ogni sezione: `sections[<NN>].liquid_source`, `sections[<NN>].is_library_match`
(true/false), `sections[<NN>].library_slug`/`section_definition_id` (se match), `sections[<NN>].slug`
(per le custom: `acquire-<kind>-<short_hash>`), `sections[<NN>].field_values` (gli `extracted_fields`
da applicare come valori dell'istanza).

---

### Phase 4 — SAVE TO LIBRARY (no theme write)

**Goal:** persistere il template in libreria. **Nessun `push_theme_asset`, nessun template JSON sul
tema.** Solo i tool di libreria `mcp__working_suite__*`.

Esegui nell'ordine:

1. **Per ogni sezione custom (non library_match), idempotente:**
   - Prima `list_section_definitions` con `slug = "acquire-<kind>-<short_hash>"` (e
     `page_kind = target_page_type`). Se torna già 1 riga → riusa quel `section_definition_id` (idempotenza:
     non duplicare se la skill gira due volte sullo stesso URL).
   - Altrimenti `create_private_section`:
     ```
     mcp__working_suite__create_private_section
       {
         slug:         "acquire-<kind>-<short_hash>",   // lowercase kebab
         version:      "1.0.0",
         title:        "<nome leggibile sezione>",
         category:     "<kind>",                         // es. hero, faq, cta_band
         page_kinds:   ["<target_page_type>"],           // 1..6 entries
         liquid_source:"<liquid completo CON {% schema %}>",
         workspace_id: <workspace_id dal contesto>
       }
     ```
     ⚠️ **NON passare `fields`/`schema_json` a mano**: il tool li DERIVA dal `{% schema %}` del
     `liquid_source` con `parseLiquidSchema` → la sezione atterra SEMPRE editabile, MAI con `fields:[]`.
     Passa override espliciti solo se vuoi forzare un set di campi diverso (qui: non serve).
   - Salva l'`id` ritornato in `sections[<NN>].section_definition_id`.

2. **Crea il page_schema** (logica import-page 5.4) — questo è il "template" in libreria:
   ```
   mcp__working_suite__create_page_schema
     {
       workspace_id: <workspace_id>,
       store_id:     <store_id>,                 // SOLO bucket di libreria, NON usato per chiamare Shopify
       page_type:    <target_page_type>,         // pdp|advertorial|listicle|quiz|home
       slug:         <slug kebab del template, es. "acquire-<dominio>-<kind>">,
       title:        <titolo leggibile del template>,
       created_via:  "skill_clone_url",          // enum valido per un acquire-da-URL (alt. "manual")
       source_metadata: {
         acquired_from: <source_url>,
         acquire_skill: "acquire-competitor-page",
         sections_count: <N>
       }
     }
   ```
   > **`created_via`** — i valori ammessi dall'enum (CHECK su `page_schemas.created_via`) sono:
   > `skill_clone_template`, `skill_compose_marketplace`, `skill_clone_url`, `skill_convert_pagefly`,
   > `skill_convert_gempages`, `skill_convert_screenshot`, `skill_create_quiz`, `manual`. Per un acquire
   > da URL competitor usa **`skill_clone_url`** (oppure `manual` come ripiego). NON inventare valori
   > fuori da questo enum (l'insert fallirebbe sul CHECK).

   Conflitto slug (una row ATTIVA con stesso `store_id`+`page_type`+`slug` esiste già) → cambia slug
   aggiungendo suffisso `-v2`/`-v3` e riprova (NON sovrascrivere).
   Salva l'`id` ritornato in `page_schema_id`.

3. **Aggiungi le section_instance in ordine** (logica import-page 5.5) — una per sezione, nell'ordine di
   apparizione:
   ```
   mcp__working_suite__add_section_instance
     {
       page_schema_id:        <page_schema_id>,
       liquid_filename:       "<slug-o-library-slug>.liquid",   // nome file della sezione
       section_definition_id: <id della section_definition>,    // private appena creata o library match
       position:              <0-based, in ordine; o ometti per append>,
       field_values:          { ...extracted_fields della sezione... }
     }
   ```
   Una `add_section_instance` per ogni sezione, `position` crescente (0,1,2,…) per preservare l'ordine.

⚠️ In NESSUN punto di questa fase chiami `push_theme_asset`, scrivi `templates/*.json` sul tema, o tocchi
lo store. Tutto vive in libreria (DB Working Suite). Non c'è `finalize_import` (non esiste un `import_id`
in questo flow).

---

### Phase 5 — STOP (handoff)

Quando tutte le sezioni sono state salvate (private section create/riusate + page_schema + tutte le
section_instance), **fermati** con questo messaggio (sostituisci i placeholder):

```
Template «<title>» salvato in libreria (tipo <page_type>).
NON ho toccato il tuo store. Per pubblicarlo usa l'azione «Adatta al mio prodotto» / «Pubblica».
```

Aggiungi un riepilogo conciso (testo, niente AskUserQuestion):

```
- Pagina acquisita:   <source_url>
- Tipo template:      <target_page_type>
- Sezioni salvate:    <N> (<N_library> da libreria · <N_custom> custom workspace)
- page_schema_id:     <id>

Nessun file scritto sul tema Shopify. Niente è stato pubblicato.
```

Poi termina. Non proporre push, non proporre creazione prodotto/page, non chiedere altre pagine.

---

## Troubleshooting rapido

| Sintomo | Causa | Fix |
|---|---|---|
| Tool `mcp__working_suite__*` assenti | MCP non configurato per la sessione | STOP: "MCP non configurato; riapri la chat builder." Niente CLI/curl. |
| Tentazione di chiamare `check_connection` | Abitudine dalle skill che scrivono sul tema | NON farlo: questa skill è read-only sullo store. `store_id` è solo il bucket di libreria. |
| `WebFetch` del competitor torna vuoto | Pagina JS-rendered | Usa il tool di studio MCP (render reale) o chiedi all'utente l'HTML salvato. MAI fetchare lo store dell'utente. |
| `create_page_schema` fallisce sul CHECK `created_via` | Valore fuori enum | Usa `skill_clone_url` (o `manual`). Vedi enum in Phase 4. |
| `create_page_schema` ritorna `conflict` | Slug già attivo per store+page_type | Cambia slug (`-v2`/`-v3`) e riprova. Non sovrascrivere. |
| `create_private_section` ritorna `conflict` | `(slug, version, workspace_id)` già esiste | Idempotenza: riusa la section esistente (la trovi con `list_section_definitions slug=…`). |
| Sezione atterra con `fields:[]` (non editabile) | `{% schema %}` mancante o malformato nel `liquid_source` | Correggi lo schema Liquid (il tool deriva i campi da lì); ricrea. |

---

## File correlati (references)

Questa skill **non** ha una propria cartella `references/`: riusa quelle delle skill esistenti.

- `../clone-competitor-store/references/competitor-discovery.md` — studio competitor, estrazione palette/font/logo (Fase 4 clone). Riusato per UNA pagina.
- `../clone-competitor-store/references/visual-replication.md` — ricostruzione design 1:1 da screenshot+URL (mobile-first, classi scoped).
- `../clone-competitor-store/references/css-scraping.md` — scaricare CSS pubblico del competitor (read-only verso il competitor, non verso lo store).
- `../clone-competitor-store/references/editability-and-app-blocks.md` — editability hard rule + `@app` block per PDP.
- `../clone-competitor-store/references/pdp-main-configuration.md` — diagnostic PDP main-custom + slot `@app`.
- `../create-new-pdp/references/section-schema-patterns.md` — pattern schema editabile.
- `../create-new-pdp/references/section-naming.md` — convenzioni naming sezioni.
- `import-page/SKILL.md` (sul VPS, via `ssh … cat /srv/shopify-pdp-builder/skills/import-page/SKILL.md`) — Phase 5.2 (library match / `create_private_section` parseLiquidSchema-derived) + 5.4-5.5 (`create_page_schema` + `add_section_instance`). **Ignora** 5.1 `check_connection`, 5.2.c `push_theme_asset`, 5.3 template-sul-tema, 5.5 `finalize_import`.
- `import-page/references/liquidify-rules.md` / `library-mapping.md` / `section-detection.md` (sul VPS) — HTML chunk → Liquid + quando riusare library vs custom.
