---
name: create-new-funnel
description: Crea una nuova pagina funnel Shopify da zero (advertorial, listicle, quiz, o custom). Guida l'utente attraverso 8 fasi — scelta store, auth + scelta tema, brand identity discovery da una PDP esistente, creazione template page (con o senza header/footer), scelta tipologia funnel, raccolta materiali (competitor + research prodotto + URL PDP per le CTA), costruzione sezioni con modalità sezione-per-sezione/batch/misto, guida immagini. Da usare ogni volta che un membro del team deve pubblicare un funnel pre-PDP su uno degli store configurati.
---

# /create-new-funnel — Orchestrator

Guida passo-passo per creare una **pagina funnel** Shopify da zero. Il funnel è una `templates/page.*.json` — non un prodotto — che ospita contenuti storytelling / listicle / quiz portando l'utente verso una PDP esistente.

Segui le 9 fasi **in sequenza**. `AskUserQuestion` come gate fra una fase e la successiva.

## Convenzioni (valide per TUTTE le fasi)

### Phase tag (priorità massima)

Ogni fase si apre con una riga isolata `<wsa-phase id="<id>" />` come **PRIMISSIMA cosa** della tua risposta — niente commenti, niente code-fence, niente prosa prima. Senza il tag la roadmap nella sidebar Working Suite non avanza.

| Fase                                        | Tag                                       |
| ------------------------------------------- | ----------------------------------------- |
| 1 — Scelta store                            | `<wsa-phase id="select-store" />`         |
| 2 — Auth + tema                             | `<wsa-phase id="auth-check" />`           |
| 3 — Brand identity                          | `<wsa-phase id="brand-discovery" />`      |
| 4 — Tipo funnel                             | `<wsa-phase id="funnel-type" />`          |
| 5 — Struttura funnel                        | `<wsa-phase id="funnel-structure" />`     |
| 6 — Studio prodotto + angle                 | `<wsa-phase id="product-angle" />`        |
| 7 — Creazione template                      | `<wsa-phase id="create-template" />`      |
| 8 — Costruzione sezioni                     | `<wsa-phase id="build-sections" />`       |
| 9 — Guida immagini                          | `<wsa-phase id="images-guide" />`         |
| 10 — Link su Shopify (solo sotto WSA)       | `<wsa-phase id="link-shopify" />`         |

### AskUserQuestion

Per domande con **opzioni discrete** (Sì/No, scelte di lista, conferme): usa SEMPRE `AskUserQuestion`. Working Suite lo renderizza come pannello cliccabile sotto la chat.

**Non duplicare la domanda nel testo prima del tool.** Massimo 1-2 righe di contesto reale se serve davvero.

Per "una sola opzione e serve conferma": `AskUserQuestion` con `Procedi con <nome>` + `Indietro`. Per "una sola e niente da confermare": vai avanti.

Domande aperte (token, URL, slug, testo libero): chiedi in testo — l'utente userà la barra input in basso.

**Gestione della risposta** (importante — leggi attento): Working Suite invia la scelta dell'utente come un normale user message contenente l'**etichetta dell'opzione cliccata**. Esempio: se hai mostrato `Advertorial` + `Listicle` + `Quiz`, quando l'utente clicca `Advertorial` riceverai `Advertorial` come prossimo messaggio. **Quel messaggio È la risposta alla tua AskUserQuestion** — il protocollo CLI non passa un `tool_result` strutturato, vedi solo il testo dell'opzione.

→ **REGOLA UNICA, ASSOLUTA:** in nessuna circostanza scrivere frasi che presuppongono un'azione di interruzione/cancellazione/abbandono da parte dell'utente. Vietate parole come:

- **annullat-** (annullata, annullato, annullare, annullamento)
- **interrott-** (interrotta, interrotto, interruzione)
- **cancellat-** (cancellata, cancellato)
- **ferm-** (fermato, fermata, fermarsi)
- **sospes-** (sospeso, sospesa)
- **abbandonat-** (abbandonato, abbandonata)
- frasi tipo "L'utente ha annullato la domanda", "Resto in attesa che tu riprenda", "Sembri aver cambiato idea"

Il motivo: il CLI claude `-p` ti passa la risposta utente come `user_message`, non come `tool_result`. Il tuo istinto è interpretare quel testo come "tool annullato". È SBAGLIATO. Quel testo è la risposta al tool. Procedi normalmente.

→ Se la risposta NON corrisponde a nessuna opzione mostrata, interpretala come correzione: nuovo `AskUserQuestion` mirato. Senza usare le parole dell'elenco sopra.

→ Prima riga di ogni fase: tag `<wsa-phase id="..." />` + 1-2 righe di contesto + tool. Nessun preamboli che suggerisca interruzione pregressa.

Esempi:

| Input ricevuto              | SBAGLIATO                                       | GIUSTO                                                                              |
| --------------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------------- |
| "Advertorial"               | "Domanda annullata."                            | `<wsa-phase id="funnel-type" />`<br/>Tipo selezionato: Advertorial. Vado avanti.    |
| "no scegli tu"              | "L'utente ha annullato la domanda."             | (nuovo AskUserQuestion con default proposto)                                        |
| "torna indietro"            | "Operazione interrotta."                        | (nuovo AskUserQuestion con conferma di tornare alla fase precedente)                |

