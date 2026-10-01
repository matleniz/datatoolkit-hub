> Archived 2026-10-01 from the never-merged `qa-exercises` branch of datatoolkit-web
> (QA run of 2026-09-28). All six GAP findings were filed and closed as MAT-160;
> this report is kept as a historical record, not as an open backlog.

# QA report — qa-exercises (public EDA / cleaning exercises via Studio UI)

**Worker:** `qa-exercises`
**App:** Studio UI http://127.0.0.1:5190 (API http://127.0.0.1:8790) — already running; not restarted.
**Method:** Playwright headless Chromium 1440×900; upload via file input; attempt each exercise using only the Studio UI (load → clean → analyse → transform → export).
**Artifacts:** `~/dtk-qa/qa-exercises/` (scripts + downloaded data); screens `~/dtk-qa/screens/qa-exercises/`.
**Scope:** 10 web-sourced data-analysis / data-cleaning / EDA exercises with downloadable CSVs, increasing difficulty.
**Known already (not re-filed):** non-CSV uploads detected as CSV; no UI for CSV load options; feature-selection / ffill / knn / bin / group_agg / cyclical not in step picker; `?` sentinel not detected; MAT-145 / MAT-146 / MAT-147. (+1 group_agg seen on drinks-by-continent.)

## Summary

Covered: UCI iris, guipsamora Euro12 + Chipotle, justmarkham drinks, seaborn tips, palmerpenguins, EDS-217 messy field survey, LaunchCode cleaning practice, TidyTuesday coffee ratings, Kaggle Learn–style SF Building Permits (open-data sample).

Studio solves clean tabular EDA end-to-end (**yes=3**: iris, tips tip_rate, penguins). Most real cleaning labs are **partial=7** (load/analyse OK, finishing transform blocked). **no=0**.

New gaps vs student exercises: **Filter rows** missing from picker (engine has it), **currency/$ strip**, **regex/range parse**, **overall % missing**, **threshold dropna(axis=1)**, and **punctuation unify** in standardize_text.

| Severity | Count |
|---|---|
| BLOCKER | 0 |
| BUG | 0 |
| GAP | 6 |
| UX | 0 |

### Exercise ladder

| # | id | Source | Solved? | Question | What was missing / broken |
|---|---|---|---|---|---|
| 1 | `01-iris-eda` | https://archive.ics.uci.edu/dataset/53/iris (classic ED… | **yes** | Load iris CSV, confirm ~150×5, inspect sepal_length dis… | — |
| 2 | `02-euro12-filter` | https://github.com/guipsamora/pandas_exercises/tree/mas… | **partial** | Load Euro_2012_stats_TEAM.csv; how many teams? which sc… | Can load and inspect Goals distribution (16×35), but Filter rows is not in the step picker — cannot keep only Goals>6 or project a filtered  |
| 3 | `03-drinks-groupby` | https://github.com/justmarkham/DAT8/blob/master/data/dr… | **partial** | Which continent has the highest mean beer_servings? Imp… | Missing dock shows continent NaNs; can impute. Cannot compute mean beer_servings by continent — group_agg not in step picker (+1 known, not  |
| 4 | `04-chipotle-prices` | https://github.com/guipsamora/pandas_exercises/tree/mas… | **partial** | Load chipotle.tsv; clean item_price (strip $); total re… | TSV loads (tab sniff, 4622×5). item_price stays text with '$' prefix; Formula numeric-only — cannot compute revenue in Studio alone. |
| 5 | `05-tips-tiprate` | https://github.com/mwaskom/seaborn-data/blob/master/tip… | **yes** | Create tip_rate=tip/total_bill; correlate with size; on… | — |
| 6 | `06-penguins-clean` | https://github.com/allisonhorst/palmerpenguins… | **yes** | Handle missing bill_*/sex/body_mass; one-hot island; sc… | — |
| 7 | `07-messy-field-survey` | https://eds-217-essential-python.github.io/course-mater… | **partial** | Standardize site→6 labels; fill n_replicates blanks wit… | After Standardize text, site still shows distinct=24 / "24 spellings" (casing groups exist but site-a vs site_a remain). Filter rows absent  |
| 8 | `08-launchcode-cleaning` | https://education.launchcode.org/data-analysis-curricul… | **partial** | Clean practice CSV: missing names, duplicate email colu… | Column `$transaction_total` stays text with '$…' values — no currency strip/cast in UI; duplicate email columns ['email', 'email.1']; no mer |
| 9 | `09-tt-coffee` | https://github.com/rfordatascience/tidytuesday/tree/mas… | **partial** | Clean altitude ranges ('1950-2200','1600 - 1800 m') to … | Missing dock + target work; altitude is free-text ranges ('1950-2200', '1600 - 1800 m'); harvest_year mixes '2012' and '2013/2014'. No regex |
| 10 | `10-sf-permits-missing` | https://www.kaggle.com/learn/data-cleaning + https://da… | **partial** | Kaggle Learn ex1-style: % missing cells; drop columns w… | Per-column missing works; No overall % missing cells metric (Kaggle Learn percent_missing); No drop-columns-where-missing-fraction>threshold |

