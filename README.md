# bioRxiv Sweeper

Sweep the most recent **bioRxiv** and **medRxiv** preprints for the subject areas you care about
and get back a single, readable digest — so you can see where the frontier of the life sciences is
currently probing without reading every paper.

It's a **single self-contained HTML file**. No install, no build step, no server: double-click
`index.html` (or open it in Chrome) and go. Everything runs in your browser.

This is the browser-native sibling of [arXiv Sweeper](../arXiv%20Sweeper/) — same three-level digest
and report design, but it reads bioRxiv/medRxiv instead of arXiv and runs as a web app instead of a
Python CLI.

## What a sweep produces

A three-level summary rendered live in the page (and printable to PDF), with hyperlinked APA-7 citations:

| Level | What it answers |
|---|---|
| **Total** | One synthesis across all your subject areas — themes, convergences, where things are moving. |
| **By-topic** | Two layers: a per-server synthesis (bioRxiv / medRxiv) above per-category syntheses (e.g. Neuroscience), generated when 2+ categories in a server are selected. |
| **By-paper** | Each paper's abstract, pulled in verbatim from the API (no LLM call). |

The **cover page** shows run metadata (subject areas, paper count, provider/model, total tokens, an
estimated API cost, any truncation) and a **papers-by-topic** chart: a colored bar list beside a nested
sunburst (inner ring = server, outer ring = category).

## Use it

1. Open **`index.html`** in a browser (Chrome/Edge recommended).
2. Pick **Claude** or **Gemini** and paste your API key (stored only in your browser; see below).
3. Pick subject areas (check a server heading to select the whole set), choose how many weeks back,
   and click **Run sweep**.
4. Read the report. Use **Print / Save as PDF** for a compact US-Letter PDF, or the light/dark toggle.

### API keys

You supply your own LLM key. It is saved in your browser's `localStorage` (per provider) and sent
**directly** from your browser to the provider — nothing passes through any server of ours.

- **Claude** — get a key at [console.anthropic.com](https://console.anthropic.com/). Requests use
  `claude-opus-4-8` and the `anthropic-dangerous-direct-browser-access` header (required for direct
  browser calls).
- **Gemini** — get a key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey). Requests
  use `gemini-3.1-pro-preview`.

Use the **Clear** button to remove a stored key. Because the key lives in the browser, treat this as a
personal, local tool — don't host it on a shared machine with your key saved.

### If the LLM step is blocked when opening from `file://`

All three endpoints (bioRxiv API, Anthropic, Google) allow direct browser calls, so the double-click
`file://` path works. If your browser is configured to block cross-origin requests from `file://`,
serve the folder locally instead (still zero install):

```
python -m http.server 8000     # then open http://localhost:8000
```

## How it works

- **Fetch** — queries `api.biorxiv.org/details/{server}/{start}/{end}/{cursor}/json?category=…`, which
  filters by subject area over a date window (cursor-paginated, 30/page). Papers are deduped by DOI,
  keeping the highest version. A safety ceiling of 500 papers/category bounds cost; hitting it is noted
  as truncation on the cover.
- **Summarize** — builds the three levels bottom-up. Levels 1–2 cite papers with `[[doi]]` tokens; a
  citation-integrity check regenerates (up to 2×) and then strips any DOI that wasn't actually fetched,
  so a hallucinated citation can never reach the report.
- **Render** — the report (design ported from the arXiv Sweeper handoff) is built as live DOM; the
  **Print** button uses the browser's own Save-as-PDF with a compact `@media print` stylesheet.

## Limitations

- Abstracts + metadata only — no PDF full-text analysis.
- The bioRxiv API is date-window based, so "topics" are its subject categories; there is no free-text
  keyword search.
- The rare preprint returned without an abstract shows a short stand-in rather than an LLM-generated one.
- No persistence beyond the report on screen and your saved key.

## License

MIT.
