# Tipping Behavior in NYC Yellow Taxis

CIS 5450 final project (Fall 2026). Group: Ziyue Xu, Yuzhi Zheng, Lin Ge, Yiwen Yan.

**Question.** Among NYC yellow taxi trips with a positive recorded card tip, which time, location, trip, and weather characteristics are associated with a tip of at least 25% of the fare, and how well can that be predicted at trip completion?

## Notebooks

| Notebook | Owner | Contents | Open |
| --- | --- | --- | --- |
| `notebooks/01_data_pipeline.ipynb` | Ziyue Xu | Download, DuckDB sampling, cleaning, zone + weather joins, target and split, export | [Colab](https://colab.research.google.com/github/DaisyXUUUUU/CIS-5450-Project/blob/main/notebooks/01_data_pipeline.ipynb) |
| EDA and feature engineering | Yuzhi Zheng | | |
| Hypothesis tests and bootstrap | Lin Ge | | |
| Modeling and evaluation | Yiwen Yan | | |

## Data

Data files are **not** stored in git. `01_data_pipeline.ipynb` downloads the raw files to Google Drive
(`MyDrive/CIS5450_project/data/raw/`) and writes the analysis table to `data/clean/trips_clean_v1.parquet`, plus a full CSV copy (`trips_clean_v1.csv`) and a 10,000-row preview CSV for opening in Excel or Sheets. Use the Parquet file in code, since CSV loses column types.
Teammates can read that file from a shared Drive folder. Column meanings and the list of columns that must not be used as features are in [`docs/data_dictionary.md`](docs/data_dictionary.md).

Sources: [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) (yellow taxi, April to June 2026, plus the Taxi Zone Lookup Table) and the [Open-Meteo Historical Weather API](https://open-meteo.com/en/docs/historical-weather-api).

## Study design (fixed in the proposal)

- Credit-card trips only, 300,000 sampled per month, seed 5450; trips with `tip_amount = 0` are dropped.
- Target `high_tip = tip_amount / fare_amount >= 0.25`; 20% and 30% are sensitivity checks.
- Split by pickup time: train April 1 to May 15, validation May 16 to 31, test June.
- `tip_amount`, `total_amount`, and anything derived from them are never features.
