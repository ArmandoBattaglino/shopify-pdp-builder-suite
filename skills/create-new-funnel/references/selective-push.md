# Push asset — pubblicare i file del funnel via MCP

Gen-2: tutti i push passano dal tool MCP `mcp__working_suite_shopify_admin__push_theme_asset`. NESSUN Shopify CLI, NESSUN `.env`, NESSUN token Theme Access, NESSUN prompt token in chat. Lo store è già connesso (Custom App, stesso Admin token dell'Analytics), decifrato lato-app dal tool MCP; la skill usa SOLO `store_id` dal contesto di sessione.

Regola: pubblica **solo i file che hai effettivamente toccato**, indicati per **chiave esatta**. Mai glob, mai wildcard, mai un push dell'intero tema.

## Prerequisiti (dal contesto di sessione)

- `store_id` — l'UUID `workspace_stores` salvato in Fase 1.
- `theme_id` — il `main_theme_id` restituito da `mcp__working_suite_shopify_admin__check_connection` in Fase 2 (tema pubblicato/live). Non esiste più alcun elenco temi.

Se i tool `mcp__working_suite_shopify_admin__*` non sono disponibili (mcp=no) → STOP: "MCP non configurato per questa sessione; riapri la chat builder." NON fare fallback a curl/CLI.

## Chiamata standard

```
mcp__working_suite_shopify_admin__push_theme_asset {
  store_id: <UUID workspace_stores>,
  theme_id: <main_theme_id>,
  assets: [
    { key: "templates/page.<nome>.json",          content: "<contenuto completo del file>" },
    { key: "sections/<prefisso><NN>-<role>.liquid", content: "<contenuto completo del file>" }
  ]
}
```

- `assets[]` — una entry per file. Una sola chiamata può contenere **fino a 50 asset** (batch). Se ne servono di più, spezza in più chiamate.
- `key` — la **chiave esatta** dell'asset nel tema. Forme valide:
  - `sections/<file>.liquid`
  - `templates/page.<nome>.json`
  - `layout/<nome>.liquid` (es. un layout chromeless custom)
  - `snippets/<nome>.liquid`
- `content` — il contenuto **integrale** del file generato dalla skill. La skill è la source of truth: niente pull, niente merge lato remoto.

## Pattern per tipo di cambiamento

### Hai modificato UNA sezione
```
assets: [ { key: "sections/<prefisso><NN>-<role>.liquid", content: "<...>" } ]
```

### Hai creato un template nuovo + sezioni nuove
```
assets: [
  { key: "templates/page.<nome>.json",          content: "<...>" },
  { key: "sections/<prefisso>01-hero.liquid",    content: "<...>" },
  { key: "sections/<prefisso>02-problem.liquid", content: "<...>" }
  // ... una entry per ogni sezione
]
```

### Hai creato un layout chromeless custom
```
assets: [
  { key: "layout/<nome>.liquid",         content: "<...>" },
  { key: "templates/page.<nome>.json",   content: "<...>" }
]
```

## Retry sugli errori transitori

`push_theme_asset` può fallire per `429` / `502` / `503` / `504` (transitori). Ri-invoca la **stessa** chiamata con gli **stessi** argomenti fino a 3 tentativi, backoff 10s → 20s → 40s.

## Quando NON fare retry (segnala subito)

- `401` / `403` → `check_connection` non più valida → STOP, manda l'utente a `/configurations/stores` a (ri)connettere la Custom App. (NON "rigenera token".)
- `Liquid syntax error` / errore di validazione asset → il file è rotto, NON è stato pubblicato. Correggi il `content` e ripeti.
- `Theme not found` / `404` sul `theme_id` → ri-esegui `check_connection` per ottenere il nuovo `main_theme_id`.
- `404` / chiave non valida su un asset → `key` errata. Fixa il path esatto.

## Cosa NON fare

- ❌ Shopify CLI (`@shopify/cli theme push/pull/list`) — eliminato in Gen-2.
- ❌ Glob/wildcard nelle `key` (es. `sections/*`) — sempre chiavi esatte.
- ❌ `.env`, `SHOPIFY_CLI_THEME_TOKEN`, token Theme Access — non esistono più.
- ❌ Fallback a `curl` verso l'Admin API — usa il tool MCP.

## Lettura di un asset esistente

Gen-2 non fa pull di una copia di lavoro. Se proprio serve leggere un singolo asset già presente sul tema, serve un tool Admin-API GET dedicato — NON usare la CLI.
