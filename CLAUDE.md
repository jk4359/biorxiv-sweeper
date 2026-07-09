# CLAUDE.md

Guidance for working in this repo. `SPEC.md` is the governing product spec — read it for
*what* the app does and why. This file covers *how* the code is built, run, and changed.

## What this is

A single self-contained web app that sweeps recent bioRxiv/medRxiv preprints for selected
subject areas and produces a three-level LLM digest (total → by-topic → by-paper) with
hyperlinked APA citations. Runs entirely client-side in the browser.

## Architecture

- **Everything lives in `index.html`.** Vanilla JS, no build step, no backend, no dependencies.
  Prompts, CSS, the subject-area taxonomy, and the report template are all inlined.
- Three files total: `index.html` (the app), `SPEC.md` (governing spec), `README.md`.
- Two external network calls, both directly from the browser:
  1. **bioRxiv API** (`https://api.biorxiv.org/details/{server}/{start}/{end}/{cursor}/json?category=…`)
     for preprint metadata + abstracts.
  2. The **LLM provider** — Anthropic (`api.anthropic.com`) or Google (`generativelanguage.googleapis.com`).
     API keys are entered in the UI, stored only in `localStorage`, and sent straight to the provider
     (Anthropic requires the `anthropic-dangerous-direct-browser-access` header).

## Run it

Open `index.html` in a browser. No server needed.

- The window/profile must be able to reach `api.biorxiv.org`. If a run reports "No papers were
  fetched," the usual cause is the fetch being blocked at the network layer (extension, proxy, or a
  browser profile whose network can't reach bioRxiv) — not an app bug.

## Test

The script exposes pure helpers via `module.exports` (bottom of `index.html`, guarded by
`typeof module !== "undefined"`) so they can be `require`d and unit-tested under Node without a DOM.
Only DOM-free logic is exported (parsing, citation formatting, `fetchCategory`, report building, etc.).
Quick syntax check of the inline script:

```sh
sed -n '/^<script>/,/^<\/script>/p' index.html | sed '1d;$d' > /tmp/app.js && node --check /tmp/app.js
```

## Key conventions & gotchas

- **`MODELS` / `EFFORTS` config block is the single source of truth** (top of the `<script>`, "Models
  & config" section). Each `MODELS` entry carries `id`, `label`, `price` (per-1M in/out for the
  cover-page cost *estimate*), and `maxOut` (the model's hard output ceiling). `PRICING` and the model
  dropdown are derived from it — add a model there and the UI + cost math pick it up automatically.
  Claude models listed must support adaptive thinking + the `effort` parameter (the request shape the
  app sends).
- **`[[DOI]]` citation-token contract.** LLM syntheses cite papers with `[[<doi>]]` tokens. Every cited
  DOI must be in the fetched set; `generateWithCitations` regenerates up to `CITATION_RETRIES` times,
  then strips any lingering unknown token before rendering. Never loosen this — fabricated citations are
  a hard non-goal.
- **Synthesis calls stream and request the model's full `maxOut`.** `_claude` parses the Anthropic SSE
  stream (accumulate `text_delta`, capture `stop_reason` + usage); `_gemini` is a plain JSON call.
  Rationale: `max_tokens` is a ceiling, not a target, so requesting the model max costs nothing extra
  and guarantees adaptive thinking can't starve the answer. There is **no app-imposed token cap**.
- **No fetch cap.** `fetchCategory` paginates until the window is exhausted (`cursor >= total` or an
  empty page). The only stop beyond that is a runaway backstop that trips only if the API serves pages
  past its own reported `total`; when it does, the topic is flagged "may be incomplete" on the cover.
- **Failures are loud, never silent.** A synthesis that can't complete (hit the model's hard ceiling,
  returned no text, or the request 400'd — e.g. input exceeding the context window) throws and renders
  as a visible per-topic gap, rather than a blank section. The by-paper section (abstracts, verbatim, no
  LLM) is always complete regardless.
- **Effort is the cost/quality dial** (Claude only; Gemini has no effort control, so its UI control is
  hidden). Higher effort → deeper syntheses + more thinking tokens.

## When changing things

- Preserve the single-file, no-build, no-dependency constraint.
- Match the surrounding code's style (compact vanilla JS, inlined styles in the report template).
- If a change touches the fetch/synthesis path, verify against a real run — this app's failure modes
  (blocked fetch, token starvation, context-window overflow) only show up end-to-end, not in unit tests.
- Where this file and `SPEC.md` disagree on product behavior, `SPEC.md` governs.