Step-picker ops seen: Parse dates, Rename, Cast, Drop duplicates, Replace sentinels, Standardize text, Impute, Clip, One-hot, Ordinal, log1p, Scale, Datetime parts, Derive, Drop columns, Formula.
Absent (known + new): Bin, Forward fill, Group aggregate, Cyclical, Select k best, Impute (KNN), **Filter rows**.

## Findings

### [GAP] Filter rows op not offered in step picker (Euro12 / messy-survey labs)
- **dataset**: Euro_2012_stats_TEAM.csv; messy_field_survey.csv
- **steps**: Workbench → + Step → look for Filter rows / filter_rows
- **expected**: filter_rows available (engine lists it) to keep Goals>6 or dropna(subset=measurement columns)
- **actual**: Step picker OP_STAGE omits filter_rows. Observed ops: Parse dates, Rename, Cast, Drop duplicates, Replace sentinels, Standardize text, Impute, Clip, One-hot, Ordinal, log1p, Scale, Datetime parts, Derive, Drop columns, Formula.
- **screenshot**: `/home/matleniz/dtk-qa/screens/qa-exercises/02-euro12-bench.png`
- **suspected layer**: web front `src/bench/stages.ts` OP_STAGE (engine has filter_rows)

### [GAP] No currency / money-string cleaning path (Chipotle + LaunchCode)
- **dataset**: chipotle.tsv; cleaning_data_practice.csv
- **steps**: Upload → workbench → inspect item_price or $transaction_total → Cast / Formula / Standardize
- **expected**: Strip leading '$' (and spaces) and cast to float so quantity*price / transaction totals are computable
- **actual**: Columns stay kind=text with values like '$2.39 ' / '$50.32'. Formula is numeric-only; Cast does not parse money.
- **screenshot**: `/home/matleniz/dtk-qa/screens/qa-exercises/04-chipotle-bench.png`
- **suspected layer**: both: no money parse transform; front Formula whitelist has no string replace

### [GAP] standardize_text leaves site-a vs site_a distinct (EDS-217 needs 6 sites)
- **dataset**: messy_field_survey.csv
- **steps**: Standardize text on site → Inspector (distinct / Value groups)
- **expected**: 6 canonical sites after normalizing case, whitespace, and hyphen/underscore punctuation
- **actual**: distinct=24; Value groups collapse casing only ("site_d"←Site_D/SITE_D). Hyphenated site-a remains separate from site_a. Map a value exists but requires hand-entering every variant.
- **screenshot**: `/home/matleniz/dtk-qa/screens/qa-exercises/07-messy-bench3.png`
- **suspected layer**: engine standardize_text (no punct unify) + front mapping UX

### [GAP] No regex / range-parse transform for messy altitude strings (TidyTuesday coffee)
- **dataset**: tt_coffee.csv
- **steps**: Workbench → altitude (values like 1950-2200, 1600 - 1800 m) → Cast / Formula
- **expected**: Extract numeric mid-altitude (common TidyTuesday cleaning step)
- **actual**: altitude stays text; Formula cannot parse ranges/units. harvest_year also mixes 2012 and 2013/2014.
- **screenshot**: `/home/matleniz/dtk-qa/screens/qa-exercises/09-coffee-bench2.png`
- **suspected layer**: engine (no string extract op) / Formula whitelist