### Push asset (MCP — riusalo)

Tutti i push passano dal tool MCP `mcp__working_suite_shopify_admin__push_theme_asset`. NESSUN Shopify CLI, NESSUN `.env`, NESSUN token in chat.

```
mcp__working_suite_shopify_admin__push_theme_asset {
  store_id: <workspace_stores UUID dal contesto di sessione (Fase 1)>,
  theme_id: <main_theme_id risolto da check_connection in Fase 2>,
  assets: [
    { key: "templates/page.<funnel.template_name>.json", content: "<contenuto completo del file>" },
    { key: "sections/<funnel.prefix><NN>-<role>.liquid",   content: "<contenuto completo del file>" }
  ]
}
```

- **Chiavi esatte** (`sections/...liquid`, `templates/page....json`, `layout/....liquid`), MAI glob, MAI `--only`, MAI `sections/*`.
- Una chiamata per asset, oppure un array `assets[]` batched (max 50 asset per chiamata).
- Include SOLO i file del nuovo funnel (template + sezioni col prefisso scelto + eventuale layout chromeless).
- `content` = contenuto integrale del file generato dalla skill (la skill è la source of truth).

### Errori transitori MCP (retry obbligatorio)

`push_theme_asset` può fallire per cause **transitorie**: `429`, `502`, `503`, `504`. Non sono errori tuoi.

**Comportamento richiesto**: fino a **3 tentativi** con backoff esponenziale (10s → 20s → 40s) ri-invocando lo stesso `push_theme_asset` con gli stessi argomenti.

