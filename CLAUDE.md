# Bengaluru Multi-Modal Urban Mobility Analytics — CA3 Activity 2 (Big Data Analytics)

Handoff doc for picking this project up in a new session with minimal
re-discovery. Read this fully before touching anything.

## What this is

A college Big Data Analytics assignment (MCA AIML, III Sem, course
25MCAAIE3042). It started as "Smart Urban Taxi Analytics" on the Kaggle
NYC Taxi Trip Duration dataset (a friend's Colab notebook, ported into a
local pipeline + Flask dashboard). On **2026-10-23** the user asked for it
to be refactored into a **Bengaluru multi-modal (Bike / Auto / Cab)
mobility project** with dynamic fares, traffic-aware ETAs, surge, and a
role-based dashboard (User / Driver / Admin). That refactor is done and
verified.

Handbook PDF (`MCA_AIML_2026-27(Odd)_III_Sem_Handbook.pdf`) has the
official rubric at pages 72-73 — 5 criteria × 3 marks = 15 marks:
Dataset Selection & Preprocessing, MapReduce Implementation, Data
Analytics & Visualization, ML Application, Report/Insights/Viva.
Syllabus wording explicitly allows **"simulate or implement"** MapReduce.

## The NYC version is archived, not deleted

`_archive_nyc_taxi/` holds the complete previous version: `pipeline/`,
`backend/`, `models/` (+ metrics), `outputs/dashboard_data.json`, the old
MapReduce CSVs, and the old `cleaned_taxi_data.csv` / `mapreduce_input.tsv`
(moved, not copied). The raw `nyc_taxi_trip_duration.csv` and the
notebook are untouched in the project root. The old job1-3 folders were
removed from `pipeline/mapreduce/` via Recycle Bin (copies are in the
archive).

**Deliverables regenerated for Bengaluru (2026-10-23):**
`deliverables/Bengaluru_Urban_Mobility_Report.{docx,pdf}` (14 pages, python-docx)
and `deliverables/Bengaluru_Urban_Mobility_Presentation.{pptx,pdf}` (17 slides,
pptxgenjs, navy/amber style kept from the NYC deck; speaker notes included).
Screenshots were captured from the live dashboard with headless Chrome + CDP
(light theme). The old NYC report/deck were moved to
`_archive_nyc_taxi/deliverables/`. The old
`Smart_Urban_Taxi_Analytics_Project.zip` was moved to `_archive_nyc_taxi/`; the new
submission zip is `Bengaluru_Urban_Mobility_Project.zip` (36 files, ~10 MB: code, models,
small outputs, deliverables; big CSVs left out, `run_all.py` regenerates them; tested from a
clean extract). Still NYC-era: the body of `MANUAL_TODO.txt` (notice at its top).

## Architecture

```
pipeline/mobility_model.py          SINGLE SOURCE OF TRUTH: 10 zones, traffic curve, vehicle speeds,
                                    fare rules, surge, night surcharge. Used by generator AND backend.
pipeline/00_generate_bengaluru_data.py  synthetic trips -> outputs/bengaluru_mobility_raw.csv (~250.7k rows,
                                    incl. ~1.5% injected dirty rows). Seed 42, deterministic.
pipeline/01_clean_and_engineer.py   raw -> outputs/cleaned_bengaluru_mobility.csv + outputs/mapreduce_input.tsv
pipeline/schema.py                  12-column TSV index contract (shared by cleaning + all mappers)
pipeline/mapreduce/job{1,2,3,4}_*/  mapper.py + reducer.py per job, Hadoop-Streaming contract
pipeline/02_run_mapreduce.py        mapper | sorted() shuffle | reducer subprocess chain -> outputs/mapreduce/*.csv
pipeline/03_train_models.py         2 sklearn models -> models/*.joblib + *_metrics.json
pipeline/04_precompute_dashboard.py outputs/dashboard_data.json, split {common, user, driver, admin}
pipeline/run_all.py                 runs steps 00-04 (~10s)

backend/app.py                      Flask: /, /api/health, /api/dashboard?role=, /api/predict/trip, /api/predict/demand
backend/templates/index.html        role switcher (User / Driver / Admin views)
backend/static/css/style.css        dataviz palette tokens; light/dark via prefers-color-scheme AND data-theme toggle
backend/static/js/dashboard.js      lazy per-role fetch + Chart.js, trip planner, demand form, theme toggle
```

