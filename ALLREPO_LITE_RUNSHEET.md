# ALLREPO lite gold — runsheet (BUILD 2026-11-16f, LITE CONFIG v1.3)
Read with `/lite/ALLREPO_LITE_STATUS.md` (decisions, figures) and `/merge/ALLREPO_LITE_HANDOFF.md`.
Files (/lite): `allrepo_lite_config.py`, `allrepo_lite_run.py`, `allrepo_lite_selftest.py`, `ALLREPO_LITE_M.m`,
`ALLREPO_LITE_IDS_EXPORT.dax`, `ALLREPO_LITE_VALIDATION.dax`; `/merge/guard_files.py` (BUILD 2026-11-16a).

**What changes.** A second bake reads the full gold and writes `ALLREPO_lite`:
- Columns: AmountBuy, AmountSell, FO_ID, MarkToMark_EUR, MTMCash, MTMSecurity, SumAmountBuySell_EUR, TransactionId dropped;
  eight EUR amounts rounded to EUR 1,000 (RiskNet, RiskBrut, CashAmount, NotionalBond, MTMSecurity, MTMCash, AmountBuy,
  AmountSell `_EUR`; same names, still EUR, Decimal); every other column unchanged; hist files carry `_SourceFile`.
- Rows, split in two parts that together hold every gold row once:
  - **core** (`hist\`, `current\`, loaded by default): repo (07L) on every date — except CCPs (Category FCG21) and
    intra-group CPYs, whose repo rows are kept on **month-ends + the last 90 days**; securities lending (07K) on month-ends
    + the last 90 days.
  - **extension** (`ext_hist\`, `ext_current\`): everything else (07F, 07Y, 07M, 10C; CCP / intra-group repo and 07K on
    other dates). Loaded only with `ALLREPO_SCOPE = full` — into the same FACT_ALLREPO table.
- No measure reads a non-07L row (16 Nov SX3), and the latest dates are complete for every client: the pages' current
  figures do not change. What core leaves out: CCP / intra-group repo history and 07K history on ordinary days older
  than 90 days (PX windows starting on such a day for those clients; daily charts of them).
**What does not change:** the full gold, its job, its folders; no measure, no relationship.
**Rollback at any point:** paste the saved live FACT_ALLREPO text back, refresh (step 0). Full data in the new layout:
`ALLREPO_SCOPE = full`, refresh.

---

## Step 0 — before anything (5 min)
1. FACT_ALLREPO > Advanced Editor > copy all > save as `H:\Documents\Python\ALLREPO_Lite\FACT_ALLREPO_M_LIVE_20261116.m`.
   **This is the rollback.**
2. Copy the working `.pbix` to `..._pre_lite_20261116.pbix`.
3. Reference figures (keep the DAX Studio results): `/dax/PX_CHECK.dax` (all rows) · `ALLREPO_LITE_PREFLIGHT.dax`
   LP1 · P6a at 4 Nov 2026: K4 (17A / 17B / total), R1c, R face · P5D at one CPH you know, latest date: collateral total
   and top issuer · FACT_ALLREPO refresh time alone (last baseline 25 min).

## Step 1 — install and selftest (office desktop, 5 min)
1. `H:\Documents\Python\ALLREPO_Lite\`: copy the three `.py` files (code only; the job's files go in the lite root).
2. `allrepo_lite_config.py`: check the three folder lines (`LITE_ROOT` beside `ALLREPO_gold`, never inside it).
3. `python allrepo_lite_selftest.py` → last line **`SELFTEST 93/93 PASS`**, exit 0. (Sandbox: 29 bugs put back one at a
   time — rounding, incremental, guards, split rule, window, id list — every one turned the selftest red.)

## Step 2 — L6 type test — DONE 16 Nov
Load times equal (gold 48 s · double 52 s · int64 49 s on one month); rounded columns reach VALUE encoding as Double →
`ROUND_STORE = "double"` (no type change in the model).

## Step 3a — export the intra-group ids (2 min)
DAX Studio on the working .pbix: `ALLREPO_LITE_IDS_EXPORT.dax`, Output > File > CSV, save as
`H:\Documents\Python\ALLREPO_Lite\allrepo_lite_intragroup_ids.csv` (beside the config). Ids only. Not through Excel.
Re-export whenever IsIntraGroup changes; the next bake rebuilds every file by itself.

## Step 3 — first full lite run (production folders; expected minutes)
`python allrepo_lite_run.py`
- `prescan i/n …` lines (dates of every gold file; cached in the manifest afterwards), then one block per gold file.
- Report header: `split:` (the rule) · `window:` latest date, first date of the 90 days, number of month-ends ·
  `intra-group ids: listed N (k padded) · found in the gold M`.
- Every gold file gives one `Built:` line: `rows = core … + ext …`, ending **`verified`** = both parts re-read and
  compared with the gold value by value (copied columns identical, rounded = rounding of gold, core + ext = gold rows,
  each row in the part the rule gives it). A file that fails is not published.
- `Rows in the lite folders: core … + extension … (core x %) · per year core / ext` = **the row counts to expect in the
  model** (core per year for steps 4 and 5).
- `Lite folder: … core … MB + extension … MB on disk (gold … MB)`. Last line `STATUS PASS`, exit 0.
- Report: `ALLREPO_lite\_reports\allrepo_lite_last.txt` (+ dated copy). Run again at once: `built 0`.

## Step 4 — pilot in the working .pbix (no swap yet)
1. Create the parameter **ALLREPO_SCOPE** (M block P), current value **full**.
2. New blank query `FACT_ALLREPO_LITE_TEST`: M block A, `PilotYears = {"2025", "2026"}`. Load. No relationships.
3. **Full scope = live:** run `ALLREPO_LITE_VALIDATION.dax`. Expected: LV1 empty · LV1b lite rows = live rows · LV2 all
   `same` · LV3 `DatesOverBound` 0 everywhere · LV4 live vs lite = rounding only · LV5 only AmountBuy, AmountSell.
4. **Core scope:** set ALLREPO_SCOPE = core, refresh FACT_ALLREPO_LITE_TEST alone (stopwatch). LV1b `Lite rows` = the bake
   report's core rows for 2025 + 2026. (LV1 now lists the dates that lose CCP / intra-group / 07K rows — expected.)
5. Extrapolate: core refresh time × (all core rows / pilot core rows). **Will decides go / no-go.** Delete the test query.

## Step 5 — swap
1. ALLREPO_SCOPE = **core**. FACT_ALLREPO > Advanced Editor > replace all with M block A (`PilotYears = {}`).
2. New blank query `ALLREPO_SCOPE_INFO` = M block B (loaded, no relationship). On the pages that read ALLREPO (P5D, PX,
   P6a): a small card / text box with `ALLREPO_SCOPE_INFO[Note]` (9 pt, muted), so the loaded scope is always stated.
3. Refresh all (evening routine), stopwatch FACT_ALLREPO.
4. After checks: LP1 rows per year = the bake report's **core** rows per year · PX_CHECK.dax = step 0 within the rounding
   (fund case: repo, not CCP / intra-group) · P6a K4 / R1c / face = step 0 within the rounding · P5D latest-date figures =
   step 0 within the rounding · View Metrics: FACT_ALLREPO size. Record refresh time and sizes in the status file.
5. FACT_ALLREPO_HIST stub: unchanged.

## Step 6 — guard tests on the lite query (20 min)
1. `guard_files.py` (BUILD 2026-11-16a), `TEST_ROOT` = `H:\Documents\Python\_guardtest`.
2. Duplicate FACT_ALLREPO → `FACT_ALLREPO_LITE_GT`; change only `LiteRoot` to `H:\Documents\Python\_guardtest\ALLREPO_LITE`
   (scope core). Keep it loaded for the test; delete it after.
3. `python guard_files.py ALLREPO_LITE <case>`, refresh the GT table only: ok succeeds · g1 `G1 no … in …\current` ·
   g2 `G2 year in both hist and current` · g3 `G3 current\…: missing critical {Haircut}` · g4 `G4 column type differs
   between files: RiskNet_EUR`. Then `python guard_files.py ALLREPO_LITE clean`.

## Step 7 — schedule (Task Scheduler)
- `python allrepo_lite_run.py`, **Start in** `H:\Documents\Python\ALLREPO_Lite`, weekdays **10:30** (gold job 10:15,
  ~2 min), "run as soon as possible after a missed start" ticked.
- Daily expectation: `Built:` for the current month (`gold changed`) and usually one older month (`window moved`: its
  dates leave the 90 days); in January–March the window reaches the hist year, so the hist file is rebuilt too (minutes).
  `STATUS PASS`, exit 0. Exit 2 = act (the failing file's previous lite files are kept; the report says why).

## Later
- **Full data for an analysis:** ALLREPO_SCOPE = full, refresh (~22 min); back to core afterwards. Same table, same pages.
- **Change the split** (another category, another code, another window): edit the SPLIT section of the config, run the
  bake (every file rebuilds by itself: `config changed`), refresh. No M edit.
- **Re-include a column / undo a rounding:** edit `DROP_COLUMNS` / `ROUND_COLUMNS`, run, refresh. No M edit.
- **Back to the full gold layout:** paste the step-0 text into FACT_ALLREPO, refresh.
- **Annual roll:** when the gold job moves 2026 to hist, the next lite run builds the hist 2026 pair, then removes the
  current 2026 pairs (written first, removed after: a refresh in between fails G2 loudly, never misses data).
- Exit codes: 0 PASS · 1 WARN · 2 FAIL.
