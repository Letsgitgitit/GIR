# ALLREPO lite gold — runsheet (BUILD 2026-11-16d)
Read with `/lite/ALLREPO_LITE_STATUS.md` (decisions, LITE CONFIG v1) and `/merge/ALLREPO_LITE_HANDOFF.md`.
Files: `allrepo_lite_config.py`, `allrepo_lite_run.py`, `allrepo_lite_selftest.py` (/lite) · `ALLREPO_LITE_M.m` ·
`ALLREPO_LITE_VALIDATION.dax` · `guard_files.py` (/merge, BUILD 2026-11-16a adds `ALLREPO_LITE`).

**What changes:** a second bake reads the full gold and writes `ALLREPO_lite\{hist,current}` (LITE CONFIG v1.1):
AmountBuy / AmountSell dropped, plus the six gold columns the live query already leaves out (FO_ID, MarkToMark_EUR,
MTMCash, MTMSecurity, SumAmountBuySell_EUR, TransactionId); the six EUR amounts rounded to EUR 1,000 (same names, still
EUR); every other column copied unchanged; hist files carry `_SourceFile` (same value the live M adds today). The model's
FACT_ALLREPO query then reads the lite folders: same columns as today minus AmountBuy / AmountSell.
**Lite root layout (as ALLREPO_gold_v2):** `hist\` and `current\` (read by the model, top level only), and the job's own
`_staging\` (files being written), `_reports\` (`allrepo_lite_last.txt` + dated copies), `_lite_manifest.json`,
`_lite_run.lock` (only while a run is going).
**What does not change:** the full gold, its job, its folders; no measure, no relationship, no page.
**Rollback at any point:** paste the saved live FACT_ALLREPO text back, refresh (step 0).

---

## Step 0 — before anything (5 min)
1. FACT_ALLREPO > Advanced Editor > copy all > save as `H:\Documents\Python\ALLREPO_Lite\FACT_ALLREPO_M_LIVE_20261116.m`.
   **This is the rollback.**
2. Copy the working `.pbix` to `..._pre_lite_20261116.pbix`.
3. Reference figures to compare after the swap (write them down or keep the DAX Studio results):
   - `/dax/PX_CHECK.dax` (fund case 6 → 11 Nov) — every row.
   - `ALLREPO_LITE_PREFLIGHT.dax` LP1 (rows and dates per year).
   - P6a at as-of 4 Nov 2026: K4 (17A / 17B / total), R1c weighted haircut, R face. P5D at one CPH you know, same date:
     collateral total and the top issuer.
   - The refresh time of FACT_ALLREPO alone (right-click the table > Refresh data, stopwatch). Last baseline: 25 min.

## Step 1 — install and selftest (office desktop, 5 min)
1. Create `H:\Documents\Python\ALLREPO_Lite\` and copy the three `.py` files into it (code only; the job's files go in
   the lite root).
2. Open `allrepo_lite_config.py` and check the three folder lines. `LITE_ROOT` must sit beside `ALLREPO_gold`, not
   inside it; the bake refuses to start otherwise.
3. Command prompt in that folder:
   `python allrepo_lite_selftest.py` → last line **`SELFTEST 57/57 PASS`**, exit 0.
   (In the sandbox, 18 bugs were put back one at a time; every one turned the selftest red.)

## Step 2 — L6 type test: Decimal vs Whole number for the six rounded columns (30 min)
Writes to scratch folders only. Use the latest complete month (example: 2026-10).
```
python allrepo_lite_run.py --only 2026-10 --round-store double --lite-root H:\Documents\Python\ALLREPO_Lite\_L6\double
python allrepo_lite_run.py --only 2026-10 --round-store int64  --lite-root H:\Documents\Python\ALLREPO_Lite\_L6\int64
```
Both runs end with `STATUS PASS`. Then, in a **new empty .pbix**, create three blank queries, one per file:
```
let Source = Parquet.Document(File.Contents("<path>")) in Source
```
- `L6_GOLD` = `G:\...\ALLREPO_gold_v2\current\FACT_ALLREPO_gold_2026-10.parquet`
- `L6_DOUBLE` = `H:\...\_L6\double\current\FACT_ALLREPO_lite_2026-10.parquet`
- `L6_INT64` = `H:\...\_L6\int64\current\FACT_ALLREPO_lite_2026-10.parquet`

Close & Apply. Time each table's refresh on its own (right-click > Refresh data). DAX Studio > View Metrics: for each
table, the table size, plus Total Size and Cardinality of RiskNet_EUR, RiskBrut_EUR, CashAmount_EUR, NotionalBond_EUR,
MTMSecurity_EUR, MTMCash_EUR. Note: L6_INT64 shows those six as Whole number; in the real model, measures that SUM them
are unaffected, but a column-level format string set on the Decimal column is lost (re-set it if any).
**Will decides:** `ROUND_STORE = "double"` or `"int64"` in `allrepo_lite_config.py`. Then delete `_L6`.

## Step 3 — first full lite run (production folders; expected minutes, not hours)
`python allrepo_lite_run.py`
- Progress line per file (`i/n side period (new) ...`), then build / verify seconds and MB per file.
- Every gold file gives one `Built:` line ending in **`verified`**. "Verified" means the lite file was re-read and
  compared with its gold file value by value: copied columns identical; rounded columns equal to the rounding of the
  gold value; row count and schema as expected. A file that fails is not published.
- `Rounding` block: per column and file, gold sum, lite sum, diff, max row error (≤ 500 by construction), largest
  daily diff, rows that became 0, nulls.
- `Lite folder: … MB on disk (gold … MB)` = the read-size reduction.
- Last line `STATUS PASS`, exit 0. Report: `ALLREPO_lite\_reports\allrepo_lite_last.txt` (+ dated copy).
- Run it again at once: `built 0`, nothing touched (that run takes seconds).

## Step 4 — L7 pilot in the working .pbix (no swap yet)
1. New blank query named exactly `FACT_ALLREPO_LITE_TEST`: paste `ALLREPO_LITE_M.m`, set `PilotYears = {"2025", "2026"}`.
   Load it. No relationships.
2. Time its refresh on its own (right-click > Refresh data). Rows ≈ 2025 + 2026 rows of LP1.
3. DAX Studio: run `ALLREPO_LITE_VALIDATION.dax`. Expected:
   - LV1 empty; LV1b lite rows = live rows on those dates.
   - LV2 every row `same`.
   - LV3: copied columns 0 / 0 dates; rounded columns `DatesOverBound` 0 (the bound is arithmetic: half a step per row).
     The `MaxAbsDayDiffEUR` figures are the actual size of the rounding in the model.
   - LV4: live vs lite on the P6a figures (Part F shown for reference); differences = rounding only.
   - LV5: only AmountBuy and AmountSell (live only), plus the six rounded columns if int64 was chosen.
     (FO_ID, MarkToMark_EUR, MTMCash, MTMSecurity, SumAmountBuySell_EUR, TransactionId are in neither table.)
4. View Metrics: FACT_ALLREPO_LITE_TEST size vs FACT_ALLREPO's 2025 + 2026 share. Extrapolate the full refresh time:
   pilot time × (all rows / pilot rows).
5. **Will decides go / no-go.** Then delete FACT_ALLREPO_LITE_TEST.

## Step 5 — swap
1. FACT_ALLREPO > Advanced Editor > replace all with `ALLREPO_LITE_M.m` (`PilotYears = {}`). Done.
2. Refresh all (evening routine). Stopwatch FACT_ALLREPO.
3. After checks:
   - LP1 rows and dates per year = step 0 (exact).
   - PX_CHECK.dax: every row = step 0 within the rounding (EUR m figures to the first decimal; counts exact).
   - P6a K4 / R1c / face, P5D figures = step 0 within the rounding.
   - View Metrics: FACT_ALLREPO size and the six columns' sizes (vs 1,884 MB for the two MtM legs today).
   - Record refresh time and sizes in `ALLREPO_LITE_STATUS.md`.
4. FACT_ALLREPO_HIST stub: unchanged (still reads one gold hist footer, 0 rows).

## Step 6 — guard tests on the lite query (V6, 20 min)
1. Copy `guard_files.py` (BUILD 2026-11-16a) next to the others; `TEST_ROOT` stays `H:\Documents\Python\_guardtest`.
2. Duplicate FACT_ALLREPO → `FACT_ALLREPO_LITE_GT`; change only the `LiteRoot` line to
   `H:\Documents\Python\_guardtest\ALLREPO_LITE`. Keep it loaded for the test (the guards must evaluate); delete it after.
3. For each case: `python guard_files.py ALLREPO_LITE <case>` then refresh FACT_ALLREPO_LITE_GT only:
   | case | expected |
   |---|---|
   | ok | refresh succeeds |
   | g1 | fails `G1 no FACT_ALLREPO_lite_*.parquet in …\current` |
   | g2 | fails `G2 year in both hist and current: <year>` |
   | g3 | fails `G3 FACT_ALLREPO_lite_…: missing critical {Haircut}` |
   | g4 | fails `G4 column type differs between files: RiskNet_EUR` |
4. `python guard_files.py ALLREPO_LITE clean`, delete FACT_ALLREPO_LITE_GT.

## Step 7 — schedule (Task Scheduler)
- Action: `python allrepo_lite_run.py`, **Start in** = `H:\Documents\Python\ALLREPO_Lite`.
- Weekdays **10:30** (the ALLREPO gold job starts 10:15 and takes ~2 min), "run as soon as possible after a missed
  start" ticked. If the gold job has not finished, nothing is lost: the lite job builds what changed next time.
- Daily expectation: `Built:` only for the current month (`gold changed`), `STATUS PASS`, exit 0. Exit 2 = act (the
  failing file's previous lite file is kept; the report says which and why).
- V8 (incremental) is covered by the selftest (T5, T6, T6b) and by the first scheduled day's report: one `Built:` line.

## Later
- **Re-include a column / undo a rounding:** edit `DROP_COLUMNS` / `ROUND_COLUMNS`, run `allrepo_lite_run.py` (every file
  rebuilds by itself: `config changed`), refresh. No M edit (the M takes the lite files' own column set).
- **Back to full granularity:** paste the step-0 text into FACT_ALLREPO, refresh. The full gold stays complete.
- **Annual roll:** when the gold job moves 2026 from current to hist, the next lite run builds `hist\…_2026` and then
  removes `current\…_2026-MM` (written first, removed after: a refresh in between fails G2, it never misses data).
- Exit codes: 0 PASS · 1 WARN (e.g. stale lock removed, gold file changed during the build → rebuilt next run) · 2 FAIL.