Run: `python backend\app.py` → `http://127.0.0.1:5000` (`#user`, `#driver`,
`#admin` deep-link). Full regen: `python pipeline\run_all.py`.
`outputs/pipeline_run_log.txt` = terminal output of the last full run
(useful for a report appendix).

## Data: synthetic, and that must be stated

No open trip-level Bengaluru bike/auto/cab dataset exists, so data is
generated from `mobility_model.py` rules. Be upfront about this in any
report/viva text. Key rules:
- Zones (real coords): Indiranagar, Koramangala, Whitefield, HSR Layout,
  Electronic City, MG Road, Hebbal, Kempegowda Airport (BLR — real lat
  13.199, slightly outside the brief's ~13.15 bound), Majestic (KSR
  Station), Jayanagar. Zone `kind` (residential/tech/mixed/transit) drives
  hour-of-day pickup patterns.
- Traffic: morning peak 08:30-11:30 (1.5→2.2×), evening 17:30-21:30
  (1.6→2.4×), night 22-06 0.85×; weekend dampened; per-zone "choke"
  factor; rain ×1.15.
- Vehicles: congestion sensitivity bike 0.55 / auto 0.8 / cab 1.0 (+cab
  extra slowdown >1.6×); rain speed retained bike 0.75 / auto 0.9 / cab 0.95.
- Fares: bike ₹20 incl 2 km + ₹8/km + ₹1.5/delay-min; auto ₹30 incl 1.8 km
  + ₹15/km + ₹2/min; cab ₹60 incl 3 km + ₹18→22/km (rises with traffic) +
  ₹3/min. Night surcharge auto 1.5× (Bengaluru meter rule), bike/cab 1.2×.
- Surge: Low 1.0 / Medium 1.1 / High 1.3, +0.2 peak, +0.3 rain, clipped
  [1.0, 2.2]. In the API the demand level comes from Model B.
- Date range 2025-01-01..2025-06-30 (181 days, 25 rainy, more in May/June).

## The 4 MapReduce jobs

1. **job1_modal_demand_by_hour** — (hour|vehicle) → count. 72 rows.
2. **job2_avg_duration_and_fare_by_zone** — (pickup_zone|vehicle) →
   count, avg duration, avg fare, total fare. 30 rows.