### [GAP] Missing-values tool lacks overall % missing cells (Kaggle Learn metric)
- **dataset**: sf_permits_sample.csv (5k×55 sample of SF Building Permits)
- **steps**: Open workbench → Missing values tool
- **expected**: Dataset-level percent_missing = total_missing / total_cells * 100 (Kaggle Learn Data Cleaning ex1)
- **actual**: Dock shows per-column missing counts/fractions only; no overall cell-missing percentage.
- **screenshot**: `/home/matleniz/dtk-qa/screens/qa-exercises/10-sf-bench2.png`
- **suspected layer**: web front ResultView for missing_values key (and/or key metrics)

### [GAP] No threshold drop for high-missing columns (dropna axis=1 style)
- **dataset**: sf_permits_sample.csv
- **steps**: + Step → Drop columns (manual multi-select only)
- **expected**: Drop all columns with missing fraction > 0.5 in one step (Kaggle Learn exercise)
- **actual**: Only manual drop_columns; with 55 columns this makes the exercise tedious/impractical in the UI.
- **screenshot**: `/home/matleniz/dtk-qa/screens/qa-exercises/10-sf-bench2.png`
- **suspected layer**: engine drop_columns has no min_non_null threshold; front has no bulk helper

## What worked well

- Iris / tips / penguins: Sources → target → Missing/Distribution/Correlation docks → Impute / One-hot / Scale / Derive-or-Formula → Export parquet + manifest (fitted steps).
- Chipotle `.tsv` sniffed as `sep \t` (4622×5).
- Messy survey `collection date` auto-kinded as date; Formula successfully added `temperature_f`.
- SF permits 5k×55 opened; Missing-values dock lists per-column missing.
- Inspector Value groups surface casing variants after standardize_text (useful hint, even if punctuation remains).

## Per-exercise detail

### `01-iris-eda` (difficulty 1) — solved: **yes**
- **source URL**: https://archive.ics.uci.edu/dataset/53/iris (classic EDA; also seaborn/sklearn)
- **question**: Load iris CSV, confirm ~150×5, inspect sepal_length distribution, set target=species, scale numerics, export.
- **dataset / workspace**: `iris.csv` / `qa-exercises-iris`
- **missing / broken**: —
- **notes**: shape= · dist=ok:Distribution | column_distribution | Select a column to see its distribution. · missing=ok:Missing values | missing_values | train · all columns | sepal_length | — | sepal_width | — | petal_length | — | petal_width | — | · export=True:2 steps · 2 fitted
Output dir: /home/matleniz/dtk-qa/qa-exercises/exports/qa-exercises-iris
train: processed/train.parqu · export confirmed 150 rows parquet
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/01-iris-sources.png`, `/home/matleniz/dtk-qa/screens/qa-exercises/01-iris-bench.png`

### `02-euro12-filter` (difficulty 2) — solved: **partial**
- **source URL**: https://github.com/guipsamora/pandas_exercises/tree/master/02_Filtering_%26_Sorting/Euro12
- **question**: Load Euro_2012_stats_TEAM.csv; how many teams? which scored most Goals? filter teams with Goals>6; select Team+Goals+Shooting Accuracy+Yellow Cards.
- **dataset / workspace**: `Euro_2012_stats_TEAM.csv` / `qa-exercises-euro12`
- **missing / broken**: Can load and inspect Goals distribution (16×35), but Filter rows is not in the step picker — cannot keep only Goals>6 or project a filtered column subset.
- **notes**: shape= · goals_dist=ok:Distribution | column_distribution | bound to Goals | n 16 · missing 0 · distinct 8 · picker_ops_sample=['Cast column types', 'Clip outliers', 'Datetime parts', 'Derive column', 'Drop columns', 'Drop duplicates', 'Formula', 'I
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/02-euro12-bench.png`

