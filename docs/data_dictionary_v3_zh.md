# 数据字典：最终分析表（56 列）

本表由 `notebooks/01_data_part_presentation.ipynb` 生成，共 796,989 行 × 56 列，输出文件为 `trips_clean.parquet`。
- 金额单位：美元
- 时间：纽约本地时间
- 列顺序：与 notebook 输出一致

**特征建议**
- ✅ 可以直接当特征
- ⚠️ 可以当特征，但要先用**训练集**做处理（填补、截断）或者在几列里选一列
- ❌ 不能当特征（结果变量、目标变量、常数或记账用的列）

**防泄漏原则**：模型的时间点是“行程已结束、乘客还没付款”。`tip_amount`、`total_amount`，以及所有由它们算出、又含有小费信息的列，一律不能当特征。

| # | 列名 | dtype | 含义 | 来源 | 特征建议 |
| --- | --- | --- | --- | --- | --- |
| 1 | `VendorID` | int32 | 计价器供应商：1 = Creative Mobile（约 18%），2 = Curb | TLC | ✅ 类别变量 |
| 2 | `tpep_pickup_datetime` | datetime64[us] | 上车时间 | TLC | ❌ 用派生出的时间列（#33–36、#49–50） |
| 3 | `tpep_dropoff_datetime` | datetime64[us] | 下车时间 | TLC | ❌ 用 `duration_min` |
| 4 | `passenger_count` | float64 | 乘客数；原值为 0 的 3,047 行已设为缺失 | TLC | ⚠️ 用训练集填补 |
| 5 | `trip_distance` | float64 | 行程距离（英里），范围 0–100 | TLC | ✅ |
| 6 | `RatecodeID` | int64 | 费率：1 标准，2 JFK，3 Newark，4 Nassau/Westchester，5 议价 | TLC | ✅ 类别变量 |
| 7 | `store_and_fwd_flag` | object | Y 表示行程先存在车上、之后才上传 | TLC | ✅（信息量很少） |
| 8 | `PULocationID` | int32 | 上车区域编号（1–263） | TLC | ✅ 类别变量 |
| 9 | `DOLocationID` | int32 | 下车区域编号（1–263） | TLC | ✅ 类别变量 |
| 10 | `payment_type` | int64 | 支付方式，全部是 1（信用卡） | TLC | ❌ 常数 |
| 11 | `fare_amount` | float64 | 按时间和里程计价的车费，> 0；是 `tip_pct` 的分母 | TLC | ✅ |
| 12 | `extra` | float64 | 杂项附加费（高峰 $2.50、夜间 $1）；Vendor 1 还把拥堵费、CBD 费、机场费也算了进来，两个 vendor 记账口径不同 | TLC | ❌ 改用 `other_charges` |
| 13 | `mta_tax` | float64 | MTA 税，几乎都是 $0.50 | TLC | ⚠️ 已包含在 `other_charges` 里，不要重复使用 |
| 14 | `tip_amount` | float64 | 信用卡小费，> 0 且不超过车费 | TLC | ❌ 结果变量 |
| 15 | `tolls_amount` | float64 | 过路费 | TLC | ⚠️ 已包含在 `other_charges` 里，不要重复使用 |
| 16 | `improvement_surcharge` | float64 | 改进附加费，几乎都是 $1 | TLC | ⚠️ 已包含在 `other_charges` 里，不要重复使用 |
| 17 | `total_amount` | float64 | 总金额，**包含小费** | TLC | ❌ 含结果 |
| 18 | `congestion_surcharge` | float64 | 纽约州拥堵费，$0 或 $2.50 | TLC | ⚠️ 已包含在 `other_charges` 里，不要重复使用 |
| 19 | `Airport_fee` | float64 | JFK / LGA 上车机场费 | TLC | ⚠️ 已包含在 `other_charges` 里，不要重复使用 |
| 20 | `cbd_congestion_fee` | float64 | 曼哈顿 CBD 拥堵费，$0 或 $0.75 | TLC | ⚠️ 用 `has_cbd_fee` 或 `other_charges` 代替 |
| 21 | `source_month` | object | 行程是从哪个月的文件抽出来的 | 派生 | ❌ 记账用 |
| 22 | `duration_min` | float64 | 下车时间 − 上车时间（分钟），范围 1–180 | 派生 | ✅ |
| 23 | `PU_Borough` | object | 上车所在的区 | zone 表 | ✅ 类别变量 |
| 24 | `PU_Zone` | object | 上车区域名 | zone 表 | ⚠️ 和 `PULocationID` 重复，二选一 |
| 25 | `PU_service_zone` | object | 上车服务区：Yellow Zone / Airports / Boro Zone / EWR | zone 表 | ✅ 类别变量 |
| 26 | `DO_Borough` | object | 下车所在的区 | zone 表 | ✅ 类别变量 |
| 27 | `DO_Zone` | object | 下车区域名 | zone 表 | ⚠️ 和 `DOLocationID` 重复，二选一 |
| 28 | `DO_service_zone` | object | 下车服务区（取值同 #25） | zone 表 | ✅ 类别变量 |
| 29 | `pickup_hour_ts` | datetime64[us] | 上车时间向下取整到整点，用作天气 JOIN 的键 | 派生 | ❌ JOIN 键 |
| 30 | `temp_c` | float64 | 上车所在小时的气温（°C） | Open-Meteo | ✅ |
| 31 | `precip_mm` | float64 | 截至 `pickup_hour_ts` 那一小时的降水量（mm） | Open-Meteo | ✅ |
| 32 | `is_rain` | Int8 | `precip_mm ≥ 0.1`（约 16.2% 的行程） | 派生 | ✅（和 #31 二选一或一起用） |
| 33 | `pickup_hour` | int8 | 上车小时（0–23） | 派生 | ✅ |
| 34 | `pickup_dow` | int8 | 星期几，0 = 周一 … 6 = 周日 | 派生 | ✅ |
| 35 | `is_weekend` | int8 | 周六或周日上车（约 25.7%） | 派生 | ✅ |
| 36 | `is_late_night` | int8 | 0:00–5:59 上车（约 6.5%），是 `is_night` 的子集 | 派生 | ✅ |
| 37 | `avg_speed_mph` | float64 | 平均车速 = 60 × 距离 ÷ 时长；> 80 mph 的只有 3 行 | 派生 | ⚠️ 截断阈值用训练集定 |
| 38 | `is_airport` | int8 | 上车或下车在 JFK（132）/ LGA（138）（约 9.4%） | 派生 | ✅ |
| 39 | `is_newark` | int8 | 上车或下车在 Newark（1）（约 0.2%） | 派生 | ✅ |
| 40 | `same_borough` | int8 | 上车和下车在同一个区（约 87.8%） | 派生 | ✅ |
| 41 | `has_cbd_fee` | int8 | `cbd_congestion_fee > 0`（约 76.1%） | 派生 | ✅ |
| 42 | `is_short_trip` | int8 | 距离 ≤ 2 英里（约 59.5%） | 派生 | ✅ |
| 43 | `tip_pct` | float64 | 小费 ÷ 车费，≤ 1 | 派生 | ❌ 结果变量 |
| 44 | `high_tip_20` | int8 | 小费 ÷ 车费 ≥ 20%（80.8%） | 派生 | ❌ 目标变量（敏感性分析） |
| 45 | `high_tip_25` | int8 | 小费 ÷ 车费 ≥ 25%，即 proposal 里的目标（63.1%；train / val / test 分别为 63.3 / 62.4 / 63.0%） | 派生 | ❌ 目标变量 |
| 46 | `high_tip_30` | int8 | 小费 ÷ 车费 ≥ 30%（34.3%） | 派生 | ❌ 目标变量（敏感性分析） |
| 47 | `pickup_date` | datetime64[us] | 上车日期，用作聚类标准误和 day-block bootstrap 的单位 | 派生 | ❌ 记账用 |
| 48 | `split` | object | train = 4/1–5/15，val = 5/16–31，test = 6 月 | 派生 | ❌ 记账用 |
| 49 | `is_rush_hour` | int8 | 周一至周五 16:00–19:59 上车（$2.50 高峰附加费时段）；节假日也算，因为计价器照收（约 21.1%） | 派生 | ✅ |
| 50 | `is_night` | int8 | 20:00–5:59 上车（$1 夜间附加费时段）（约 28.1%） | 派生 | ✅ |
| 51 | `pre_tip_bill` | float64 | `total_amount − tip_amount` = `fare_amount + other_charges`，即付小费前应付的全部金额 | 派生 | ✅（和 `fare_amount + other_charges` 二选一，不要三列一起用） |
| 52 | `tip_pct_bill` | float64 | 小费 ÷ `pre_tip_bill` | 派生 | ❌ 结果变量 |
| 53 | `high_tip_bill_20` | int8 | 小费 ÷ `pre_tip_bill` ≥ 20%（63.5%） | 派生 | ❌ 目标变量 |
| 54 | `high_tip_bill_25` | int8 | 小费 ÷ `pre_tip_bill` ≥ 25%（9.4%） | 派生 | ❌ 目标变量 |
| 55 | `high_tip_bill_30` | int8 | 小费 ÷ `pre_tip_bill` ≥ 30%（2.6%） | 派生 | ❌ 目标变量 |
| 56 | `other_charges` | float64 | `pre_tip_bill − fare_amount`：车费以外的所有费用（附加费、税、各项 fee、过路费），两个 vendor 口径一致 | 派生 | ✅（不要和 #12、#13、#15、#16、#18–20 一起用） |