3. **job3_top_congestion_corridors** — peak-hour, inter-zone only;
   (pickup|dropoff) → trips, avg speed, avg duration, avg delay, delay
   share, status. 90 rows. Status thresholds **CRITICAL < 13.75 km/h,
   SEVERE < 15** (calibrated to this data's speed spread: 21/46/23).
4. **job4_demand_dataset** — (date|hour|dow|zone|is_raining) → count.
   38,158 rows. Feeds Model B. Rain rides in the key (per-date attribute,
   doesn't split cells).

## Models (keep these settings)

RandomForest `n_estimators=80, max_depth=12, min_samples_leaf=5`,
`random_state=42`, `joblib.dump(..., compress=3)` — per the original size
decision. Sizes now: duration 1.6 MB, demand 3.8 MB.

- **Model A (duration)** features exactly as the brief:
  `vehicle_type_encoded, distance_km, hour, day_of_week, traffic_factor`
  + `is_raining` (added so the bike rain penalty is learnable).
  **Target is `duration_min / physics_eta`**, prediction = output ×
  `mm.expected_duration_minutes(...)`. Reason: a raw-minutes target gave
  identical ETAs for all 3 vehicles on long/rare trips (e.g. airport at
  night, 52 km → 77.8 min for all) because trees can't extrapolate and
  long bike/auto trips are rare. Don't revert. Fare is NOT modelled — it
  is computed from predicted ETA via the rate card.
- **Model B (demand level)** features `hour, day_of_week, is_weekend,
  is_peak_hour (hour+0.5 in peak window), zone_code, is_raining` on Job 4
  output; tertile labels (≤4 Low, ≤8 Medium, >8 High).

## Current metrics (last real run — cite exactly)

- Raw 250,750 rows → cleaned **247,951** (750 duplicates, 600 missing
  zone, 600 zero/negative distance, 600 duration/timestamp mismatch, 249
  outliers >99.9th pct = 185 min removed; 600 missing fares imputed from
  rate card; 600 upper-cased/space-padded vehicle names normalised)
- Mode share: bike 33.2% / auto 34.7% / cab 32.1%
- Model A: **MAE 2.995 min, R² 0.957** (bike 1.88 / auto 2.77 / cab 4.38).
  Physics formula alone also scores MAE 2.99 — honest caveat: on
  synthetic data generated by that formula, ML can't beat it. Shown in
  the Admin ML card.
- Model B: **accuracy 68.0%, weighted F1 0.687**; classes Low 15,442 /
  Medium 12,220 / High 10,496
- KPIs: revenue ₹5.36 Cr, avg fare ₹216, avg duration 26.9 min, avg
  distance 7.78 km, avg speed 17.4 km/h, avg surge 1.39×, peak hour
  19:00, busiest zone Koramangala, 21 critical corridors (slowest:
  Koramangala→HSR Layout 13.1 km/h — Silk Board effect)
- Insights worth quoting: rain slows bikes +47% per km vs cab +31%;
  ₹/trip-minute DROPS in peak jams (delay charges don't keep pace with
  time); auto is pricier than cab at night (1.5× meter).

## Verification done (2026-10-23)

`run_all.py` clean with `-W error::FutureWarning`, 0 warnings. Live
server: all GET routes + both POST endpoints 200; bad role / bad zone /
bad hour / garbage body → 400. Chrome: all three views rendered in dark
and light, trip planner + demand form submitted, theme toggle re-renders
charts. Not tested at phone width in a real browser (CSS has 900px/600px
breakpoints).

## Still open (user's to do)

1. Report, deck and zip are all Bengaluru now. If code/deliverables change,
   rebuild the zip (script logic: same 36-file list) and commit + push.
2. Fill in real names/date on title pages (`[Your Name]` etc.) — never guess.
3. Submission deadline per handbook: 06-11-2026 (verify against handbook p.5).
4. Optional hosting / real Hadoop via WSL2 — user declined WSL; don't re-suggest.

## Environment gotchas (Windows, this machine)

- Bash tool = Git Bash. Heredocs containing `'` / backslashes can break or
  get mangled (bit this session twice) — prefer the Write/Edit tools for
  file content, and PowerShell for Windows-native ops.
- No LibreOffice/poppler. Office is installed → docx/pptx→PDF via COM
  (`Word.Application` `.SaveAs(path,17)`, `PowerPoint.Application`
  `.SaveAs(path,32)`). PyMuPDF (`fitz`) for rendering PDF pages.
- node `docx` / `pptxgenjs` not global — `npm init -y && npm install` in a
  scratch folder.
- pandas is a pre-release build: use `["object", "str", "category"]` in
  `select_dtypes`.
- Python deps global (pandas, numpy, scikit-learn, joblib, Flask). No
  xgboost, WSL, Hadoop, Docker. Java 17 present.
- Git repo (since 2026-10-24), pushed to private GitHub repo
  `AnoonMR/bengaluru-urban-mobility-analytics`, branch `main`. `.gitignore`
  excludes big CSVs, the archive, zips, the NYC notebook and the handbook.
- Flask debug reloader does NOT reload models — restart the server after
  re-running the pipeline. Kill with `netstat -ano | grep ":5000 "` →
  `taskkill //F //PID <pid> //T`.
- Chrome automation: ref-based clicks on the submit button didn't always
  submit; coordinate clicks (from a fresh screenshot) did.

## User preferences

- Manual/installation steps go in a txt file (`MANUAL_TODO.txt`), not run.
- Declined WSL/Ubuntu/Docker for real Hadoop — don't re-litigate.
- Wants deliverables as real files (docx/pptx + PDFs).
- Global rule: **never `rm` / `Remove-Item`** — Recycle Bin via
  `[Microsoft.VisualBasic.FileIO.FileSystem]::DeleteFile/DeleteDirectory(..., 'OnlyErrorDialogs', 'SendToRecycleBin')`.
