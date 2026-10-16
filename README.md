# HiveScale by HiveMinds

**Make every bid count.** HiveScale is an internal bid optimization calculator for Cost Cap and tROAS campaigns. It calculates the billing ratio, applies the HiveMinds optimization rules, and explains the recommended bid step by step.

It runs entirely in the browser. There is no backend, no login, no API keys and no external services. Nothing you enter is stored permanently or sent anywhere, and fonts are bundled with the app.

## The rules

| | Cost Cap | tROAS |
|---|---|---|
| Billing ratio | Actual budget consumption ÷ (Current bid × Conversions) | Minimum ROAS bid ÷ Achieved ROAS |
| Below 0.80 | Decrease bid by 20% (× 0.80) | Increase bid by 20% (× 1.20) |
| 0.80 to 1.20, inclusive | Maintain | Maintain |
| Above 1.20 | Increase bid by 20% (× 1.20) | Decrease bid by 20% (× 0.80) |

Recommendations should only be applied once the best-practice checks are complete. Until then, results are marked **Provisional**.

All rules live in one file: `src/lib/rules.ts`. The best-practice checklist lives in `src/lib/checklist.ts` and can be edited as HiveMinds guidance changes. Keep item `id`s stable when only the wording changes.

## Requirements

- Node.js 18.18 or later (tested on Node 22)
- npm 9 or later

## Setup and local development

```bash
git clone <your-repo-url> hivescale
cd hivescale
npm ci            # installs the exact versions in package-lock.json
npm run dev       # starts the dev server at http://localhost:5173
```

## Testing

```bash
npm test          # runs all tests once (Vitest + Testing Library, jsdom)
npm run test:watch
npm run typecheck
```

The suite covers the calculation engine (every threshold boundary for both modes, zero conversions, zero achieved ROAS, zero spend, invalid and negative inputs, ROAS percentage and multiplier conversion, bid limits, simulations, floating-point boundary cases), formatting, exports, and the interface (live recalculation, mode switching, readiness states, reset, copy, CSV export, print, session history and keyboard navigation).

## Production build

```bash
npm run build     # type-checks, then builds static files into dist/
npm run preview   # serves dist/ at http://localhost:4173
```

## Deploying to Vercel

### Option A: from GitHub (recommended)

1. Create an empty repository on GitHub.
2. Push the project:
   ```bash
   git init
   git add .
   git commit -m "HiveScale v1"
   git branch -M main
   git remote add origin https://github.com/<you>/hivescale.git
   git push -u origin main
   ```
3. In Vercel, choose **Add New → Project** and import the repository.
4. Vercel detects Vite from `vercel.json`. The settings are:
   - Install command: `npm ci`
   - Build command: `npm run build`
   - Output directory: `dist`
5. Click **Deploy**. Every push to `main` redeploys automatically.

### Option B: Vercel CLI

```bash
npm i -g vercel
vercel          # first deploy, creates a preview URL
vercel --prod   # promotes to production
```

No environment variables are needed.

Note: a Vercel deployment is public by default. If this tool should stay internal, enable Vercel's deployment protection or host it behind your company's access controls.

## Project structure

```
src/
  lib/                     Pure logic, no React
    rules.ts               Thresholds, actions, 20% adjustment (single source of truth)
    engine.ts              Billing ratio, classification, recommendation, limits, simulations
    validation.ts          Input parsing, ROAS format conversion
    evaluate.ts            Combines validation + engine into one evaluation and readiness state
    explain.ts             Explanation text and the step-by-step breakdown
    exporters.ts           Copy summary, CSV, file download, clipboard
    format.ts              Currency, ratio, ROAS and percentage formatting
    fields.ts              Form state shapes and defaults
    checklist.ts           Best-practice checklist (edit here)
    __tests__/             Engine, validation, formatting and export tests
  state/
    calculatorState.ts     Reducer: independent Cost Cap and tROAS state, session history
    useStickyIfFits.ts     Keeps the results column in view only when it fits
  components/              UI components
  test/                    Interface tests
  App.tsx                  Page composition and actions
  styles.css               Design tokens, layout, responsive and print styles
```

Every surface (panel, breakdown, copy, CSV and history) reads from the same evaluation object, so they cannot disagree.

## Behaviour notes

- **Precision.** Calculations use full precision. Rounding only happens for display. Billing ratios show 2 decimals; if rounding would land exactly on 0.80 or 1.20 while the true value is outside the Maintain range (for example 0.7951), extra decimals are shown so the display never contradicts the action.
- **Floating-point boundaries.** Some exact boundary inputs produce values like 0.7999999999999998 in binary arithmetic. A tolerance of 1e-9 is used only when comparing against the thresholds, so these are treated as exactly 0.80.
- **ROAS formats.** Values are stored internally as multipliers. In percentage format, 250 means 250% (2.5x), never 250x. Switching format converts the values you've entered.
- **Bid limits** are optional and unset by default. They only change the final proposed bid, never the billing ratio or the action, and the rule-based bid stays visible.
- **Simulations** are mathematical previews of bid values only. They don't predict conversions, spend, CPA or ROAS and never replace the official 20% recommendation.
- **Session history** is held in memory and is lost when the tab is closed or reloaded.
- **CSV export** includes a UTF-8 byte-order mark so Excel shows ₹ and € correctly, and neutralises spreadsheet formulas in campaign names.

## Known limitations

- Responsive layout is verified in a real browser, not by the automated tests (jsdom has no layout engine).
- Session history is intentionally not persisted. Add storage only if HiveMinds decides it's appropriate for campaign data.
- Currency choice is per Cost Cap calculation; there is no currency conversion.
