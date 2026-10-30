# TB-Paraná-DW: Tuberculosis Data Warehouse & Risk Classification

A data warehouse that combines **tuberculosis case notifications**, **climate**, and **socioeconomic** data for every municipality in Paraná, Brazil (2018–2023). It supports multidimensional analysis and a municipality-level **risk classification model**.

Developed for the *Data Preparation and Integration* course in the B.Sc. in Data Science and AI at Universidade Estadual de Londrina (UEL), supervised by Prof. Daniel Kaster.

---

## Why

Tuberculosis incidence depends on more than health-system data. Population density, income, and climate are all plausible drivers. These variables come from different agencies, in different formats, at different granularities. This project integrates them into a single dimensional model so they can be analyzed together.

## Data sources

| Source | Content | Raw format |
|---|---|---|
| DATASUS | TB cases per municipality per year | CSV |
| IBGE | Population density | CSV |
| IBGE | Municipal GDP (2017–2023) | XLSX → converted to XML |
| INMET | Weather stations (metadata) | JSON |
| INMET | Daily measurements: temperature, humidity, precipitation | CSV (one per station/year) |

## Pipeline

```
raw files ──► Python preprocessing ──► staging tables (Oracle) ──► star schema ──► analytical SQL
                                                                         │
                                                                         └──► ML: risk classification
```

1. **Preprocessing (Python):** encoding and character cleanup. The GDP spreadsheet is converted to XML and read in the database with `XMLTABLE`. Daily data from all INMET stations in Paraná is consolidated into one file.
2. **Staging:** raw data is loaded into permissive temporary tables, including `XMLTYPE` and a JSON-constrained `CLOB`.
3. **Star schema:** data is transformed into dimension and fact tables.
4. **Analysis:** OLAP and analytical queries run over the warehouse.
5. **Modeling:** a classifier predicts municipality risk levels.

## Star schema

<img width="2125" height="1833" alt="ER diagram - tuberculosis DW" src="https://github.com/user-attachments/assets/12b920ad-cb97-47ee-85ee-396fdb9392a5" />

- **Fact:** `fato_tuberculose` (TB cases per municipality × year)
- **Dimensions:** `dim_tempo` (year, with a pandemic flag for 2020–2022), `dim_municipio`, `dim_estacao`, `dim_clima` (yearly aggregates per station), `dim_socioeconomico` (GDP per capita, total GDP, population density)

**Design decision:** in the first version, the fact table reached climate only through `dim_clima`. That did not guarantee each municipality was linked to the correct weather station, so `sk_estacao` was added to the fact table as an explicit, mandatory foreign key.

## Analytical queries

There are 15 queries in [`SCRIPTS/SQL/consultas.sql`](SCRIPTS/SQL/consultas.sql), including:
- window functions (`LAG`, `RANK`, `NTILE`, 3-year moving averages) for municipal trends
- OLAP operators (`ROLLUP`, `CUBE`, `GROUPING SETS`), e.g. cases by region × year and pandemic × GDP band
- a simple view and a materialized view for consolidated indicators
- SQL/XML and SQL/JSON exports

## Risk classification

Notebook: [`ML/TB_pipeline_completo.ipynb`](ML/TB_pipeline_completo.ipynb). Developed together with Sofia Gutschow Casal.

**Task:** given a municipality's indicators in year *t*, predict its TB risk category in year *t+1*.
- Low/None: ≤ 5 cases · Moderate: 6–20 · High: > 20
- Dataset: 1,383 instances (285 municipalities × 2018–2022), heavily imbalanced (1,105 Low · 169 Moderate · 109 High)

**Data quality.** The integrated data already had 54 real missing values from weather-station gaps. As part of the assignment, we also injected noise into about 6% of the cells in five numeric columns (missing values and out-of-range outliers) and into about 3% of the region names (text inconsistencies). Every injected error is logged in `ML/dados/log_ruido_injetado.csv`. Cleaning then:
- flags values outside domain ranges (e.g. humidity outside 0–100%) as missing
- normalizes text
- imputes with the regional median, falling back to the global median

**Derived features:** previous-year cases, 2-year moving average of cases, cases × population density, and year-over-year change in GDP per capita.

**Model and validation:** Random Forest (`class_weight="balanced"`) tuned with `GridSearchCV`. All validation uses `GroupKFold` with **municipality as the group**. The same municipality appears in several years, so a random split would put, say, Londrina 2018 in training and Londrina 2019 in testing. The model would recognize the city instead of learning the pattern. Grouping keeps each test fold made of unseen municipalities.

**Results** (balanced accuracy, 4-fold grouped CV):

| Model | Balanced accuracy |
|---|---|
| Persistence baseline: *"next year = this year's category"* (no ML) | 0.819 |
| Random Forest, original features | 0.821 |
| Random Forest, + derived features | **0.843** |
| + feature selection (importance < 0.02 removed) | 0.841 |

**Reading:**
- TB risk is highly persistent. A rule that simply repeats this year's category already reaches 0.82, so the model's real added value is about +2.4 points, all of it from the temporal features.
- On a held-out set of municipalities, the *Moderate* class remains hard (recall 0.22). It sits between two thresholds and most of its errors flip to a neighboring class.
- The reported CV scores are the best scores from the grid search, without nested CV, so they are slightly optimistic.

Natural next steps: model incidence per 100k inhabitants instead of absolute counts, try ordinal or regression formulations that respect the Low < Moderate < High order, and use nested CV.

## Repository structure

```
DADOS/
  BRUTOS/          raw source files
  PROCESSADOS/     cleaned / consolidated files
SCRIPTS/
  PYTHON/          preprocessing and database loading
  SQL/             staging tables, star schema, ETL, analytical queries
ML/
  TB_pipeline_completo.ipynb   target, features, noise, cleaning, modeling
  dados/                       DW export (vw_municipio_tb.csv), intermediate datasets, noise log, results
```

> The raw daily INMET files are not versioned because of their size. The consolidated output (`clima_pr_2018_2023.csv`) is included.

## How to run

1. Start an Oracle database (developed with Oracle XE).
2. Run `SCRIPTS/SQL/tabelas_temp.sql`, then `SCRIPTS/PYTHON/carga_dados.py` to load the staging tables.
3. Run `SCRIPTS/SQL/esquema_estrela.sql`, then `SCRIPTS/SQL/temporarias_para_dw.sql`.
4. Run the queries in `SCRIPTS/SQL/consultas.sql`.
5. ML part (no database needed): open `ML/TB_pipeline_completo.ipynb` from inside `ML/`. It reads `dados/vw_municipio_tb.csv`.

## Tech

Python (pandas, NumPy, oracledb, scikit-learn, seaborn) · Oracle SQL · dimensional modeling
