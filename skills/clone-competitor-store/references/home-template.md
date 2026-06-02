# Home template — gestione `index.<slug>.json`

Riferimento per Fase 6.x.7 di `/clone-competitor-store`. Spiega come gestire la creazione e l'assegnazione della homepage del nuovo store.

## Contesto

In Shopify la homepage è renderizzata dal template `templates/index.json` (default). Il theme può supportare più template di tipo `index` (es. `index.minimal.json`, `index.full.json`) e ogni store può scegliere quale usare via Theme Settings.

Quando la skill costruisce una "Home", la scelta è:
1. **Sovrascrivere `index.json`** → la home del competitor diventa la home dello store, irreversibile senza altro push
2. **Creare `index.<store.slug>-home.json`** → coesiste con `index.json` come alternativa, l'utente la attiva manualmente da Theme Settings

Per default la skill sceglie l'**opzione 2** (più sicuro). L'opzione 1 è disponibile solo su esplicita richiesta utente.

## Differenza con PDP/funnel

| Aspetto | PDP | Funnel | Home |
|---------|-----|--------|------|
| File template | `templates/product.<slug>.json` | `templates/page.<slug>.json` | `templates/index.<slug>.json` |
| Entity Shopify richiesta | Product | Page | (nessuna — la home è "implicita") |
| Assignment | Sidebar Product → Theme template | Sidebar Page → Theme template | Theme settings → Customize → Home → Theme template |
| Multipla istanza | Sì (un template per N prodotti) | Sì (un template per N pagine) | No (1 sola homepage attiva alla volta) |

## Procedura standard (opzione 2 — Recommended)

### 1. Creazione template

```liquid
{
  "sections": {
    "01-announcement-bar": { "type": "<store-slug>-home-01-announcement-bar", "settings": {} },
    "02-hero":             { "type": "<store-slug>-home-02-hero", "settings": {} },
    "03-featured-product": { "type": "<store-slug>-home-03-featured-product", "settings": {} },
    ...
  },
  "order": [
    "01-announcement-bar",
    "02-hero",
    "03-featured-product",
    ...
  ]
}
```

Salva in `templates/index.<store.slug>-home.json`.

Push via MCP `mcp__working_suite_shopify_admin__push_theme_asset` con `assets:[{ key, content }]` per chiave esatta (template + sezioni home), `store_id` + `theme_id = main_theme_id`. Retry 429/502/503/504 con backoff 10s/20s/40s (×3); 401/403 → STOP → riconnetti la Custom App. NESSUN comando CLI.

### 2. Istruzioni utente per assegnare

Mostra:

```
=== Assegna come homepage ===

Il template è creato come `index.<store.slug>-home`. Per attivarlo come 
homepage del tuo store:

1. Admin → Online Store → Themes → <theme name> → Customize.
2. In alto a sinistra, click sul picker delle pagine → seleziona "Home page".
3. Nella sidebar destra (top), trova il dropdown "Theme template".
4. Apri il dropdown e seleziona `<store.slug>-home`.
5. Save (in alto a destra).

Fatto. Ora visitando il dominio root vedrai questa home.

⚠️ Per tornare alla home precedente in qualsiasi momento, ripeti lo step 4 
selezionando un template diverso (es. `Default` per index.json originale).
```

Aspetta che l'utente confermi l'assegnazione + visualizzi la home live.

### 3. Verifica

Smoke check:
- Apri `https://<store.shopify_domain>/` (root)
- Verifica che renderizzi le sezioni della nuova home
- Smoke mobile + desktop

## Procedura alternativa (opzione 1 — sovrascrivere index.json)

Solo su esplicita richiesta utente. Step:

### 1. Backup di `index.json` esistente

```bash
cp <workdir>/templates/index.json <workdir>/templates/index.backup-pre-clone.json
```

