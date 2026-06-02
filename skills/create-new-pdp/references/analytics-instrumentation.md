# Analytics instrumentation — Working Suite SDK

**Tutte le sezioni che generi devono essere strumentate** in modo che gli eventi arrivino nella dashboard Analytics di Working Suite (`/app/<ws>/analytics`). Senza strumentazione la PDP/Funnel funziona ma non vediamo cosa fanno i visitatori → nessun dato per ottimizzare.

Il SDK lato theme è già installato sui tuoi store registrati in `analytics_sources` (workspace → store domain → ingest endpoint). Espone `window.wsa.track(name, payload)`. Non devi installare nulla, devi solo chiamarlo dai punti giusti del DOM.

## Event catalog (M1)

Solo questi nomi sono accettati dall'endpoint ingest (`/api/webhooks/ingest`). Eventi con nomi non in lista vengono droppati silenziosamente.

| Event             | Auto SDK | Da emettere dalla skill | Quando                                   |
| ----------------- | -------- | ----------------------- | ---------------------------------------- |
| `page_view`       | ✅       | no                      | Pageload — gestito dal SDK               |
| `session_entry`   | ✅       | no                      | Prima visita della sessione              |
| `scroll_depth`    | ✅       | no                      | 25/50/75/100% — gestito dal SDK          |
| `add_to_cart`     | ✅       | no                      | Submit `<form action="/cart/add">`       |
| `begin_checkout`  | ✅       | no                      | `/checkout` page — auto                  |
| `purchase`        | ✅       | no                      | Order confirmation — auto                |
| **`cta_click`**   | ❌       | **sì**                  | Click su ogni CTA principale             |
| **`step_view`**   | ❌       | **sì** (solo quiz)      | Ogni step quiz quando entra in viewport  |
| **`step_answer`** | ❌       | **sì** (solo quiz)      | Click su un pulsante risposta            |
| **`step_back`**   | ❌       | **sì** (solo quiz)      | Click su "Indietro"                      |
| **`step_complete`** | ❌     | **sì** (solo quiz)      | Submit finale del quiz                   |
| `funnel_abandon`  | ✅       | no                      | Tab close / visibilitychange — auto      |

## Pattern di emissione

### CTA click (PDP + Funnel)

Tutti i bottoni primari/secondari che portano a un'azione (Aggiungi al carrello, Acquista ora, Scopri di più, link a PDP da funnel, ecc.) devono essere strumentati.

```liquid
<a
  class="<prefix>-cta"
  href="{{ section.settings.cta_url }}"
  data-wsa-cta-id="{{ section.settings.cta_id | default: section.id }}"
  data-wsa-cta-position="{{ section.settings.cta_position | default: 'hero' }}"
>
  {{ section.settings.cta_label }}
</a>

<script>
  (function() {
    if (!window.wsa) return;
    document.querySelectorAll('[data-wsa-cta-id]').forEach(function(el) {
      el.addEventListener('click', function(e) {
        var t = e.currentTarget;
        window.wsa.track('cta_click', {
          props: {
            cta_id: t.dataset.wsaCtaId,
            cta_label: (t.textContent || '').trim().slice(0, 200),
            cta_position: t.dataset.wsaCtaPosition || null,
            template: {{ template.suffix | default: template.name | json }}
          }
        });
      });
    });
  })();
</script>
```

Regole:
- `cta_id` è SEMPRE uno slug stabile (`hero-primary`, `closing-buy`, `quiz-result-buy`), non il testo del bottone. Stable across copy changes.
- `cta_position` è una macro-zona (`hero`, `mid`, `closing`, `sticky-bottom`). Aiuta a vedere quale posizione converte meglio.
- Aggiungi `data-wsa-cta-id` ad **ogni** anchor/button che porta da PDP a checkout o da funnel a PDP/checkout.

### Quiz — `step_view` (quando uno step appare)

Una sezione per ogni step del quiz. Inserisci questo script DENTRO la sezione, in modo che si esegua solo quando quella sezione è renderizzata.

