# Hearth — project notes

Hearth is a family budget web app, a single file (`index.html`, vanilla JS/HTML/CSS) served by
GitHub Pages at **hearth.stillwater.place** (CNAME → `bcgordon.github.io`). Data lives in
Supabase (project **Home Finance**, table `households`, one JSON `data` record per household).
Parchment and dark themes, Zilla Slab, 16px base text.

This repo is **public**. Never commit household data, codes, balances, income figures or keys
beyond the publishable Supabase key already in `index.html`.

## Version
Live: **v4.3.1** (Nov 25, 2026: Card cycle reconciling, running total, Start fresh). Bump patch/minor from here unless BC names a version.

## Standing rules
1. **Propose or mock before building.** Build only after BC says **"build it and ship it"**.
   Mock with Hearth's real data where possible (a working prototype on the side branch is fine).
2. **Ship = merge to `main`.** Pages builds from `main`. Work on the session's side branch, then
   fast-forward `main` on "build it and ship it". No PRs needed for Hearth unless BC asks.
3. **Bump the version on every app push, in two places that must match:** `APP_VERSION` in
   `index.html` and `version.txt`. The app compares them to show "Update ready · Reload".
4. **Syntax-check before shipping:** extract the inline `<script>` and run `new Function(code)`.
5. **After pushing, confirm the deploy:** a Pages run for the new commit finished successfully,
   and the live `version.txt` and `APP_VERSION` show the new version. If no build fires,
   a trivial re-commit forces one. If HTTPS breaks (GitHub cert), re-save the custom domain in
   Settings → Pages.
6. **Castle of cards.** Projection, CC-cycle and register code is fragile: change it phase by phase,
   compare the forecast before/after on a copy of the real data, and report what moved.
7. **Supabase:** Claude can read it directly (connector). Reads are fine; **writes to the record
   need BC's OK first**, and should be narrow (one entry/key), guarded, and verified after.
8. Test changes in Playwright against a copy of the record with Supabase writes blocked
   (`page.route` aborting non-GET requests).

## Data model (`record`)
- `entries` keyed by quarter (`2026_Q4`): `{id, ts, amt, cat, type, payMethod, note, pending?,
  fundCoverage?, fundBillId?}`. `payMethod`: `cc`, `checking`, `income`, `credit` (negative amt).
- `bills`: custom scheduled bills; defaults are `DEFAULT_BILLS` in code minus `deletedBills`.
  Fields: `freq` (monthly/weekly/biweekly/quarterly/annual/once), `type` (in/out/save),
  `paidBy`, `day`, `startDate` (weekly/biweekly), `startMonth/Year`, `endMonth/Year`,
  `isFundTransfer`, `eom`, `cat` (report category, optional; guessed from the name if absent).
- Per-occurrence stores (easy to confuse): `billOverrides` (`bill_override_{id}_{y}_{m}_{d}` →
  amount or `'skip'`), `ccOverrides` (`cc_override_{y}_{m}`, a complete payment amount),
  `fundCovers` (`fund_cover_{id}_{y}_{m}_{d}`), `dismissedEvents` (`{y}-{m}-{d}-{Name_With_Underscores}`,
  month 0-indexed: the "skip" on upcoming events). `billPending`.
- `config`: `ccCycleStart/End`, `budget`, `names`. `budgets`: quarterly category overrides.
- `monthBudgets['YYYY-MM'][category]` = a one-month monthly amount (e.g. December gifts). The
  month's everyday target rises by the difference (`monthTarget`), and so does the CC forecast
  estimate for the cycle that closes in that month. `watchDone[key]` = ts (watch list).

## How the numbers are defined (keep these consistent)
- **Everyday spending** = card entries (cc + credits), not fund-covered, not checking/income.
  Target **$3,500/month** by default (`PACE_TARGET`, editable on the Budget tab, stored in
  `config.everydayTarget`, applied by `applyEverydayTarget()`); the forecast uses the same figure.
- **Weekly** (`renderWeekly`): Monday–Sunday, **catch-up** targets: what's left of each month's
  target/category budget spread over its remaining days. Weekly limits for every budgeted
  category except Groceries, Gas & transit and subscriptions.
- **Budget colours** (`_paceStatus`): red only when more than $1 past the budget; met or within $1
  is amber ("At budget"); otherwise green, or amber when ahead of pace.
- **Dashboard pacing** and **Budget tab**: calendar month. Card bills are separate from the $3,500.
  Category budgets sum below $3,500 on purpose; the gap is the **cushion** row.
- **Budget tab**: monthly summary (calendar month), **Your monthly plan** (income − everyday target −
  card bills − checking bills − savings = planned to keep; this month and the next two), and
  **Everyday budgets** (budget, so far, 3-month average, flag when the average is >20% over;
  edit sheet: From now on → `budgets` (quarterly ×3) or This month only → `monthBudgets`).
- **Card cycle**: Capital One's cycle is the **25th–24th**, paid on the 19th. `CC_CYCLE_DAY = 25`;
  `refreshCCCycle()` rolls it automatically (no manual dates in Settings any more). The Balance tab
  card check, the History → Card cycle view and the forecast all use it.
- **History**: Register (calendar month, newest first, filters, pending toggles, Mark all posted,
  search across all months), Card cycle (each cycle vs the statement amount in `ccOverrides`),
  Reports (P&L folded in as tap-to-expand Income/Expenses). Quarter/Month/CC Cycle/P&L views retired.
- **Reconciling** (`renderCycles`, `cycleGap`): running total = Σ(statement − Hearth total) over closed
  cycles, minus explained amounts (manual `ccRecon[key].explained`, plus card payments made from
  checking, auto via bill category "Card payment"), minus `ccRecon.baseline.amt` ("Start fresh").
  `ccRecon['{payYear}_{payMonth}'] = {note, explained, ts}`. Reconciling never changes amounts or the forecast.
- **CC payment forecast** (`forecastCCPayment`): override wins → closed cycle uses actual →
  open/future cycle uses max(actual so far + bills still to post, $3,500 + scheduled card bills).
- **Reports** (`reportFigures`): income = paychecks + logged income (not fund draws);
  expenses = card spending incl. fund-covered + card bills + checking bills; **not** expenses:
  sinking-fund transfers, card payments, **allowances** (BC's call: transfers). Taxes stay in.
- Bills respect `billOverrides` skips/amounts and `dismissedEvents` everywhere new code sums them
  (see `_paceBillsByDay`). Monthly `eom` bills fall on the last day of the month.

## Known open items
- Cycle dates and CC autopay schedule are partly hard-coded in `getScheduledEvents`.
- Some older views (P&L, quarter tracker) count slightly differently from the dashboard
  (e.g. checking entries); align them when touched.
- Data hygiene: occasional hand-logged entries duplicating scheduled card bills.
- Watch list (`getWatchItems`) groups by the first word of the note; `record.watchDone[key]` = ts
  hides a merchant until it charges again.
