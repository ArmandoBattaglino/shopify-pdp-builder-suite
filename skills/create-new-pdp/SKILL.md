---
name: create-new-pdp
description: Crea una nuova PDP Shopify da zero. Guida l'utente attraverso 7 fasi — scelta store, verifica auth, duplicazione template e sezioni con nuovo prefisso, raccolta materiali di ricerca sul prodotto, popolamento testi sezione-per-sezione mantenendo i layout esistenti, indicazioni sulle immagini da caricare, verifica finale. Da usare ogni volta che un membro del team deve pubblicare una PDP per un nuovo prodotto su uno degli store configurati (Nimea, Glowria, o altri aggiunti a config/stores.json).
---

# /create-new-pdp — Orchestrator

Guida passo-passo per creare una PDP Shopify da zero. Segui le 7 fasi **in sequenza**, mai saltare. `AskUserQuestion` come gate fra una fase e la successiva.

## Convenzioni (valide per TUTTE le fasi)

### Phase tag (priorità massima)

Ogni fase deve aprirsi con una riga isolata `<wsa-phase id="<id>" />` come **PRIMISSIMA cosa** della tua risposta — niente commenti, niente code-fence, niente prosa prima. Senza il tag, la roadmap nella sidebar Working Suite resta indietro e l'operatore non sa che sei avanzato.

| Fase                                        | Tag                                       |
| ------------------------------------------- | ----------------------------------------- |
| 1 — Scelta store                            | `<wsa-phase id="select-store" />`         |
| 2 — Auth + tema                             | `<wsa-phase id="auth-check" />`           |
| 3 — Duplicazione template                   | `<wsa-phase id="duplicate-template" />`   |
| 4 — Raccolta materiali                      | `<wsa-phase id="research-materials" />`   |
| 5 — Riscrittura testi                       | `<wsa-phase id="rewrite-content" />`      |
| 6 — Guida immagini                          | `<wsa-phase id="images-guide" />`         |
| 7 — Verifica finale                         | `<wsa-phase id="final-check" />`          |
| 8 — Link su Shopify (solo sotto WSA)        | `<wsa-phase id="link-shopify" />`         |

### AskUserQuestion

Per domande con **opzioni discrete** (Sì/No, scelte di lista, conferme): usa SEMPRE il tool `AskUserQuestion`. Working Suite lo renderizza come pannello cliccabile sotto la chat.

**Non duplicare la domanda nel testo prima del tool** — l'utente la vede già nel pannello. Limita la prosa a 1-2 righe di contesto reale se serve.

Per "una sola opzione disponibile e serve conferma": `AskUserQuestion` con `Procedi con <nome>` + `Indietro`. Per "una sola opzione e niente da confermare": vai avanti senza chiedere.

Domande aperte (token, URL, slug, testi liberi): chiedi in testo — Working Suite mostra l'input testuale in basso.

**Gestione della risposta** (importante — leggi attento): Working Suite invia la scelta dell'utente come un normale user message contenente l'**etichetta dell'opzione cliccata**. Esempio: se hai mostrato opzioni `Glowria (Malta)` + `Nimea`, quando l'utente clicca la prima riceverai `Glowria (Malta)` come prossimo messaggio. **Quel messaggio È la risposta alla tua AskUserQuestion** — il protocollo CLI non passa un `tool_result` strutturato, quindi vedi solo il testo dell'opzione.

→ **REGOLA UNICA, ASSOLUTA:** in nessuna circostanza scrivere frasi che presuppongono un'azione di interruzione/cancellazione/abbandono da parte dell'utente. Questo include — ma NON è limitato a — qualsiasi formulazione che contenga parole come:

- **annullat-** (annullata, annullato, annullare, annullamento, ...)
- **interrott-** (interrotta, interrotto, interruzione, ...)
- **cancellat-** (cancellata, cancellato, cancellazione)
- **ferm-** (fermato, fermata, fermo, fermarsi)
- **sospes-** (sospeso, sospesa, sospensione)
- **abbandonat-** (abbandonato, abbandonata)
- frasi tipo "L'utente ha annullato la domanda", "Resto in attesa che tu riprenda", "Fammi sapere quando vuoi riprendere", "Sembri aver cambiato idea"