```liquid
<div
  class="<prefix>-step"
  data-wsa-step="{{ section.settings.step_number }}"
  data-wsa-step-id="{{ section.settings.step_id }}"
  data-wsa-step-label="{{ section.settings.heading | escape }}"
>
  <h2>{{ section.settings.heading }}</h2>
  <!-- options sotto -->
</div>

<script>
  (function() {
    if (!window.wsa) return;
    var container = document.querySelector(
      '[data-wsa-step="{{ section.settings.step_number }}"]'
    );
    if (!container) return;

    // Emette step_view quando lo step diventa visibile (≥50% in viewport).
    // IntersectionObserver evita di sparare lo step_view di TUTTI gli step
    // quando la pagina ha più step su una stessa scroll (quiz "single page")
    // o quando l'utente torna indietro.
    var seen = false;
    var obs = new IntersectionObserver(function(entries) {
      entries.forEach(function(entry) {
        if (entry.isIntersecting && entry.intersectionRatio >= 0.5 && !seen) {
          seen = true;
          window.wsa.track('step_view', {
            step: container.dataset.wsaStep,
            step_label: container.dataset.wsaStepLabel || null,
            props: { step_id: container.dataset.wsaStepId }
          });
        }
      });
    }, { threshold: [0.5] });
    obs.observe(container);
  })();
</script>
```

Per quiz "single page" dove tutti gli step sono sezioni separate sulla stessa pagina, l'IntersectionObserver fa esattamente quello che vuoi: spara `step_view` solo quando lo step entra effettivamente in vista. Per quiz "step-by-step" (uno step per pageload tipo SPA), va bene anche un trigger su DOMContentLoaded.

### Quiz — `step_answer` (click su una risposta)

Ogni opzione di risposta è un `<button>` con `data-wsa-answer` + `data-wsa-answer-label`. Lo script intercetta il click e emette l'evento PRIMA della navigazione successiva.

```liquid
<div
  class="<prefix>-step"
  data-wsa-step="{{ section.settings.step_number }}"
  data-wsa-step-id="{{ section.settings.step_id }}"
  data-wsa-step-label="{{ section.settings.heading | escape }}"
>
  {% for block in section.blocks %}
    {% if block.type == 'answer' %}
      <button
        type="button"
        class="<prefix>-answer"
        data-wsa-answer="{{ block.settings.value }}"
        data-wsa-answer-label="{{ block.settings.label | escape }}"
        data-next-step="{{ block.settings.next_step | default: '' }}"
      >
        {{ block.settings.label }}
      </button>
    {% endif %}
  {% endfor %}
</div>

<script>
  (function() {
    if (!window.wsa) return;
    var stepContainer = document.querySelector(
      '[data-wsa-step="{{ section.settings.step_number }}"]'
    );
    if (!stepContainer) return;

    stepContainer.querySelectorAll('[data-wsa-answer]').forEach(function(btn) {
      btn.addEventListener('click', function(e) {
        var t = e.currentTarget;
        window.wsa.track('step_answer', {
          step: stepContainer.dataset.wsaStep,
          step_label: stepContainer.dataset.wsaStepLabel || null,
          answer: t.dataset.wsaAnswer,
          answer_label: (t.dataset.wsaAnswerLabel || t.textContent || '').trim().slice(0, 200) || null,
          props: { step_id: stepContainer.dataset.wsaStepId }
        });
      });
    });
  })();
</script>
```

Regole importanti:
- `step` è SEMPRE il numero posizionale dello step (`"1"`, `"2"`, …`"11"`). È la chiave di ordering nei grafici.
- `step_id` è uno slug stabile della domanda (`hydration`, `goal`, `budget`). Resta uguale anche se rinumeri gli step (es. aggiungi uno step nel mezzo). È la chiave di join.
- `answer` è uno slug stabile della risposta (`yes`, `no`, `4_6`, `over_60`). NON il testo umano. Stable across copy changes.
- `answer_label` è il testo umano visibile sul bottone — è quello che vediamo in dashboard.
- Limita label a 200 caratteri (`.slice(0, 200)`) per evitare payload accidentalmente lunghi.
- Se per qualche ragione un label è vuoto, manda `null` esplicito (non stringa vuota).

