# goit-rdb-hw-07: часові ряди та JSONB у задачах ML

## Дані
- **Time-series:** NYC TLC Yellow Taxi Trip Records, офіційні Parquet: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
  - файли `yellow_tripdata_2024-02.parquet` і `yellow_tripdata_2024-03.parquet` з `https://d37ci6vzurychx.cloudfront.net/trip-data/`, повністю;
  - notebook завантажує їх сам у `data/` (кеш), тому в репозиторій Parquet не додається;
  - два місяці, а не один: `trips_lag_30d` потребує 30 днів історії, а з одним місяцем фільтр `trips_lag_30d IS NOT NULL` залишив би 1 день на зону;
  - **stub-schema fallback:** якщо TLC недоступний, автоматично використовується stub із завдання (60 днів × 5 зон). Режим друкується в розділі 1.2 (`Режим: STUB` або `Режим: TLC ...`).
- **JSONB:** синтетичний детермінований event-log `t7_hw_events`: 50 000 подій, 5 типів (`page_view`, `click`, `add_to_cart`, `checkout`, `recommendation_shown`) з різними схемами payload.

## Time-series features (`t7_hw_taxi_daily` → `v_t7_hw_taxi_features`)
- `t7_hw_taxi_daily`: денний ряд `(pickup_zone_id, ts_day)` з **calendar spine**. Кожна зона має кожен день періоду; день без поїздок — `trip_count = 0`, `avg_fare = NULL`. Без spine `LAG(…, 7)` бере «7 рядків тому», а не «7 днів тому» (демонстрація в розділі 2.2).
- Ознаки, усі з `PARTITION BY pickup_zone_id`:
  - lag: `trips_lag_1d`, `trips_lag_7d`, `trips_lag_30d`;
  - rolling: `rolling_avg_fare_7d` (`RANGE INTERVAL '6 days' PRECEDING`), `rolling_std_fare_30d` (`'29 days'`);
  - expanding: `cumulative_trips`, `expanding_avg_trips`.
- `t7_hw_taxi_trips_typed`: production-style `TIMESTAMPTZ` (`AT TIME ZONE 'America/New_York'`) з перевіркою поїздок через перехід на літній час.

## JSONB-частина
- Запити: `@>`, `?`, `?|`, `jsonb_path_query`, `jsonb_path_exists`.
- `EXPLAIN (ANALYZE, BUFFERS)` до і після `GIN (payload jsonb_path_ops)`, до і після expression B-tree `((payload ->> 'device'))`. Додатково — `jsonb_ops` для `?` і виміряна ціна індексів на запис.
- `t7_hw_checkouts_norm`: нормалізована таблиця checkout-ів; один агрегат у JSONB- і normalized-версіях.
- Bonus: MongoDB aggregation pipeline як pseudocode (markdown).

## Запуск
1. Відкрити `hw7_timeseries_jsonb.ipynb` у Google Colab.
2. **Runtime → Restart session and run all.** Завантаження двох місяців і `COPY` ~6 млн рядків займає кілька хвилин. Усі об'єкти перестворюються через `DROP ... IF EXISTS`.
3. Зберегти notebook з output (**File → Download .ipynb**).

Примітки до середовища (враховані в першій клітинці):
- pgserver на Python 3.13+ ставиться з cp312-wheel з тегом `abi3`;
- SQLAlchemy URI з `postgresql+psycopg2://`;
- службові SQL-скрипти виконуються через «сирий» курсор psycopg2, бо SQLAlchemy 2.1 інакше сприймає `%` (модуль у seed) як плейсхолдер.
