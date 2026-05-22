---
name: import-page
description: Importa una pagina esistente (PDP, advertorial, listicle, quiz, home, altra) da un'altra piattaforma (PageFly, GemPages, Shopify nativo, Funnelish, HeyFlow, o qualsiasi URL pubblico) e la trasforma in una page_schema strutturata di Working Suite con sezioni Liquid editabili. Guida l'operatore attraverso 6 fasi — input URL/HTML, fetch, riconoscimento piattaforma, analisi semantica delle sezioni, generazione Liquid + push sul tema Shopify, riepilogo finale. Da usare quando un cliente onboarda con una pagina già fatta altrove e vuole portarla nel sistema senza ricostruirla da zero.
---

# /import-page — Orchestrator

Guida passo-passo per importare una pagina esistente da URL pubblica o HTML grezzo. Segui le 6 fasi **in sequenza**, mai saltare. `AskUserQuestion` come gate fra fasi.

## Pre-requisito CRITICO

Working Suite passa nel contesto iniziale i parametri della sessione di import:

```
import_id:             <uuid della row in builder_imports>
source_kind:           "url" | "html_paste"
source_url:            <https://... — solo se source_kind=url>
source_html:           <stringa HTML grezza — solo se source_kind=html_paste>
ownership_confirmed:   true   ← se false, RIFIUTA l'import (vedi Phase 1)
store_id:              <uuid workspace_stores>
store_domain:          <store>.myshopify.com
workspace_id:          <uuid>
target_page_type:      "pdp" | "advertorial" | "listicle" | "quiz" | "home" | "other"
target_slug:           <slug della nuova pagina>
target_title:          <titolo della nuova pagina>
```

Se mancano dati critici (in particolare `ownership_confirmed=true`), **fermati immediatamente** e segnala l'errore.

## Convenzioni (valide per TUTTE le fasi)

### Phase tag (priorità massima)

Ogni fase deve aprirsi con una riga isolata `<wsa-phase id="<id>" />` come **PRIMISSIMA cosa** della tua risposta — niente commenti, niente code-fence, niente prosa prima. Senza il tag, la roadmap nella sidebar Working Suite resta indietro e l'operatore non sa che sei avanzato.

| Fase                                          | Tag                                       |
| --------------------------------------------- | ----------------------------------------- |
| 1 — Input validation                          | `<wsa-phase id="input-validation" />`     |
| 2 — Fetch della pagina                        | `<wsa-phase id="fetch-page" />`           |
| 3 — Riconoscimento piattaforma                | `<wsa-phase id="platform-detection" />`   |
| 4 — Analisi semantica sezioni                 | `<wsa-phase id="section-analysis" />`     |
| 5 — Generazione Liquid + push                 | `<wsa-phase id="liquidify-push" />`       |
| 6 — Verifica finale                           | `<wsa-phase id="final-check" />`          |

### AskUserQuestion

Per domande con opzioni discrete (Sì/No, conferme, scelte): usa SEMPRE `AskUserQuestion`. Non duplicare la domanda nel testo prima del tool.

**Gestione della risposta**: la scelta dell'utente arriva come user message con l'etichetta dell'opzione cliccata. Quel messaggio È la risposta alla tool_use. **In nessuna circostanza scrivere frasi che presuppongono annullamento/interruzione** — quel testo è la risposta, procedi normalmente.

Vietato usare termini come: annullat-, interrott-, cancellat-, ferm-, sospes-, abbandonat-, "resto in attesa", "fammi sapere quando vuoi riprendere".

### Tool obbligatori per questa skill

- `WebFetch` (Phase 2) — scaricare HTML da URL
- `Bash` con `curl` (Phase 2 fallback) — se WebFetch fallisce su Cloudflare/JS challenges
- `Read` / `Write` (Phase 5) — leggere library + scrivere sezioni Liquid
- `mcp__working_suite_shopify_admin__*` (Phase 5) — push file Liquid sul tema dell'utente

### Reference file (leggere prima di iniziare la fase corrispondente)