### Quiz — `step_back` (click su Indietro)

```liquid
<button
  type="button"
  class="<prefix>-back"
  data-wsa-step="{{ section.settings.step_number }}"
  data-wsa-step-id="{{ section.settings.step_id }}"
>
  ← Indietro
</button>

<script>
  (function() {
    if (!window.wsa) return;
    document.querySelectorAll('.<prefix>-back').forEach(function(btn) {
      btn.addEventListener('click', function(e) {
        var t = e.currentTarget;
        window.wsa.track('step_back', {
          step: t.dataset.wsaStep,
          props: { step_id: t.dataset.wsaStepId }
        });
      });
    });
  })();
</script>
```

### Quiz — `step_complete` (submit finale)

Sull'ultimo step del quiz (es. inserimento email + click su "Mostra risultato"):

```liquid
<form class="<prefix>-final" action="..." method="post">
  <input type="email" name="email" required>
  <button type="submit">Mostra il mio risultato</button>
</form>

<script>
  (function() {
    if (!window.wsa) return;
    var form = document.querySelector('.<prefix>-final');
    if (!form) return;

    form.addEventListener('submit', function() {
      window.wsa.track('step_complete', {
        step: '{{ section.settings.step_number }}',
        props: {
          step_id: '{{ section.settings.step_id }}',
          total_steps: {{ section.settings.total_steps | default: 0 }}
        }
      });
    });
  })();
</script>
```

## Riepilogo schema fields obbligatori

Quando crei sezioni quiz, lo schema `{% schema %}` di OGNI step deve avere:

```json
{
  "settings": [
    { "type": "number", "id": "step_number",  "label": "Numero step (ordering)", "default": 1 },
    { "type": "text",   "id": "step_id",      "label": "Slug step (chiave join)", "default": "step-1" },
    { "type": "text",   "id": "heading",      "label": "Testo domanda", "default": "..." },
    { ... altri settings layout ... }
  ],
  "blocks": [
    {
      "type": "answer",
      "name": "Risposta",
      "settings": [
        { "type": "text", "id": "value", "label": "Slug risposta (chiave aggregazione)", "default": "..." },
        { "type": "text", "id": "label", "label": "Testo bottone risposta", "default": "..." },
        { "type": "text", "id": "next_step", "label": "Step successivo (opzionale)", "default": "" }
      ]
    }
  ]
}
```

## Cosa NON serve fare

- **Page view / scroll depth / session entry**: gestiti automaticamente dal SDK quando il theme è installato. Non emetterli a mano.
- **Add to cart / purchase**: il SDK ascolta `form[action*="/cart/add"]` submit events + i webhook order di Shopify. Niente codice custom necessario.
- **Funnel abandon**: il SDK ascolta `visibilitychange` + `pagehide` + chiusura tab. Auto-fired.

## Convenzioni di naming

- `cta_id` come slug stabile, mai testo del bottone
- `step_id` come slug stabile (descrittivo della domanda)
- `answer` come slug stabile (descrittivo della scelta)
- `step` come numero posizionale (1, 2, 3, …)
- Tutti gli `id`/`label` UI sono `image_picker`/`text`/`richtext` in schema, MAI hardcoded nel markup. La skill già scrive markup editabile dal theme editor: passa i `step_id`/`answer.value` come settings esposti, così l'operatore può modificarli senza editare Liquid.

## Test post-deploy

Quando finisci una sezione strumentata:

1. Dopo il push via `push_theme_asset`, apri l'URL live della PDP (`https://<store.shopify_domain>/products/<product.slug>`)
2. Apri DevTools → Network → filtra per `ingest`
3. Naviga la sezione (scroll su step, click su risposta, click su CTA)
4. Verifica che la request a `/api/webhooks/ingest` contenga gli event corretti col payload pieno
5. Apri `/app/<ws>/analytics` in dashboard Working Suite e verifica che gli eventi siano arrivati (≤ 1 minuto di lag)
