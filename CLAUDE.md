# CLAUDE.md — Budbeslut

## What this product is

Budbeslut is an AI-assisted due diligence tool for Swedish apartment buyers
(bostadsrätter in bostadsrättsföreningar, BRF). The user flow is
Hemnet listing → Budbeslut analysis → bidding decision.

Core philosophy: **Evidence > Assumptions > Recommendations.**
No hallucinations. No hidden scoring. No fake market values.

## Language rules

- Talk to the user (the founder) in **Swedish**.
- Code, identifiers, comments, commit messages and PR descriptions: **English**.
- All user-facing UI text and report text: **Swedish**.
- The extraction prompt sent to the model stays in **Swedish**, because the
  annual reports are Swedish.

## Tech stack

- Built in Lovable, synced two-way with this GitHub repo.
- React + TypeScript, TanStack Start (`createServerFn` server functions).
- Supabase via `supabaseAdmin` (tables include `analyses`, `brf_reports`;
  storage bucket `brf-reports`).
- PDF text extraction with `unpdf`, per page.
- Metric extraction via the Lovable AI gateway (`google/gemini-2.5-flash`).
- Google Geocoding is server-side only; the API key must never reach the
  frontend.

### Lovable sync

- Always work on a separate branch. The founder merges into the default
  branch, which then syncs to Lovable.
- Do not touch generated Supabase integration files unless required.
- Follow the repo's existing convention for database migrations.

## Key files

- `brf-report.ts` — metric keys and labels, `statedNumber`, BRF status
  (`assessBrf`), fee increase risk (`feeIncreaseRisk`), facts, assumptions and
  recommendations. Pure functions: this is where tests belong.
- The server file containing `extractBrfReport`, `linkBrfReport`,
  `getBrfReport` and `extract()` — PDF handling, the extraction prompt,
  quote verification, signing and storage.

## Non-negotiable rules

1. **Never estimate, derive or invent values** in the extraction step. Only
   values explicitly stated in the report. Missing values are shown as
   "Not found in uploaded report." (Swedish UI equivalent where applicable).
2. **Every extracted value keeps its source page and confidence.** The
   verbatim-quote check against the cited page in `extract()` must stay.
3. **No points without evidence.** A factor with no explicit evidence
   contributes nothing; it must never default to a scoring category.
4. **Forbidden outputs:** market value, fair value, max bid, premium/discount
   valuation, appreciation forecasts. Price analysis must always state
   "Area-based comparison only. Not a market valuation."
5. **Do not change scoring thresholds or point values** without explicit
   approval from the founder. Propose changes; do not apply them.
6. **Engine versioning.** Any change that can alter extraction output or
   scores must bump `ENGINE_VERSION`. Saved reports display their stored
   result and version and are never silently recomputed.

## Fee increase risk (Avgiftshöjningsrisk) — the founder wants to keep this

| Factor | Points |
| --- | --- |
| Debt per sqm 6 000–10 000 kr | +1 |
| Debt per sqm > 10 000 kr | +2 |
| Savings per sqm 130–250 kr | +1 |
| Savings per sqm < 130 kr | +2 |
| Interest sensitivity 6–10 % | +1 |
| Interest sensitivity > 10 % | +2 |
| Liquidity Moderate / Weak | +1 / +2 |
| Cash flow Neutral / Negative | +1 / +2 |
| Planned major renovations | +1 |
| > 50 % of loans maturing within 24 months | +1 |
| Leasehold (tomträtt) | +1 |
| Negative equity | +2 |

Levels: 0–2 Low, 3–5 Medium, 6+ High. No level if fewer than 3 factors.

### Debt-free associations (skuldfria föreningar) — intentional advantage

- Debt-free is detected from debt per sqm = 0 **or** explicit wording such as
  "Föreningen har inga lån".
- Interest sensitivity is shown as "Ej tillämpligt – föreningen har inga lån"
  with 0 points.
- −2 points, never below 0.
- With no hard flag (negative equity, leasehold): positive cash flow → Low.
  Otherwise High is capped at Medium, unless cash flow is Negative **and**
  liquidity is Weak.
- Any adjustment is explained in the report.

## Swedish annual report pitfalls (check these whenever touching extraction)

- **Short-term loans:** loans repricing within 12 months are booked as current
  liabilities. Total debt = long-term + short-term portion. Never compute debt
  or debt ratio from long-term debt alone.
- **Savings:** use the key figure "sparande per kvm". "Avsättning till yttre
  fond" is a book allocation, not savings.
- **Result:** depreciation is a non-cash cost. The relevant figure is result
  before depreciation, not "årets resultat".
- **Units:** kr vs tkr vs mkr can be mixed in one report.
- **Number formats:** decimal comma, space as thousands separator, minus sign
  `−` or `-`, and parentheses for negatives.
- **Years:** never let a year (e.g. "2024:") be parsed as a value.
- **Area basis:** per-sqm figures may use total area or bostadsrätt area.
- **K2 vs K3** affects results and comparability; cash flow statements are
  not mandatory under K2.
- Long reports are truncated before extraction; the audit report is often
  near the end.

## Testing

- Unit-test the pure functions in `brf-report.ts`.
- Tests must never call the AI gateway or Supabase. Secrets
  (`LOVABLE_API_KEY`, Supabase keys) are not available in the cloud
  environment.
- Planned: a golden set in `tests/golden/` — stored extraction output per
  annual report plus the human-annotated expected values and risk level.
  Every scoring change is run against it and the diff is reported.
- Reference case: **Brf Vale nr 25 (Sveavägen 117B)** is debt-free and was
  judged Low risk by the human reviewer.

## Status

- v1.1 changes (no-default scoring for cash flow and liquidity, debt-free
  detection and floor, savings definition in the prompt, year stripping in
  `statedNumber`, `temperature: 0`, stored fee risk + engine version, signing
  key check) are specified and may or may not be merged yet. Check the code
  before assuming.
- Next: golden set and regression tests, then loan tranche extraction for
  refinancing risk.

## Do not build

- Any valuation, bid suggestion or price forecast.
- A single composite 0–100 score.
- ML-trained risk scoring.
- Open-ended chat over the annual report without citations.
- Neighbourhood scores (crime, demographics, "area quality").
- Scraping Hemnet or broker sites.

## Working conventions

- Small, focused changes; one concern per branch.
- After each task, summarise in Swedish what changed, which files, and how
  the founder can verify it (e.g. with Brf Vale nr 25).
- When unsure about a Swedish accounting term or a product rule, ask rather
  than guess.
