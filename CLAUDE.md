# Hearth — project notes

Hearth is a family budget web app, a single file (`index.html`, vanilla JS/HTML/CSS) served by
GitHub Pages at **hearth.stillwater.place** (CNAME → `bcgordon.github.io`). Data lives in
Supabase (project **Home Finance**, table `households`, one JSON `data` record per household).
**Look (v4.4.0): Typewriter**, after myStereo's favourite view: cream paper cards on a pale mint page with Hermes-mint keycaps, band and tab bar (light, mint
sampled from a Hermes typewriter: #a0d8c8 / #b8f0df / #88c0b0), blue ink,
Deep Teal (dark), American Typewriter (on every household device; Courier Prime fallback), a seafoam
band with a chrome strip on each header, cream keycaps for main actions, flat thin-ruled surfaces.
Only Light / Dark / Match phone (`hft_mode`). Icon: teal fireplace with mint flame and piggy bank
(BC's artwork, full-bleed): `apple-touch-icon.png` (180), `hf-icon.png` (512), favicon inline; `hearth-mark.png` = the fireplace cut out (transparent), used
for the Dashboard tab and the sidebar logo. The skin is the `TYPEWRITER SKIN` block at the end of
the `<style>`; colours are the existing tokens (`--coral` = red ink, `--sky` = seafoam ink, etc.).

This repo is **public**. Never commit household data, codes, balances, income figures or keys
beyond the publishable Supabase key already in `index.html`.

## Version
Live: **v4.6.7** (Nov 26, 2026: Calm with crackle, breathing flame and piggy, Toss a worry, timer). Bump patch/minor from here unless BC names a version.

## Standing rules
1. **Propose or mock before building.** Build only after BC says **"build it and ship it"**.
   Mock with Hearth's real data where possible (a working prototype on the side branch is fine).
2. **Ship = merge to `main`.** Pages builds from `main`. Work on the session's side branch, then
   fast-forward `main` on "build it and ship it". No PRs needed for Hearth unless BC asks.
3. **Bump the version on every app push, in two places that must match:** `APP_VERSION` in
   `index.html` and `version.txt`. The app compares them to show "Update ready · Reload".
4. **Syntax-check before shipping:** extract the inline `<script>` and run `new Function(code)`.
5. **After pushing, confirm the deploy** (if the Pages *deploy* step fails or sits queued, push a
   small new commit rather than waiting on a re-run; seen on Nov 25, 2026 during a GitHub Actions/Pages incident; a stuck re-run never left the queue): a Pages run for the new commit finished successfully,
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
- `bills`: **every** scheduled bill, paycheck and savings transfer (v4.6.3: `DEFAULT_BILLS` in code is empty;
  the old defaults were copied into `record.bills` with their `hc_*` ids, which also sit in `deletedBills`).
  Fields: `freq` (monthly/weekly/biweekly/quarterly/annual/once), `type` (in/out/save),
  `paidBy`, `day`, `startDate` (weekly/biweekly), `startMonth/Year`, `endMonth/Year`,
  `isFundTransfer`, `eom`, `cat` (report category, optional; guessed from the name if absent).
- Per-occurrence stores (easy to confuse): `billOverrides` (`bill_override_{id}_{y}_{m}_{d}` →
  amount or `'skip'`), `ccOverrides` (`cc_override_{y}_{m}`, a complete payment amount),
  `fundCovers` (`fund_cover_{id}_{y}_{m}_{d}`), `dismissedEvents` (`{y}-{m}-{d}-{Name_With_Underscores}`,
  month 0-indexed: the "skip" on upcoming events). `billPending`.
- `config`: `ccCycleStart/End`, `budget`, `names`, `everydayTarget` (code default is a generic 3000),
  `may6Payment` (the one-off May 2026 card payment). `fundTargets[id]` = each sinking fund's target. `budgets`: quarterly category overrides.
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
- **iPad sidebar** (`renderSidebarGlance`): + Add expense keycap, plain nav, and At a glance cards
  (this week, this month, planned to keep, checking now + lowest ahead, card cycle + next payment,
  statements vs Hearth, watch list), all from the same functions as the dashboard; each opens its screen.
  iPad is for detailed review; iPhone for entries and quick checks.
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
- Next (v4.4.x) to do: Settings day pickers (statement close / autopay) are display-only — wire them
  carefully (cycle code); tidy older screens (Balance, Funds, Bills).
- v4.6.0 (from Monarch / Rocket Money): **merchant memory** on Add (`merchantMemory`, the latest entry
  per note fills category/type/pay); **Subscriptions** screen (`renderSubscriptions`: scheduled bills in
  the two subscription categories + logged merchants charged 2+ times lately; price flags; hides
  merchants marked Done on the watch list); **calendar** on Balance (`renderCalendar`, from
  `buildProjection`); **Goals** on Budget (`record.goals`, `{id,name,kind:save|payoff,target,start,current,monthly}`;
  the Emergency goal always reads the fund balance). Still queued: safe to spend.
- v4.6.1: merchant suggestions are grouped by vendor (`merchantGroups`): case/punctuation/"&" ignored,
  a name that starts another absorbs it, one-letter typos join, and `record.merchantAliases`
  (`{variant: name}`) joins the rest by hand. Entries' notes are never rewritten.
- v4.6.2: **hide the iPad sidebar** (« in the logo row, » tab to reopen; `body.sb-hidden`, `hft_sb_hidden`).
  **Demo mode** (`enterDemo`/`exitDemo`, Settings › Show someone, and "See a demo" on the lock screen):
  `buildDemoRecord()` makes a made-up household from today's date; while `demoMode` is on, `cloudGet`/`cloudSet`
  only touch `demoRecord` in memory and `callSheetsSync` does nothing. Any new code that writes anywhere
  must respect `demoMode` too. Add tiles and the dashboard's budget labels now count `monthBudgets`.
- v4.6.3: no household figures in the code. Bills, paychecks, fund targets, the everyday target and the
  May 2026 payment live in the record; new code must read them from there, never hard-code amounts.
  (Older commits in git history still contain the old figures.)
- v4.6.4: **Version history** (Settings › Account; `VERSION_HISTORY` array, newest first, plain words, no
  figures). **Add a line for every release.** **Text size** (Settings; `hft_text` std/lg/xl → CSS `--tz`
  1/1.1/1.2 applied as `zoom` on `.screen` and the sidebar's children; default Larger). The iPad/desktop
  sidebar is `position: fixed` (sticky never worked: `#app` has `overflow-x:hidden`); `#ipad-content`
  gets `margin-left: 300px` while the sidebar shows. `Runaway Anchor` is a bill category.
- v4.6.5: the Balance calendar shows card bills (💳 next to the date; "On the card" list when a day is
  tapped) for reference only. `buildProjection` still excludes `paidBy: 'cc'` bills on purpose: they're
  inside the card payment on the 19th, so they never change a day's checking total or balance bar.
- v4.6.6: **Calm** (`openCalm`/`closeCalm`, `#calm`): Deep Teal full screen, `calm-hearth.png` (the 640px
  fireplace with the flame dimmed) + `calm-flame.png` (the flame cut out, animated: flicker + sway),
  a glow over the logs, a halo and the words "breathe in / breathe out" on a 10 s cycle (5 in, 5 out),
  "I'm calm now" lower right. Opened from the dashboard (under the quote) and the iPad sidebar.
- v4.6.7 Calm: the **flame** wrapper breathes (smaller on the 5 s breathe in, bigger on the breathe out);
  the **piggy** (`calm-piggy.png`, cut out of `calm-hearth.png`) swells and glows on the breathe in, with
  breath clouds (`.calm-puff`) drawn into its snout on the in and puffed out on the out. **Sound** (Web Audio,
  no files): crackle + brown-noise hum, 🔈 toggle (`hft_calm_sound`); no breath chimes (BC: visual cue only);
  three soft chimes only when a **timer** (∞/1/3/5 min) ends. **Toss a worry**: typed text flies into the fire
  and burns with a flare, whoosh and sparks; nothing saved. Screen **wake lock** while Calm is open. No
  floating embers except the sparks while a worry burns.
- The Camry loan is serviced by Royal Credit Union; due on the 3rd from Dec 2026 (the old bill ends Oct 2026).
- Cycle dates and CC autopay schedule are partly hard-coded in `getScheduledEvents`.
- Some older views (P&L, quarter tracker) count slightly differently from the dashboard
  (e.g. checking entries); align them when touched.
- Data hygiene: occasional hand-logged entries duplicating scheduled card bills.
- Watch list (`getWatchItems`) groups by the first word of the note; `record.watchDone[key]` = ts
  hides a merchant until it charges again.
