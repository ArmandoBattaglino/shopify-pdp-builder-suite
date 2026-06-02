# Push selettivo — pubblicare solo i file modificati via MCP

Regola: **non fare mai un push completo**, e mai usare la CLI Shopify. Il push avviene esclusivamente con il tool MCP `mcp__working_suite_shopify_admin__push_theme_asset`, una `key` esatta per asset (mai glob). Includi SOLO i file che hai effettivamente toccato (template + sezioni del nuovo prefisso), mai asset di altri prodotti/template.

## Chiamata standard

```
mcp__working_suite_shopify_admin__push_theme_asset({
  store_id: <UUID workspace_stores dal contesto di sessione>,
  theme_id: <main_theme_id risolto da check_connection in Fase 2>,
  assets: [
    { key: "sections/<prefisso>-NN-...liquid", content: "<contenuto completo del file>" },
    { key: "templates/product.<nome>.json",    content: "<contenuto completo del file>" }
  ]
})
```

## Parametri

- `store_id`: l'UUID dello store in `workspace_stores`, preso dal contesto di sessione (salvato in Fase 1). È la chiave con cui il tool decifra lato-app l'Admin token (stesso token dell'Analytics, Custom App). NESSUN `.env`, NESSUN Theme Access token.
- `theme_id`: sempre il `main_theme_id` (tema pubblicato/live) restituito da `check_connection` in Fase 2. Non esiste più alcun elenco temi: il target è sempre il main/live.
- `assets[]`: array di `{ key, content }`.
  - `key`: il path **esatto** dell'asset (`sections/<file>.liquid` oppure `templates/product.<nome>.json`). Mai un glob, mai una wildcard.
  - `content`: il contenuto **completo** del file (non un diff).
  - Una `key` per asset; puoi batchare più asset in un'unica chiamata, **max 50** per chiamata.

## Pattern per tipo di cambiamento

### Hai modificato UNA sezione
```
assets: [ { key: "sections/<prefisso>-<suffix>.liquid", content: "..." } ]
```

### Hai creato un template nuovo + sezioni nuove
```
assets: [
  { key: "templates/product.<nome>.json", content: "..." },
  { key: "sections/<prefisso>-01.liquid", content: "..." },
  { key: "sections/<prefisso>-02.liquid", content: "..." }
  // ... una entry per ogni sezione duplicata (max 50 per chiamata)
]
```

### Hai modificato uno snippet condiviso
```
assets: [ { key: "snippets/<nome-snippet>.liquid", content: "..." } ]
```

**Attenzione**: se modifichi uno snippet usato anche da altri template (es. berberina vs crema-occhiaie), il cambio impatta anche quelli. Duplica lo snippet con un nuovo nome se vuoi isolare.

## Cosa NON fare

- ❌ Glob/wildcard nelle `key` (es. `sections/*`). Solo path esatti, uno per asset.
- ❌ Mandare un diff invece del contenuto completo del file.
- ❌ Pushare asset che non hai toccato (rischio di sovrascrivere lavoro altrui o template di altri prodotti).
- ❌ Fallback a `curl`/CLI Shopify se il tool MCP fallisce o è assente. Se i tool `mcp__working_suite_shopify_admin__*` non sono disponibili → STOP e riapri la chat builder.

## Retry su errori transitori

`push_theme_asset` può fallire per cause transitorie: `429`, `502`, `503`, `504`, errori di rete/timeout. In questi casi ritenta con backoff **10s → 20s → 40s** (fino a 3 tentativi), ripetendo la stessa chiamata.

Errori NON transitori (stop immediato, niente retry):
- `401` / `403` → connessione Admin non valida (`check_connection` fallita). Rimanda l'utente a /configurations/stores a (ri)connettere la Custom App.
- `Liquid syntax error` / `Invalid Liquid` → la modifica è rotta. Correggi il file e ripeti.
- `404` su `theme_id` → il `main_theme_id` non è più valido. Ri-esegui `check_connection`.
- `key` non valida → fixa la chiave esatta dell'asset.

## Verifica dopo il push

Il tool ritorna esito positivo per ogni asset accettato. Se un asset fallisce la validazione Liquid, quel file NON viene applicato: correggi il Liquid e rilancia la chiamata per quell'asset. Conferma sempre visivamente sull'URL live della PDP.