### `03-drinks-groupby` (difficulty 3) — solved: **partial**
- **source URL**: https://github.com/justmarkham/DAT8/blob/master/data/drinks.csv
- **question**: Which continent has the highest mean beer_servings? Impute missing continent; compute mean alcohol by continent.
- **dataset / workspace**: `drinks_by_country.csv` / `qa-exercises-drinks`
- **missing / broken**: Missing dock shows continent NaNs; can impute. Cannot compute mean beer_servings by continent — group_agg not in step picker (+1 known, not re-filed).
- **notes**: shape= · missing=ok:Missing values | missing_values | train · all columns | country | — | beer_servings | — | spirit_servings | — | wine_servings | — · continent_menu=continent · text
Inspect
Add to selection
shift-click
Distribution
Standardize text…
One-hot…
Impute…
Rename…
Cast type…
Set  · imputed continent · +1 seen: group_agg not in step picker (known, not filed as new finding)
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/03-drinks-bench.png`

### `04-chipotle-prices` (difficulty 4) — solved: **partial**
- **source URL**: https://github.com/guipsamora/pandas_exercises/tree/master/01_Getting_%26_Knowing_Your_Data/Chipotle
- **question**: Load chipotle.tsv; clean item_price (strip $); total revenue quantity*item_price; how many unique item_name?
- **dataset / workspace**: `chipotle.tsv` / `qa-exercises-chipotle`
- **missing / broken**: TSV loads (tab sniff, 4622×5). item_price stays text with '$' prefix; Formula numeric-only — cannot compute revenue in Studio alone.
- **notes**: detected=FILE
DETECTED (FILE_INSPECT)
ROLE
chipotle.tsv
order_id, quantity, item_name, choice_description, item_price
csv · sep "\\t" · utf- · shape= · item_price_menu=item_price · text
Inspect
Add to selection
shift-click
Distribution
Standardize text…
One-hot…
Rename…
Cast type…
Set as tar · formula_try=fail:Page.wait_for_function: Timeout 20000ms exceeded.:NEW STEP · CUSTOM FORMULA
← All transforms
Formula
Your own column from a · inspector=Locator.click: Timeout 30000ms exceeded.
Call log:
  - waiting for locator(".grid-th").filter(has_text=re.compile(r"^item_price$")
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/04-chipotle-sources.png`, `/home/matleniz/dtk-qa/screens/qa-exercises/04-chipotle-bench.png`