Il motivo: il protocollo CLI di claude `-p` riceve la risposta dell'utente alla tua `AskUserQuestion` come un nuovo `user_message`, NON come un `tool_result` strutturato. Tu vedi solo il testo della risposta dopo la `tool_use`. Il tuo istinto è interpretare l'arrivo del testo come "non hai usato il tool, hai annullato". È **sbagliato**. Quel testo È la risposta al tool. Procedi normalmente trattando il testo come la scelta dell'utente fra le opzioni mostrate.

→ Se la risposta dell'utente NON corrisponde a nessuna opzione mostrata (è una frase libera tipo "no scegli tu", "torna indietro", uno store diverso), interpretala come correzione del flow: chiedi conferma con un nuovo `AskUserQuestion` mirato. **Senza** mai usare le parole dell'elenco sopra.

→ Stessa regola alla **prima riga di ogni fase**, anche dopo un resume: la prima cosa che scrivi è il tag `<wsa-phase id="..." />`, poi 1-2 righe di contesto, poi il tool. **Nessun preamboli** che suggerisca che il lavoro era stato fermato o interrotto, anche se al modello sembra "naturale" da dire.

Esempi di risposte CORRETTE dopo aver ricevuto un input utente:

| Input ricevuto                | Risposta SBAGLIATA                                  | Risposta GIUSTA                                                                    |
| ----------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------- |
| "Procedi con Glowria (Malta)" | "Domanda annullata. Riprendiamo quando vuoi."       | `<wsa-phase id="auth-check" />`<br/>Verifico il token e elenco i temi disponibili. |
| "no scegli tu"                | "L'utente ha annullato la domanda."                 | (nuovo AskUserQuestion con le stesse opzioni o conferma di proseguire con default) |
| "torna indietro"              | "Operazione interrotta — riprendiamo dal precedente." | (nuovo AskUserQuestion con conferma di tornare alla fase precedente)              |

### Push selettivo (template — riusalo ovunque)

Il push avviene via MCP, mai via CLI. Chiama `mcp__working_suite_shopify_admin__push_theme_asset` passando lo `store_id` dal contesto di sessione e il `theme_id` = `main_theme_id` risolto in Fase 2 da `check_connection`:

```
mcp__working_suite_shopify_admin__push_theme_asset({
  store_id: <workspace_stores UUID dal contesto>,
  theme_id: <main_theme_id>,
  assets: [
    { key: "templates/product.<nome>.json", content: "<contenuto completo del file>" },
    { key: "sections/<prefisso>-NN-....liquid", content: "<contenuto completo del file>" }
  ]
})
```

Regole:
- **Chiavi esatte, mai glob.** Ogni `key` è il path esatto dell'asset (`sections/<prefisso>-NN-...liquid` oppure `templates/product.<slug>.json`). Includi SOLO i file del nuovo prefisso (template + sezioni duplicate), mai sezioni/template di altri prodotti.
- `content` è il contenuto **completo** del file (non un diff). Una `key` per asset.
- Una chiamata per asset, oppure un'unica chiamata con `assets[]` batchato (**max 50** per chiamata).
- `theme_id` è sempre `main_theme_id` (il tema live/pubblicato) restituito da `check_connection`.

### Errori transitori MCP push (retry obbligatorio)