- `references/platform-detection.md` (Phase 3) — heuristic per riconoscere PageFly/GemPages/Funnelish/HeyFlow/Shopify nativo
- `references/section-detection.md` (Phase 4) — come dividere il DOM in sezioni semantiche
- `references/liquidify-rules.md` (Phase 5) — HTML chunk → Liquid + `{% schema %}` block
- `references/library-mapping.md` (Phase 5) — quando usare sezione library vs creare custom workspace-private
- `../create-new-pdp/references/auth-pattern.md` (Phase 5) — auth Shopify (Custom App access token)
- `../create-new-pdp/references/selective-push.md` (Phase 5) — pattern push selettivo + retry
- `../create-new-pdp/references/section-schema-patterns.md` (Phase 5) — editability hard rule
- `../create-new-pdp/references/section-naming.md` (Phase 5) — filename convention `<store>-<page>-<NN>-<slug>.liquid`
- `../create-new-pdp/references/analytics-instrumentation.md` (Phase 5) — tracking `data-wsa-cta-id`

---

## Phase 1 — Input validation

`<wsa-phase id="input-validation" />`

**Goal**: confermare i parametri di import + validare ownership.

**Cosa fai:**
1. Conferma all'operatore i parametri ricevuti dal contesto:
   - `source_kind` (URL o HTML paste)
   - `source_url` / `source_html` (snippet primi 80 char di hash)
   - `target_page_type`, `target_slug`, `target_title`
   - `store_id`, `store_domain`
2. **Verifica `ownership_confirmed === true`**. Se false:
   - Risposta secca: "Impossibile procedere. La conferma di proprietà della pagina non è stata data. Rilancia l'import dal builder e spunta la casella di conferma."
   - STOP. Non andare a Phase 2.
3. Se i dati sono completi e ownership confermata, usa `AskUserQuestion` con opzioni:
   - "Procedi: analizza la pagina e portala in {target_title}"
   - "Indietro: chiudi senza importare"
4. Su "Procedi" → Phase 2.

---

## Phase 2 — Fetch della pagina

`<wsa-phase id="fetch-page" />`

**Goal**: ottenere l'HTML grezzo della pagina sorgente.

**Cosa fai:**
1. Se `source_kind === "html_paste"`: l'HTML è già nel contesto come `source_html`. Vai direttamente a Phase 3.
2. Se `source_kind === "url"`:
   - Usa `WebFetch` con `url: source_url` e `prompt: "Restituisci l'HTML completo della pagina così com'è — non riassumere, non interpretare. Mantieni tutti i tag, attributi data-*, classi, script src, meta tag."`
   - Se `WebFetch` fallisce (timeout, 403 Cloudflare challenge, JS rendering required), fallback con:
     ```bash
     curl -sL -A "Mozilla/5.0 (WorkingSuite Importer/1.0)" --max-time 20 "<source_url>"
     ```
   - Se anche curl fallisce: chiedi all'operatore via `AskUserQuestion` se vuole:
     - "Riprova fetch (potrebbe essere temporaneo)"
     - "Incolla l'HTML manualmente" — l'operatore apre la pagina nel browser, Save As HTML, paste nel prossimo messaggio
3. Salva l'HTML in memoria (variabile interna). NON salvare su disco — è dato temporaneo del contesto chat.
4. Avvisa l'operatore: "Pagina scaricata, {size_kb} KB. Procedo all'analisi."

**Validation:**
- Se HTML < 1 KB → pagina sospetta (forse 404 / empty body). Avvisa e chiedi conferma.
- Se HTML > 5 MB → troppa roba (forse upload sbagliato). Avvisa e chiedi conferma.

---

## Phase 3 — Riconoscimento piattaforma

`<wsa-phase id="platform-detection" />`

**Goal**: identificare quale tool ha generato la pagina sorgente (PageFly v3, PageFly v4, GemPages, Shopify default, Funnelish, HeyFlow, unknown).

**Cosa fai:**
1. Leggi `references/platform-detection.md` per le heuristic complete.
2. Cerca nell'HTML:
   - `<meta name="generator" content="…">`
   - Script src patterns (`pagefly.io`, `gempages.app`, `funnelish.com`, `heyflow.com`)
   - Specific CSS class patterns (es. `pf-*` per PageFly, `gp-*` per GemPages)
   - `<link rel="canonical">` host
3. Output: una di queste etichette:
   - `pagefly` | `gempages` | `shopify_native` | `funnelish` | `heyflow` | `unknown`
4. Comunica all'operatore in 1 riga: "Pagina identificata come **{platform}**. Procedo all'analisi semantica."
5. **Non chiedere conferma**. Vai direttamente a Phase 4 (la piattaforma è solo info utile, non blocca nulla).

---

## Phase 4 — Analisi semantica sezioni

`<wsa-phase id="section-analysis" />`

