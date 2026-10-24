# Game Lifecycle Decay Analytics

Analiza degradacji sentymentu w cyklu życia gry na danych recenzji Steam („Klątwa Mainstreamu").

## Hipoteza

Wraz ze wzrostem łącznego wolumenu recenzji i oddalaniem się od premiery średnia ocen gry systematycznie spada. Tytuł opuszcza bańkę lojalnych fanów i trafia do niedopasowanych graczy mainstreamowych.

## Wartość biznesowa

Narzędzie prognostyczne dla inwestorów i producentów, które pozwala przewidzieć długoterminowy rating tytułu na podstawie pierwszych 48 godzin od premiery.

## Stack

- **Parquet** – przechowywanie danych
- **DuckDB** – profilowanie, czyszczenie, window functions
- **Polars** – feature engineering
- **scikit-learn** – modelowanie

## Architektura danych (Parquet)

Dane są partycjonowane przez `partition_cols=['app_id']` lub `year` (wyciągnięty z `timestamp_created`).

Cel: sekwencyjna analiza szeregów czasowych per gra bez wczytywania reszty biblioteki Steam do RAM.

## Profilowanie i czyszczenie (DuckDB)

**Detekcja anomalii czasowych:**

- `timestamp_created > now()`
- `timestamp_updated < timestamp_created`

**Krzywa sentymentu** (window function):

```sql
SELECT
    app_id,
    timestamp_created,
    AVG(recommended::INT) OVER (
        PARTITION BY app_id
        ORDER BY timestamp_created
    ) AS running_sentiment
FROM reviews;
```

## Feature engineering (Polars)

- `daysSinceFirstReview`
- dynamika napływu recenzji (rolling window 7-dniowe)
- cechy pierwszego 5% wolumenu ocen

## Model (scikit-learn)

`RandomForestRegressor` prognozujący docelowy rating gry (np. po 50k ocen) na bazie pierwszych 2k recenzji.

## Struktura projektu

```
game-lifecycle-decay-analytics/
├── data/
│   ├── raw/
│   └── parquet/
├── sql/
├── notebooks/
├── src/
└── README.md
```

## Uruchomienie

```bash
git clone https://github.com/<user>/game-lifecycle-decay-analytics.git
cd game-lifecycle-decay-analytics
pip install duckdb polars scikit-learn pyarrow
```