`push_theme_asset` può fallire a volte per cause **transitorie** (non per errore tuo): `429 Too Many Requests`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`, errori di rete/timeout lato Admin API. Sono normali — succede sotto carico o per blip di rete.

**Comportamento richiesto**: non fermarti al primo fallimento. Fai **fino a 3 tentativi** con backoff: 10s → 20s → 40s.

Pattern (logico):
1. Tentativo 1 → se errore transitorio (429/502/503/504/network), attendi 10s.
2. Tentativo 2 → se ancora transitorio, attendi 20s.
3. Tentativo 3 → se ancora transitorio, attendi 40s e, se fallisce di nuovo, segnala all'utente.

Ogni tentativo è una nuova chiamata `mcp__working_suite_shopify_admin__push_theme_asset` con gli stessi `{ store_id, theme_id, assets }`.

**Quando NON fare retry** (errori NON transitori — fallimento immediato e segnala all'utente):
- `401 Unauthorized` / `403 Forbidden` → la connessione Admin non è valida → `check_connection` è fallita. Stop, rimanda l'utente a **/configurations/stores** a (ri)connettere la Custom App. (NON "rigenera token".)
- `Liquid syntax error` / `Invalid Liquid` → la modifica è rotta. Stop, leggi il file, fixa, ripeti.
- `Theme not found` / `404` su `theme_id` → il `main_theme_id` non è più valido. Stop, ri-esegui `check_connection` per risolvere il tema corrente.
- `404` / asset key non valida → la `key` non corrisponde a un asset gestibile. Stop, fixa la chiave esatta.

In modalità manutenzione (post-launch), applica lo stesso retry sulle chiamate `push_theme_asset` puntuali.

### Regole di intoccabilità

1. **Mai modificare il template base o le sue sezioni.** Il template base (es. `product.berberina-pills.json`) e le sue sezioni (es. `cboe-pdp-05.liquid`) appartengono a un prodotto live diverso. Si duplicano in file nuovi con prefisso nuovo; ogni `Edit`/`Write` avviene sui duplicati.
2. **Mai riscrivere il markup esistente** durante la riscrittura testi. Layout, CSS, JS si preservano; modifichi solo i `default` dei `{% schema %}`. Fonte: `references/workflow-faithful-rebuild.md`.
3. **Ogni sezione duplicata deve essere editabile dal theme editor.** Se è hardcoded (markup legacy), liquidify automaticamente — vedi Fase 3.5. Pattern: `references/section-schema-patterns.md`.
4. **Mai scrivere dentro `~/.claude/`.** Il cwd corrente è il plugin skin: tutto quello che modifichi vive lì.
5. **Conferma con l'utente prima di ogni azione irreversibile** (creazione file, push live, rinomina).

## Setup ambient (Working Suite)

Sotto WSA — riconoscibile da `$WSA_INTERNAL_KEY` valorizzato — il wrapper ti consegna l'ambiente pronto:

- **cwd**: `~/Desktop/shopify-pdp-builder/` (plugin skin per-workspace).
- **`config/stores.json`**: SEMPRE in `./config/stores.json` (relativo al cwd). Read diretto, niente fallback.
- **Connessione store**: lo store è già connesso (Custom App, stesso Admin token dell'Analytics), decifrato lato-app dal tool MCP; la skill usa SOLO lo `store_id` (UUID `workspace_stores`) dal contesto di sessione. **NESSUN `.env`, NESSUN token Theme Access, NESSUN prompt token in chat.**
- **Stores mancanti**: lo `stores.json` è in **sola lettura** sotto WSA (rigenerato a ogni init). Se serve uno store non in lista, rimanda l'operatore a **/configurations/stores** nella dashboard a connettere la Custom App, poi ricaricare la chat.

### MCP guard (priorità massima)

Tutte le operazioni sul tema passano dai tool `mcp__working_suite_shopify_admin__*` (`check_connection`, `push_theme_asset`). Se questi tool **non sono disponibili** nella sessione (mcp=no) → **STOP** con: "MCP non configurato per questa sessione; riapri la chat builder." **NON** fare fallback a `curl`/CLI Shopify.

---

## Fase 1 — Scelta store

🏷️ Prima riga: `<wsa-phase id="select-store" />`

`Read config/stores.json` (path relativo al cwd). Mostra la lista all'utente via `AskUserQuestion` — una opzione per store, niente "Altro" sotto WSA.

Salva i campi dello store scelto in memoria: `store.name`, `store.shopify_domain`, e soprattutto `store.store_id` (l'UUID `workspace_stores` dello store scelto — è la chiave che passi a TUTTI i tool MCP). Il `theme_id` NON viene preso da qui: si risolve in Fase 2 dal `main_theme_id` di `check_connection`.

---

## Fase 2 — Verifica connessione + risoluzione tema

🏷️ Prima riga: `<wsa-phase id="auth-check" />`

1. **Verifica connessione** — chiama:
   ```
   mcp__working_suite_shopify_admin__check_connection({ store_id: <store.store_id dal contesto, salvato in Fase 1> })
   ```
   - Se il risultato **non è connesso** → **STOP** con: "Connessione Shopify Admin non valida per questo store. Vai su /configurations/stores e (ri)connetti la Custom App." Non procedere oltre.
2. **Risolvi il tema** dal risultato: il campo `main_theme_id` è il tema **pubblicato/live**. Salvalo in `store.theme_id` — è il `theme_id` che userai per TUTTI i push.
   - Se `main_theme_id` è `null` → **STOP**: "Tema principale non risolto."
   - Non esiste più alcun `theme list`: il tema su cui operi è sempre il main/live restituito da `check_connection`.

---

## Fase 3 — Duplicazione template

🏷️ Prima riga: `<wsa-phase id="duplicate-template" />`

### 3.1 Nome nuovo template

`AskUserQuestion`: "Che nome vuoi dare al nuovo template PDP?"

Vincoli: kebab-case, solo `[a-z0-9-]`, esempi `crema-borse-occhiaie-pdp`, `siero-vitamina-c-pdp`. Verifica che `templates/product.<nome>.json` non esista già.

### 3.2 Template base

```bash
ls templates/ | grep '^product\..*\.json$'
```

(`templates/` è relativo al cwd `~/Desktop/shopify-pdp-builder/`, come `config/stores.json`.)

`AskUserQuestion` con una opzione per template trovato. Se ne esiste uno solo oltre al default `product.json`, assumi quello senza chiedere.

### 3.3 Prefisso sezioni

Genera default dalle iniziali (vedi `references/section-naming.md`):
- `crema-borse-occhiaie-pdp` → `cboe`
- `siero-vitamina-c-pdp` → `svc`

`AskUserQuestion`: prefisso `<default>`? Conferma o alternativo.

### 3.4 Duplicazione

1. Leggi il template base JSON.
2. Estrai le sezioni referenziate (`"type": "<nome>"` che corrisponde a un file `sections/<nome>.liquid`).
3. Identifica il prefisso del base (inizio comune dei nomi sezione).
4. Per ogni sezione col prefisso base:
   - `Read sections/<old>-<suffix>.liquid` → `Write sections/<nuovo>-<suffix>.liquid`.
   - Aggiorna il nome schema (`"name"` in `{% schema %}` e `presets[0].name`) con uppercase del nuovo prefisso. Le classi CSS interne (`.cboe-...`) restano invariate.
5. Scrivi `templates/product.<nome>.json` come copia del base, sostituendo i `"type"` col nuovo prefisso.
6. Riepilogo all'utente + conferma prima del push.

**Mai modificare i file originali.**

### 3.5 Liquidify automatico (OBBLIGATORIO, prima del push)

Ogni sezione duplicata deve essere editabile dal theme editor. Pattern: `references/section-schema-patterns.md`.

Per ogni file duplicato:

1. **Detection** (deterministica, no domande):
   - Leggi schema → settings/blocks non vuoti?
   - Testi visibili nel markup referenziati come `{{ section.settings.<id> }}` / `{{ block.settings.<id> }}`?
   - Immagini via `| image_url`, no `<img src="https://cdn.shopify.com/...">` hardcoded?
   - Se sì a tutto → sezione già editabile, lascia invariata.
   - Altrimenti → marca `needs_liquidify`.

2. **Se needs_liquidify**: liquidifica la copia (mai l'originale):
   - Estrai testi visibili, URL immagine (`<img src>`, `srcset`, `background-image: url(...)` inline), link, `alt`.
   - Per ciascuno: setting con `id` semantico (`hero_heading`, `hero_image`, `cta_primary_url`...) e `default` = valore estratto.
   - Tipi: `richtext` per paragrafi formattati, `image_picker` per immagini, `url`+`text` per CTA, `blocks` per gruppi ripetuti.
   - Sostituisci nel markup: testi → `{{ section.settings.<id> }}`, immagini → `{{ section.settings.<id> | image_url: width: <W> }}`.
   - NON modificare: tag HTML, classi CSS, `data-*`, `<script>`, `<style>`, JSON-LD, Liquid logic.

3. Riepilogo all'utente: quali sono già editabili, quali liquidificate.

### 3.6 Push iniziale

Push selettivo via `push_theme_asset` (vedi convenzioni): un'unica chiamata MCP con `assets[]` contenente il template `templates/product.<nome>.json` + tutte le sezioni duplicate `sections/<prefisso>-NN-...liquid` (chiavi esatte, contenuto completo di ogni file, max 50 asset per chiamata). `theme_id` = `store.theme_id` (il `main_theme_id` risolto in Fase 2).

### 3.7 Creazione Product in Shopify Admin

Istruzioni all'utente:
```
1. Admin → Products → Add product
2. Title: "<nome del nuovo prodotto>"
3. Theme template (sidebar destra): seleziona <nome>
4. Save
5. Torna qui col lo slug del prodotto (es. crema-borse-occhiaie-ufficiale)
```

Salva lo slug in `product.slug`. URL live: `https://<store.shopify_domain>/products/<product.slug>`.