**Goal**: dividere il DOM della pagina in sezioni semantiche, ognuna con un kind, un chunk HTML, e i field estratti.

**Cosa fai:**
1. Leggi `references/section-detection.md` per le regole di splitting.
2. Pulisci l'HTML:
   - Rimuovi `<script>`, `<style>`, `<noscript>`, `<iframe>` ads
   - Normalizza immagini in URL assoluti (rispetto al base_url della source_url)
   - Decodifica HTML entities
   - Isola `<body>`, scarta header/footer se identificabili
3. Identifica sezioni semantiche. Per ogni sezione produci un oggetto:
   ```json
   {
     "kind": "hero" | "usp_grid" | "product_main" | "product_gallery" | "reviews" | "faq" | "cta_band" | "testimonials" | "features_list" | "comparison_table" | "rich_text" | "image_text" | "footer" | "header" | "unknown",
     "title_for_human": "Hero con immagine sfondo",
     "html_chunk": "<section>...</section>",
     "extracted_fields": {
       "headline": "Addio occhiaie",
       "subheadline": "Risultati in 14 giorni",
       "cta_label": "Acquista ora",
       "cta_url": "/products/crema-borse-occhiaie",
       "background_image": "https://cdn.shopify.com/.../hero.jpg"
     },
     "suggested_library_slug": "hero-basic" | null,
     "confidence": 0.0-1.0
   }
   ```
4. Mostra all'operatore una lista numerata delle sezioni identificate (`AskUserQuestion` per conferma):
   ```
   Ho identificato 5 sezioni:
   1. Hero (confidence 0.92) → suggerisco library hero-basic
   2. USP Grid 3 colonne (0.85) → suggerisco library usp-grid
   3. Reviews carousel (0.78) → custom (non in library)
   4. FAQ accordion (0.91) → suggerisco library faq-accordion
   5. CTA band (0.65) → custom

   Procedo a generare i file Liquid?
   ```
   Opzioni: "Procedi", "Mostra dettagli di una sezione", "Indietro".

5. Su "Procedi" → Phase 5.

---

## Phase 5 — Generazione Liquid + push sul tema

`<wsa-phase id="liquidify-push" />`

**Goal**: per ogni sezione, generare il file Liquid corretto, opzionalmente creare section_definition privata, push selettivo sul tema Shopify, generare template JSON Shopify.

**Cosa fai:**

### 5.1 Verifica auth Shopify
Leggi `../create-new-pdp/references/auth-pattern.md`. Verifica che lo store abbia Custom App connessa (`mcp__working_suite_shopify_admin__check_connection`). Se no, **STOP** con errore "Custom App non connessa per questo store. Vai su /configurations/stores e completa la connessione."

### 5.2 Per ogni sezione identificata in Phase 4

Leggi `references/library-mapping.md` per le regole library lookup vs custom.

Loop per ogni sezione (in ordine di apparizione nella pagina):

**a) Match library?**
- Se `suggested_library_slug !== null` E quella sezione esiste nel workspace (via `mcp__working_suite__list_section_definitions`) → riusa quella library.
- Genera filename con pattern v1: `<store_prefix>-<page_type_short>-<NN>-<library_slug>.liquid`
- Push file Liquid riusando il sorgente library + applica `extracted_fields` come `default` nei settings dello `{% schema %}` (leggi `references/liquidify-rules.md` §"Default-override pattern").

**b) No library match (custom)?**
- Leggi `references/liquidify-rules.md` per la conversione HTML chunk → Liquid + schema.
- Genera filename: `<store_prefix>-<page_type_short>-<NN>-import-<kind>.liquid`
- Costruisci il Liquid:
  - Markup wrapper con classe CSS unica (`ws-import-<kind>-<short_hash>`)
  - Tutti i testi/immagini/link convertiti in `{{ section.settings.X }}`
  - `{% schema %}` block con settings derivati da `extracted_fields`
  - Editability hard rule rispettata (vedi `../create-new-pdp/references/section-schema-patterns.md`)
- Crea anche un record di section_definition privata via `mcp__working_suite__create_private_section` con:
  - `slug: "import-<kind>-<short_hash>"`
  - `version: "1.0.0"`
  - `visibility: "workspace"`
  - `workspace_id: <from context>`
  - `liquid_source: <generated>`
  - `category: <kind>`
  - `page_kinds: [<target_page_type>]`