## 补充说明

- **目标变量的 ≥ 判断**：所有 `high_tip_*` 和 `high_tip_bill_*` 都用 ≥，并且换算成整数“分”来比较（tip × 100 ≥ 阈值 × 分母），这样正好等于阈值的小费不会因为浮点误差被误判。
- **`high_tip_bill_20` 要注意**：`tip_pct_bill` 的中位数正好是 20%，约 47% 的行程小费恰好等于账单的 20%（四舍五入到分），也就是直接点了预设按钮。用 ≥ 判断时，这些行程都算 1，所以 `high_tip_bill_20`（63.5%）主要反映“有没有按预设付”。另外，预设小费四舍五入到分时如果往下舍（例如 $2.674 记成 $2.67），会落到 20% 以下，被算成 0。`_25`（9.4%）和 `_30`（2.6%）离预设较远，不受这两点影响，但正例很少，类别不平衡比较明显。
- **金额类特征怎么选**（三选一，避免重复计算）：
  1. 只用 `pre_tip_bill`
  2. 用 `fare_amount` + `other_charges`
  3. 用 `fare_amount` + 各项费用列（不推荐，因为 Vendor 1 的 `extra` 口径不同）
- **建模时才做的处理**：填补、截断、one-hot、标准化都只在训练集上拟合，然后应用到 val 和 test。
- **占比数字的来源**：所有百分比都是根据 2026-11-19 在 Colab 运行得到的 `trips_clean.parquet`（796,989 × 56）核对的。核对结果：split 行数为 395,144 / 127,349 / 274,496；`other_charges` 没有负值；`fare + other_charges = pre_tip_bill` 全部成立。