---

## Fase 4 — Raccolta materiali

🏷️ Prima riga: `<wsa-phase id="research-materials" />`

Chiedi all'utente (checklist):

1. **PDP competitor** (1-3 URL/screenshot) — linguaggio, claim, struttura.
2. **Transcript video** (1-3) — tono di voce, punti di dolore, frasi ad alta conversione.
3. **Research prodotto**: ingredienti chiave, claim principali, target audience, differenziazione, proof points.
4. **Angolo marketing attuale** (1-3 esempi di ad: copy + headline + immagine/video).
5. **Brand assets**: palette HEX, font, logo URL CDN.

### 4.1 Angle analysis (OBBLIGATORIO, non saltabile)

Dopo aver ricevuto i materiali, prima della riscrittura:

1. **Identifica il MAIN ANGLE** dei competitor. L'angle è il frame narrativo (NON il claim). Esempi:
   - Ingrediente scientifico esclusivo
   - Routine naturale di 5 minuti
   - Alternativa economica al trattamento da €300
   - Before/after reale di persone come te
   - Risolve X in N giorni — garantito

2. **Presenta il main angle** con evidenze:
   ```
   ➤ ANGLE: <una riga>
      Evidenze:
      - Competitor A (URL): <come lo usa>
      - Competitor B (URL): <come lo usa>
      - Ads attuali: <come lo rinforzano>
   ```