**c) Push file via Shopify Admin API**
- Usa `mcp__working_suite_shopify_admin__push_theme_asset` (riusa pattern `selective-push.md`).
- Retry 3x su 502/503/504 con backoff 10s/20s/40s.
- Mai usare glob — sempre asset key esatti.

### 5.3 Template JSON Shopify

Quando tutte le sezioni sono pushate, genera il template:
- Se `target_page_type === "pdp"` → `templates/product.<target_slug>.json`
- Se `target_page_type === "home"` → `templates/index.json` (chiedi conferma! sovrascrive home esistente)
- Altrimenti → `templates/page.<target_slug>.json`

Struttura JSON:
```json
{
  "sections": {
    "<filename-1-no-ext>": { "type": "<filename-1-no-ext>", "settings": {} },
    "<filename-2-no-ext>": { "type": "<filename-2-no-ext>", "settings": {} }
  },
  "order": ["<filename-1-no-ext>", "<filename-2-no-ext>"]
}
```

Push del template via stesso pattern.

### 5.4 Crea page_schema in Working Suite

Usa `mcp__working_suite__create_page_schema` (NB: la skill `import-page` ha created_via='skill_import_page' — l'adapter Gate 2-4 lo gestisce).

Parametri:
- `workspace_id`, `store_id`, `page_type`, `slug`, `title` (dal contesto)
- `created_via: "skill_import_page"`
- `source_metadata: { import_id, source_kind, source_url, detected_platform, sections_count }`

Per ogni section_instance, chiama `mcp__working_suite__add_section_instance` con il filename pushato + section_definition_id corretto + field_values estratti.

### 5.5 Update audit log

Chiama `mcp__working_suite__finalize_import` con `import_id` (dal contesto) + `page_schema_id` (appena creato).

---

## Phase 6 — Verifica finale

`<wsa-phase id="final-check" />`

**Goal**: riepilogare cosa è stato fatto e dare all'operatore il link per aprire la nuova pagina nel composer.

**Cosa fai:**

1. Riepilogo all'operatore (testo, no AskUserQuestion):
   ```
   ✅ Import completato.

   - Pagina sorgente: {source_url o "HTML paste"}
   - Piattaforma rilevata: {platform}
   - Sezioni create: {N} (di cui {N_library} dalla library, {N_custom} custom workspace)
   - File Liquid pushati sul tema: {N+1} (sezioni + template JSON)
   - page_schema_id: {id}

   Apri la pagina nel composer per rifinire i testi e le immagini:
   /app/{ws_slug}/builder/page-schemas/{page_schema_id}/edit

   Quando sei pronto, premi "Pubblica" dal dettaglio per propagare le modifiche su Shopify.

   ATTENZIONE: se la pagina sorgente usava app blocks (Katching Bundles, ReCharge, Judge.me),
   non sono stati copiati automaticamente. Riaggiungili dall'editor Shopify nelle sezioni
   che hanno lo slot @app disponibile.
   ```

2. Se ci sono stati WARNING durante l'import (es. immagini non scaricabili, JS non eseguito, sezioni con confidence < 0.5), elencali in fondo.

3. STOP. L'import è completo.

---

## Cose che questa skill NON fa (limiti v1)

- **App blocks (Katching/ReCharge/Judge.me) NON copiati**: l'operatore li riaggiunge manualmente dall'editor Shopify.
- **Customer Reviews con dati reali**: la skill rileva la sezione ma NON copia il contenuto recensioni (per copyright + frequente fetch dinamico).
- **Cart/Checkout custom**: non importati (sono pagine system Shopify, non modificabili da template).
- **A/B test**: non gestito qui. Arriva in Fase 5 Builder v2.
- **Quiz multi-step**: il quiz ha logica branching che richiede la skill `create-quiz` dedicata (Fase 6 Builder v2). Per ora questa skill gestisce solo singole landing.
- **Pagine completamente client-side rendered** (es. Funnelish con tutto in JS): la `WebFetch` cattura solo HTML iniziale. Se la pagina ha contenuto post-render, va in `html_paste` da Save As.

---

## Errore standard

Se in qualsiasi fase qualcosa fallisce in modo non recuperabile:
1. Fermati immediatamente.
2. Chiama `mcp__working_suite__finalize_import` con `import_id` + `status='failed'` + `error_message=<descrizione>`.
3. Comunica all'operatore: "Import fallito durante {phase}. Errore: {message}. Nessun file pushato sul tema."
4. Ricorda: NIENTE termini di "annullamento" — è un fail, non una cancellazione.