### `05-tips-tiprate` (difficulty 5) — solved: **yes**
- **source URL**: https://github.com/mwaskom/seaborn-data/blob/master/tips.csv
- **question**: Create tip_rate=tip/total_bill; correlate with size; one-hot sex; export.
- **dataset / workspace**: `tips.csv` / `qa-exercises-tips2`
- **missing / broken**: —
- **notes**: shape=244 × 7 · editor=NEW STEP · ENCODE & TRANSFORM | ← All transforms | Derive column | Add a column combining two columns (ratio, difference, product, da · formula_fallback=ok
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/05-tips-bench2.png`

### `06-penguins-clean` (difficulty 6) — solved: **yes**
- **source URL**: https://github.com/allisonhorst/palmerpenguins
- **question**: Handle missing bill_*/sex/body_mass; one-hot island; scale numerics; target=species.
- **dataset / workspace**: `penguins.csv` / `qa-exercises-penguins`
- **missing / broken**: —
- **notes**: missing=ok:Missing values | missing_values | train · all columns | species | — | island | — | bill_length_mm | 2 · 1% | bill_depth_mm | 2 ·  · bill_length_mm:Impute:ok · body_mass_g:Impute:ok · sex:Impute:ok · island:One-hot:ok · bill_length_mm:Scale:ok
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/06-penguins-bench.png`

### `07-messy-field-survey` (difficulty 7) — solved: **partial**
- **source URL**: https://eds-217-essential-python.github.io/course-materials/coding-colabs/4d_cleaning_messy_data.html
- **question**: Standardize site→6 labels; fill n_replicates blanks with 1; drop rows missing measurements; add temperature_f; classify pH.
- **dataset / workspace**: `messy_field_survey.csv` / `qa-exercises-messy4`
- **missing / broken**: After Standardize text, site still shows distinct=24 / "24 spellings" (casing groups exist but site-a vs site_a remain). Filter rows absent — cannot dropna(subset=measurement cols). temperature_f formula works (applied). n_replicates constant-fill=1 not clearly driven (default impute used). No pH classify/bin in picker (bin known-absent).
- **notes**: collection date auto-kind=date · standardize_text applied; inspector distinct=24 with value groups for strip+lower casing · temperature_f column created via Formula · picker has no Filter rows / Bin
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/07-messy-bench3.png`, `/home/matleniz/dtk-qa/screens/qa-exercises/07-messy-formula-ok.png`

### `08-launchcode-cleaning` (difficulty 8) — solved: **partial**
- **source URL**: https://education.launchcode.org/data-analysis-curriculum/cleaning-spreadsheets/exercises/index.html
- **question**: Clean practice CSV: missing names, duplicate email columns, datetime, $transaction_total→number, drop password.
- **dataset / workspace**: `cleaning_data_practice.csv` / `qa-exercises-launch2`
- **missing / broken**: Column `$transaction_total` stays text with '$…' values — no currency strip/cast in UI; duplicate email columns ['email', 'email.1']; no merge-columns helper
- **notes**: shape=603 × 13 · headers=['first_name', 'last_name', 'email', 'employer', 'username', 'password', 'email.1', 'ipv4_address', 'user_agent', 'datetime', 'compa · money_menu=$transaction_total · identifier | Inspect | Add to selection | shift-click | Distribution | Rename… | Cast type… | Set as target  · money_insp=Inspector | Click a column header (right-click for its menu), a row number or a cell. Or start a step from + Step in the pipeline · drop_password=ok · impute_last=menuitem /Impute/ missing; menu=last_name · identifier
Inspect
Add to selection
shift-click
Distribution
Rename…
Cast type…
Set  · parse_dt=menuitem /Parse dates/ missing; menu=datetime · date
Inspect
Add to selection
shift-click
Distribution
Rename…
Cast type…
Set as ta
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/08-launchcode-sources.png`, `/home/matleniz/dtk-qa/screens/qa-exercises/08-launchcode-bench2.png`

### `09-tt-coffee` (difficulty 9) — solved: **partial**
- **source URL**: https://github.com/rfordatascience/tidytuesday/tree/master/data/2020/2020-07-07
- **question**: Clean altitude ranges ('1950-2200','1600 - 1800 m') to numeric mean; normalize harvest_year; impute farm/region; correlate scores vs altitude.
- **dataset / workspace**: `tt_coffee.csv` / `qa-exercises-coffee2`
- **missing / broken**: Missing dock + target work; altitude is free-text ranges ('1950-2200', '1600 - 1800 m'); harvest_year mixes '2012' and '2013/2014'. No regex/range-mean transform — cannot finish TT cleaning.
- **notes**: shape=1339 × 43 · missing=ok:Missing values | missing_values | train · all columns | total_cup_points | — | species | — | owner | 7 · 1% | country_of_origin | · altitude_menu=altitude · text | Inspect | Add to selection | shift-click | Distribution | Standardize text… | One-hot… | Impute… | Rename… | · altitude_insp=Inspector | Click a column header (right-click for its menu), a row number or a cell. Or start a step from + Step in the pipel · harvest_year=harvest_year · text
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/09-coffee-bench2.png`

### `10-sf-permits-missing` (difficulty 10) — solved: **partial**
- **source URL**: https://www.kaggle.com/learn/data-cleaning + https://data.sfgov.org/Housing-and-Buildings/Building-Permits/i98e-djp9 (5k sample)
- **question**: Kaggle Learn ex1-style: % missing cells; drop columns with >50% NA; impute remaining; report cleaned shape.
- **dataset / workspace**: `sf_permits_sample.csv` / `qa-exercises-sf2`
- **missing / broken**: Per-column missing works; No overall % missing cells metric (Kaggle Learn percent_missing); No drop-columns-where-missing-fraction>threshold (dropna axis=1)
- **notes**: shape=5000 × 55 · missing=ok:Missing values | missing_values | train · all columns | permit_number | — | permit_type | — | permit_type_definition | 106 · 2% | · has_dataset_pct=False
- **screens**: `/home/matleniz/dtk-qa/screens/qa-exercises/10-sf-bench2.png`