3. `AskUserQuestion`: "Usi questo angle, proponimi alternative, o hai un angle in mente?"

4. Se alternative: proponi 2-3 angle (a) usati in adiacenza, (b) sostenibili dai nostri proof points, (c) differenzianti rispetto al main. `AskUserQuestion` finale.

Salva in `pdp.angle` — bussola della Fase 5.

### 4.2 Riepilogo + conferma

Bullet sintetico: nome prodotto, angle, 3 claim, 3 proof, palette. `AskUserQuestion`: "Materiali completi, procedo con la scrittura testi?"

---

## Fase 5 — Riscrittura testi

🏷️ Prima riga: `<wsa-phase id="rewrite-content" />`

**Le sostituzioni avvengono nei `default` dei `{% schema %}`** (settings/blocks), non nel markup HTML. Markup/CSS/JS della sezione restano identici. Tutti i testi coerenti con `pdp.angle`.

### 5.0 Modalità (obbligatorio)

`AskUserQuestion`:
- **A — Sezione per sezione** (consigliata per prima PDP): riscrivi, mostra diff, push solo quella sezione, l'utente verifica live, passa alla successiva.
- **B — Batch**: tutte parallele, diff completo unico, approvazione globale, push selettivo unico.
- **C — Misto**: cambia modalità in corsa quando vuoi.

### 5A — Sotto-flusso sezione per sezione

Per ogni sezione del template nuovo, in ordine di apparizione:

1. **Read** del file duplicato → identifica ruolo (hero, FAQ, testimonial, ...) usando `references/image-specs-per-section.md`.
2. **Proponi testi nuovi** coerenti con `pdp.angle`, stesso numero di bullet/card/FAQ. Diff before/after chiaro:
   ```
   — Headline
      prima: "..."
      dopo:  "..."
   ```
   Se la research non basta per una struttura (es. 5 card, 3 claim): chiedi input mirato invece di inventare.
3. **Aspetta conferma/correzioni** prima di scrivere.
4. **Applica** modifiche SOLO ai `default` dello schema:
   - Poche sostituzioni → `Edit` con `old_string`/`new_string` esatti.
   - Molte → `Bash` con `python3 <<'PY' ... PY` heredoc (parsa schema JSON, modifica `default`, riscrivi blocco).
   - Non toccare: markup HTML, classi CSS, `<script>`, `<style>`, `data-*`, URL immagini in `image_picker` default (verranno sostituite in Fase 6), Liquid tags.
   - Se trovi testi ancora hardcoded nel markup: segnala e completa la liquidify su quella sezione (non patchare nel markup).
5. **Verifica no-regressioni**: prima del push, conferma di non aver toccato file `<vecchio-prefisso>-*` o `templates/product.<vecchio-nome>.json`.
6. **Push selettivo** della sola sezione modificata.
7. Utente verifica `https://<store.shopify_domain>/products/<product.slug>` mobile + desktop.
8. Aggiustamenti minimi (font-size, white-space, spelling) → incrementali, mai rifare la sezione.
9. Prima della prossima sezione: `AskUserQuestion` "Continua sezione-per-sezione, passa a batch, o misto?" Rispetta la scelta.

### 5B — Sotto-flusso batch

1. **Read tutte** le sezioni duplicate in parallelo (multiple Read in un messaggio).
2. **Pianifica riscrittura integrata** — testi tra sezioni coerenti, stesso linguaggio, claim ripetuti con stessa terminologia.
3. **Diff completo unico** raggruppato per sezione.
4. Approvazione globale o correzioni puntuali → applica.
5. Verifica no-regressioni → push selettivo unico con tutte le sezioni.
6. Utente check completo. Aggiustamenti successivi → puntuali (anche se il primo push era batch).

### 5C — Misto

L'utente cambia modalità in corsa. Rispetta la scelta, continua dalla prossima sezione non ancora lavorata.

---

## Fase 6 — Guida immagini

🏷️ Prima riga: `<wsa-phase id="images-guide" />`

Le immagini sono `image_picker` nello schema. L'utente le carica **dal theme editor**, nessun replace di URL CDN nei file. Specs: `references/image-specs-per-section.md`.

Per ogni sezione con `image_picker`:

1. `grep image_picker` nello schema → conta + ruoli.
2. Brief all'utente:
   ```
   Sezione: <nuovo-prefisso>-<suffix>.liquid
   - Ruolo immagine: <cosa deve mostrare>
   - Ratio: <es. 4:5>, dimensioni: <1200×1500>, peso max: <150 KB>
   - Campo theme editor: "<label image_picker>"
   ```
3. Istruzioni:
   ```
   1. Admin → Online Store → Themes → Customize (tema live)
   2. Top-left picker → Products → <prodotto>
   3. Seleziona la sezione → click sul campo immagine → Upload → Save
   ```
4. Conferma visiva su URL live. Nessun push necessario.

---

## Fase 7 — Verifica finale

🏷️ Prima riga: `<wsa-phase id="final-check" />`

Checklist:
- [ ] Testi validati su tutte le sezioni
- [ ] Tutte le immagini caricate e renderizzate
- [ ] Mobile 375px ok
- [ ] Desktop 1440px ok
- [ ] DevTools console pulita (no errori Liquid, no 404 immagini)
- [ ] URL diretto ok: `https://<store.shopify_domain>/products/<product.slug>`
- [ ] Custom domain (se presente)

Per ogni problema → torna alla fase corrispondente. Quando tutto ✅: dichiara la PDP pronta. Suggerisci catalogo/collezione, pricing/inventory/variants, bundling (Katching: vedi `references/workflow-faithful-rebuild.md`).