(Backup locale, non pushato. Se serve la versione live attuale dell'`index.json`, recuperala con un tool Admin-API GET dedicato — NON con la CLI.)

### 2. Conferma esplicita (sovrascrive la home live)

Prima di procedere, conferma con `AskUserQuestion`: "Confermi di sovrascrivere la homepage live? Non è reversibile senza un altro push." Solo dopo il sì, procedi.

### 3. Sovrascrivi `index.json` con la nuova struttura

Stesso JSON dell'opzione 2 ma salvato in `templates/index.json` invece di `index.<store.slug>-home.json`.

### 4. Push via MCP (chiavi esatte: index + sezioni home)

```
mcp__working_suite_shopify_admin__push_theme_asset
  {
    store_id: <store.id>,
    theme_id: <store.theme_id>,            // = main_theme_id da check_connection
    assets: [
      { key: "templates/index.json",                       content: "<nuova struttura home>" },
      { key: "sections/<store-slug>-home-01-<role>.liquid", content: "<liquid>" },
      { key: "sections/<store-slug>-home-02-<role>.liquid", content: "<liquid>" }
      // ... una entry per ogni sezione home, chiave ESATTA (mai glob), max 50 per chiamata
    ]
  }
```

⚠️ Retry su 429/502/503/504 con backoff 10s → 20s → 40s (max 3). Su 401/403 → STOP, riconnetti la Custom App su /configurations/stores.

### 5. Conferma utente

Mostra:
```
✓ index.json sovrascritto sul tema live.
La homepage è cambiata. Per tornare indietro: ripristina dal backup locale 
`templates/index.backup-pre-clone.json` e ripusha.
```

## Casi speciali

### Tema con homepage dinamica multi-segment

Alcuni temi (Dawn 12+, Horizon) supportano "home segments" con A/B testing nativo o targeting per audience. Se vedi nelle settings del tema riferimenti a `audience_id` o `home_variant`, segnala all'utente: "Il tema supporta multi-home; l'assegnazione del nuovo template come default va fatta nella tab specifica delle homepage variants — verifica in Customize."

### Tema senza supporto multi-template index

Temi vecchi/custom potrebbero non avere il dropdown "Theme template" sulla home. In quel caso, opzione 2 non è praticabile e l'unica strada è sovrascrivere `index.json`. Spiega all'utente la situazione e chiedi conferma esplicita.

### Conflitto con app "Page builder" (PageFly, GemPages, Shogun)

Se lo store usa una app page builder per la homepage attuale, il loro template potrebbe interferire con l'assegnazione del nostro nuovo template. Avvisa l'utente: "Vedo che la home attuale è gestita da [app]. Quando assegnerai il nostro template, l'app potrebbe perdere il controllo sulla home. Verifica che vada bene per te."

## Sezioni tipiche di una home competitor

Per riferimento, ecco le sezioni comuni che troverai in una home di competitor US:

| # | Tipo sezione | Frequenza | Note |
|---|--------------|-----------|------|
| 01 | Announcement bar | 90% | Sale + offer count |
| 02 | Hero (image/video full-width) | 99% | Headline + CTA principale |
| 03 | USP strip (3-4 box brevi) | 80% | Spedizione, garanzia, ingredienti |
| 04 | Featured product / hero product | 70% | Card prodotto top con CTA |
| 05 | Press features / "as seen in" | 50% | Logos riviste/podcast |
| 06 | Customer reviews wall | 75% | Testimonial scroll/grid |
| 07 | How it works (3 steps) | 60% | Visual timeline |
| 08 | Bundle / Best sellers | 65% | Multi-product showcase |
| 09 | Before/after | 50% | Foto comparison + claim |
| 10 | Founder story / brand voice | 30% | Storytelling section |
| 11 | FAQ | 40% | Accordion |
| 12 | Newsletter / email capture | 70% | Lead gen |
| 13 | Footer | 100% | Link policies + social + email |

Quando crei la home, mappa le sezioni del competitor su questa lista (alcune saranno presenti, altre no).

## Differenze rispetto a PDP / Funnel

- La home può essere **più "vetrina"** che funnel: meno claim verticali su un singolo prodotto, più presentazione del brand.
- Le CTA della home tipicamente puntano a **multiple destinazioni**: PDP principale, collection, bundle, blog.
- Newsletter signup è quasi sempre presente e va connesso a una mailing list (Klaviyo o Shopify Email) — la skill genera il form ma la connessione effettiva la fa l'utente fuori-skill.

## Checklist post-build home

- [ ] Tutte le sezioni renderizzano mobile + desktop
- [ ] CTA principale (hero) punta alla PDP creata in Fase 6 (PDP)
- [ ] Featured product/best sellers puntano a prodotti reali (non placeholder)
- [ ] Newsletter form non rotto (form action valida)
- [ ] Footer ha email + telefono + indirizzo da `store.*`
- [ ] Logo nel header è `store.logo_url` (non quello del competitor)
- [ ] Eventuale cookie banner non sovrappone il hero in modo grave (Mobile)
