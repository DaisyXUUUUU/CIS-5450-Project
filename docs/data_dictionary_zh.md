# 数据字典：`trips_clean_v1.parquet`

English version: [data_dictionary.md](data_dictionary.md)

每一行是一条清洗后的刷卡黄色出租车行程，由 `notebooks/01_data_pipeline.ipynb` 生成。共 796,989 行、48 列，覆盖 2026 年 4 月到 6 月。
写代码时请读 Parquet 文件；CSV 只用来打开查看，因为它会丢掉日期和整数类型。

## 不能当模型特征的列

| 列名 | 原因 |
| --- | --- |
| `tip_amount` | 就是要预测的结果。 |
| `total_amount` | 包含小费。 |
| `tip_pct` | 结果的比例形式。 |
| `high_tip`、`high_tip_20`、`high_tip_30` | 目标变量。 |
| `payment_type` | 筛选后全是 1（信用卡），没有信息。 |
| `tpep_dropoff_datetime` | 只用来计算 `duration_min`。 |
| `split`、`source_month`、`pickup_hour_ts` | 记录用的辅助列。 |

## 全部列

### TLC 原始字段

| 列名 | 类型 | 含义 |
| --- | --- | --- |
| `VendorID` | 整数 | 计价器供应商（1 = Creative Mobile，2 = Curb，6 = Myle，7 = Helix）。类别变量。 |
| `tpep_pickup_datetime` | 日期时间 | 上车时间，纽约本地时间。小时、星期、周末、深夜都从它算出。 |
| `tpep_dropoff_datetime` | 日期时间 | 下车时间。不当特征。 |
| `passenger_count` | 小数 | 乘客数。原来的 0 已改为缺失（3,047 行，0.38%），在建模 Pipeline 里只用训练集填补。 |
| `trip_distance` | 小数 | 计价器记录的里程（英里），0 < 里程 ≤ 100。 |
| `RatecodeID` | 整数 | 费率类型：1 标准，2 JFK，3 Newark，4 Nassau/Westchester，5 议价，6 拼车。99（未知）已删除。类别变量。 |
| `store_and_fwd_flag` | 文本 | Y 表示记录先存在车上、之后才上传。 |
| `PULocationID`、`DOLocationID` | 整数 | TLC 上车、下车区域编号（清洗后为 1 到 263）。JFK = 132，LaGuardia = 138，Newark = 1。 |
| `fare_amount` | 小数 | 按时间和里程计的车费，> 0。是 `tip_pct` 的分母。 |
| `extra` | 小数 | 杂项附加费（高峰、夜间等）。 |
| `mta_tax` | 小数 | MTA 税。 |
| `tolls_amount` | 小数 | 过路费。 |
| `improvement_surcharge` | 小数 | 改进附加费。 |
| `congestion_surcharge` | 小数 | 纽约州拥堵附加费。 |
| `Airport_fee` | 小数 | 在 LaGuardia 和 JFK 上车收取的机场费。 |
| `cbd_congestion_fee` | 小数 | 曼哈顿中央商务区（CBD）拥堵收费。 |

### 清洗和 JOIN 生成的列

| 列名 | 类型 | 含义 |
| --- | --- | --- |
| `source_month` | 文本 | 行程是从哪个月的文件里抽出来的。 |
| `duration_min` | 小数 | 行程时长（分钟）= 下车时间 − 上车时间，范围 1 到 180（Table 1 第 3 项）。 |
| `PU_Borough`、`DO_Borough` | 文本 | 上车、下车所在的区（Manhattan、Queens 等），来自 zone 对照表（第 12–13 项）。 |
| `PU_Zone`、`DO_Zone` | 文本 | 上车、下车的街区名称。 |
| `PU_service_zone`、`DO_service_zone` | 文本 | 服务区类型：Yellow Zone、Boro Zone、Airports 或 EWR。 |
| `pickup_hour_ts` | 日期时间 | 上车时间向下取整到整点，用来 JOIN 天气。不当特征。 |
| `temp_c` | 小数 | 该小时的气温（°C），来自 Open-Meteo（第 20 项）。 |
| `precip_mm` | 小数 | 截至 `pickup_hour_ts` 的那一小时累计降水量（mm），这一小时在上车前已经结束。 |
| `is_rain` | 整数 | `precip_mm ≥ 0.1` 时为 1（第 21 项，H2 分组）。全部行程中 16.2% 为 1。 |

### 衍生特征（proposal Table 1）

| 列名 | 类型 | 含义 |
| --- | --- | --- |
| `pickup_hour` | 整数 | 上车小时，0 到 23（第 1 项）。 |
| `pickup_dow` | 整数 | 星期几，0 = 周一 … 6 = 周日（第 2 项）。 |
| `is_weekend` | 整数 | 周六或周日上车为 1（第 14 项）。 |
| `is_late_night` | 整数 | 0:00 到 5:59 上车为 1（第 15 项，H1 分组）。 |
| `avg_speed_mph` | 小数 | 平均车速（英里/小时）= 60 × `trip_distance` / `duration_min`（第 16 项）。超过 80 的值可疑（只有 3 行），如何截断或删除只用训练集决定。 |
| `is_airport` | 整数 | 上车或下车在 JFK 或 LaGuardia 为 1（第 17 项，H3 分组）。 |
| `is_newark` | 整数 | 上车或下车在 Newark 为 1。Newark 收费规则不同，H3 中排除。 |
| `same_borough` | 整数 | 上下车在同一个区为 1（第 18 项）。 |
| `has_cbd_fee` | 整数 | 收了 CBD 拥堵费（`cbd_congestion_fee > 0`）为 1（第 19 项）。 |
| `is_short_trip` | 整数 | 里程 ≤ 2 英里为 1（H4 分组）。 |

### 目标变量和切分

| 列名 | 类型 | 含义 |
| --- | --- | --- |
| `tip_pct` | 小数 | 小费比例 = `tip_amount / fare_amount`，0 < 比例 ≤ 1。不当特征。 |
| `high_tip` | 整数 | `tip_pct ≥ 25%` 为 1。**主目标变量**，整体约 63% 为 1。 |
| `high_tip_20`、`high_tip_30` | 整数 | 阈值换成 20% 和 30% 的版本，只用于敏感性分析。 |
| `pickup_date` | 日期 | 上车日期。按天聚类的标准误和 day-block bootstrap 都以它为单位。 |
| `split` | 文本 | `train`（4/1–5/15，395,144 行）、`val`（5/16–31，127,349 行）、`test`（6 月，274,496 行）。 |