---

## Fase 8 — Link su Shopify (solo sotto WSA)

🏷️ Prima riga: `<wsa-phase id="link-shopify" />`

Se `$WSA_INTERNAL_KEY` vuoto: salta la fase.

```bash
test -n "$WSA_INTERNAL_KEY" && echo "WSA_ACTIVE" || echo "WSA_INACTIVE"
```

Se attiva: chiedi l'URL PDP live (testo libero o `AskUserQuestion`), salva in `$PRODUCT_URL`, poi:

```bash
curl -sS -X POST "$WSA_INTERNAL_BASE/link-shopify" \
  -H "Content-Type: application/json" \
  -H "X-WSA-Key: $WSA_INTERNAL_KEY" \
  -d "$(printf '{"url":"%s"}' "$PRODUCT_URL")"
```

Risposta:
- `ok: true` + `productLinked: true` → sessione collegata al prodotto in `workspace_products`. Conferma all'utente, chiudi.
- `ok: true` + `productLinked: false` → URL salvato ma handle non corrisponde. Suggerisci: "Vai su Configurazioni → Prodotti e aggiungi `<handle>` per collegare anche le analytics, oppure proseguiamo senza."
- `ok: false` → mostra errore + suggerisci link manuale dalla pagina sessione Builder.

---

## Modalità manutenzione (post-launch)

Una sessione PDP non si chiude mai. Dopo che la Fase 8 ha collegato la PDP su Shopify, la sessione resta **aperta a tempo indeterminato** per fix successivi: typo nei testi, sostituzione immagini, aggiornamento prezzi/varianti, refresh stagionali, A/B testing su una sezione, ecc.

### Detection (deterministica)

Al resume di una sessione (`claude --resume <id>`), il transcript precedente contiene già le fasi che hai attraversato. Per capire in che stato sei:

- Cerca nel transcript il tag `<wsa-phase id="link-shopify" />` seguito dalla conferma del curl all'endpoint `/link-shopify` (response `ok: true`).
- Se presente → **sei in modalità manutenzione**. Non rifare le fasi 1-7.
- Se assente → continui dal punto dove la roadmap si era fermata (modalità normale).

### Comportamento in modalità manutenzione

