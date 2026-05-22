# Platform detection — heuristics

Leggi all'inizio della **Phase 3** della skill `import-page`. Output finale: una di
queste etichette → `pagefly` | `gempages` | `shopify_native` | `funnelish` | `heyflow` | `unknown`.

## Strategia generale

Cerca segnali in 3 luoghi, **in quest'ordine**:
1. `<meta name="generator">` (più affidabile, raro)
2. Script `src=...` (molto affidabile per tool esterni)
3. CSS class patterns sul `<body>` o sui top-level container

Non chiedere all'operatore di confermare — la detezione è solo info utile per
adattare l'analisi nelle phase successive, non un blocker.

## Pattern per piattaforma

### PageFly

Segnali:
- Script src contiene `pagefly-build.com` / `cdn.pagefly.io` / `pagefly.app`
- CSS class prefisso `pf-` sui container (es. `pf-23-section`, `pf-row`)
- Inline marker: `<div data-pf-type="...">` o `data-pf-id="..."`
- Liquid markers nel sorgente theme: `{% section 'pagefly-...' %}`

Sotto-tipi:
- PageFly v3 (legacy): più markup inline, classi tipo `pf-1`-`pf-99`
- PageFly v4+ (attuale): markup più semantico, classi tipo `pf-section-X`

Per la skill v1 non distinguiamo le sub-versioni — `pagefly` come unico label.

### GemPages

Segnali:
- Script src contiene `gempages.app` / `cdn.gempages.com`
- CSS class prefisso `gp-` (es. `gp-component`, `gp-row`)
- `data-gp-component` su elementi top-level

### Shopify nativo (theme default come Dawn, Impulse, Sense, ecc.)

Segnali:
- Niente script tool-esterno
- CSS class pattern `shopify-section` sui container generati dal template JSON
- `<meta name="shopify-checkout-api-token" content="...">`
- `<meta name="shopify-digital-wallet" ...>`
- Liquid section ids visibili: `section.<id>`, `template-product`, `template-page`

### Funnelish

Segnali:
- Script src contiene `funnelish.com` / `cdn.funnelish.io`
- Hostname della source URL = `*.funnelish.io` o dominio custom puntato Funnelish
- CSS class pattern `fnl-*`
- `<meta name="funnel-id" content="...">`

### HeyFlow

Segnali:
- Script src contiene `heyflow.app` / `cdn.heyflow.com`
- Class pattern `hf-*`
- `<meta name="generator" content="HeyFlow">`

### Unknown / fallback

Se nessuno dei pattern matcha, usa `unknown`. La skill prosegue con l'analisi
semantica generica (Phase 4) che funziona su qualsiasi HTML ben formato.

## Implementation hint per Claude

Pseudocode (Claude esegue tutto questo come logica interna, NON serve scrivere
code reale — è descrittivo):

```
function detectPlatform(html):
  if html contains "<meta name=\"generator\" content=\"" → extract value
    if value matches "PageFly|GemPages|Funnelish|HeyFlow|Shopify" → return that
  
  if html contains "pagefly-build.com" or "cdn.pagefly.io" → return "pagefly"
  if html contains "gempages.app" or "cdn.gempages.com" → return "gempages"
  if html contains "funnelish.com" or "cdn.funnelish.io" → return "funnelish"
  if html contains "heyflow.app" or "cdn.heyflow.com" → return "heyflow"
  if html contains "shopify-checkout-api-token" → return "shopify_native"
  
  return "unknown"
```

## Perché serve sapere la piattaforma?

Influenza scelte nella Phase 4 (section detection):
- **PageFly / GemPages**: il DOM è già diviso in "rows" / "sections" col tool's markup
  → usa quel markup come guida primaria per il splitting
- **Shopify nativo**: i `<section class="shopify-section" id="shopify-section-...">`
  sono i confini esatti delle sezioni nel template originale
  → 1 shopify-section = 1 sezione nostra
- **Funnelish / HeyFlow**: niente markup tool-specifico, il splitting è 100% semantico
  → leggi il DOM come fosse "qualsiasi pagina web", usa headings + landmark roles
- **Unknown**: stesso treatment di Funnelish/HeyFlow (semantic only)
