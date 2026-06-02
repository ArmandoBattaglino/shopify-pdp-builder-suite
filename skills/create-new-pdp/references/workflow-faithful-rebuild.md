# Faithful rebuild — la regola d'oro

Quando popoli una sezione duplicata con i contenuti di un nuovo prodotto, **non riscrivere il markup**. La sezione ha già layout, CSS, JS, media query e responsive logic testati e ottimizzati per mobile/desktop. L'utente li vuole preservati al 100%.

## Cosa fai SEMPRE

- **Sostituire testi**: titoli, sottotitoli, paragrafi, liste, label, badge, alt text, caption, bottone CTA.
- **Sostituire URL immagini/video**: solo il valore `src=`, `poster=`, `href=` che punta a CDN Shopify o a media esterni.
- **Sostituire URL interni**: `/products/<slug>` → nuovo slug del prodotto target.
- **Aggiornare il nome della sezione**: `schema.name` e `presets[0].name` nel blocco `{% schema %}`, per distinguerla nel theme editor.
- **Aggiornare array JS inline** (es. `benefits`, `reviewData`, `imgs`) **solo nei valori stringa**, non nella struttura.

## Cosa NON fai MAI

- ❌ Rimuovere o aggiungere classi CSS (`gp-`, `gps-`, `.promo-top-image`, ecc.).
- ❌ Toccare i selettori CSS nelle `<style>` block (hash GemPages tipo `gps-615148538410762923` devono restare identici per non rompere scoping).
- ❌ Modificare la struttura dei nodi HTML (niente `<div>` in più o in meno).
- ❌ Toccare le media query o le regole `@media`.
- ❌ Modificare JavaScript di interattività (click handler, Swiper init, accordion logic, show-more counter).
- ❌ Sostituire un elemento con un altro diverso (es. `<img>` → `<video>` senza che l'utente l'abbia chiesto esplicitamente).
- ❌ "Migliorare" il codice anche se vedi occasioni.

## Eccezioni consentite, solo su richiesta esplicita dell'utente

- Aggiustare `font-size` / `white-space` / `line-height` se l'utente dice che un testo va a capo male o è troppo grande/piccolo.
- Cambiare `<img>` in `<video>` se l'utente fornisce un URL video e chiede lo switch.
- Aumentare/diminuire dimensione di un'immagine-logo se l'utente dice "troppo grande / troppo piccola".

In questi casi: applica **la modifica minima** (1-4 righe CSS/HTML), lascia tutto il resto intatto.

## Tecniche pratiche

### Edit mirato
Per sostituzioni puntuali (1-5 testi), usa `Edit` con `old_string` abbastanza univoco da non collidere. Includi abbastanza contesto prima/dopo.

### Python heredoc multi-replace
Per sezioni con molti testi (FAQ a 7 domande, carousel a 10 recensioni, liste benefits), usa un blocco Python con `Bash`:

```bash
python3 <<'PY'
p = "/absolute/path/to/sections/<sezione>.liquid"
c = open(p).read()
repls = [
  ("TESTO VECCHIO 1", "TESTO NUOVO 1"),
  ("TESTO VECCHIO 2", "TESTO NUOVO 2"),
  # ...
]
for old, new in repls:
    n = c.count(old)
    assert n == 1, f"FAIL ({n}): {old[:60]}"
    c = c.replace(old, new)
open(p, "w").write(c)
print("OK", len(repls), "replacements")
PY
```

L'`assert n == 1` è critico: se una stringa non è univoca, il replace salterebbe silenziosamente il valore sbagliato. L'assert blocca subito.

### JS array sostituzione completa
Per array JS lunghi (`reviewData`, `benefits`), usa una regex non-greedy con `re.DOTALL` che cattura da `const reviewData = [` fino a `];`, e rimpiazza tutto il blocco. Esempio:

```python
import re
pattern = re.compile(r'const reviewData = \[.*?\];', re.DOTALL)
c = pattern.sub(new_array_literal, c, count=1)
```

## Check dopo ogni sostituzione

1. Conferma che il count dei replace sia quello atteso (tipicamente 1 per testo).
2. Apri il file modificato con `Read` e controlla che struttura/indentazione siano intatte.
3. Push selettivo via `push_theme_asset` (MCP) della sola `key` `sections/<file>.liquid` con il contenuto completo del file (vedi `selective-push.md`).
4. Chiedi all'utente conferma visiva sulla PDP live prima di passare alla sezione successiva.