**Primo turno dopo il resume** (il primo messaggio dell'operatore in modalità manutenzione):

1. Emetti il tag fase corrente per la roadmap (riusa l'ultima fase attraversata, di solito `link-shopify`) — non aggiungere fasi nuove.
2. Saluta con un riassunto breve e mirato:
   ```
   PDP <product.slug> pronta e collegata. Cosa vuoi modificare?
   ```
3. `AskUserQuestion` con macro-categorie:
   - **Testi** (cambio copy in una o più sezioni)
   - **Immagini** (sostituire/aggiungere/togliere immagini)
   - **Layout/struttura** (riordinare sezioni, aggiungerne, toglierne)
   - **Settings prodotto** (prezzo, varianti, inventory — punto a Shopify Admin perché non sono editabili dal tema)
   - **Altro** (problema specifico)

**Per ogni fix scelto:**

1. Chiedi all'operatore **cosa esattamente** vuole cambiare (e in quale sezione se non chiaro).
2. Se non sai a quale file appartiene la modifica, fai un grep mirato per identificarla — NON rifare la lista template/sezioni come in Fase 3.
3. Applica il fix puntuale: `Edit` o `python3 heredoc` sul singolo file.
4. **Push selettivo** del solo file modificato via `push_theme_asset` (vedi convenzioni). Il contenuto generato dalla skill è la fonte di verità: non c'è alcun pull del tema.
5. Chiedi all'operatore di verificare sull'URL live.
6. Quando confermato: `AskUserQuestion` "Altre modifiche?" con opzioni:
   - **Sì, ne ho un'altra** → torna alla scelta macro-categoria
   - **No, è tutto** → conferma chiusura sessione di manutenzione e basta

**Cosa NON fare in modalità manutenzione:**

- Non ripercorrere le fasi 1-7 (store già scelto, tema già risolto, template già duplicato, ecc.). Tutti questi dati sono già in memoria via `--resume`.
- Non c'è alcun pull del tema: Gen-2 non scarica una copia di lavoro, il contenuto generato dalla skill è la fonte di verità.
- Non riproporre la modalità sezione-per-sezione vs batch — per un fix singolo si fa direttamente puntuale.
- Non emettere nuovi tag `<wsa-phase>` per fasi che hai già completato — confonderebbe la roadmap.

**Nota collaborativa**: una stessa sessione può essere riaperta da operatori diversi del workspace (RLS workspace-based). Se l'utente che apre adesso non sei "lo stesso" della creazione (es. messaggi sotto altro nome nel transcript), comportati comunque normale: il contesto del lavoro è in transcript, e gli `AskUserQuestion` valgono per chiunque clicchi.

## Strumentazione analytics (OBBLIGATORIO)

Ogni PDP creata da questa skill **deve** essere strumentata per la dashboard Analytics di Working Suite. Il SDK `window.wsa.track(name, payload)` è già installato sul theme degli store registrati in `analytics_sources` — la skill chiama il SDK dai punti giusti del DOM.

**Cosa generare nelle sezioni:**

- Su ogni **CTA principale** della PDP (Aggiungi al carrello, Acquista, Scopri di più, sticky bottom): aggiungi `data-wsa-cta-id="<slug>"` + `data-wsa-cta-position="<hero|mid|closing|sticky-bottom>"` + uno snippet `<script>` inline che emette `cta_click` con `cta_id`/`cta_label`/`cta_position`/`template`. Pattern completo in `references/analytics-instrumentation.md`.
- Lascia automatici (no codice necessario): `page_view`, `session_entry`, `scroll_depth`, `add_to_cart` (form submit standard `/cart/add`), `begin_checkout`, `purchase`.
- Nessun evento quiz su PDP (quelli appartengono al funnel quiz — vedi skill funnel).

**Quando applicare:** in Fase 3 (duplicazione + liquidify), per ogni sezione duplicata che contiene CTA principali, aggiungi anche la strumentazione. In Fase 5 (riscrittura testi), se l'operatore aggiunge nuove CTA via theme editor, l'`data-wsa-cta-id` deve essere un `text` setting dello schema così resta editabile.

Reference: `references/analytics-instrumentation.md` (event catalog completo, pattern script, convenzioni naming).

## References

- `workflow-faithful-rebuild.md` — regola d'oro + tecniche sostituzione + form Katching
- `selective-push.md` — come pubblicare gli asset via `push_theme_asset` (MCP)
- `section-naming.md` — convenzioni prefissi
- `image-specs-per-section.md` — dimensioni/ratio per ruolo sezione
- `section-schema-patterns.md` — liquidify, setting types, edge case
- `analytics-instrumentation.md` — event catalog Working Suite + pattern strumentazione (OBBLIGATORIO)

## Troubleshooting

| Sintomo                          | Causa                                  | Fix                                                  |
| -------------------------------- | -------------------------------------- | ---------------------------------------------------- |
| `check_connection` non connesso / push 401/403 | Custom App non connessa / Admin token non valido | Operatore (ri)connette la Custom App su /configurations/stores |
| `check_connection` con `main_theme_id` null | Tema principale non risolto      | STOP: tema principale non risolto; operatore verifica il tema live |
| Tool `mcp__working_suite_shopify_admin__*` assenti | MCP non configurato per la sessione | STOP: riapri la chat builder (no fallback CLI/curl)  |
| Push fallisce "Liquid syntax"    | Sostituzione ha rotto Liquid           | Read file, localizza errore, Edit per riparare       |
| Push fallisce 429/502/503/504    | Errore transitorio Admin API           | Retry con backoff 10s/20s/40s (3 tentativi)          |
| Sezione non appare su PDP live   | Template JSON o sezione non pushata    | Verifica `templates/product.<nome>.json` + push      |
| Stile diverso dall'originale     | Hash legacy toccato per errore         | Ripristina file originale + rifai solo testi schema  |
| Product non usa il nuovo template| Template suffix non selezionato        | Admin → Product → Theme template → seleziona         |
