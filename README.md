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

## Dane surowe (kamień milowy 1)

Pełny zrzut leży w `data/raw/steam-reviews/` (663 plików Parquet, około 23,5 GB). Próbka 2 000 000 wierszy: `data/raw/steam_reviews_2m.parquet`. Oba katalogi danych są poza gitem. Schemat, statystyki i anomalie techniczne liczą pełny zrzut. Anomalie biznesowe, szum tekstu i rozkłady liczą próbkę 2 000 000 wierszy. Próbka jest zestawem roboczym tych sekcji: nie pojedynczy `part.0` i nie cały zrzut około 24 GB. Notatniki: `notebooks/m1_profil_jakosci_danych.ipynb` (pełny profil) oraz `notebooks/m1_schemat_statystyki_anomalie_techniczne.ipynb` (schemat, statystyki i anomalie techniczne pełnego zrzutu).

Pełny zrzut pobiera `snapshot_download` z pakietu `huggingface_hub` (jest na liście instalacji poniżej) do `data/raw/steam-reviews`. To samo wywołanie jest w notatniku i startuje, gdy w katalogu nie ma 663 plików `*.parquet`. Rozmiar zrzutu to około 23,5 GB.

```python
from huggingface_hub import snapshot_download

snapshot_download(
    repo_id="adde023/steam-reviews",
    repo_type="dataset",
    allow_patterns=["*.parquet"],
    local_dir="data/raw/steam-reviews",
)
```

Próbka 2 000 000 wierszy nie należy do tego repozytorium. Notatnik czyta ją z `data/raw/steam_reviews_2m.parquet`.

Nazwy kolumn w pliku różnią się od opisu kamienia milowego. Mapowanie jest w notatniku.

### Pochodzenie zbioru

Zbiór jest wtórnym zrzutem recenzji Steam. Karta Kaggle [kieranpoc/steam-reviews](https://www.kaggle.com/datasets/kieranpoc/steam-reviews) opisuje 113 883 717 recenzji oraz pola autora, czasu gry, języka i głosów zgodne z plikiem Parquet. Karta nie zawiera kodu zbierania, sumy kontrolnej ani okna czasowego pobrania. Lustro tej karty datuje zbiór na listopad 2023. Repozytorium [adde023/steam-reviews](https://huggingface.co/datasets/adde023/steam-reviews) powstało 2026-03-12 i opisuje konwersję plików Kaggle do Parquet. Po konwersji w pliku pozostała kolumna `__null_dask_index__`: indeks Dask, nie pole Steam.

Etykiety licencji są rozbieżne. Kaggle podaje MIT (stan karty na 2026-11-05). README na Hugging Face podaje CC BY-NC 4.0. Żadna z etykiet nie stanowi zgody Valve. [Regulamin Steam Web API](https://steamcommunity.com/dev/apiterms) wiąże dostęp z licencją pojedynczej aplikacji i ogranicza ruch do 100 000 żądań na dobę. Publiczny zrzut 113 milionów wierszy pozostaje poza tym zakresem.

`author_steamid` oraz `recommendationid` mają typ `DOUBLE`, nie `BIGINT`. Wartości SteamID64 leżą w pobliżu 7,66e16, powyżej 2^53 (9 007 199 254 740 992). W tym zakresie float64 reprezentuje co szesnastą liczbę całkowitą, więc młodsze bity zapisu znikają. Dla recenzji `148919350` podgląd zbioru pokazuje `76561198826729330`, a plik Parquet przechowuje `76561198826729328`. `recommendationid` w zaobserwowanym zakresie, rzędu 1,5e8, mieści się w 53-bitowej mantysie i pozostaje dokładną liczbą całkowitą; typ kolumny pozostaje nieodpowiedni dla identyfikatora 64-bitowego. `COUNT(DISTINCT author_steamid)` liczy zachowane wartości float. Zgodność `COUNT(*)` z deklarowaną liczbą 113 883 717 identyfikuje zrzut i nie rozstrzyga o braku zaokrągleń w poszczególnych polach.

## Uruchomienie

```bash
git clone https://github.com/<user>/game-lifecycle-decay-analytics.git
cd game-lifecycle-decay-analytics
pip install duckdb polars scikit-learn pyarrow huggingface_hub pandas jupyter
```
