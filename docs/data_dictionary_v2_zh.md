# 数据字典：`trips_clean_v2.parquet`

796,989 行 × 54 列，由 `notebooks/01b_add_features_v2.ipynb` 读 v1 加 6 列生成（前 48 列 = v1，未改）。金额单位为美元，时间为纽约本地时间。

分母：`tip_pct` 用 **fare**（`fare_amount`，只含车费）；`tip_pct_bill` 用**账单** = `total_amount − tip_amount`（小费前应付的全部金额）。用哪套当主目标待和 Ricky 确认。

特征一栏：✅ 可以 ⚠️ 可以但要处理 ❌ 不能

| # | 列名 | 类型 | 含义 / 公式 | 来源 | 特征 |
| --- | --- | --- | --- | --- | --- |
| 1 | `VendorID` | int | 计价器供应商，1 = Creative Mobile（18%），2 = Curb | TLC | ✅ 类别 |
| 2 | `tpep_pickup_datetime` | datetime | 上车时间 | TLC | ❌ 用派生列 |
| 3 | `tpep_dropoff_datetime` | datetime | 下车时间 | TLC | ❌ |
| 4 | `passenger_count` | float | 乘客数，0 已改为缺失（3,047 行） | TLC | ⚠️ 用训练集填补 |
| 5 | `trip_distance` | float | 里程（英里），0–100 | TLC | ✅ |
| 6 | `RatecodeID` | int | 费率：1 标准，2 JFK，3 Newark，4 Nassau/Westchester，5 议价 | TLC | ✅ 类别 |
| 7 | `store_and_fwd_flag` | str | Y = 先存车上再上传 | TLC | ✅ 信息很少 |
| 8 | `PULocationID` | int | 上车区域编号 1–263 | TLC | ✅ 类别 |
| 9 | `DOLocationID` | int | 下车区域编号 1–263 | TLC | ✅ 类别 |
| 10 | `payment_type` | int | 全是 1（信用卡） | TLC | ❌ 常数 |
| 11 | `fare_amount` | float | 按时间和里程计的车费，> 0；`tip_pct` 的分母 | TLC | ✅ |
| 12 | `extra` | float | 杂项附加费（高峰 $2.50、夜间 $1）；Vendor 1 还重复包含拥堵费、CBD 费、机场费 | TLC | ⚠️ Vendor 1 先减去这三项 |
| 13 | `mta_tax` | float | MTA 税，几乎都是 $0.50 | TLC | ✅ |
| 14 | `tip_amount` | float | 信用卡小费，> 0 且 ≤ fare | TLC | ❌ 结果 |
| 15 | `tolls_amount` | float | 过路费 | TLC | ✅ |
| 16 | `improvement_surcharge` | float | 改进附加费，几乎都是 $1 | TLC | ✅ |
| 17 | `total_amount` | float | 总额，**含小费** | TLC | ❌ |
| 18 | `congestion_surcharge` | float | 纽约州拥堵费，$0 或 $2.50 | TLC | ✅ |
| 19 | `Airport_fee` | float | JFK / LGA 上车机场费 | TLC | ✅ |
| 20 | `cbd_congestion_fee` | float | 曼哈顿 CBD 拥堵费，$0 或 $0.75 | TLC | ✅ 与 #41 二选一 |
| 21 | `source_month` | str | 抽样来源月份 | 派生 | ❌ |
| 22 | `duration_min` | float | 下车 − 上车（分钟），1–180 | 派生 | ✅ |
| 23 | `PU_Borough` | str | 上车所在区 | zone 表 | ✅ 类别 |
| 24 | `PU_Zone` | str | 上车街区名 | zone 表 | ⚠️ 与 #8 重复 |
| 25 | `PU_service_zone` | str | Yellow Zone / Airports / Boro Zone / EWR | zone 表 | ✅ 类别 |
| 26 | `DO_Borough` | str | 下车所在区 | zone 表 | ✅ 类别 |
| 27 | `DO_Zone` | str | 下车街区名 | zone 表 | ⚠️ 与 #9 重复 |
| 28 | `DO_service_zone` | str | 同 #25，下车 | zone 表 | ✅ 类别 |
| 29 | `pickup_hour_ts` | datetime | 上车时间取整到小时，天气 JOIN 键 | 派生 | ❌ |
| 30 | `temp_c` | float | 该小时气温（°C） | Open-Meteo | ✅ |
| 31 | `precip_mm` | float | 上车前那一小时降水（mm） | Open-Meteo | ✅ |
| 32 | `is_rain` | int | `precip_mm ≥ 0.1`（16.2%） | 派生 | ✅ |
| 33 | `pickup_hour` | int | 上车小时 0–23 | 派生 | ✅ |
| 34 | `pickup_dow` | int | 星期，0 = 周一 … 6 = 周日 | 派生 | ✅ |
| 35 | `is_weekend` | int | 周六或周日（25.7%） | 派生 | ✅ |
| 36 | `is_late_night` | int | 上车 0:00–5:59（6.5%），是 #50 的子集 | 派生 | ✅ |
| 37 | `avg_speed_mph` | float | 60 × 里程 / 时长；> 80 的有 3 行 | 派生 | ⚠️ 用训练集定截断 |
| 38 | `is_airport` | int | 上车或下车在 JFK（132）/ LGA（138）（9.4%） | 派生 | ✅ |
| 39 | `is_newark` | int | 上车或下车在 Newark（1）（0.2%） | 派生 | ✅ |
| 40 | `same_borough` | int | 上下车同一个区（87.8%） | 派生 | ✅ |
| 41 | `has_cbd_fee` | int | `cbd_congestion_fee > 0`（76.1%） | 派生 | ✅ |
| 42 | `is_short_trip` | int | 里程 ≤ 2 英里（59.5%） | 派生 | ✅ |
| 43 | `tip_pct` | float | `tip_amount / fare_amount`，≤ 1 | 派生 | ❌ |
| 44 | `high_tip` | int | `tip_pct ≥ 0.25`，**原定主目标**（约 63%） | 派生 | ❌ |
| 45 | `high_tip_20` | int | `tip_pct ≥ 0.20`，敏感性分析 | 派生 | ❌ |
| 46 | `high_tip_30` | int | `tip_pct ≥ 0.30`，敏感性分析 | 派生 | ❌ |
| 47 | `pickup_date` | datetime | 上车日期，聚类标准误 / bootstrap 单位 | 派生 | ❌ |
| 48 | `split` | str | train 4/1–5/15，val 5/16–31，test 6 月 | 派生 | ❌ |
| 49 | `is_rush_hour` | int | 周一至周五上车 16:00–19:59，节假日也算（21.1%） | v2 派生 | ✅ |
| 50 | `is_night` | int | 上车 20:00–5:59（28.1%） | v2 派生 | ✅ |
| 51 | `tip_pct_bill` | float | `tip_amount / (total_amount − tip_amount)`，中位数 0.20 | v2 派生 | ❌ |
| 52 | `high_tip_bill` | int | 小费 > 账单 20%（四舍五入到分）+ 1 分，即 `tip > round(0.20 × bill, 2) + 0.015`（约 16.4%） | v2 派生 | ❌ |
| 53 | `high_tip_bill_25` | int | 同上，25%（约 4.8%） | v2 派生 | ❌ |
| 54 | `high_tip_bill_30` | int | 同上，30%（约 1.0%） | v2 派生 | ❌ |

备注：账单（`total_amount − tip_amount`）本身付款前就已知，可以在建模时算出来当特征，但算完要丢掉 `total_amount`。
