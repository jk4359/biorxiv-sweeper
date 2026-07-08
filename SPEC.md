# bioRxiv Sweeper — Product Spec

Single source of truth for this app. Adapted from the arXiv Sweeper spec; where they differ, this
document governs bioRxiv Sweeper.

## Purpose

Sweep the most recent bioRxiv and medRxiv preprints for user-selected subject areas and summarize them
at three levels, so the reader can see where the frontier of the life sciences is currently probing —
without reading every paper. Delivered as a single self-contained HTML file that runs in any modern
browser, entirely client-side.

## Core behavior

1. The user selects one or more **subject areas**. A subject area is a bioRxiv or medRxiv category
   (e.g. `neuroscience`, `infectious diseases`). The **server** (bioRxiv / medRxiv) is the top-level
   group; the category is the subcategory.
2. The tool queries the **bioRxiv API** (`https://api.biorxiv.org/details/{server}/{start}/{end}/{cursor}/json?category={slug}`)
   for all papers in the selected category over a date window (default 1 week, configurable in weeks),
   newest-first via cursor pagination (30/page). Results are deduped by DOI, keeping the highest
   version. A per-category safety ceiling (500) bounds latency and cost; hitting it is recorded as
   truncation. The window is bounded by explicit start/end dates, so there is no rollout prompt.
3. Summaries are generated with an **LLM backend**, selectable per run: **Claude** (default,
   `claude-opus-4-8`) or **Gemini** (`gemini-3.1-pro-preview`). The key is entered in the UI, stored in
   `localStorage`, and sent directly from the browser to the provider (Anthropic requires the
   `anthropic-dangerous-direct-browser-access` header).
4. Output is a report rendered as live DOM in the page, printable to **PDF** via the browser's own
   print (US Letter, 0.75in margins, compact 10pt body via `@media print`).
5. The report's **cover** is titled with the generation date + time and carries run metadata: subject
   areas, paper count, any truncation, the provider/model, total tokens (input/output), and an
   **estimated** API cost. It also shows a **papers-by-topic** visualization: a bar list of papers per
   category (grouped and colored by server) beside a **nested sunburst** (inner ring = server, outer
   ring = category). Counts are per topic.

## The three summary levels

| Level | Content | Linking |
|---|---|---|
| 1. Total | One synthesis across all selected areas: themes, convergences, where the frontier is moving | Hyperlinked APA in-text citations |
| 2. By-topic | Two layers: a per-server synthesis (generated only when 2+ categories in that server are selected) above a per-category synthesis of what that subfield is probing now | Hyperlinked APA in-text citations |
| 3. By-paper | The paper's abstract, pulled verbatim from the API (no LLM call). If the API returned no abstract, a short stand-in is shown | Hyperlink to the preprint page |

Report order: Total → By-topic → By-paper.

## Citations

- Style: **APA 7** for preprints, e.g.:
  `Author, A., & Author, B. (2026). Title of paper. bioRxiv. https://www.biorxiv.org/content/<doi>v<n>`
- Authors arrive already surname-first (`Surname, Initials`), so no surname heuristic is needed.
- Every citation links to the preprint's `https://www.biorxiv.org/content/<doi>v<version>` page
  (medRxiv: `www.medrxiv.org`). In-text citations at levels 1–2 use APA author–year form, hyperlinked
  to the in-page by-paper entry.

## Constraints

- Single self-contained `index.html`: vanilla JS, no build step, no backend. Prompts, report template,
  CSS, and taxonomy are inlined.
- Respect the bioRxiv API: polite concurrency (≤3 in-flight) across categories; page sequentially
  within a category.
- Never fabricate citations: every cited DOI must come from the fetched set; regenerate up to 2× then
  strip any lingering unknown DOI before rendering.
- LLM keys live only in `localStorage` and are sent directly to the provider; provide a clear-key
  control.

## Non-goals (v1)

- No PDF full-text analysis (abstracts + metadata only).
- No free-text keyword search (the API is date + category based).
- No persistence beyond the on-screen report and the saved key; no scheduling.
