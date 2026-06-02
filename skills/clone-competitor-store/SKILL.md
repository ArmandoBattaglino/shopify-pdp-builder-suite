---
name: clone-competitor-store
description: Replica integrale di uno store competitor (di solito US) sul nostro tema Shopify (di solito mercato IT). Diversa dalle skill /create-new-pdp e /create-new-funnel che creano UNA pagina alla volta — questa orchestra la creazione di TUTTE le pagine necessarie (funnel + PDP + home + opzionali) in una singola sessione, con brand discovery globale fatta UNA volta dai URL competitor e poi riusata per ogni pagina. Da usare ogni volta che identifichi un competitor che funziona e vuoi portarlo sul tuo store, replicandolo pixel-perfect ma con identità di brand diversa.
---

# /clone-competitor-store — Orchestrator

Sei una guida passo-passo per clonare un competitor su uno store Shopify. Costruisci tutto il necessario (funnel + PDP + home + pagine extra) in **un'unica sessione orchestrata**, in ordine vincolante: **Funnel → PDP → Home → Extras**.

Segui le 8 fasi **in sequenza**. Non saltare fasi. Usa `AskUserQuestion` come gate tra una fase e la successiva.

## Principi generali (HARD RULES)

- **🚫 Anti-hallucination assoluto** (regola più importante): replica SOLO quello che l'utente fornisce. Niente sezioni "che ci starebbero bene", niente testi "tipici di una hero", niente claim "che il competitor probabilmente dice". Se l'utente non te lo dà, non lo metti. Se ti manca un'informazione, FERMA e chiedila — non inventare.
- **📸 Screenshot per il LAYOUT, NON per il testo**: gli screenshot mobile + desktop servono per ricostruire il design visivo (struttura, spacing, tipografia, layout). I TESTI invece li fornisce l'utente direttamente (HTML salvato o copy/paste manuale dal sito competitor). Claude non fa OCR sui testi degli screenshot — il downscaling rende OCR inaffidabile e porta a hallucination.
- **🤝 Workflow autorizzato + transformative**: la skill assume un caso d'uso legittimo (clone autorizzato dal cliente proprietario del competitor, oppure analisi competitive con scope di reskinning) dove il prodotto finale è un **derivative work** (Step 1 lingua originale → Step 2 traduzione IT + adattamento al brand del cliente). I testi forniti dall'utente sono input dell'utente da strutturare, non riproduzione fatta da Claude.
- **📋 Testi forniti dall'utente prima della costruzione**: prima di scrivere qualsiasi liquid, l'utente fornisce i testi delle sezioni in formato strutturato (HTML salvato + opzionale, oppure copy/paste manuale per sezione). Claude li valida con l'utente, poi li struttura nei `default` dello schema senza alterarli.
- **🔄 Due step di costruzione (literal → IT)**: ogni pagina viene costruita **prima in lingua originale** (esattamente come il competitor), pushata e verificata 1:1 con il competitor. **Solo dopo conferma di identità visuale** si procede con la traduzione IT come step 2 separato. Mai mischiare i due step.
- **♻️ Ogni pagina è indipendente**: il flow Step 1 (literal lingua originale) → Step 2 (IT) si applica **per OGNI pagina del loop di Fase 6**, indipendentemente dallo stato delle pagine precedenti. Non saltare Step 1 sulla 2ª pagina perché la 1ª è già stata tradotta nella stessa sessione, perché il custom domain è italiano, perché "il workflow l'hai capito". Anche se hai costruito 5 pagine prima, la 6ª riparte da Step 1 lingua originale come la prima. Nessuna "ottimizzazione" del flow è ammessa — produce sempre hallucination o testi inventati.
- **🧹 Collision detection PRIMA della creazione**: ogni volta che stai per creare un template/sezioni con un certo prefisso store, verifica con `ls` che NON esistano già file con quei nomi nel workdir. Se ci sono → STOP e chiedi all'utente: cleanup totale, suffix versione `-v2`, o stop. **Mai sovrascrivere "alla cieca"** — si finisce con sezioni orfane numerate due volte (es. `-03-editorial.liquid` + `-03-problem.liquid`) che creano il caos.
- **🔒 Mai toccare template/sezioni esistenti.** Tutto vive in file nuovi con prefisso store-specifico (`<store-slug>-<page-type>-<NN>-<role>.liquid`). Mai sovrascrivere `index.json`, `product.<existing>.json`, sezioni del tema base.
- **Costruzione SEQUENZIALE in ordine fisso**: **Funnel → PDP → Home → Extras**. Motivo: il funnel ha CTA → PDP, la home ha link → PDP. Senza ordine ti tocca rework.
- **Brand discovery UNA volta sola** (Fase 4) — gli output sono **base di partenza** per tutte le pagine. Però ogni pagina specifica può avere palette propria diversa dalla globale (è frequente negli advertorial competitor): in 6.x.5.0 la skill estrae la palette pagina-specifica dall'HTML/CSS della pagina e la usa come override per QUELLA pagina.
- **Replica visiva 1:1**: ogni sezione si costruisce **da zero** (no riuso sezioni esistenti del tema). HTML+CSS scritti per matchare lo screenshot/markup competitor.
- **🎨 CSS scraping opzionale per fedeltà ~90%**: se l'utente lo attiva in Fase 6.x.4.bis, la skill scarica i file CSS pubblici del competitor via `curl` (sono asset pubblici, mandati al browser di chiunque visiti il sito) e ne riusa le regole nelle sezioni clonate. Questo porta la fedeltà visiva da ~70% (ricostruzione a vista) a ~90% (CSS sorgente riapplicato). Senza CSS scraping, la skill fa il suo meglio dagli screenshot ma valori esatti di padding/margin/font-weight saranno approssimazioni.
- **Tutto editabile dal theme editor — REGOLA CRITICA**: niente hardcoded nel markup. Testi/immagini/link/colori sempre come `settings`/`blocks` nello schema. Per le sezioni PDP è OBBLIGATORIO `"blocks": [{"type": "@app"}]` per supportare Katching Bundles, subscription pickers, app review. Fonte di verità: `references/editability-and-app-blocks.md`.
- **🛍️ PDP above-the-fold custom 1:1**: per le PDP, la sezione `main` (`product-information` nativa Shopify) NON va lasciata col config di default. Va analizzata vs competitor sopra-piega elemento per elemento (pills, rating, price, variant picker, quantity, buy buttons, accelerated checkout, bundle widget, trust badges) e configurata di conseguenza: `disabled: true` per blocchi native non presenti nel competitor, custom CSS per stylizzare il button add-to-cart, slot `@app` per Katching/subscription/recensioni se il competitor li usa. Errore tipico da NON fare: lasciare il prezzo Shopify default visibile quando il competitor lo nasconde, lasciare i bottoni Apple Pay/Google Pay quando il competitor mostra solo "Add to cart". Fonte di verità: `references/pdp-main-configuration.md`.
- **🔍 PDP main quasi sempre custom**: `product-information` native va bene SOLO per PDP super-essenziali. Le PDP DTC moderne hanno bundle picker, free gift selector, mystery gift, urgency, trust strip, accordion product details — tutti elementi che NON esistono in product-information default. Prima di scegliere `main_strategy`, esegui un **diagnostic dell'HTML competitor** in Fase 6.x.2.0: cerca signal `kaching-bundle`, `free-gift`, `countdown-timer`, `recharge-subscriptions`, `trust-strip`, accordion `<details>`. Se trovi UNO di questi → `main_strategy = "custom"`. Default sicuro in dubbio: custom.
- **📸 PDP — input materiali in 2 batch**: per le PDP NON basta un full-page lungo. La buy box dx ha 6-8 elementi in ~600px, illeggibili compressi. Workflow PDP-specific: Batch 1 (above-the-fold prioritario) → screenshot HD della SOLA buy box dx + gallery sx + lista esplicita degli elementi della sidebar (titolo/stelle/bullet/bundle/free gift/ATC/accordion/trust). Batch 2 (below-the-fold) → multi-section o full-page del corpo. Senza batch 1 separato, la skill costruisce la buy box "a memoria" → hallucination garantita.
- **🪜 Default action: LITERAL sempre** (anti-asimmetria PDP/funnel): per ogni testo che vedi negli screenshot/HTML competitor, il comportamento di base è **literal nei `default` dello schema**, indipendentemente dal tipo di pagina (funnel, PDP, home, extras) e dalla classificazione informativa L1/L2/L3/L4. Non auto-convertire MAI un testo in placeholder. La classificazione L1-L4 è solo una mappa per discutere con l'utente: i testi L4 (narrativi/testimonial/about) li proponi come "vuoi convertire in placeholder?" e l'utente decide. Senza conferma esplicita: literal. Errore tipico: skill che sulla PDP mette placeholder per "scriverli con voce brand" mentre nel funnel ha messo literal — risultato asimmetrico e PDP non identica al competitor.
- **🔁 Workflow testi INVARIANTE per ogni tipo di pagina**: il modo di raccogliere e applicare i testi è IDENTICO per funnel, PDP, home, extras. L'utente fornisce HTML salvato (o copy/paste) → la skill mette i testi letterali nei `default` dello schema. **NIENTE asimmetrie** tra tipi di pagina. Le differenze tra tipi riguardano la STRUTTURA (PDP main custom, batch 1+batch 2, home assignment, funnel layout chromeless) — NON il modo di trattare i testi.
- **⭐ Sezioni reviews: replica esattamente quello che il competitor ha**: niente più, niente meno. Alcuni competitor hanno solo carousel testimonial, alcuni solo grid con breakdown, alcuni entrambe in posizioni separate, alcuni nessuna sezione dedicata (solo star rating accanto al titolo via app), alcuni testimonial integrati in altre sezioni. **Non aggiungere una carousel "perché è il pattern tipico"** se il competitor non ce l'ha. **Non rimuovere una grid** se invece c'è. Identifica le sezioni reviews effettivamente presenti negli screenshot e replica solo quelle. Vedi `references/pdp-main-configuration.md` sezione "Sezioni reviews" per i pattern di riferimento.
- **Mobile-first**: il traffico arriva da ads (>80% mobile). Ogni sezione deve girare benissimo a 375px.
- **Un asset per chiamata (o batch espliciti), sempre per chiave esatta**. Ogni push avviene via il tool MCP `mcp__working_suite_shopify_admin__push_theme_asset`, passando `assets: [{ key, content }]` con la **chiave esatta** del file (es. `sections/<prefix>-02-hero.liquid`, `templates/page.<slug>.json`). MAI glob/wildcard nelle chiavi. Una chiamata per asset, oppure un array `assets[]` batchato (max 50 asset per chiamata). Include SOLO i file della pagina corrente.
- **Auth Gen-2 (Custom App, niente CLI/Theme Access)**: lo store è già connesso (Custom App, stesso Admin token dell'Analytics), decifrato lato-app dal tool MCP; la skill usa SOLO `store_id` dal contesto di sessione. NESSUN `.env`, NESSUN token Theme Access, NESSUN prompt token in chat.
- **MCP guard**: se i tool `mcp__working_suite_shopify_admin__*` non sono disponibili (mcp=no) → STOP: "MCP non configurato per questa sessione; riapri la chat builder." NON fare fallback a curl/CLI.
- **Conferma prima di azioni irreversibili** (creazione file, push live, override homepage).

## Lettura dello stato iniziale

All'avvio devi trovare il file `config/stores.json` del plugin.

**REGOLA ASSOLUTA** — senza eccezioni, senza interpretazione:

1. **NON creare MAI nessun file dentro `~/.claude/skills/`.** Quella cartella contiene solo i symlink di sviluppo. Se stai per scrivere un file il cui path contiene `.claude/skills/`, fermati subito: è sbagliato.
2. **`stores.json` vive SOLO nella cartella del plugin**, cioè `<plugin-root>/config/stores.json`. Mai altrove.
3. **`stores.example.json` esiste già** nel repo — non va mai creato né copiato dalla skill.

**Procedura deterministica** (esegui nell'ordine, senza saltare step):

```bash
# Step A — prova path canonico
test -f "$HOME/Desktop/shopify-pdp-builder/config/stores.json" && echo "FOUND:$HOME/Desktop/shopify-pdp-builder/config/stores.json"

# Step B — risolvi via symlink della skill (se A non ha trovato nulla)
PLUGIN_ROOT=$(dirname "$(dirname "$(readlink "$HOME/.claude/skills/clone-competitor-store")")")
test -f "$PLUGIN_ROOT/config/stores.json" && echo "FOUND:$PLUGIN_ROOT/config/stores.json"

# Step C — prova il path plugin standard di Claude Code
test -f "$HOME/.claude/plugins/shopify-pdp-builder/config/stores.json" && echo "FOUND:$HOME/.claude/plugins/shopify-pdp-builder/config/stores.json"
```

Comportamento:

- Se **uno qualsiasi** degli step stampa `FOUND:<path>` → usa quel path con `Read` per leggere `stores.json` e procedi con la Fase 1.
- Se **tutti e tre** falliscono → **FERMATI**. Non scrivere nulla. Chiedi all'utente in chiaro: "Non trovo `config/stores.json`. Dimmi il path assoluto del plugin `shopify-pdp-builder` sul tuo disco." Attendi risposta, poi ricontrolla.

**Mai** usare `Write` o `Edit` per creare `stores.json` o `stores.example.json`. Se manca, è un problema di setup utente, non un file da generare dalla skill.

---

## Fase 1 — Scelta store

Mostra all'utente la lista degli store configurati. Usa `AskUserQuestion` con un'opzione per store + "Altro".

Se "Altro": chiedi nome, `shopify_domain` (*.myshopify.com), path workdir. Aggiungi voce a `stores.json`. **NON** chiedere theme ID né path env: il theme id arriva da `check_connection` (Fase 2) e l'auth è gestita dalla Custom App via MCP.

Salva in memoria: `store.name`, `store.shopify_domain`, `store.workdir_path`, e — soprattutto — `store.id` (lo **UUID di `workspace_stores`** preso dal contesto di sessione del builder). `store.id` è l'unico identificatore che la skill passa ai tool MCP. Se manca dal contesto → STOP: "store_id assente dal contesto di sessione; riapri la chat builder dallo store giusto."

**Genera lo slug store** (kebab-case del nome, es. `Glowria` → `glowria`, `Nimea Beauty` → `nimea`). Salva in `store.slug`. Sarà il prefisso di tutte le sezioni e i template creati dalla skill.

---

## Fase 2 — Verifica connessione Shopify Admin (MCP) + risoluzione tema

Gen-2: niente CLI, niente `.env`, niente Theme Access. La connessione è una Custom App (stesso Admin token dell'Analytics) decifrata lato-app dal tool MCP. La skill verifica la connessione e ricava il tema principale in **una sola chiamata MCP**.

1. **Guard MCP**: se i tool `mcp__working_suite_shopify_admin__*` non sono disponibili → STOP: "MCP non configurato per questa sessione; riapri la chat builder." NON fare fallback a curl/CLI.
2. **Verifica connessione**: chiama il tool MCP

   ```
   mcp__working_suite_shopify_admin__check_connection
     { store_id: <store.id> }   // l'UUID di workspace_stores salvato in Fase 1
   ```

3. **Se NON connesso** (campo `connected: false`, oppure errore 401/403) → STOP con il messaggio esatto:
   > "Connessione Shopify Admin non valida per questo store. Vai su /configurations/stores e (ri)connetti la Custom App."

   NON suggerire mai "rigenera token", NON leggere alcun file di auth CLI.
4. **Risoluzione tema (NIENTE `theme list`)**: il tema su cui operare è il **tema pubblicato/live**, che arriva dal campo del risultato `main_theme_id` di `check_connection`. Salva:
   - `store.theme_id = <result.main_theme_id>`
   - `store.theme_name = <result.main_theme_name>` (se presente nel risultato)

   Se `main_theme_id` è `null` → STOP: "Tema principale non risolto per questo store."
5. Non c'è alcun **pull**: Gen-2 non scarica una working copy del tema. I contenuti generati dalla skill (file `.liquid` + `templates/*.json` che scrivi in locale nel workdir) sono la **fonte di verità** e vengono spinti sul tema con `push_theme_asset`. Se un singolo asset esistente dovesse essere letto, servirebbe un tool Admin-API GET dedicato — NON usare la CLI.

---

## Fase 3 — Definizione scope pagine

`AskUserQuestion` multi-select: "Quali pagine dobbiamo replicare dal competitor?"

Opzioni base (almeno una obbligatoria):
- **Funnel** — pre-PDP page (advertorial / listicle / quiz)
- **PDP** — pagina prodotto
- **Home** — homepage del competitor

⚠️ Le pagine extra (contattaci, traccia ordine, FAQ, about, blog, collection) **non vengono chieste qui** — vengono chieste alla FINE (Fase 7). Out of scope sempre: cart, checkout.

Se l'utente seleziona Funnel, sotto-domanda con `AskUserQuestion`: "Che tipo di funnel?"
- Advertorial / Listicle / Quiz / Altro

Salva in `scope.funnel_type` ∈ {`advertorial`, `listicle`, `quiz`, `other`} oppure `null`.

Comunica all'utente l'ordine di costruzione che seguirai:
```
Costruirò in quest'ordine vincolante:
  1. Funnel <type>          (perché ha CTA → PDP)
  2. PDP                     (perché ha URL che useremo poi)
  3. Home                    (perché linka alla PDP)
  4. (Pagine extra alla fine, opzionali)
```

Salva in `scope.pages_ordered` la lista delle pagine in ordine corretto (es. `["funnel-advertorial", "pdp", "home"]`).

---

## Fase 4 — Brand identity discovery globale

Questa fase gira **UNA volta sola**. I suoi output sono usati da TUTTE le pagine costruite in Fase 6.

Fonte di verità: `references/competitor-discovery.md`.

### 4.1 Raccolta URL competitor

Chiedi all'utente: "Mandami i URL competitor di TUTTE le pagine che dobbiamo replicare (una per ogni pagina selezionata in Fase 3)."

Esempio:
```
- Funnel advertorial:  https://competitor.com/pages/why-it-works
- PDP:                 https://competitor.com/products/cream-name
- Home:                https://competitor.com/
```

Salva in `competitor.urls` (oggetto chiave-valore: `{ funnel: "...", pdp: "...", home: "..." }`).

### 4.2 Estrazione automatica

Per ogni URL competitor, usa `WebFetch` per scaricare HTML + CSS inline + foglie statiche referenziate. Estrai (vedi `references/competitor-discovery.md` per i dettagli del processo):

- **Palette colori** (top 5-7 colori non-neutri ricorrenti, prioritizzando CTA + heading)
- **Font family** (heading + body, normalizzando da Google Fonts URL o `font-family:` inline)
- **Logo URL** (cercando `<img>` nel header con alt/class identificative — `class*="logo"`, `alt*="logo"`)
- **Tono di voce / copy keyword** (estratto dai testi heading/CTA dominanti)
- **Layout grid** (max-width container, padding standard, breakpoint mobile)

### 4.3 Riepilogo + conferma

Presenta all'utente un blocco strutturato:

```
=== Brand discovery dal competitor ===

PALETTE (estratti, ordinati per ricorrenza):
  primary       #E36B6E       (CTA buttons, badges)
  cta           #1A1A1A       (CTA secondario, dark mode)
  heading       #1A1A1A
  body          #4A4A4A
  background    #FAF6F2
  accent        #D4A574       (gold detail)

FONT:
  heading       Playfair Display, serif
  body          Poppins, sans-serif

LOGO:           https://competitor.com/cdn/shop/files/logo.png

TONO DI VOCE:   editoriale, confidenziale, focus su risultati
```

Usa `AskUserQuestion`:
- **Conferma** → procedi
- **Correggo un valore** → chiedi quale e applica
- **Riestrai (URL diverso)** → ricomincia 4.1

Salva in `brand.*`:
- `brand.palette.primary`, `.cta`, `.heading`, `.body`, `.background`, `.accent`
- `brand.fonts.heading`, `.body`
- `brand.logo_url_competitor`
- `brand.tone`

---

## Fase 5 — Identità nuovo brand

Domande per i dati che NON estrarre dal competitor (questi sono nostri).

`AskUserQuestion` o testo libero:

1. **Nome del brand**: uguale al competitor o nuovo?
   - Se "uguale": riusa `brand.competitor_name` estratto da Fase 4 (titolo della home o `<meta property="og:site_name">`).
   - Se "nuovo": chiedi nome.
   - Salva in `store.brand_name`.

2. **Logo**: chiedi all'utente di:
   ```
   Carica il logo del brand su Shopify:
     1. Admin → Settings → Files → Upload files → seleziona il logo (preferibilmente PNG con transparenza, min 512×128).
     2. Una volta caricato, click destro sul logo → Copy link.
     3. Incolla qui l'URL CDN (formato `https://cdn.shopify.com/s/files/...`).
   ```
   Salva in `store.logo_url`.

3. **Email assistenza**: testo libero. Default: `info@<dominio>` se conoscibile dallo store. Salva in `store.support_email`.

4. **Telefono**: testo libero. Salva in `store.phone`.

5. **Indirizzo legale completo** (per footer + pagine legali): testo libero. Salva in `store.legal_address`.

6. **Lingua testi**: conferma `IT` come default. La skill TRADUCE i testi competitor ENG → IT secondo `references/localization-it.md`. Salva in `store.locale`.

Riepilogo + `AskUserQuestion` di conferma prima di passare alla costruzione.

---

## Fase 6 — Costruzione iterativa pagine

Loop sull'array `scope.pages_ordered` calcolato in Fase 3, in ordine fisso (Funnel → PDP → Home).

Per ogni pagina, esegui il sub-workflow 6.x:

### 6.x.1 — Input materiali competitor

**Strategia di input**: gli screenshot servono per il **layout visivo** (capire come sono fatte le sezioni, ordine, spacing, tipografia, colori). I **testi** invece arrivano dall'utente direttamente in formato strutturato — gli screenshot full-page compressi sono inaffidabili per OCR e Claude finirebbe per inventare. Sono due input separati, raccolti in 2 fasi.

**Le PDP richiedono workflow speciale** perché la buy box dx ha 6-8 elementi piccoli concentrati in ~600px di altezza, illeggibili in un full-page compresso. Per le PDP la skill chiede materiali in **2 batch**: prima above-the-fold (HD), poi below-the-fold.

#### Caso A — Pagina NON-PDP (funnel, home, extras)

```
=== Pagina <N>/<TOT>: <tipo pagina> ===

Per ricostruire il LAYOUT visuale 1:1 mi servono:
  1. Screenshot DESKTOP della pagina (uno full-page lungo va bene, oppure 
     5-10 screenshot di sezione singola — preferisci la seconda opzione 
     se la pagina è molto lunga, perché ogni sezione resta in alta risoluzione)
  2. Screenshot MOBILE della pagina (stesso discorso)
  3. URL competitor (per riferimento)

Gli screenshot servono solo per capire il design — i testi me li darai 
separatamente in Fase 6.x.5.a in formato strutturato (HTML salvato o 
copy/paste manuale) per evitare errori di lettura.
```

#### Caso B — Pagina PDP (workflow speciale in 2 batch)

⚠️ **Per le PDP NON ti basta un full-page compresso**: la buy box dx (titolo / stelle / bullet / bundle / free gifts / ATC / accordion / trust strip) è una zona densissima di 600×800px che in un full-page lungo diventa illeggibile.

**Batch 1 — Above-the-fold (PRIORITARIO)**:

```
=== PDP <N>/<TOT>: above-the-fold ===

Per la buy box devo lavorare con HD. Mi servono:

DESKTOP (~1440px):
  1. Screenshot HD della SOLA buy box dx (tutta la sidebar destra, 
     dal titolo al primo accordion) — risoluzione naturale, NO compressione
  2. Screenshot HD della gallery sx (immagine principale + thumbnail rail)
  3. Screenshot HD del header/announcement bar

MOBILE (~390px iPhone):
  4. Screenshot HD del primo viewport mobile (tutto quello che vedi senza scrollare)
  5. Screenshot HD del secondo viewport (1° scroll, di solito gallery → buy box mobile)

LISTA ESPLICITA degli elementi che vedi nella sidebar dx (in ordine top→bottom):
  Mandami una lista tipo:
    - Pills "BESTSELLER" + "ZERO ALLERGENI"
    - Title H1
    - Stars 4.8/5 + "2,431 reviews"
    - Bullet x4 (icona ✓)
    - Bundle box "BUY 1 / BUY 2 / BUY 3" con sconti
    - Free gift selector "Choose your gift"
    - Add to cart button (verde acceso)
    - Express checkout (Shop Pay)
    - Trust strip (5 icone: free shipping, 30-day return, ecc.)
    - Accordion: "PRODUCT DETAILS" + "INGREDIENTS" + "HOW TO USE"

  È importante perché alcuni elementi sono nascosti agli screenshot 
  (collapsed, hover-only, dependent on bundle selection).
```

**Batch 2 — Below-the-fold**:

```
=== PDP <N>/<TOT>: below-the-fold ===

Ora il resto della pagina:
  - Screenshot multi-section (5-10 screenshot per le sezioni sotto la buy box: 
    benefits, how it works, ingredients, comparison, reviews, FAQ, ecc.)
  - Stesso su mobile

Se la PDP non ha sezioni below-the-fold complesse (è una PDP corta), basta 
un full-page lungo dei rimanenti pixel.
```

**URL competitor**: confermami quello già salvato in Fase 4 o passamene uno aggiornato.

⚠️ **Se l'utente fornisce screenshot di scarsa qualità** (sfocati, viewport tagliato): chiedi sostituzione prima di procedere. Per il layout serve qualità visibile.

Salva in:
- `current_page.screenshots.atf_desktop` (above-the-fold buy box dx + gallery sx)
- `current_page.screenshots.atf_mobile`
- `current_page.screenshots.below_fold_desktop` (multi-section o full-page below)
- `current_page.screenshots.below_fold_mobile`
- `current_page.atf_elements_list` (lista esplicita elementi sidebar dx)
- `current_page.url`

### 6.x.2 — Analisi visiva e mappatura sezioni

#### 6.x.2.0 — HTML diagnostic checklist (SOLO se la pagina è PDP)

⚠️ **Obbligatorio per PDP**: prima di mappare le sezioni, esegui un **diagnostic check sull'HTML salvato** del competitor (Formato A di 6.x.5.a, se l'utente lo ha già fornito; altrimenti chiedilo subito) per capire se la sezione `main` della PDP **deve essere completamente custom** o se basta il `product-information` nativo Shopify.

Le PDP DTC moderne hanno quasi sempre sezione main **custom**: bundle picker (Katching/Bundler), free gift selectors, mystery gift, urgency cards, trust strip, accordion product details. Tutti elementi che NON esistono in `product-information` default e devono essere costruiti come blocchi custom o ospitati come `@app` slot.

**Procedura del diagnostic** (Bash + grep sull'HTML salvato):

```bash
HTML_FILE="<path-to-saved-html>"

echo "=== Diagnostic main-custom signals ==="
echo ""

echo "→ Bundle picker (Katching/Bundler/FastBundle):"
grep -oEi "(kaching-bundle|kaching-bundles-block|bundler-bundle|fast-bundle|<bundle-|class=\"[^\"]*bundle[^\"]*\")" "$HTML_FILE" | head -5

echo ""
echo "→ Free gift / Mystery gift:"
grep -oEi "(free-gift|gift-tier|mystery-gift|mystery-box|gift-selector|claim-your-gift)" "$HTML_FILE" | head -5

echo ""
echo "→ Urgency / countdown:"
grep -oEi "(countdown-timer|urgency-card|stock-counter|low-stock|<countdown|data-countdown)" "$HTML_FILE" | head -5

echo ""
echo "→ Subscription apps (ReCharge/Smartrr/Skio/Bold):"
grep -oEi "(recharge-subscriptions|smartrr|skio-|bold-subscriptions|subscribe-and-save|<subscription-)" "$HTML_FILE" | head -5

echo ""
echo "→ Trust strip / icons row:"
grep -oEi "(trust-strip|trust-badges|guarantee-row|shipping-icons|free-shipping-badge|class=\"[^\"]*badges-row[^\"]*\")" "$HTML_FILE" | head -5

echo ""
echo "→ Product details accordion:"
grep -oEi "(product-details-accordion|<details[^>]*>|class=\"[^\"]*accordion[^\"]*product[^\"]*\"|INGREDIENTS|HOW TO USE)" "$HTML_FILE" | head -5
```

**Regola**: se TROVI almeno UNO di questi signal nell'HTML del competitor → la sezione `main` della PDP **DEVE essere costruita come custom**:

```
current_page.main_strategy = "custom"
```

Significa che la sezione `main` del template JSON userà `product-information` come scheletro MA con custom liquid blocks aggiuntivi che replicano bundle/gift/urgency/trust/accordion. Inoltre alcuni elementi richiederanno slot `@app` (Katching, ReCharge) che l'utente attiverà manualmente dal theme editor a fine build.

Vedi `references/pdp-main-configuration.md` per il pattern dettagliato di main custom + fix #4 sotto per le sezioni reviews doppie.

Se invece il diagnostic NON trova niente → main standard product-information va bene:

```
current_page.main_strategy = "native"
```

⚠️ **Default sicuro**: se sei in dubbio, scegli `"custom"`. Il costo di costruire una main custom inutilmente è basso (manciata di blocchi custom_liquid in più). Il costo di lasciare main native quando il competitor ha bundle/gift è alto (la PDP è visivamente diversa, conversione collassa).

Mostra il risultato del diagnostic all'utente:

```
=== Diagnostic main-custom — risultati ===

Bundle picker:        ✓ trovato (<kaching-bundles-block>)
Free gift:            ✓ trovato ("free-gift" + "gift-tier")
Mystery gift:         ✗ non rilevato
Urgency:              ✓ trovato (countdown-timer)
Subscription:         ✗ non rilevato
Trust strip:          ✓ trovato (.trust-strip)
Accordion details:    ✓ trovato (<details> + "INGREDIENTS")

→ La sezione main DEVE essere CUSTOM (5 signals positivi su 6)
  La skill costruirà product-information con blocchi custom per:
  - Bundle picker (slot @app per Katching)
  - Free gift selector (block custom_liquid o @app a seconda del tema)
  - Urgency countdown (block custom_liquid)
  - Trust strip (block custom_liquid)
  - Accordion product details (block custom_liquid con <details>)
```

`AskUserQuestion`: "Confermi `main: custom` oppure forziamo `main: native`?"

Salva in `current_page.main_strategy` e passa alla mappatura sezioni 6.x.2.

---

#### 6.x.2.1 — Mappatura sezioni (per tutti i tipi di pagina)

Usa `WebFetch` sull'URL competitor + analizza **soprattutto gli screenshot** ricevuti per ricostruire la **lista ordinata di sezioni** che compongono la pagina.

**Regola anti-hallucination**: la lista sezioni deve corrispondere a quello che si VEDE nello screenshot. Niente sezioni "tipiche del template" aggiunte senza che siano visibili. Se vedi 7 sezioni → la lista ha 7 sezioni. Niente di più. Niente di meno.

Esempi di sezioni comuni (USA SOLO COME RIFERIMENTO MENTALE per riconoscerle, NON come template fisso):

- announcement-bar, hero, problem-hook, testimonial-quote, solution-intro, benefits-grid, ingredients-detail, social-proof, faq, cta-offer, trust-badges, footer (advertorial)
- hero-banner, main (gallery+info+buy), ingredients-strip, testimonial-carousel, how-it-works, comparison-table, faq, reviews-wall, closing-cta (PDP)
- announcement-bar, hero-video, featured-product, usp-strip, press-features, testimonial-grid, cta-bundle, blog-teaser, newsletter-signup, footer (home)

Salva in `current_page.sections` (array ordinato). Per ogni sezione:
- `nn`: numero (zero-padded, es. "01")
- `role`: ruolo semantico (es. "hero", "faq", "cta-offer")
- `notes`: descrizione visiva breve di quello che SI VEDE (es. "headline + subheadline + bottone coral + foto donna sorridente a destra")

Mostra all'utente la lista mappata + `AskUserQuestion`: "Confermi la struttura o vuoi modificarla? (Aggiungi/rimuovi/riordina/rinomina)"

### 6.x.3 — Creazione template Shopify

Genera nome template (kebab-case):
- **Funnel** → `<store.slug>-<funnel.type>` (es. `glowria-advertorial`)
- **PDP** → `<store.slug>-pdp` (es. `glowria-pdp`)
- **Home** → `<store.slug>-home` (es. `glowria-home`)
- **Extras** → `<store.slug>-<page-name>` (es. `glowria-contact`, `glowria-tracking`)

Genera prefisso sezioni (vedi `../create-new-pdp/references/section-naming.md`):
- `current_page.section_prefix = <store.slug>-<page-type-short>` (es. `glowria-adv`, `glowria-pdp`, `glowria-home`)

#### 6.x.3.a — Collision detection (OBBLIGATORIO, prima di creare qualsiasi file)

⚠️ **STOP critico**: prima di scrivere qualsiasi file, verifica se ESISTONO GIÀ template/sezioni con i nomi che stai per usare. Senza questo check rischi di sovrascrivere lavoro precedente, lasciare sezioni orfane (file in locale ma non più referenziate dal template), o accumulare detriti che confondono il theme editor.

**Esegui detection con `Bash`** sul workdir locale:

```bash
cd "<store.workdir_path>"

# Template di destinazione
ls "templates/<template-target>.json" 2>/dev/null

# Sezioni con il prefisso che andrai a creare
ls sections/<current_page.section_prefix>-*.liquid 2>/dev/null
```

Se almeno UNO dei due comandi trova qualcosa → c'è collisione. **Non procedere** alla creazione. `AskUserQuestion`:

- **(a) Cambia slug → versione `-v2` (Recommended in Gen-2)** → riusa lo stesso approccio dei nomi ma con suffix versione:
  - `target_slug = <slug>-v2` → template `<store.slug>-pdp-v2`, prefisso sezioni `<store.slug>-pdp-v2-` (incrementa il numero se `-v2` già esiste: `-v3`, ecc.)
  - Procedi a 6.x.3.b coi nomi nuovi (chiavi asset nuove → nessun overwrite del lavoro precedente)
  - I file vecchi restano intatti sia in locale che sul tema live (l'utente può cancellarli a mano dopo dal theme editor / file manager se vuole)
  - ⚠️ **Perché in Gen-2 il default è "Cambia slug" e non un cleanup**: `push_theme_asset` (MCP) **scrive/aggiorna** asset per chiave, ma **non cancella** asset remoti. Non esiste (ancora) un tool `delete-theme-asset`. Quindi sovrascrivere lo stesso prefisso lascia comunque eventuali orfani vecchi sul tema. Cambiare slug evita del tutto il problema delle chiavi orfane.

- **(b) Stop e usa il vecchio** → la skill termina, l'utente lavora sul template esistente con le altre skill (`/create-new-pdp` per editing, theme editor per modifiche dirette)

⚠️ Se l'utente dice "vai avanti, sovrascrivi" senza scegliere una delle opzioni: rifiuta. Spiega che sovrascrivere lo stesso prefisso in Gen-2 **non cancella** le sezioni vecchie dal tema (`push_theme_asset` non elimina asset) → restano file orfani che confondono il theme editor. Insisti su "Cambia slug" o "Stop".

> **Nota orphan cleanup (Gen-2 gap noto)**: la cancellazione di asset orfani dal tema live richiederebbe un futuro tool `mcp__working_suite_shopify_admin__delete_theme_asset` (Admin API `themeFilesDelete`). Finché non esiste, la strategia anti-orfani è **non riusare lo stesso slug** (opzione a). Non tentare cancellazioni via CLI.

#### 6.x.3.b — Creazione template scheletro

Crea il template Shopify scheletro (solo dopo che la collision detection è passata):

| Tipo pagina | File template | Layout | Note |
|-------------|---------------|--------|------|
| Funnel | `templates/page.<store.slug>-<type>.json` | `theme.chromeless` o `theme` (chiedi) | `AskUserQuestion`: "Vuoi header/footer? (Recommended chromeless per landing ads)" |
| PDP | `templates/product.<store.slug>-pdp.json` | (default) | Schema specifico PDP |
| Home | `templates/index.<store.slug>-home.json` | `theme` | NON sovrascrive `index.json`. L'utente lo assegnerà come homepage in Fase 6.x.7 |
| Extras | `templates/page.<store.slug>-<name>.json` | `theme` | Standard layout |

Scrivi `templates/<file>.json` con scheletro:
```json
{
  "sections": {},
  "order": [],
  "layout": "<layout>"
}
```

(Il layout key va omesso se è quello di default per il tipo template.)

Push dello scheletro vuoto via MCP (una chiamata, chiave esatta del template):
```
mcp__working_suite_shopify_admin__push_theme_asset
  {
    store_id: <store.id>,
    theme_id: <store.theme_id>,          // = main_theme_id da check_connection (Fase 2)
    assets: [
      { key: "templates/<file>.json", content: "<contenuto JSON completo del template scheletro>" }
    ]
  }
```

⚠️ **Retry su ogni `push_theme_asset`**: in caso di errore transitorio (HTTP 429/502/503/504) ritenta con backoff 10s → 20s → 40s (max 3 tentativi). Su 401/403 → la connessione è caduta: NON ritentare, STOP e manda l'utente a /configurations/stores a riconnettere la Custom App.

### 6.x.4 — Scelta modalità di costruzione sezioni

`AskUserQuestion`: "Come costruiamo le sezioni di questa pagina?"

- **Modalità A — One-shot (Recommended)** → costruisco TUTTE le sezioni in un'unica passata, push unico, tu verifichi alla fine. Più veloce, replica al volo. Tipica scelta quando hai screenshot chiari.
- **Modalità B — Sezione per sezione** → ogni sezione: write + push + verifica live + conferma → prossima. Più sicura per pagine complesse/lunghe.

Salva in `current_page.build_mode` ∈ {`A`, `B`}.

---

### 6.x.4.bis — CSS scraping del competitor (OPZIONALE, +20-25% di fedeltà visuale)

Senza CSS scraping, la skill ricostruisce HTML+CSS della sezione "a vista" dagli screenshot — fedeltà ~70%. Con CSS scraping che attinge ai file CSS sorgente del competitor (pubblicamente accessibili tramite `curl` come fa qualsiasi browser), la fedeltà sale a ~90%.

Fonte di verità: `references/css-scraping.md`.

`AskUserQuestion`: "Vuoi attivare CSS scraping per fedeltà visuale ~90% (+5-10 minuti) o saltare e accettare ~70%?"

- **Sì, attiva CSS scraping (Recommended per design complessi)** → procedi a 6.x.4.bis.1
- **No, salta** → vai a 6.x.5

#### 6.x.4.bis.1 — Estrazione CSS sorgente

```bash
# 1) Leggi l'HTML della pagina competitor
curl -sL "<current_page.url>" -A "Mozilla/5.0 (compatible; CSS-scraper)" \
  -o "/tmp/cb-comp-<NN>.html"

# 2) Estrai i <link rel="stylesheet"> e i <style> inline dall'HTML
#    (puoi farlo con grep o regex via Bash + python3)
```

Per ogni `<link rel="stylesheet" href="...">` trovato:
- Risolvi href relativo → URL assoluto (prepend competitor domain)
- `curl -sL "<css-url>" -o "/tmp/cb-comp-<NN>-<n>.css"`

Concatena tutti i CSS scaricati + i `<style>` inline in un file unico:
```bash
cat /tmp/cb-comp-<NN>-*.css > "/tmp/cb-comp-<NN>-aggregated.css"
# Aggiungi anche i <style> inline estratti dall'HTML
```

Salva il path in `current_page.competitor_css_path`.

Mostra all'utente:
```
✓ CSS scraping completato:
  - File HTML scaricato: /tmp/cb-comp-<NN>.html (XXX KB)
  - N file CSS scaricati: /tmp/cb-comp-<NN>-aggregated.css (YYY KB)
  - Z regole CSS rilevate
```

#### 6.x.4.bis.2 — Identificazione classi competitor per ogni sezione

Per ogni sezione mappata in 6.x.2, identifica nel HTML del competitor il **selettore CSS principale** che la rappresenta. Tre opzioni in ordine di affidabilità:

**Opzione 1 (preferita) — Tu identifichi la classe via screenshot di Inspect Element**

`AskUserQuestion`: "Per ogni sezione, mandami lo screenshot del browser DevTools → Elements panel con l'elemento root della sezione selezionato (così vedo `<section class='nome-classe-competitor'>` o simili). Oppure incolla la prima riga HTML dell'elemento root per ogni sezione."

L'utente apre DevTools, ispeziona la sezione, copia l'HTML del nodo root (es. `<section class="hero-banner-1234 above-fold">`).

Salva in `current_page.sections[<NN>].competitor_selector` per ogni sezione.

**Opzione 2 — Auto-discovery via WebFetch + euristiche**

Se l'utente non vuole fare Inspect manualmente: usa `WebFetch` per leggere l'HTML scaricato (`/tmp/cb-comp-<NN>.html`) e identifica i `<section>`, `<div>` di alto livello che corrispondono alle sezioni mappate in 6.x.2 (matching per ordine di apparizione + heading visibile).

Approssimativo, da validare con l'utente prima di procedere.

**Opzione 3 — Skip e ricostruisci a vista**

Se il competitor usa framework con classi auto-generate (CSS modules tipo `.css-1abc2de` di styled-components o Emotion) le classi sono linkate a markup specifico che noi non riproduciamo identico. In quel caso CSS scraping non aiuta sui selettori → torna alla modalità "ricostruisci a vista" (~70% fedeltà). Avvisa l'utente.

#### 6.x.4.bis.3 — Stato pronto per la build

Salva in `current_page.css_scraping_active = true` e procedi a 6.x.5 con accesso a:
- `current_page.competitor_css_path`: file CSS aggregato del competitor
- `current_page.sections[<NN>].competitor_selector`: selettore root per ogni sezione

Il flusso di build di 6.x.5.b userà queste informazioni per riapplicare le regole CSS del competitor sulle nostre classi scoped.

---

### 6.x.5 — STEP 1: Literal clone (lingua originale, input-driven anti-hallucination)

⚠️ **GUARD CRITICO — RESET DI SESSIONE PER OGNI NUOVA PAGINA** ⚠️

**Sei in 6.x.5 di una NUOVA pagina del loop di Fase 6.** Indipendentemente da quello che è successo nelle pagine precedenti di questa sessione (es. l'advertorial completato sia in Step 1 EN che Step 2 IT, la home parzialmente costruita, ecc.), **questa pagina riparte da zero col flow standard**:

1. Prima Step 1 (literal in lingua originale del competitor)
2. Verifica side-by-side col competitor live
3. Solo dopo conferma → Step 2 (traduzione IT)

**NON saltare lo Step 1 anche se** (lista non esaustiva di scuse che la skill potrebbe darsi):
- "Le pagine precedenti erano già state tradotte in IT, quindi anche questa va in IT diretta"
- "Il custom domain è italiano, quindi il cliente vuole IT subito"
- "L'utente ha già confermato la traduzione del flow per l'advertorial, sicuramente vale anche per la PDP"
- "La PDP ha meno testi del funnel, posso unire Step 1 e Step 2"
- "Per velocità faccio direttamente IT mantenendo struttura"

⛔ **Tutte queste sono scuse che bypassano il flow e producono pagine costruite senza verifica anti-hallucination**. Il flow Step 1 → Step 2 è **inderogabile per OGNI pagina**, sempre.

**Self-check obbligatorio** prima di procedere a 6.x.5.0:
- [ ] Sto per scrivere `default` in **lingua originale del competitor** (di solito EN)? Se sto per scrivere IT, fermati: sei nel posto sbagliato del flow
- [ ] L'utente ha fornito i testi della pagina (HTML salvato o copy/paste)? Se no, sono in 6.x.5.a, non più avanti
- [ ] La verifica visuale 1:1 con il competitor (6.x.5.d) è ancora da fare? Se l'ho saltata, torno a farla

Se uno di questi check fallisce → STOP. Riavvolgi al punto giusto del flow. Non ri-pushare in IT senza prima EN + verifica.

---

**Contesto operativo del workflow** (importante leggere): la skill costruisce un **clone autorizzato** del sito competitor sul tema dell'utente, partendo da contenuti **forniti direttamente dall'utente** (non scrappati da Claude). Il prodotto finale è un **derivative work**: lo Step 1 produce una bozza in lingua originale che lo Step 2 traduce in italiano e adatta al brand del cliente. L'utente è il proprietario del workspace target e ha autorizzazione del cliente per questo lavoro di clone + traduzione (use case tipico: agenzia che porta uno store competitor di un proprio cliente nel mercato IT). I testi forniti dall'utente sono input dell'utente, non riproduzione fatta da Claude — vanno strutturati nei `default` dello schema senza alterarli.

**Obiettivo dello Step 1**: replicare la pagina del competitor **esattamente identica**, sia visivamente che testualmente, **mantenendo i testi nella lingua originale del competitor** (di solito EN). Niente traduzione qui. Niente reinterpretazione. Niente "sezioni che ci starebbero bene". Solo quello che l'utente fornisce.

La traduzione IT è uno STEP 2 separato (Fase 6.x.6) che parte SOLO dopo che hai verificato che la pagina clonata è 1:1 visivamente identica al competitor.

#### 6.x.5.0 — Estrazione palette **pagina-specifica** (sostituisce/override la palette globale di Fase 4)

⚠️ **Critico**: la palette estratta in Fase 4 (brand discovery globale) è una **base di partenza** che vale per la home/identità globale del competitor. **Non è la palette di questa pagina specifica**. È molto comune nei competitor advertorial avere palette pagina-specifica diversa dal resto dello store (es. CTA verde acceso su un sito globalmente pastello, sticky bar nera con CTA giallo, accent ciano per emotional design). Costruire sezioni usando la palette globale produce risultati **visivamente sbagliati** anche se i testi sono perfetti.

Prima di costruire qualsiasi sezione di questa pagina, estrai i colori esatti che il competitor usa SU QUESTA PAGINA SPECIFICA.

**Procedura** (da fare **sempre**, anche se l'utente non ha attivato CSS scraping in 6.x.4.bis):

##### 1. Se hai HTML salvato (Formato A di 6.x.5.a, già richiesto)

Leggi l'HTML salvato della pagina e cerca:

- **CSS custom properties** definite in `:root` (vincolante: queste SONO la palette ufficiale del competitor):
  ```bash
  grep -E "^\s*--[a-zA-Z0-9_-]+:\s*#[0-9a-fA-F]" "<html-file>"
  # Oppure in python3:
  python3 <<'PY'
  import re
  with open("<html-file>") as f:
      content = f.read()
  # Cerca CSS variables in <style> e in :root selectors
  vars = re.findall(r'(--[a-zA-Z0-9_-]+):\s*(#[0-9a-fA-F]{3,8}|rgb[a]?\([^)]+\))', content)
  for name, value in vars:
      print(f"{name}: {value}")
  PY
  ```
- **Inline style su `<section>`/`<div>` di alto livello**: `<section style="background: #00C853; color: #fff;">` — pesa molto, è visibilmente applicato.
- **Hex/rgb più frequenti** nel CSS specifico della pagina (esclusi grayscale puri come `#000`/`#fff`/`#ccc`).

##### 2. Se CSS scraping è attivo (6.x.4.bis)

Stessa estrazione ma sui file CSS aggregati `/tmp/cb-css-<slug>/aggregated-<NN>.css`. Le `:root { --xxx }` definite lì sono ancora più affidabili (vivono sul tema, non solo nella pagina).

##### 3. Mostra la differenza all'utente

Presenta una tabella con tre colonne:

```
=== Palette pagina <NN> vs palette globale Fase 4 ===

VARIABILE          | PAGINA (competitor reale) | GLOBALE Fase 4 | NOTA
-------------------+---------------------------+----------------+----------
--adv-cta-bg       | #00C853 (verde acceso)    | #FF6B35 (orange) | OVERRIDE → uso il verde pagina
--adv-accent       | #34BBE5 (ciano)           | #FF6B35         | OVERRIDE → ciano
--adv-sticky-bg    | #111111 (nero)            | (n/a)           | NUOVO → uso nero
--adv-text         | #333333                   | #1A1A1A         | OVERRIDE → grigio competitor
--adv-card-bg      | #F7F9FA                   | #FAF6F2         | quasi uguale, OK
```

`AskUserQuestion`:
- **Sì, usa la palette di pagina (Recommended)** → `current_page.palette` = palette estratta. Override completo della palette globale per QUESTA pagina.
- **No, mantieni palette globale Fase 4** → `current_page.palette` = `brand.palette` (rischio: visual diverso dal competitor)
- **Mix custom**: l'utente sceglie variabile per variabile

Salva in `current_page.palette` la palette finale da usare in 6.x.5.b. **Tutti i `default` dei `setting type:"color"` delle sezioni di questa pagina useranno questa palette**, NON la `brand.palette` globale.

⚠️ La `brand.palette` globale resta comunque utile per: header/footer (se condivisi tra le pagine), home (se non ha override propri), pagine extra. Solo questa pagina specifica usa il proprio override.

#### 6.x.5.a-pre — Livelli di replica (L1-L4)

⚠️ **Default action**: per ogni testo che vedi negli screenshot/HTML competitor, il **comportamento di base è LITERAL nei `default`** (popolazione esatta dello schema con il testo del competitor). Questo vale per **TUTTI i tipi di pagina** (funnel, PDP, home, extras) **senza eccezioni e senza differenze**.

L1-L4 è una **classificazione facoltativa** che la skill può mostrare all'utente come riepilogo. Solo dopo che l'utente classifica esplicitamente certi pezzi come L4 e dice "questi voglio placeholder", la skill può convertire da literal a placeholder. **Senza conferma esplicita dell'utente, ogni testo va literal.**

**Errore tipico da NON fare**: applicare automaticamente "L4 → placeholder" senza chiedere all'utente. Se il competitor ha 4 paragrafi narrativi, la skill li mette literal nei `default`. Sarà l'utente a decidere se vuole convertirli in placeholder per "scriverli con la sua voce" o lasciarli literal.

**Workflow di raccolta testi è INVARIANTE per tutti i tipi di pagina**:
- Funnel/advertorial → user fornisce HTML salvato → testi letterali nei `default`
- PDP → user fornisce HTML salvato → testi letterali nei `default`
- Home → user fornisce HTML salvato → testi letterali nei `default`
- Extras → user fornisce HTML salvato → testi letterali nei `default`

Le differenze tra tipi di pagina riguardano la **STRUTTURA** (PDP ha sezione main custom, home ha index template, funnel ha layout chromeless), **NON il modo di trattare i testi forniti dall'utente**. Se il modello sente "PDP è speciale" → applica la specialità SOLO alla struttura, MAI ai testi. I testi vanno literal sempre, identico al funnel.

---

**Distinzione (informativa, non auto-applicata)**: "literal clone in lingua originale" si riferisce al posizionamento dei testi nei `default`. Diverse categorie di testo possono ricevere trattamenti diversi MA solo se l'utente conferma esplicitamente:

```
=== LIVELLI DI REPLICA ===

L1 — Struttura / CSS / Design
     Esempio: layout grid, padding, font-family, colori, border-radius.
     Trattamento: replica 1:1 (pattern di design pubblico, totalmente legittimo).
     Nello schema: nessun "testo" da inserire — vive nel CSS della sezione.

L2 — Labels funzionali / CTA / Headings strutturali (≤ 8 parole)
     Esempio: "Add to cart", "Frequently Asked Questions", "What's inside",
     "Shipping & Returns", "Most popular", "Free", "30-day guarantee".
     Trattamento: literal nei `default` dello schema (sono pattern marketing
     standard del mercato, non originali del competitor).

L3 — Bullet / badge / micro-claim brevi (frasi-claim ≤ 12 parole)
     Esempio: "Eliminate foul odour", "100% natural ingredients", 
     "Fast-acting formula", "Travel size friendly".
     Trattamento: literal nei `default` (sono claim brevi che condensano
     un beneficio, non scrittura creativa lunga del competitor).

L4 — Paragrafi narrativi / testimonial reali / individual reviews / about copy
     Esempio: "Dirty Secret" 4 paragrafi storytelling, recensione di Sarah
     che racconta la sua esperienza, biografia founder, lunga descrizione 
     del processo produttivo.
     Trattamento: NON literal. Inserisci PLACEHOLDER esplicito nei `default`
     tipo "[Paragrafo narrativo da scrivere — vedi competitor: <breve descrizione 
     del topic>]". L'utente compila in Step 2 con la sua voce/i suoi dati.
     Motivo: (a) i testi narrativi del competitor sono originali creativi; 
     riprodurli sotto altro brand sembra plagio anche dopo traduzione. 
     (b) i testimonial sono UGC di clienti del competitor — riprodurli sotto
     altro brand è problematico per identità di terzi. (c) Pratica: una PDP
     IT che traduce paragrafo per paragrafo un narrative US suona come "sito
     tradotto", convertendo male — l'utente la riscriverà comunque, tanto 
     vale lasciare placeholder dall'inizio.
```

**Decision tree** durante 6.x.5.a/b (RISPETTARE):

```
Per ogni testo che vedi nello screenshot/HTML competitor:

Step 1 — DEFAULT ACTION: literal nel `default` dello schema.
         Indipendentemente da L1/L2/L3/L4. Il testo va copiato così com'è.

Step 2 — Classificazione informativa (per riepilogo all'utente):
   - L1 (struttura/CSS): non è un "testo nei default" — vive nel CSS
   - L2 (label/CTA ≤ 8 parole): literal
   - L3 (bullet/microclaim ≤ 12 parole): literal
   - L4 (paragrafi narrativi/testimonial/about): literal di default,
     MA puoi proporre all'utente "vuoi convertire questo in placeholder
     per scriverlo con la tua voce in Step 2?"

Step 3 — Se utente esplicitamente dice "L4 → placeholder per questo pezzo":
         allora sostituisci il default literal con
         "[descrizione del topic — da scrivere]".
         SOLO con conferma esplicita.

Default sicuro: literal sempre. Mai auto-conversione a placeholder.
```

**Esempi pratici**:

| Testo competitor | Categoria | Trattamento |
|------------------|-----------|-------------|
| "Add to cart" | L2 | `"default": "Add to cart"` |
| "Frequently Asked Questions" | L2 | `"default": "Frequently Asked Questions"` |
| "BEST SELLER" | L2 | `"default": "BEST SELLER"` |
| "Eliminate foul odour" | L3 | `"default": "Eliminate foul odour"` |
| "100% natural ingredients" | L3 | `"default": "100% natural ingredients"` |
| "Free shipping over $50" | L3 | `"default": "Free shipping over $50"` |
| 4-paragraph "Our Dirty Secret" story | L4 | DEFAULT literal: copia i 4 paragrafi così come sono. Mostra all'utente "vuoi convertirlo in placeholder?". Se sì → `"default": "<p>[Storytelling — paragrafo 1: descrivi il problema...]</p>"`. Se no o nessuna risposta → resta literal |
| Testimonial reale "Sarah, NY: I've been using..." | L4 | DEFAULT literal MA con suggerimento all'utente: "questo è un testimonial reale del competitor. Vuoi placeholder per usare i tuoi?" |
| Long product description "Crafted with..." | L4 | DEFAULT literal. Suggerimento opzionale di placeholder all'utente |
| Founder biography paragrafo | L4 | DEFAULT literal. Suggerimento opzionale di placeholder |

A fine 6.x.5.a la skill mostra all'utente il riepilogo dei testi **classificati come L4** (lunghi/narrativi/testimonial), in due colonne:

```
=== Testi L4 — vuoi mantenerli literal o convertirli in placeholder? ===

Sezione 04 (story) — 4 paragrafi narrativi:
  LITERAL (default): testo competitor copiato così com'è
  oppure
  PLACEHOLDER: "[Storytelling — 4 paragrafi sul problema cliente]"

Sezione 07 (testimonials) — 5 quote di clienti:
  LITERAL (default): nomi + foto + quote del competitor
  oppure
  PLACEHOLDER: "[3-5 testimonial reali della tua clientela]"

Per default lascio tutto LITERAL. Dimmi quali specifici vuoi convertire 
in placeholder.
```

`AskUserQuestion`: "Conferma LITERAL per tutto, o mandami la lista di pezzi da convertire in placeholder."

⚠️ **Quando in dubbio tra L3 e L4**: il dubbio non importa — il default è literal in entrambi i casi. Lascia decidere l'utente se vuole un placeholder.

#### 6.x.5.a — Raccolta testi letterali dall'utente (NO scraping da parte di Claude)

⚠️ **Cambio di paradigma rispetto a versioni precedenti della skill**: la skill **NON estrae** testi né dagli screenshot via OCR né dal markup HTML del competitor via WebFetch. È l'**utente** che fornisce i testi. Claude li struttura nello schema senza alterarli.

Motivo del cambio:
- Anti-hallucination: gli screenshot full-page ridotti perdono leggibilità → OCR sbaglia → Claude riempie i buchi con testo plausibile → catastrofe.
- Il workflow corretto è quello dei team marketing professionali: il copy lo seleziona/copia il responsabile, non un agente automatico.
- Il modello che esegue la skill ha guardrail conservativi sull'estrazione di testi proprietari da fonti terze. Se l'utente fornisce i testi direttamente, sono "input dell'utente" — pienamente legittimi da strutturare nello schema.

**Procedura per la raccolta**:

Chiedi all'utente di fornire i testi delle sezioni in **uno** dei seguenti formati (in ordine di preferenza):

**Formato A — HTML salvato della pagina competitor (Recommended se autorizzato)**

L'utente apre la pagina competitor su Chrome → `Cmd+S` → "Webpage, Complete" → salva il file `.html` → te lo manda in chat (o ti dice il path).

Tu leggi il file con `Read`. È testo strutturato, deterministico, niente OCR. Da lì estrai per ogni sezione i testi visibili (`<h*>`, `<p>`, `<a>`, `<button>`, `<li>`, `alt=`, ecc.) e li strutturi nei `default` dello schema in 6.x.5.b.

**Formato B — Copy/paste manuale per sezione (sempre disponibile)**

Per ogni sezione mappata in 6.x.2 chiedi all'utente:
```
=== Sezione 02 — hero ===

Selezionando manualmente il testo dal sito competitor (Cmd+A su quella sezione → Cmd+C → Cmd+V qui), incollami:

  Headline:           ___
  Subheadline:        ___
  Eyebrow text:       ___
  CTA primary label:  ___
  CTA primary link:   ___ (URL completo dell'href del bottone)
  Trust badges:       ___ (uno per riga)
  Other elements:     ___ (qualsiasi altro testo visibile)

Salta le righe che non esistono nella sezione.
```

L'utente incolla. Tu strutturi.

**Formato C — Copy/paste libero, tu strutturi (per pagine semplici)**

L'utente incolla un blob di testo per sezione (senza struttura precisa). Tu identifichi heading, subheading, CTA, ecc. dal contesto (lo capisci dalla formattazione — heading di solito è la prima riga, CTA è in stile diverso). Mostra all'utente la struttura proposta e fai confermare prima di procedere.

**Regole comuni a tutti i formati**:

1. **I testi vanno nei `default` dello schema così come l'utente li fornisce**. Niente parafrasi. Niente correzioni di typos. Niente "miglioramenti".
2. **Lingua originale**: se l'utente fornisce inglese, lo schema mostra inglese. Se francese, francese. La traduzione IT è Step 2.
3. **Non aggiungere sezioni/campi che l'utente non ha fornito**. Se la mappatura ha 3 bullet e l'utente ne fornisce 3, lo schema ha 3 bullet. Niente di più.
4. **Se mancano testi** per qualche sezione mappata: chiedi esplicitamente all'utente "mi mancano i testi di queste sezioni: <lista>". Non inventare. Non procedere fino a quando non li hai.
5. **Recensioni / testimonial**: l'utente decide se passare i testimonial reali del competitor (anche con nomi/foto), placeholder generici ("[testimonial]"), o se rimuovere la sezione (preferibile se non ha testimonianze proprie). Mai trasferire automaticamente nomi/foto reali — sono identità di terzi.
6. **Footer / disclaimer legali**: chiedi se l'utente vuole il testo del competitor come riferimento di struttura (poi sostituiremo coi suoi dati aziendali in Step 2) o se vuole già metterli in versione finale (suoi dati).

Salva i testi raccolti in `current_page.user_provided_texts[<NN>]` per ogni sezione.

**Mostra all'utente il riepilogo** dei testi raccolti, sezione per sezione, e chiedi con `AskUserQuestion`:
- **Tutto corretto, procedi alla build** → vai a 6.x.5.b
- **Correggo alcune righe** → l'utente indica cosa cambiare
- **Mi manca questa sezione** → l'utente la fornisce e ripresenti

#### 6.x.5.b — Build sezioni con testi forniti dall'utente

Per ogni sezione in `current_page.sections`, scrivi `sections/<current_page.section_prefix>-<NN>-<role>.liquid` rispettando le regole specificate sotto.

**Inserisci nei `default` dello schema ESATTAMENTE i testi che l'utente ha fornito in 6.x.5.a**, lingua originale. Non tradurre. Non parafrasare. Non aggiungere sezioni o campi che non l'utente ha fornito.

**Regole per ogni sezione costruita** (rispettare TUTTE):

#### A) HTML + CSS scoped 1:1 con il competitor

- Markup mobile-first (parti dal layout 375px), poi `@media (min-width: 750px)` per tablet, `@media (min-width: 1200px)` per desktop. Match preciso degli screenshot mobile + desktop ricevuti in 6.x.1.
- Tutte le classi CSS dentro la sezione iniziano col prefisso file: tutte le classi di `glowria-adv-02-hero.liquid` iniziano con `.glowria-adv-02-` (no name conflicts cross-section).
- Stili dentro `<style>` inline al file `.liquid` (NO `assets/*.css` esterni, a meno che strettamente necessario).
- Niente framework esterni (no Tailwind, no Bootstrap).
- Font: usa `brand.fonts.heading` per H1/H2/H3, `brand.fonts.body` per paragrafi. Importa con `@import url('https://fonts.googleapis.com/css2?family=...')` in cima alla `<style>`.
- Colori: usa `brand.palette.*` come default dei `settings`, mai hardcoded nel markup. Esempio CTA button:
  ```liquid
  <style>
    .{{ section.id }}-cta {
      background: {{ section.settings.cta_bg | default: 'BRAND_PRIMARY_HEX' }};
      color: {{ section.settings.cta_text | default: '#fff' }};
    }
  </style>
  ```
  Dove `BRAND_PRIMARY_HEX` è il valore di `brand.palette.primary` di Fase 4 hardcoded come **default** nel CSS — l'utente può poi cambiarlo dal theme editor (settings color).

**Se CSS scraping è attivo** (`current_page.css_scraping_active === true`): per ogni sezione, prima di scrivere CSS proprio, **estrai dal file `current_page.competitor_css_path` le regole rilevanti** che colpiscono il `competitor_selector` di quella sezione e i suoi figli. Riapplica quei valori (padding, margin, font-size, font-weight, line-height, color, background, border-radius, box-shadow, ecc.) sulle nostre classi scoped. Vedi `references/css-scraping.md` per la procedura dettagliata di estrazione + riapplicazione.

**Se la pagina è una PDP** (`current_page.type === "pdp"`): la sezione `main` (`product-information` nativa Shopify) richiede configurazione speciale per matchare l'above-the-fold del competitor. Vedi `references/pdp-main-configuration.md` per:
- Checklist analisi above-the-fold competitor (pills, badge, rating, price, variant picker, quantity, buy buttons, accelerated checkout, bundle widget, urgency, trust badges)
- Mapping checklist → configurazione blocchi JSON del template (`disabled: true/false` per native blocks, `block_order` corretto)
- Custom CSS per stylizzare il button add-to-cart native (colore/font/border-radius/dimensioni del competitor)
- Slot `@app` per Katching Bundles / subscription / recensioni se il competitor li usa
- Block `hide_native_price` per nascondere il prezzo native quando il competitor non lo mostra above-the-fold
- Istruzioni utente per attivare manualmente app blocks (Katching, ReCharge, ecc.) dal theme editor

⚠️ **Critico per PDP**: se il competitor ha un bundle widget (es. "Buy 1 / Buy 2 / Buy 3" con sconti), un subscription picker, recensioni custom, ecc., questi vanno **replicati 1:1 above-the-fold**. Anche se sono gestiti da app esterne (non costruite da noi), la skill deve **predisporre lo slot** (block `@app` nel `block_order` giusto) e istruire l'utente come attivarli dal theme editor. Senza questi elementi, la PDP è un guscio vuoto rispetto al competitor — il valore principale di una PDP advertorial-driven è esattamente nel bundle widget + CTA stilizzato.

#### B) Editabilità COMPLETA dal theme editor (HARD RULE)

Vedi `references/editability-and-app-blocks.md` per dettaglio. Riassunto operativo:

- **Testi** → schema `settings`:
  - `type: "text"` per heading singola riga (con `default` popolato dal testo IT tradotto)
  - `type: "textarea"` per testo multiriga semplice
  - `type: "richtext"` per paragrafi con bold/italic/link inline (WYSIWYG nell'editor)
  Render via `{{ section.settings.<id> }}`.

- **Immagini** → schema `settings`, `type: "image_picker"` (MAI `<img src="cdn-url">` hardcoded). Default vuoto. Markup gestisce stato vuoto con SVG placeholder:
  ```liquid
  {% if section.settings.hero_image %}
    <img src="{{ section.settings.hero_image | image_url: width: 1600 }}"
         alt="{{ section.settings.hero_image.alt | escape }}" loading="lazy">
  {% else %}
    <div class="img-placeholder">[Hero desktop — consigliato 1920×800]</div>
  {% endif %}
  ```

- **Link / CTA** → coppia di `settings`:
  - `type: "url"` (es. `cta_url`) — default = `funnel.cta_url` se è un funnel, altrimenti vuoto
  - `type: "text"` (es. `cta_label`) — testo bottone, default tradotto IT

- **Colori** → `type: "color"` con default da `brand.palette.*`. L'utente può override.

- **Liste ripetibili** (FAQ items, testimonial, benefit cards, bullet points, gallery items) → schema `blocks` con `type` dedicato, iterazione con `{% for block in section.blocks %}` nel markup. `max_blocks` sensato (es. 12 per FAQ, 6 per testimonial).

- **App blocks support per PDP** (CRITICO) → ogni sezione del template PDP che ospita "logica prodotto" (main, buy-buttons area, bundles area) deve dichiarare nel suo `blocks` array dello schema:
  ```json
  "blocks": [
    { "type": "@app" },
    { "type": "<custom-block-1>" },
    { "type": "<custom-block-2>" }
  ]
  ```
  Questo permette al theme editor di **inserire blocchi di app** (es. Katching Bundles, subscription pickers, recensioni Judge.me/Loox) negli slot voluti senza modificare il liquid. **Senza `@app` non funziona Katching Bundles**.

- **Default sensati**: ogni `setting` e `block` ha `default` popolato con i **testi letterali estratti in 6.x.5.a** (lingua originale del competitor) / colori brand / immagini placeholder. Così la pagina live mostra subito esattamente quello che vede il competitor, senza passare dall'editor.

#### C) Schema completo di esempio (template per ogni sezione)

```liquid
{% schema %}
{
  "name": "<NOME LEGGIBILE SEZIONE>",
  "tag": "section",
  "class": "<prefix>-<NN>",
  "settings": [
    { "type": "text",         "id": "heading",        "label": "Titolo",         "default": "..." },
    { "type": "richtext",     "id": "body",           "label": "Paragrafo",      "default": "<p>...</p>" },
    { "type": "image_picker", "id": "hero_image",     "label": "Immagine hero" },
    { "type": "url",          "id": "cta_url",        "label": "Link CTA",       "default": "" },
    { "type": "text",         "id": "cta_label",      "label": "Testo CTA",      "default": "Scopri di più" },
    { "type": "color",        "id": "cta_bg",         "label": "Colore CTA",     "default": "#E36B6E" }
  ],
  "blocks": [
    { "type": "@app" },
    {
      "type": "faq_item",
      "name": "Domanda FAQ",
      "settings": [
        { "type": "text", "id": "question", "label": "Domanda", "default": "..." },
        { "type": "richtext", "id": "answer", "label": "Risposta", "default": "<p>...</p>" }
      ]
    }
  ],
  "max_blocks": 12,
  "presets": [
    { "name": "<NOME LEGGIBILE SEZIONE>" }
  ]
}
{% endschema %}
```

#### D) Aggiornamento template JSON

Per ogni sezione creata, aggiungi entry al template `templates/<file>.json`:

```json
"<NN>": {
  "type": "<prefix>-<NN>-<role>",
  "settings": {}
}
```

E aggiungi la chiave all'array `order` nella posizione corretta.

#### 6.x.5.c — Push literal

##### Modalità A (one-shot)

Scrivi tutti i file `.liquid` in parallelo (multiple `Write` in un singolo messaggio). Aggiorna il template JSON con tutte le sezioni in `sections{}` + `order[]`. Poi un solo push MCP batchato con tutte le chiavi esatte (template + ogni sezione):

```
mcp__working_suite_shopify_admin__push_theme_asset
  {
    store_id: <store.id>,
    theme_id: <store.theme_id>,           // = main_theme_id da check_connection
    assets: [
      { key: "templates/<file>.json",                content: "<contenuto JSON completo del template>" },
      { key: "sections/<prefix>-01-<role>.liquid",   content: "<contenuto liquid completo>" },
      { key: "sections/<prefix>-02-<role>.liquid",   content: "<contenuto liquid completo>" }
      // ... una entry per ogni sezione, chiave ESATTA (mai glob), max 50 asset per chiamata
    ]
  }
```

⚠️ **Batch > 50 asset**: se la pagina ha più di 50 asset (raro), spezza in più chiamate `push_theme_asset` da ≤50 ciascuna.

⚠️ **Retry obbligatorio su OGNI chiamata `push_theme_asset`** (clone = molti push in loop, alto rischio rate-limit): su 429/502/503/504 ritenta con backoff 10s → 20s → 40s (max 3 tentativi). Su 401/403 → STOP, manda l'utente a /configurations/stores a riconnettere la Custom App (NON "rigenera token", NON CLI).

##### Modalità B (sezione per sezione)

Loop: per ogni sezione, write file → aggiorna template JSON → push MCP della singola sezione + template → utente verifica live → conferma → prossima sezione.

Per ogni iterazione, una chiamata MCP con 2 asset (sezione corrente + template aggiornato):
```
mcp__working_suite_shopify_admin__push_theme_asset
  {
    store_id: <store.id>,
    theme_id: <store.theme_id>,
    assets: [
      { key: "sections/<prefix>-<NN>-<role>.liquid", content: "<contenuto liquid>" },
      { key: "templates/<file>.json",                content: "<template aggiornato con la sezione>" }
    ]
  }
```
(Stesso retry/backoff e gestione 401/403 della Modalità A — applicalo a OGNI push del loop.)

A ogni conferma di sezione, prima di passare alla successiva, `AskUserQuestion`: "Confermata. Continua sezione-per-sezione, o passo al one-shot per le restanti N?"

#### 6.x.5.d — Verifica visuale 1:1 col competitor (literal phase)

Chiedi all'utente di aprire la pagina live (in modalità "draft / unpublished theme preview" se possibile) e fare confronto **side-by-side con la pagina competitor**:

```
Apri side-by-side:
  - Sinistra:  https://<competitor.url>
  - Destra:    https://<store.shopify_domain>/<live-url>?preview_theme_id=<store.theme_id>

Verifica per ogni sezione:
  [ ] Stesso ordine sezioni (top-to-bottom)
  [ ] Stessi testi (in lingua originale, parola per parola)
  [ ] Stessi colori e tipografia
  [ ] Stesso layout mobile (375px) e desktop (1440px)
  [ ] Stessi CTA labels e link
  [ ] Niente sezioni in più o in meno
```

`AskUserQuestion`:
- **Pagina identica al competitor** → procedi a 6.x.6 (Step 2 — Localizzazione IT)
- **Ho trovato discrepanze** → l'utente le elenca, applichi fix puntuali sezione-per-sezione, ripushi, ripeti la verifica
- **Voglio rivedere l'estrazione** → torna a 6.x.5.a per la sezione interessata

⚠️ **Non procedere alla localizzazione IT finché la pagina literal non è verificata identica al competitor.** Se traduci in IT prima della verifica, eventuali discrepanze testuali si moltiplicano per due (lingua originale + traduzione → entrambe sbagliate).

---

### 6.x.6 — STEP 2: Localizzazione IT

⚠️ **GUARD CRITICO — VERIFICA INGRESSO STEP 2** ⚠️

Prima di entrare in 6.x.6 per QUESTA pagina, devi aver completato:

- [ ] **6.x.5.0** — Estrazione palette pagina-specifica → conferma utente
- [ ] **6.x.5.a** — Raccolta testi LETTERALI in lingua originale dall'utente (HTML salvato o copy/paste) → conferma utente
- [ ] **6.x.5.b** — Build sezioni con testi in **lingua originale** (EN/altro) nei `default` dello schema
- [ ] **6.x.5.c** — Push selettivo del template + sezioni
- [ ] **6.x.5.d** — Verifica side-by-side con competitor live → utente conferma "pagina identica al competitor"

Se UNO di questi check non è stato fatto per QUESTA pagina specifica → **NON entrare in 6.x.6**. Torna al step mancante.

**Self-check**: stai per leggere file `.liquid` con `default` in lingua originale (EN) per tradurli? Se i `default` sono già in IT, vuol dire che hai già fatto Step 2 (o hai saltato Step 1) — verifica.

**Errore tipico da NON fare**: in una sessione lunga dove la pagina precedente è già stata costruita+tradotta, partire una nuova pagina e andare dritto in 6.x.6 perché "tanto il workflow l'ho capito". **Ogni pagina ha il suo ciclo Step 1 → Step 2 indipendente**, niente shortcut.

---

**Obiettivo dello Step 2**: tradurre i testi delle sezioni dall'originale (EN/altro) a italiano, mantenendo:
- **Struttura identica**: stesse sezioni, stesso ordine, stessi `settings`/`blocks`
- **Layout identico**: nessuna modifica al markup HTML/CSS
- **Tono di voce**: come definito in `brand.tone` di Fase 4

Niente nuovi contenuti. Niente aggiunte. Solo traduzione dei testi che sono GIÀ nei `default` dello schema.

#### 6.x.6.a — Traduzione default schema

Per ogni sezione, leggi il file `.liquid` esistente, identifica nel `{% schema %}` tutti i `default` testuali (settings tipo `text`/`textarea`/`richtext` + `block.settings.<id>` con `default`), e proponi la traduzione IT.

**Regole**:
- Modifica SOLO i `default` dei `settings` e `blocks` del `{% schema %}`. NON toccare il markup, le classi CSS, gli `<script>`, `<style>`, attributi `data-*`.
- Mantieni stessa lunghezza/peso del testo originale (importante per layout — un titolo molto più lungo può rompere il design).
- Applica le regole di `references/localization-it.md`:
  - Mantieni tono di voce competitor (`brand.tone` di Fase 4)
  - Modera claim cosmetici/integratori secondo normativa IT
  - Adatta valute/unità di misura/formati indirizzo se compaiono
  - I testimonial: NON tradurre, **rimuovi** (sono di clienti del competitor, non nostri). Sostituisci con placeholder o con testimonial nostri se l'utente li ha
  - Numeri specifici (es. "47% reduction in 28 days"): conferma con utente che li abbiamo davvero, altrimenti modera o rimuovi

**Mostra all'utente diff sezione-per-sezione** in formato compatto:

```
=== Sezione 02 — hero ===
heading
  EN: "The cream that finally works"
  IT: "La crema che finalmente funziona"

subheading
  EN: "Visible results in 28 days, 100% money-back guarantee"
  IT: "Risultati visibili in 28 giorni, garanzia soddisfatto o rimborsato"

cta_label
  EN: "Get yours now"
  IT: "Ordina la tua"
```

`AskUserQuestion` di switch:
- **Approvo tutto, applica e pusha** → batch translation + push selettivo
- **Sezione per sezione, voglio rivedere** → loop con conferma per ogni sezione
- **Cambio tono di voce** → l'utente specifica più diretto/pacato/scientifico, rigenera

#### 6.x.6.b — Push translation

Un solo push MCP batchato con le sole sezioni tradotte (chiavi esatte, niente glob):
```
mcp__working_suite_shopify_admin__push_theme_asset
  {
    store_id: <store.id>,
    theme_id: <store.theme_id>,            // = main_theme_id da check_connection
    assets: [
      { key: "sections/<prefix>-01-<role>.liquid", content: "<liquid con default tradotti IT>" },
      { key: "sections/<prefix>-02-<role>.liquid", content: "<liquid con default tradotti IT>" }
      // ... una entry per ogni sezione tradotta (max 50 per chiamata)
    ]
  }
```

Niente push del template `.json` (i `default` cambiano dentro i `.liquid`, ma la struttura del template JSON resta identica).

⚠️ Stesso retry/backoff su OGNI push (429/502/503/504 → 10s/20s/40s, max 3). 401/403 → STOP → riconnetti la Custom App su /configurations/stores.

#### 6.x.6.c — Verifica final IT

Chiedi all'utente di aprire la pagina live e fare uno smoke check:

```
Verifica IT (URL: <live-url>):
  - [ ] Tutti i testi sono in italiano (no residui inglesi tranne brand name se voluto)
  - [ ] Layout intatto (testi più lunghi/corti non rompono il design)
  - [ ] Mobile 375px e desktop 1440px coerenti con la versione literal
  - [ ] Tono di voce coerente sezione per sezione
  - [ ] Nessun claim cosmetico/integratore problematico per normativa IT
```

`AskUserQuestion`:
- **Pagina IT OK** → procedi a 6.x.7 (caso speciale per tipo pagina)
- **Aggiusto questi testi** → fix puntuali e ripush

### 6.x.7 — Caso speciale per ogni tipo di pagina

#### Se la pagina è una PDP

Dopo il push del template + sezioni:
```
=== Crea il prodotto in Shopify ===

1. Admin → Products → Add product.
2. Title: scegli il nome (può essere quello del competitor tradotto IT, o nuovo).
3. Scrivi descrizione base (anche placeholder, useremo il template).
4. Aggiungi varianti, prezzi, immagini prodotto come desideri.
5. Sidebar destra → Online store → Theme template → seleziona `<store.slug>-pdp`.
6. Save.
7. Tornami con:
   - Slug del prodotto (es. `crema-corpo-autoabbronzante`)
   - URL live (`https://<store.shopify_domain>/products/<slug>`)
```

Aspetta che l'utente confermi e mandi l'URL. Salva in:
- `pages.pdp.product_slug`
- `pages.pdp.cta_url` (URL pubblico del prodotto live, useralo come default in tutti i CTA dei funnel/home costruiti dopo)

#### Se la pagina è un Funnel

Dopo il push del template + sezioni:
```
=== Crea la Page in Shopify Admin ===

1. Admin → Online Store → Pages → Add page.
2. Title: <titolo visibile della pagina, tradotto IT>.
3. Content: lascialo vuoto (le sezioni vengono dal template).
4. Sidebar destra → Theme template → seleziona `<store.slug>-<type>`.
5. Save.
6. Tornami con lo slug della page (es. `scopri-perche`).
```

Aspetta conferma e slug. Salva in `pages.funnel.page_slug`.

⚠️ Se in Fase 3 era stata richiesta anche la PDP, il funnel l'ha già nel `default` dei `cta_url` perché è stato costruito DOPO la PDP. Verifica i CTA: aprendo il funnel live tutte le CTA devono già puntare alla PDP creata.

#### Se la pagina è la Home

Dopo il push del template + sezioni:
```
=== Assegna come homepage ===

Hai 2 opzioni:

OPZIONE A (Recommended) — assegna come homepage:
  1. Admin → Online Store → Themes → <theme name> → Customize.
  2. Top-left picker → Home page.
  3. Sidebar destra → Theme template → seleziona `<store.slug>-home`.
  4. Save.

OPZIONE B — sostituisci direttamente l'index.json di default:
  Te lo posso fare io con un push MCP che sovrascrive `templates/index.json`.
  Più semplice ma SOVRASCRIVE LA HOME LIVE e non è reversibile senza un altro push.
  Conferma se vuoi questa strada.
```

`AskUserQuestion` per scegliere.

⚠️ **OPZIONE B sovrascrive la home live**: prima di pushare `templates/index.json`, conferma esplicitamente con un secondo `AskUserQuestion` ("Confermi di sovrascrivere la homepage live? L'azione non è reversibile senza un altro push."). Salva una copia locale del vecchio `templates/index.json` come backup (es. `templates/index.backup-pre-clone.json`) PRIMA di sovrascrivere.

Solo dopo la conferma, push MCP (chiavi esatte: l'index + tutte le sezioni home):
```
mcp__working_suite_shopify_admin__push_theme_asset
  {
    store_id: <store.id>,
    theme_id: <store.theme_id>,            // = main_theme_id da check_connection
    assets: [
      { key: "templates/index.json",                     content: "<nuova struttura home>" },
      { key: "sections/<store.slug>-home-01-<role>.liquid", content: "<liquid>" },
      { key: "sections/<store.slug>-home-02-<role>.liquid", content: "<liquid>" }
      // ... una entry per ogni sezione home, chiave esatta, max 50 per chiamata
    ]
  }
```
(Retry 429/502/503/504 → 10s/20s/40s ×3; 401/403 → STOP → riconnetti Custom App.) Vedi `references/home-template.md`.

#### Se la pagina è un Extra (contattaci, traccia ordine, FAQ, about, blog, collection)

Stessa procedura del Funnel (Page in Admin → assegna template). Vedi `references/page-types-extras.md` per template-specifici (es. `traccia-il-tuo-ordine` ha campo input order tracking).

### 6.x.8 — Loop alla pagina successiva

Quando l'utente conferma OK sia in 6.x.5.d (literal) sia in 6.x.6.c (IT), e ha completato il caso speciale di 6.x.7 (creazione product/page/assignment): marca `current_page.status = "done"`, passa alla prossima in `scope.pages_ordered`.

Quando tutte le pagine in `scope.pages_ordered` sono done → vai a Fase 7.

---

## Fase 7 — Pagine extra opzionali

`AskUserQuestion` multi-select: "Vuoi aggiungere altre pagine prima di chiudere?"

Opzioni (vedi `references/page-types-extras.md` per template specifici):
- **Contattaci** (`contact`) — modulo + email + telefono + indirizzo
- **Traccia il tuo ordine** (`traccia-il-tuo-ordine`) — input numero ordine + email
- **FAQ** — accordion editabile da blocks
- **Chi siamo** (`about`) — storia brand
- **Blog landing** — lista articoli
- **Collection page** — griglia prodotti

Per ogni pagina extra selezionata, ripete il sub-workflow 6.x.1 → 6.x.8 (con peso ridotto sull'analisi visiva — le extras sono più standardizzate). Vale sempre la regola: prima literal clone (lingua originale), poi traduzione IT.

Quando l'utente non vuole aggiungere altro → vai a Fase 8.

---

## Fase 8 — Verifica finale + handoff

Mostra checklist finale:

```
=== Riepilogo clone competitor ===

Pagine create:
  ✓ Funnel <type>:    https://<shop>/pages/<funnel.slug>
  ✓ PDP:              https://<shop>/products/<pdp.slug>
  ✓ Home:             assegnata come homepage (template <store.slug>-home)
  ✓ Contattaci:       https://<shop>/pages/contact
  ...

Brand applicato:
  - Logo:             <store.logo_url>
  - Email:            <store.support_email>
  - Telefono:         <store.phone>
  - Indirizzo:        <store.legal_address>

Cose da verificare manualmente:
  [ ] Tutte le pagine renderizzano mobile + desktop
  [ ] Tutti i CTA dei funnel puntano alla PDP creata
  [ ] Home → click prodotto → arriva alla PDP
  [ ] Theme editor: ogni sezione mostra settings editabili
  [ ] PDP: click "+ Add block" mostra app blocks (Katching Bundles, ecc.)
  [ ] Footer corretto su tutte le pagine
```

Suggerisci passi successivi (fuori scope skill):
- Configurare policy (Privacy, Terms, Refund, Shipping) — vedi memoria di sessione, le abbiamo già scritte per altri store
- Installare app: Klaviyo, GTM, Meta Pixel, Triple Whale
- Configurare spedizioni / pagamenti / tax dello store
- Pushare il dominio live, configurare DNS

`AskUserQuestion` finale: "Tutto OK? Posso chiudere la sessione | Ho un fix da fare prima."

Se chiudi: dichiara il clone pronto.

---

## Troubleshooting rapido

| Sintomo                                         | Causa                                              | Fix                                                                  |
|-------------------------------------------------|----------------------------------------------------|----------------------------------------------------------------------|
| `check_connection` ritorna `connected: false` / 401 / 403 | Custom App non collegata o token Admin non valido per questo store | STOP. Manda l'utente su /configurations/stores a (ri)connettere la Custom App. NON "rigenera token", NON CLI |
| `push_theme_asset` ritorna 401 / 403            | La connessione è caduta a metà sessione            | STOP, riconnetti la Custom App su /configurations/stores, poi riprova il push |
| `push_theme_asset` ritorna 429 / 502 / 503 / 504 | Rate-limit o errore transitorio Admin API          | Retry con backoff 10s → 20s → 40s (max 3 tentativi). Se persiste, riprova più tardi |
| Tool `mcp__working_suite_shopify_admin__*` assenti | MCP non configurato per la sessione                | STOP: "MCP non configurato per questa sessione; riapri la chat builder." NIENTE fallback CLI/curl |
| `check_connection` ritorna `main_theme_id: null` | Tema principale non risolto                         | STOP: "Tema principale non risolto per questo store." |
| `push_theme_asset` fallisce con "Liquid syntax error" | Schema malformato                                  | Leggi error line, `Edit` puntuale del file, ri-pusha l'asset        |
| WebFetch del competitor restituisce vuoto       | JS-rendered content / paywall                      | Chiedi all'utente screenshot full-page + URL, lavora dagli screenshot |
| Brand discovery estrae colori sbagliati         | CSS dinamico / dark mode default                   | Mostra colori estratti, lascia override utente in 4.3                 |
| PDP non mostra app blocks editabili             | Schema sezione manca `{"type": "@app"}` nei blocks | Edit schema, riaggiungi `@app`, ripusha sezione                      |
| Home non si vede sul dominio                    | Template non assegnato come homepage in Theme settings | Admin → Online Store → Themes → Customize → assegna template       |
| CTA del funnel puntano a `#`                    | PDP creata DOPO il funnel (ordine sbagliato)        | Edit setting `cta_url` di tutte le sezioni funnel, sostituisci URL PDP |
| Mobile text overflow                            | Font-size fissi in px                              | Sostituisci con `clamp(1rem, 4vw, 1.5rem)` o media query             |
| Sezioni orfane sul tema dopo riuso slug         | `push_theme_asset` non cancella asset; slug riusato | Cambia slug (`-v2`); orphan cleanup richiede un futuro tool `delete_theme_asset` |

---

## File correlati (references)

### Specifici di questa skill (in `skills/clone-competitor-store/references/`)

- `competitor-discovery.md` — come WebFetch + estrazione palette/font/logo dal markup competitor
- `visual-replication.md` — regole per ricreare design 1:1 da screenshot + URL (mobile-first, breakpoint, classi scoped)
- `css-scraping.md` — procedura per scaricare CSS pubblico del competitor e riapplicarlo (Fase 6.x.4.bis, fedeltà ~90%)
- `editability-and-app-blocks.md` — hard rule editabilità schema, blocks pattern, `@app` block per PDP (Katching Bundles)
- `pdp-main-configuration.md` — analisi above-the-fold competitor + config sezione `main` PDP (disabled native blocks, custom CSS button, slot @app per Katching/subscription)
- `localization-it.md` — principi traduzione ENG → IT, tono di voce, cosa fare con claim regulatory IT
- `home-template.md` — gestione `index.<slug>.json` + assignment manuale come homepage
- `page-types-extras.md` — template per contattaci, traccia ordine, FAQ, about, blog, collection

### Auth + push (Gen-2, MCP — NO file CLI)

- Auth: tool MCP `mcp__working_suite_shopify_admin__check_connection { store_id }` → restituisce `connected` + `main_theme_id`. NESSUN file di auth CLI, NESSUN Theme Access token. (NON leggere mai `../create-new-pdp/references/auth-pattern.md`: è il pattern CLI Gen-1, obsoleto.)
- Push: tool MCP `mcp__working_suite_shopify_admin__push_theme_asset { store_id, theme_id, assets:[{key,content}] }` → una chiave esatta per asset (max 50/chiamata), retry 429/502/503/504 con backoff 10s/20s/40s. (NON usare comandi `theme push` CLI.)

### Riusati dalle skill esistenti (path relativo, solo convenzioni — NON auth/push)

- `../create-new-pdp/references/section-naming.md` — convenzioni prefissi e naming sezioni
- `../create-new-pdp/references/section-schema-patterns.md` — pattern schema editabile
- `../create-new-funnel/references/brand-identity-discovery.md` — base per `competitor-discovery.md` (estesa)
- `../create-new-funnel/references/funnel-types.md` — strutture base advertorial/listicle/quiz
- `../create-new-funnel/references/page-template-layout.md` — chromeless vs theme layout
- `../create-new-funnel/references/funnel-image-specs.md` — specs immagini per ruolo sezione funnel