**Quando NON fare retry** (segnala subito all'utente):
- `401` / `403` → `check_connection` non più valida → STOP, manda l'utente a `/configurations/stores` a (ri)connettere la Custom App. (NON "rigenera token".)
- `Liquid syntax error` / errore di validazione asset → la modifica è rotta. Fixa il contenuto e ripeti.
- `Theme not found` / `404` sul `theme_id` → il tema principale non esiste più. Ri-esegui `check_connection` per ottenere il nuovo `main_theme_id`.
- `404` / chiave non valida sull'asset → `key` errata. Fixa il path esatto.

Se i tool `mcp__working_suite_shopify_admin__*` non sono disponibili (mcp=no) → STOP: "MCP non configurato per questa sessione; riapri la chat builder." NON fare fallback a curl/CLI.

### Regole di intoccabilità

1. **Mai toccare template / sezioni esistenti** del tema. Il funnel vive in file nuovi con un prefisso **mai uguale** a uno già presente.
2. **Brand-consistent**: colori e font dal `brand.*` di Fase 3 (estratti da una PDP esistente). Non inventare.
3. **CTA → PDP**: tutti i bottoni puntano a `funnel.cta_url` di Fase 5.1. Mai `#` o placeholder.
4. **Mobile-first**: traffico funnel è >80% mobile. Ogni sezione deve leggersi a 375px.
5. **No framework esterni**: solo CSS scoped dentro la sezione, classi con prefisso che match il nome file.
6. **Mai scrivere dentro `~/.claude/`** — il cwd corrente è il plugin skin.
7. **Conferma prima di ogni azione irreversibile.**

## Setup ambient (Working Suite)

Sotto WSA — riconoscibile da `$WSA_INTERNAL_KEY` valorizzato — il wrapper consegna l'ambiente pronto:

- **cwd**: `~/Desktop/shopify-pdp-builder/`.
- **`config/stores.json`**: SEMPRE in `./config/stores.json`. Read diretto, niente fallback.
- **Connessione Shopify**: lo store è già connesso (Custom App, stesso Admin token dell'Analytics), decifrato lato-app dal tool MCP; la skill usa SOLO `store_id` dal contesto di sessione. NESSUN `.env`, NESSUN token Theme Access, NESSUN prompt token in chat.
- **Stores mancanti / non connessi**: rimanda l'operatore a `/configurations/stores` a (ri)connettere la Custom App, poi ricaricare la chat.

Se i tool `mcp__working_suite_shopify_admin__*` non sono disponibili (mcp=no) → STOP: "MCP non configurato per questa sessione; riapri la chat builder." NON fare fallback a curl/CLI.

---

## Fase 1 — Scelta store

🏷️ Prima riga: `<wsa-phase id="select-store" />`

`Read config/stores.json`. `AskUserQuestion` — una opzione per store, niente "Altro" sotto WSA.

Salva `store.name`, `store.shopify_domain`, `store.store_id` (l'UUID `workspace_stores` — usato come `store_id` in TUTTE le chiamate MCP), `store.workdir_path`. Il `theme_id` NON si prende qui: arriva da `check_connection` in Fase 2.

---

## Fase 2 — Verifica connessione + tema principale

🏷️ Prima riga: `<wsa-phase id="auth-check" />`

1. **Guard MCP**: se i tool `mcp__working_suite_shopify_admin__*` non sono disponibili (mcp=no) → STOP: "MCP non configurato per questa sessione; riapri la chat builder." NON fare fallback a curl/CLI.
2. **Verifica connessione**: chiama `mcp__working_suite_shopify_admin__check_connection` con `{ store_id: <store.store_id> }`.
   - Se NON connesso → STOP con: "Connessione Shopify Admin non valida per questo store. Vai su /configurations/stores e (ri)connetti la Custom App."
   - Se la chiamata torna `401` / `403` → STOP, stesso messaggio (manda l'utente a `/configurations/stores` a riconnettere la Custom App). NON parlare di rigenerare token.
3. **Tema principale**: prendi `main_theme_id` dal RISULTATO di `check_connection` (è il tema pubblicato/live). Salva in `store.theme_id`. Questo valore è il `theme_id` per TUTTI i push successivi. Non esiste più alcun elenco temi.
   - Se `main_theme_id` è `null` → STOP: "Tema principale non risolto."
4. Lo store è già connesso (Custom App, stesso Admin token dell'Analytics), decifrato lato-app dal tool MCP; la skill usa SOLO `store_id` dal contesto di sessione. NESSUN `.env`, NESSUN token Theme Access, NESSUN prompt token in chat.

> Gen-2 non scarica una copia di lavoro del tema (niente pull): il contenuto generato dalla skill è la source of truth. Se in futuro servisse leggere un singolo asset esistente, serve un tool Admin-API GET dedicato — NON usare la CLI.

---

## Fase 3 — Brand identity discovery

🏷️ Prima riga: `<wsa-phase id="brand-discovery" />`

Obiettivo: estrarre palette + tipografia + stili ricorrenti da una PDP esistente del brand. Fonte: `references/brand-identity-discovery.md`.

### 3.1 Scelta PDP di riferimento

`ls templates/ | grep '^product\..*\.json$'`. `AskUserQuestion` con una opzione per template. Se uno solo oltre al default: assumi. Salva in `brand.reference_pdp`.

### 3.2 Estrazione

Read in parallelo:
- `templates/product.<reference>.json`
- 2-3 sezioni principali (hero + prima con CTA)
- `config/settings_data.json` (color_schemes + type_*_font)
- `config/settings_schema.json` (opzionale, per nomi umani)

Estrai secondo `brand-identity-discovery.md`:
- **Palette**: `primary`, `primary_text`, `secondary`, `accent`, `text`, `background` (HEX)
- **Tipografia**: `heading_font`, `body_font` (normalizzati da Shopify handle a `"Nome, fallback"`)
- **Stili ricorrenti**: border-radius, shadow, heading weight/style

### 3.3 Riepilogo + conferma

Mostra il brand profile in formato leggibile (vedi `brand-identity-discovery.md` per il layout). `AskUserQuestion`: conferma / correggo valore / cambio PDP riferimento.

Salva `brand.*` — verrà iniettato nelle sezioni in Fase 8.

---

## Fase 4 — Scelta tipo di funnel

🏷️ Prima riga: `<wsa-phase id="funnel-type" />`

`AskUserQuestion`: "Che tipo di funnel stiamo creando?"

- **Advertorial** — articolo narrativo storytelling, target freddo
- **Listicle** — "N motivi per", benefici ordinati, target semi-caldo
- **Quiz** — domande + risultato personalizzato, engagement-driven
- **Altro** — descrivi la struttura

Salva in `funnel.type` ∈ {`advertorial`, `listicle`, `quiz`, `other`}.

**Sotto Working Suite** (`$WSA_INTERNAL_KEY` valorizzato), persisti subito il tipo scelto via wrapper così la home + le anteprime mostrano "Funnel · Advertorial" senza aspettare la fine del flow:

```bash
curl -sS -X POST "$WSA_INTERNAL_BASE/set-funnel-meta" \
  -H "Content-Type: application/json" \
  -H "X-WSA-Key: $WSA_INTERNAL_KEY" \
  -d "$(printf '{"funnelType":"%s"}' "$FUNNEL_TYPE")"
```

Carica struttura base da `references/funnel-types.md` — ipotesi di partenza, raffinata in Fase 5.

Se `other`: chiedi descrizione (numero sezioni, ordine, ruoli).

---

## Fase 5 — Struttura funnel (URL destinazione + competitor)

🏷️ Prima riga: `<wsa-phase id="funnel-structure" />`

Definisci **solo la struttura** (quante sezioni, ordine, ruoli). I contenuti vengono in Fase 6.

### 5.1 URL PDP destinazione (OBBLIGATORIO)

`AskUserQuestion` o testo: "URL della PDP di destinazione? (dove portano tutte le CTA del funnel)"

Esempi: `https://nimea-beauty.it/products/crema-borse-occhiaie-ufficiale`.

Valida: `https://` + `/products/<slug>`. Salva in `funnel.cta_url` — sarà il `href` di TUTTI i bottoni.

**Sotto Working Suite**, persisti subito il target URL così la home + le anteprime mostrano il prodotto a cui il funnel punta + la thumbnail (se il prodotto è registrato in Configurazioni → Prodotti):

```bash
curl -sS -X POST "$WSA_INTERNAL_BASE/set-funnel-meta" \
  -H "Content-Type: application/json" \
  -H "X-WSA-Key: $WSA_INTERNAL_KEY" \
  -d "$(printf '{"funnelTargetUrl":"%s"}' "$FUNNEL_CTA_URL")"
```

La risposta JSON ha `funnelTargetProductLinked: true|false`. Se `false`, l'handle non risulta in `workspace_products` — di' all'operatore: "URL salvato, ma il prodotto `<handle>` non è ancora nel catalogo Working Suite. Aggiungilo da Configurazioni → Prodotti per averlo nelle anteprime / analytics." Procedi comunque col flow del funnel.

### 5.2 Esempi competitor

Chiedi 2-5 URL/screenshot di funnel competitor dello stesso `funnel.type`. Per ciascuno chiedi contesto breve (perché ti piace, cosa copieresti, cosa cambieresti).

Se l'utente non ha esempi: suggerisci Facebook Ads Library, swipefile, advertorial.com. Se `skip`: usa la struttura base di `references/funnel-types.md`.

Analizza i competitor per:
- Confermare/raffinare la struttura (aggiungere/rimuovere/riordinare sezioni)
- Identificare pattern di hook, problem, proof, CTA
- Stimare density media (word count per sezione)

### 5.3 Struttura proposta + conferma

Presenta la struttura definitiva (esempio):

```
Struttura advertorial proposta (11 sezioni, basata su 3 competitor + base):

01. announcement   — top bar urgency/offer
02. hero           — headline + foto + CTA 1
03. problem-hook   — amplifica dolore
04. story          — storia personale
05. solution-intro — presentazione prodotto
06. how-it-works   — 3 step + ingredienti
07. social-proof   — testimonianze
08. faq            — 5-7 obiezioni
09. cta-offer      — offerta + urgency + CTA 2
10. trust-badges   — garanzia / spedizione
11. footer         — disclaimer
```

`AskUserQuestion`: conferma / modifica. L'utente può aggiungere/rimuovere/riordinare/rinominare.

Salva la lista finale in `funnel.sections_plan` (array ordinato con `nn`, `role`, `notes`).

---

## Fase 6 — Studio prodotto + angle marketing

🏷️ Prima riga: `<wsa-phase id="product-angle" />`

Fase più articolata. Obiettivo: arrivare con materiale sufficiente per scrivere hook + story + proof + obiezioni che girino attorno a un angle chiaro.

### 6.1 Studio della PDP propria

`WebFetch` su `funnel.cta_url`. Estrai:
- Nome prodotto esatto
- Claim principali (H1, H2, bullet)
- Ingredienti chiave / tecnologia / brevetti
- Proof points (numeri, certificazioni, test)
- Prezzo + varianti
- Tono di voce

Riepilogo all'utente → chiedi cosa manca / cosa è sbagliato (le PDP sono ottimizzate per conversione diretta, non sempre per storytelling). Salva in `product.*`.

### 6.2 Studio PDP competitor

`AskUserQuestion` o chiedi: "2-3 URL di PDP competitor nello stesso spazio prodotto."

`WebFetch` ciascuna → estrai claim, ingredienti, proof, pricing, posizionamento.

Output: tabella sintetica vs nostro con colonne `claim_primario`, `differenziatore`, `prezzo`, `gap`.

### 6.3 Angle analysis (OBBLIGATORIO)

1. **Identifica il MAIN ANGLE** dei competitor (PDP 6.2 + funnel 5.2). L'angle è il frame narrativo, NON un claim. Esempi:
   - Ingrediente scientifico esclusivo
   - Routine naturale di 5 minuti
   - Alternativa economica al trattamento da €300
   - Before/after reale di persone come te
   - Risolve X in N giorni — garantito

2. **Presenta il main angle** con evidenze:
   ```
   ➤ ANGLE: <una riga>
      Evidenze:
      - Competitor A (URL): <claim specifico>
      - Competitor B (URL): <idem>
      - Funnel competitor X (URL): <idem>
   ```

3. **Chiedi anche gli ads attuali**: 1-3 copy ads del nostro prodotto (headline + body + concept). Se `skip`: OK.

4. `AskUserQuestion`: "Usi questo angle, proponimi alternative, o hai un angle in mente?"

5. Se alternative: proponi 2-3 angle (a) usati in adiacenza, (b) sostenibili dai nostri proof, (c) differenzianti dal main. Ognuna con nome + pitch + perché funzionerebbe meglio. `AskUserQuestion` di scelta finale.

Salva in `funnel.angle` — bussola narrativa di tutte le sezioni.

### 6.4 Research prodotto — rifinitura

Consolida (l'utente può completare/correggere):

- Nome prodotto esatto
- Ingredienti chiave (3-5, con effetto legato all'angle)
- Claim principali (3-5, in linea con l'angle)
- Target audience (chi, età, problema specifico)
- Differenziazione vs competitor (dal gap di 6.2)
- Proof points (numeri, test, recensioni, certificazioni, testimoni)
- Obiezioni tipiche (3-5, dalla Facebook Ads Library o commenti reali)

Se qualcosa è debole, segnala esplicitamente ("Proof points: nessun numero concreto — vuoi aggiungerne?").

### 6.5 Brand assets extra

Chiedi: logo URL CDN, foto prodotto custom, mockup, loghi certificazioni. Se `skip`: usa quelli da tema + PDP.

### 6.6 Riepilogo prima del template

Blocco compatto: tipo + struttura, angle, 3 claim, 3 proof, 3 obiezioni, URL CTA, palette + tipografia. `AskUserQuestion`: procedo con la creazione template?

---

## Fase 7 — Creazione template funnel

🏷️ Prima riga: `<wsa-phase id="create-template" />`

### 7.1 Partenza

`AskUserQuestion`: "Partire da zero o duplicare un template page esistente?"
- **Da zero** (default).
- **Duplica** esistente — flow analogo a Fase 3 di `/create-new-pdp` ma su `page.*.json`.

### 7.2 Nome nuovo template

`AskUserQuestion`: "Nome del nuovo template funnel?"

Proponi 2-3 default che combinano slug prodotto (da `funnel.cta_url`) + tipo:
- `crema-borse-occhiaie` + `advertorial` → `crema-borse-occhiaie-advertorial`
- `berberina-pills` + `listicle` → `berberina-pills-listicle`

Vincoli: kebab-case `[a-z0-9-]`. Verifica `templates/page.<nome>.json` non esista. Salva in `funnel.template_name`.

### 7.3 Layout — header/footer

`AskUserQuestion`: "Mantenere header e footer Shopify?"

- **No, chromeless** (consigliato) → full-screen senza navbar/footer. Layout `theme.chromeless` se esiste.
- **Sì, tema standard** → header + footer del tema sempre visibili. Layout `theme`.

Fonte: `references/page-template-layout.md`.

Se chromeless: `Glob` o `ls` per verificare `layout/theme.chromeless.liquid`. Se non esiste, `AskUserQuestion`: (a) creo layout custom ora, (b) procedo con `theme`, (c) nome layout diverso. Salva in `funnel.layout`.

### 7.4 Prefisso sezioni

Default da `funnel.template_name` (vedi `references/section-naming.md`, sezione "FUNNEL"):
- `crema-borse-occhiaie-advertorial` → `cbo-adv-` o `cbo-`
- `berberina-listicle` → `bl-` o `bb-lst-`

`AskUserQuestion`: prefisso `<default>`? Salva in `funnel.prefix`.

### 7.5 Template scheletro

```json
{
  "sections": {},
  "order": [],
  "layout": "<funnel.layout>"
}
```

Scrivi in `templates/page.<funnel.template_name>.json`. Le sezioni si aggiungeranno in Fase 8.

Se hai scelto duplicazione (7.1): applica logica di duplicazione (analogo Fase 3 PDP).

### 7.6 Push del template

Vedi convenzioni "Push asset (MCP)". Chiama `mcp__working_suite_shopify_admin__push_theme_asset` con `{ store_id: <store.store_id>, theme_id: <store.theme_id>, assets: [{ key: "templates/page.<funnel.template_name>.json", content: "<contenuto template>" }] }`.

Se hai creato un layout chromeless custom (Fase 7.3), includilo come asset aggiuntivo nello stesso array: `{ key: "layout/<funnel.layout>.liquid", content: "<contenuto layout>" }`.

### 7.7 Page in Shopify Admin

Istruzioni all'utente:
```
1. Admin → Online Store → Pages → Add page
2. Title: "<titolo visibile>"
3. Content: vuoto
4. Theme template (sidebar): seleziona <funnel.template_name>
5. Save
6. Torna qui col lo slug (es. scopri-autoabbronzante)
```

Salva in `funnel.page_slug`. URL live: `https://<store.shopify_domain>/pages/<funnel.page_slug>`.

---

## Fase 8 — Costruzione sezioni

🏷️ Prima riga: `<wsa-phase id="build-sections" />`

### 8.0 Modalità (obbligatorio)

`AskUserQuestion`:
- **A — Sezione per sezione** (consigliata): crea → push → verifica live → conferma → prossima.
- **B — Batch**: tutte in parallelo → preview unica → approvazione → push unico.
- **C — Misto leggero**: checkpoint solo sulle complesse (hero, problem, solution, proof, cta-offer, quiz-wrapper). Semplici (announcement, trust, faq, footer) in batch.

Salva in `funnel.build_mode` ∈ {A, B, C}.

### 8.1 Pianifica la lista sezioni

In base a `funnel.sections_plan` (5.3) + `funnel.angle` (6.3), produci la lista esplicita:

```
01. sections/<prefix>01-announcement.liquid  → announcement bar
02. sections/<prefix>02-hero.liquid          → hero
03. sections/<prefix>03-problem.liquid       → problem hook
...
```

`AskUserQuestion` di conferma/modifica.

### 8.2 Regole per ogni sezione

Per ogni `sections/<funnel.prefix><NN>-<role>.liquid`:

1. **Mobile-first**, `@media (min-width: 768px)` per desktop. Classi scoped con prefisso che match il file (es. `.aa-02-hero ...` in `aa-02-hero.liquid`).
2. **Stili inline** in `<style>` scoped dentro il `.liquid` (no `assets/*.css` esterni se non strettamente necessario). Usa `brand.*`:
   - CTA: `background: <brand.primary>`, `color: <brand.primary_text>`, `border-radius: <brand.button_radius>`, hover più scuro
   - Headline: `font-family: <brand.heading_font>`, `font-weight: <brand.heading_weight>`
   - Body: `font-family: <brand.body_font>`, `color: <brand.text>`
3. **Testi** dal research 6.4 + angle 6.3, density coerente al ruolo (vedi `references/funnel-types.md`).
4. **CTA** → `funnel.cta_url`. Ripeti minimo 2 volte (hero + closing).
5. **Tutto editabile dal theme editor** (regola fissa, no eccezioni). Pattern: `references/section-schema-patterns.md`.
   - Testi: `text` (singola riga), `textarea` (multiriga semplice), `richtext` (paragrafi con inline bold/italic/link).
   - Immagini: `image_picker`. Mai `<img src="https://cdn..."` hardcoded. Render via `{{ section.settings.<id> | image_url: width: <W> }}`.
   - CTA: coppia `url` + `text` per ciascun bottone. `default` di `url` = `funnel.cta_url`.
   - Liste ripetibili (FAQ, benefit, testimonial, listicle reasons, quiz questions): `blocks` con `type` dedicato + `max_blocks` sensato. Iter via `{% for block in section.blocks %}`.
   - **Default sensati**: ogni setting/block parte popolato dal research → la pagina mostra contenuto reale subito dopo il push, senza passare dall'editor.
   - **Placeholder immagini**: `image_picker` senza default; markup gestisce lo stato vuoto:
     ```liquid
     {% if section.settings.hero_image %}
       <img src="{{ section.settings.hero_image | image_url: width: 1600 }}" alt="{{ section.settings.hero_image.alt }}" loading="lazy">
     {% else %}
       <img src="data:image/svg+xml,%3Csvg…%3E" alt="IMG_PLACEHOLDER_<NN>_<role>">
     {% endif %}
     ```
6. **Schema Liquid** completo (esempio minimo — estendi per sezione):
   ```liquid
   {% schema %}
   {
     "name": "<NOME LEGGIBILE>",
     "tag": "section",
     "class": "<prefix>-<NN>",
     "settings": [
       { "type": "text", "id": "heading", "label": "Titolo", "default": "…" },
       { "type": "richtext", "id": "body", "label": "Testo", "default": "<p>…</p>" },
       { "type": "image_picker", "id": "hero_image", "label": "Immagine" },
       { "type": "url", "id": "cta_url", "label": "Link CTA", "default": "<funnel.cta_url>" },
       { "type": "text", "id": "cta_label", "label": "Testo CTA", "default": "Scopri di più" }
     ],
     "presets": [{ "name": "<NOME LEGGIBILE>" }]
   }
   {% endschema %}
   ```
7. **Aggiorna `templates/page.<funnel.template_name>.json`**:
   - Aggiungi a `sections`: `"<NN>": { "type": "<prefix><NN>-<role>", "settings": {} }`
   - Aggiungi al `order` nella posizione giusta.

### 8.3 Workflow A — sezione per sezione

Loop 01→N:

1. `Write` del file `.liquid` + `Edit`/`Read+Write` del template JSON.
2. Push via `push_theme_asset` (vedi convenzioni): un'unica chiamata con `assets: [{ key: "sections/<file>.liquid", content }, { key: "templates/page.<funnel.template_name>.json", content }]`.
3. Utente verifica `https://<store.shopify_domain>/pages/<funnel.page_slug>` mobile + desktop.
4. Aggiustamenti minimi in loop fino a conferma.
5. Prima della prossima: `AskUserQuestion` "Continua sezione-per-sezione, passa a batch, o misto?" Rispetta la scelta.

### 8.4 Workflow B — batch

1. `Write` di TUTTI i file in parallelo.
2. Scrivi il template JSON definitivo con tutte le sezioni ordinate.
3. **Diff-preview** testuale: struttura + headline + primo paragrafo per sezione (no dump markup intero).
4. Approvazione globale o correzioni puntuali.
5. Push unico via `push_theme_asset`: un solo `assets[]` con il template + tutte le sezioni (max 50 asset/chiamata; se di più, spezza in batch).
6. Utente verifica live.
7. Aggiustamenti successivi → modalità sezione-per-sezione per i fix.

### 8.5 Workflow C — misto leggero

1. Sezioni "complesse" (hero, problem, solution, proof, cta-offer, quiz-wrapper, result) → workflow A.
2. Sezioni "semplici" (announcement, trust, faq, footer, disclaimer) → batch, push unico intermedio via `push_theme_asset` (un solo `assets[]`) senza conferma individuale.
3. A fine fase: utente verifica tutto insieme sull'URL live (niente pull — il contenuto generato dalla skill è la source of truth).

### 8.6 Verifica no-regressioni

Prima di ogni push: conferma di non aver toccato file fuori dal prefisso `<funnel.prefix>`. Se sì → stop, ripristina, riprova.

---

## Fase 9 — Guida immagini

🏷️ Prima riga: `<wsa-phase id="images-guide" />`

Immagini sono `image_picker` nello schema. L'utente le carica **dal theme editor**, no URL CDN incollati in chat. Specs: `references/funnel-image-specs.md`.

Per ogni sezione con `image_picker`:

1. `grep image_picker` nello schema → conta + ruoli.
2. Brief:
   ```
   Sezione: <prefix>-<NN>-<role>.liquid
   Immagini: <N>
   Per ogni:
   - Ruolo: <es. "hero lifestyle">
   - Ratio: <4:5>, dimensioni: <1200×1500>, peso max: 150 KB (.webp)
   - Campo theme editor: "<label>"
   ```
3. Istruzioni:
   ```
   1. Admin → Online Store → Themes → Customize (tema live/pubblicato)
   2. Top-left → Pages → <titolo pagina>
   3. Sidebar sezioni → <prefix>-<NN>-<role>
   4. Per ogni campo immagine: click → Select image → Upload → Save
   ```
4. Conferma in chat quando caricato. Nessun push (le immagini sono setting Shopify, non file Liquid).
5. Conferma visiva sull'URL live.

Modalità: la stessa `funnel.build_mode` di Fase 8 (A/B/C) — l'utente può fare tutto in un passaggio nell'editor, poi checkpoint visivo finale.

---

## Verifica finale

Checklist:
- [ ] Template `templates/page.<funnel.template_name>.json` pushato
- [ ] Tutte le sezioni pushate, schema Liquid senza errori
- [ ] URL live ok: `https://<store.shopify_domain>/pages/<funnel.page_slug>`
- [ ] Mobile 375px: leggibile, bottoni cliccabili, testi non troncati
- [ ] Desktop 1440px: layout coerente
- [ ] Palette + tipografia coerenti con `brand.*` (spot check vs PDP riferimento)
- [ ] Tutte le CTA → `funnel.cta_url`
- [ ] Tutti gli `image_picker` con immagine assegnata
- [ ] DevTools console pulita
- [ ] Header/footer come da scelta

Per ogni problema: torna alla fase, fix, re-push.

Quando tutto ✅: dichiara il funnel pronto. Suggerisci UTM tracking sugli ads, Meta Pixel / GA sulla page (theme settings o code injection), click CTA end-to-end fino al checkout.

---

## Fase 10 — Link su Shopify (solo sotto WSA)

🏷️ Prima riga: `<wsa-phase id="link-shopify" />`

Solo se `$WSA_INTERNAL_KEY` valorizzato:
```bash
test -n "$WSA_INTERNAL_KEY" && echo "WSA_ACTIVE" || echo "WSA_INACTIVE"
```

Chiedi URL page funnel live, salva in `$PAGE_URL`:

```bash
curl -sS -X POST "$WSA_INTERNAL_BASE/link-shopify" \
  -H "Content-Type: application/json" \
  -H "X-WSA-Key: $WSA_INTERNAL_KEY" \
  -d "$(printf '{"url":"%s"}' "$PAGE_URL")"
```

Il wrapper salva `linked_url` + `linked_handle` della page. Il prodotto target del funnel (deciso in Fase 5.1) puoi linkarlo con una seconda chiamata identica passando `$PRODUCT_URL` — utile come fallback se in futuro estendiamo il flow.

Conferma all'utente, chiudi.

---

## Modalità manutenzione (post-launch)

Una sessione funnel non si chiude. Dopo che la Fase 10 ha collegato la page su Shopify, la sessione resta **aperta a tempo indeterminato** per fix successivi: typo, sostituzione immagini, swap CTA, refresh stagionali, ottimizzazioni A/B su una sezione.

### Detection (deterministica)

Al resume (`claude --resume <id>`):

- Se il transcript contiene `<wsa-phase id="link-shopify" />` + conferma curl `ok: true` → **modalità manutenzione**.
- Altrimenti → modalità normale, continui dalla fase non chiusa.

### Comportamento in modalità manutenzione

**Primo turno dopo resume:**

1. Emetti il tag fase corrente (riusa `link-shopify`, non aggiungere fasi nuove).
2. Saluta breve:
   ```
   Funnel <funnel.page_slug> pronto e collegato. Cosa vuoi modificare?
   ```
3. `AskUserQuestion` con macro-categorie:
   - **Testi** (copy in una o più sezioni)
   - **Immagini**
   - **Layout/struttura** (riordino sezioni, aggiungere/togliere)
   - **CTA** (cambio destinazione, copy bottone, riposizionamento)
   - **Altro**

**Per ogni fix:**

1. Chiedi all'operatore cosa esattamente cambiare (e in quale sezione se non chiaro).
2. `grep` mirato per localizzare la modifica — non rifare l'inventario sezioni come in Fase 8.
3. `Edit` o `python3 heredoc` puntuale sul singolo file.
4. Push del solo file toccato via `push_theme_asset` (`assets: [{ key, content }]`, store_id + theme_id dal contesto). Riconferma `check_connection` se la sessione è vecchia e dubiti che lo store sia ancora connesso.
5. Verifica URL live.
6. `AskUserQuestion` "Altre modifiche?" → loop o chiudi.

**Cosa NON fare:**

- Non ripercorrere fasi 1-9 (brand, tipo funnel, struttura, prodotto/angle, template, sezioni — tutto già in memoria via `--resume`).
- Non riproporre modalità A/B/C — per un fix singolo si fa puntuale.
- Non emettere nuovi tag `<wsa-phase>` per fasi già fatte.

**Nota collaborativa**: la sessione è workspace-shared. Se chi apre adesso non è il creatore originale, comportati comunque normale: il contesto è in transcript.

## Strumentazione analytics (OBBLIGATORIO)

Ogni funnel creato da questa skill **deve** essere strumentato per la dashboard Analytics di Working Suite. Il SDK `window.wsa.track(name, payload)` è già installato sul theme degli store registrati in `analytics_sources`.

**Cosa generare nelle sezioni:**

- Su ogni **CTA** del funnel (hero CTA, closing CTA, eventuali CTA intermedie): `data-wsa-cta-id="<slug>"` + `data-wsa-cta-position` + script che emette `cta_click`. Pattern in `references/analytics-instrumentation.md`. **Questo è quello che ti dice quale CTA del funnel converte di più.**
- Per funnel di **tipo `quiz`** (`funnel.type === "quiz"`): ogni step del quiz è una sezione separata che emette:
  - `step_view` (IntersectionObserver al 50% in viewport)
  - `step_answer` (click sul pulsante risposta) — DEVE includere `step`, `step_label`, `answer`, `answer_label`, `props.step_id`
  - `step_back` (click su Indietro)
  - `step_complete` (submit finale)
  - Schema fields obbligatori per OGNI step section: `step_number` (number), `step_id` (text slug), `heading` (text/richtext) + blocks `answer` con `value` (slug) e `label` (texto umano).
- Lascia automatici: `page_view`, `session_entry`, `scroll_depth`, `funnel_abandon`.

**Quando applicare:** in Fase 8 (costruzione sezioni), ogni sezione che generi deve avere lo snippet `<script>` di strumentazione appropriato. La reference contiene gli script copia-incolla per CTA / step_view / step_answer / step_back / step_complete.

**Schema editor:** tutti gli slug analytics (`cta_id`, `step_id`, `answer.value`) sono `text` setting esposti nello schema — l'operatore li vede nel theme editor e può modificarli se il setup analytics ha richieste specifiche. NON hardcoded nel markup.

Reference: `references/analytics-instrumentation.md` (event catalog completo, pattern script, convenzioni naming).

## References

- `auth-pattern.md` — [OBSOLETO Gen-1] sostituito dai tool MCP `check_connection` + `push_theme_asset`
- `selective-push.md` — push asset via MCP `push_theme_asset` (chiavi esatte, batch, retry)
- `section-naming.md` — convenzioni prefissi (PDP + funnel)
- `funnel-types.md` — struttura advertorial / listicle / quiz / other
- `brand-identity-discovery.md` — palette + tipografia da PDP esistente
- `page-template-layout.md` — chromeless vs theme
- `funnel-image-specs.md` — dimensioni/ratio per ruolo sezione
- `section-schema-patterns.md` — pattern schema editabile (setting types, blocks, richtext)
- `analytics-instrumentation.md` — event catalog Working Suite + pattern strumentazione (OBBLIGATORIO)

## Troubleshooting

| Sintomo                                | Causa                                          | Fix                                                  |
| -------------------------------------- | ---------------------------------------------- | ---------------------------------------------------- |
| Page live mostra ancora header/footer  | Layout non chromeless o tema senza chromeless  | Verifica `"layout"` nel template; crea layout custom |
| Bottoni non portano al prodotto        | `href` vuoto o `#` in qualche sezione          | Grep `href="` nelle sezioni, tutti = `funnel.cta_url`|
| Font incoerente col resto del sito     | `brand.*` non applicato a tutte le sezioni    | Ri-scansiona, applica `font-family: <brand.*>`       |
| Sezione invisibile live                | Non aggiunta all'`order` del template JSON     | Verifica array `order`                               |
| `push_theme_asset` 401/403             | `check_connection` non più valida              | STOP, manda l'utente a /configurations/stores a riconnettere la Custom App |
| `push_theme_asset` "Liquid syntax"     | Schema malformato, brace unmatched             | Leggi error line, `Edit` puntuale, re-push           |
| `push_theme_asset` 429/502/503/504     | Errore transitorio                             | Retry stesso push con backoff 10s/20s/40s (3 tentativi) |
| Tool `mcp__working_suite_shopify_admin__*` assenti | MCP non configurato per la sessione | STOP, riapri la chat builder. NO fallback CLI/curl   |
| Mobile text overflow                   | Font-size in px anziché clamp                  | `clamp(1rem, 4vw, 1.5rem)` o media query             |
