# 数据字典：`trips_clean_v2.parquet`（中文详细版）

- **文件位置**：`MyDrive/CIS5450_project/data/clean/trips_clean_v2.parquet`（v1 `trips_clean_v1.parquet` 保留在同一目录，不改）
- **生成方式**：`notebooks/01b_add_features_v2.ipynb` 只读 v1，在后面加新列，**不重新下载、不重新抽样、不重新清洗**，没有任何随机数，每次运行结果完全相同。
- **规模**：796,989 行 × 54 列 = v1 的 48 列（数值一个都没改）+ 6 个新列。
  - 注：2026-11-16 第一次运行得到的是 52 列（当时只有 4 个新列）。之后加了 `high_tip_bill_25`、`high_tip_bill_30`，并改了 `is_rush_hour` 的节假日处理（见第 5 节），需要用 PR 里最新的 notebook 重新跑一次，得到 54 列。
- **时间范围**：上车时间 2026-04-01 到 2026-06-30，纽约本地时间（不带时区）。
- **一行 = 一条**被抽样、清洗后的**信用卡付费**、**小费 > 0** 的黄色出租车行程。
- 写代码请读 Parquet；CSV 会丢掉日期和整数类型。

English short version: [data_dictionary.md](data_dictionary.md)

---

## 1. 一眼看懂：哪些列能用、哪些绝对不能用

### 1.1 绝对不能当模型特征（会泄露答案 / 没有信息）

模型的时间点是**行程结束、付款之前**。凡是用到小费、或者在付款之后才知道的东西，一律不能进特征。

| 列名 | 为什么不能用 |
| --- | --- |
| `tip_amount` | 就是要预测的结果本身。 |
| `total_amount` | = 账单 + 小费，**包含小费**。哪怕只用它和别的列做减法，也能还原出小费。 |
| `tip_pct` | 小费 ÷ fare，结果的比例形式。 |
| `tip_pct_bill` | 小费 ÷ 账单，结果的比例形式。 |
| `high_tip`、`high_tip_20`、`high_tip_30` | fare 分母下的目标标签。 |
| `high_tip_bill`、`high_tip_bill_25`、`high_tip_bill_30` | 账单分母下的目标标签。 |
| `payment_type` | 筛选后全是 1（信用卡），没有任何信息。 |
| `tpep_dropoff_datetime` | 只用来算 `duration_min`。时长本身可以用。 |
| `split` | 训练 / 验证 / 测试的标签，是记录用的。 |
| `source_month` | 记录这一行来自哪个月文件，和 split 几乎一一对应。 |
| `pickup_hour_ts` | 天气 JOIN 用的键（整点时间戳）。 |
| `pickup_date` | 聚类标准误和 day-block bootstrap 的单位。直接当特征等于让模型记住"哪一天"，测试集（6 月）的日期在训练里从没出现过。 |
| `tpep_pickup_datetime` | 原始时间戳不要直接喂给模型，用由它派生的 `pickup_hour`、`pickup_dow`、`is_weekend` 等。 |

> **"账单"本身可以吗？** 账单 = `total_amount − tip_amount`，在行程结束、付款前就显示在屏幕上，所以**作为一个数值特征是可以的**（各项费用的和也行）。但不能**在特征里同时放 `total_amount`**，也不能把它和小费放在一起算比例。v2 没有单独存账单这一列，需要的话在建模 pipeline 里用 `total_amount − tip_amount` 算出来，然后**丢掉 `total_amount`**。

### 1.2 可以用，但要小心

| 列名 | 要注意什么 |
| --- | --- |
| `extra` | **两个 vendor 的口径不同。** Vendor 1（145,315 行，18.2%）的 `extra` 里已经**重复包含**了拥堵费 2.50、CBD 0.75 和机场费，这几项在各自的列里又出现一次；Vendor 2 的 `extra` 只含高峰 / 夜间等附加费。用之前要统一：Vendor 1 用 `extra − congestion_surcharge − cbd_congestion_fee − Airport_fee`（修正后的取值主要是 0 / 1 / 2.5 / 5 / 6 / 7.5，和 Vendor 2 一致），或者干脆不用 `extra`，改用 `is_rush_hour`、`is_night`。 |
| `fare_amount`、各项费用列 | 本身是原始值，可以当特征；但 `fare_amount` 也是 `tip_pct` 的分母，在解释"fare 越高小费比例越低"时要记得这有一部分是分母造成的。 |
| `passenger_count` | 有 3,047 行缺失（0.38%，原始 0 被改成缺失）。只能**用训练集**拟合填补。 |
| `avg_speed_mph` | 有 3 行 > 80 mph（最大 217），截断阈值只能用训练集决定。 |
| `VendorID`、`RatecodeID`、`PULocationID`、`DOLocationID` | 是**类别**，不是数值，要 one-hot 或 target encoding（target encoding 只能在训练集里做，并且要做交叉折叠）。 |
| `PU_Zone`、`DO_Zone` | 类别很多（约 260 个），和 `PULocationID` / `DOLocationID` 是同一个信息，两者选一个。 |
| `is_late_night` 和 `is_night` | `is_late_night`（0–5 点）是 `is_night`（20–5 点）的子集，一起放进线性模型要注意共线性。 |
| `is_rush_hour` 和 `pickup_hour` + `pickup_dow` | `is_rush_hour` 完全由后两者决定，树模型里是冗余的，线性模型里会共线。 |
| `has_cbd_fee` 和 `cbd_congestion_fee` | 同一个信息，选一个。 |
| `is_rain` 和 `precip_mm` | 同一个信息的二值版和连续版。 |

---

## 2. 分母到底是什么（fare vs 账单）

这是整个项目最容易搞混的地方。v2 里**两套都保留**，方便对比，等 Ricky 和组里确认用哪一套。

| | fare 分母（proposal 原定，v1 起就有） | 账单分母（v2 新加，待定） |
| --- | --- | --- |
| 分母 | `fare_amount`：只含按时间和里程计的车费，**不含**任何税、附加费、过路费 | 账单 = `total_amount − tip_amount`：乘客在小费前要付的全部金额（车费 + 所有附加费 + 税 + 过路费） |
| 比例列 | `tip_pct = tip_amount / fare_amount` | `tip_pct_bill = tip_amount / (total_amount − tip_amount)` |
| 主目标 | `high_tip`：`tip_pct ≥ 0.25` | `high_tip_bill`：小费比账单的 20% **多出一分钱以上** |
| 敏感性分析 | `high_tip_20`（≥ 20%）、`high_tip_30`（≥ 30%） | `high_tip_bill_25`、`high_tip_bill_30`（同样的"多一分钱"规则） |
| 中位数 | 约 27.1% | **正好 20.0%** |
| 主目标正类比例（train / val / test） | 63.3% / 62.4% / 63.0% | 16.3% / 16.5% / 16.6% |

**为什么考虑换成账单：** 乘客在屏幕上看到的小费选项是按账单（含附加费）算的。v1 数据里，约 49% 的行程小费正好是账单的 20%（误差 1 分以内）。用 fare 做分母时，晚上 $1、高峰 $2.50 的附加费会让同样"按 20% 点选"的小费看起来比例更高，于是出现假的"凌晨 2–4 点、工作日 16–19 点小费高"的峰值。换成账单后这些峰值消失（每小时中位数都正好是 20%）。

**为什么"多一分钱"，而不是 `> 0.20`：** 小费会四舍五入到分，按 20% 预设点选的人，`tip / bill` 会因为舍入落在 0.1997 或 0.2003 这种位置，跟乘客的行为无关。所以定义为：

```
bill          = round(total_amount − tip_amount, 2)
high_tip_bill = tip_amount > round(0.20 × bill, 2) + 0.015
```

即小费至少比"账单 20% 四舍五入到分"多 2 分钱才算 1，按预设 20% 给的一律算 0。朴素版 `tip_pct_bill > 0.20` 的正类比例约 19.2–19.5%（比上面高约 3 个百分点，多出来的基本都是舍入）。25% 和 30% 版本同理。

**规则 8（`tip / fare ≤ 100%`）不用改：** v1 清洗时删掉了 1,205 行小费超过 fare 的行程。所有保留行的账单都 ≥ fare（核对过，0 行例外），所以 `tip_pct_bill` 自动 ≤ 1（实际最大 0.896，0 行 > 1）。唯一的差别是那 1,205 行按账单算可能不到 100%，但它们本来就是极端值，在报告里写一句即可，不用重新清洗。

**换不换分母不影响行数**，只影响用哪一列当目标。另外 25% 这个阈值是按 fare 选的，如果换成账单，阈值也跟着换成 20%（已经这样定义了）。

---

## 3. 全部 54 列

"来源"一栏：**TLC 原始** = 从 TLC 月度 Parquet 原样读入，没改任何值；**派生** = 我们算出来的；**JOIN** = 从别的表拼过来的。金额单位都是美元，时间都是纽约本地时间。

### 3.1 TLC 原始字段（20 列）

| # | 列名 | 类型 | 单位 / 取值 | 来源 | 含义 | 能当特征？ |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `VendorID` | int32 | 1 或 2 | TLC 原始 | 计价器供应商。1 = Creative Mobile（145,315 行），2 = Curb（651,674 行）。 | 可以（类别）。 |
| 2 | `tpep_pickup_datetime` | datetime | 秒 | TLC 原始 | 计价器开始计费的时间（上车）。所有时间派生列都由它算出。TLC 附加费也按上车时间判断。 | 不直接用，用派生列。 |
| 3 | `tpep_dropoff_datetime` | datetime | 秒 | TLC 原始 | 计价器停止的时间（下车）。 | **不能**。 |
| 4 | `passenger_count` | float64 | 1–6，缺失 3,047 行 | TLC 原始（0 → 缺失） | 司机输入的乘客数。 | 可以，训练集填补。 |
| 5 | `trip_distance` | float64 | 英里，0.01–64.84 | TLC 原始 | 计价器记录的里程。清洗规则：0 < 里程 ≤ 100。 | 可以。 |
| 6 | `RatecodeID` | int64 | 1 标准 765,555；2 JFK 25,141；3 Newark 2,586；4 Nassau/Westchester 195；5 议价 3,512 | TLC 原始 | 行程结束时的费率。99（未知）已删除。 | 可以（类别）。 |
| 7 | `store_and_fwd_flag` | str | N 796,074；Y 915 | TLC 原始 | Y = 记录先存在车上，之后才上传。 | 可以，但几乎没有信息。 |
| 8 | `PULocationID` | int32 | 1–263 | TLC 原始 | 上车区域编号。JFK = 132，LaGuardia = 138，Newark = 1。 | 可以（类别）。 |
| 9 | `DOLocationID` | int32 | 1–263 | TLC 原始 | 下车区域编号。 | 可以（类别）。 |
| 10 | `payment_type` | int64 | 全是 1 | TLC 原始 | 付款方式，1 = 信用卡。只保留信用卡，因为现金小费不记录。 | **不能**（常数）。 |
| 11 | `fare_amount` | float64 | $3.00–$390.00，中位数 $13.50 | TLC 原始 | 按时间和里程计的车费，**不含**税、附加费、过路费和小费。**是 `tip_pct` 的分母。** 清洗规则：> 0。 | 可以。 |
| 12 | `extra` | float64 | −$4.75–$16.50 | TLC 原始 | 杂项附加费。Vendor 2：高峰 $2.50、夜间 $1.00 等；**Vendor 1 还重复包含了拥堵费、CBD 费、机场费**（见 1.2）。 | 小心，先按 vendor 统一。 |
| 13 | `mta_tax` | float64 | 几乎都是 $0.50 | TLC 原始 | MTA 税。 | 可以，但几乎是常数。 |
| 14 | `tip_amount` | float64 | $0.01–$99.98，中位数 $3.51 | TLC 原始 | 信用卡小费（现金小费不记录）。清洗规则：> 0、≤ fare。 | **不能**（结果）。 |
| 15 | `tolls_amount` | float64 | $0–$53 | TLC 原始 | 过路费。 | 可以。 |
| 16 | `improvement_surcharge` | float64 | 几乎都是 $1.00 | TLC 原始 | 改进附加费。 | 可以，但几乎是常数。 |
| 17 | `total_amount` | float64 | $5.72–$471.00 | TLC 原始 | 乘客付的总额，**包含小费**（不含现金小费）。账单 = `total_amount − tip_amount`。 | **不能**。 |
| 18 | `congestion_surcharge` | float64 | $0 或 $2.50 | TLC 原始 | 纽约州拥堵附加费（曼哈顿 96 街以南起止）。 | 可以。 |
| 19 | `Airport_fee` | float64 | $0–$2.00 | TLC 原始 | 在 LaGuardia / JFK 上车的机场费。 | 可以。 |
| 20 | `cbd_congestion_fee` | float64 | $0 或 $0.75 | TLC 原始 | 曼哈顿中央商务区（CBD）拥堵收费。 | 可以（和 `has_cbd_fee` 二选一）。 |

### 3.2 清洗和 JOIN 生成的列（13 列）

| # | 列名 | 类型 | 单位 / 取值 | 来源 / 公式 | 含义 | 能当特征？ |
| --- | --- | --- | --- | --- | --- | --- |
| 21 | `source_month` | str | `2026-04` / `2026-05` / `2026-06` | 派生 | 这一行是从哪个月的 TLC 文件里抽出来的。 | **不能**（记录用）。 |
| 22 | `duration_min` | float64 | 分钟，1–171.5 | 派生：下车时间 − 上车时间 | 行程时长。清洗规则：1–180 分钟（短距离 + 长时间的慢行程**没删**，可能是真的堵车）。 | 可以。 |
| 23 | `PU_Borough` | str | Manhattan、Queens 等 | JOIN：TLC zone 对照表（按 `PULocationID`） | 上车所在的区。 | 可以（类别）。 |
| 24 | `PU_Zone` | str | 约 260 个街区名 | JOIN：同上 | 上车街区名。 | 和 `PULocationID` 二选一。 |
| 25 | `PU_service_zone` | str | Yellow Zone 717,626；Airports 60,223；Boro Zone 19,137；EWR 3 | JOIN：同上 | 上车服务区类型。 | 可以（类别）。 |
| 26 | `DO_Borough` | str | 同上 | JOIN（按 `DOLocationID`） | 下车所在的区。 | 可以（类别）。 |
| 27 | `DO_Zone` | str | 同上 | JOIN | 下车街区名。 | 和 `DOLocationID` 二选一。 |
| 28 | `DO_service_zone` | str | 同上 | JOIN | 下车服务区类型。 | 可以（类别）。 |
| 29 | `pickup_hour_ts` | datetime | 整点 | 派生：上车时间向下取整到小时 | 天气 JOIN 的键。 | **不能**（记录用）。 |
| 30 | `temp_c` | float64 | °C，−1.6–35.8 | JOIN：Open-Meteo 历史天气 API，按 `pickup_hour_ts` | 该小时的气温。JOIN 覆盖率 100%。 | 可以。 |
| 31 | `precip_mm` | float64 | mm，0–12.8 | JOIN：Open-Meteo | **截至** `pickup_hour_ts` 的那一小时的累计降水，这一小时在上车前已经结束，所以不是"未来"信息。 | 可以。 |
| 32 | `is_rain` | Int8 | 0/1，16.2% 为 1 | 派生：`precip_mm ≥ 0.1` | 是否下雨（H2 分组）。 | 可以（和 `precip_mm` 二选一或都用）。 |
| 33 | `pickup_hour` | int8 | 0–23 | 派生：上车时间的小时 | 上车小时。 | 可以。 |

### 3.3 衍生特征（proposal Table 1，9 列）

| # | 列名 | 类型 | 取值 / 比例 | 公式 | 含义 | 能当特征？ |
| --- | --- | --- | --- | --- | --- | --- |
| 34 | `pickup_dow` | int8 | 0 = 周一 … 6 = 周日 | 上车时间的星期 | 星期几。 | 可以。 |
| 35 | `is_weekend` | int8 | 25.7% 为 1 | `pickup_dow ∈ {5, 6}` | 周六或周日上车。 | 可以。 |
| 36 | `is_late_night` | int8 | 6.5% 为 1 | 上车小时 0–5 | 深夜 0:00–5:59（H1 分组）。是 `is_night` 的子集。 | 可以。 |
| 37 | `avg_speed_mph` | float64 | 中位数 8.7，最大 217 | 60 × `trip_distance` / `duration_min` | 平均车速（英里 / 小时）。> 80 的有 3 行。 | 可以，截断阈值用训练集定。 |
| 38 | `is_airport` | int8 | 9.4% 为 1 | 上车或下车区域 ∈ {132, 138} | 上车或下车在 JFK / LaGuardia（H3 分组）。 | 可以。 |
| 39 | `is_newark` | int8 | 0.2% 为 1 | 上车或下车区域 = 1 | Newark 机场。收费规则不同，H3 里排除。 | 可以。 |
| 40 | `same_borough` | int8 | 87.8% 为 1 | `PU_Borough == DO_Borough` | 上下车在同一个区。 | 可以。 |
| 41 | `has_cbd_fee` | int8 | 76.1% 为 1 | `cbd_congestion_fee > 0` | 是否收了 CBD 拥堵费。 | 可以。 |
| 42 | `is_short_trip` | int8 | 59.5% 为 1 | 0 < `trip_distance` ≤ 2 | 短途（H4 分组）。 | 可以。 |

### 3.4 目标变量和切分（v1 已有，fare 分母，6 列）

| # | 列名 | 类型 | 取值 | 公式 | 含义 | 能当特征？ |
| --- | --- | --- | --- | --- | --- | --- |
| 43 | `tip_pct` | float64 | 0 < x ≤ 1，中位数 0.271 | `tip_amount / fare_amount` | 小费占**车费**的比例。 | **不能**。 |
| 44 | `high_tip` | int8 | 约 63% 为 1 | `tip_pct ≥ 0.25` | **proposal 原定的主目标**。 | **不能**。 |
| 45 | `high_tip_20` | int8 | | `tip_pct ≥ 0.20` | 敏感性分析。 | **不能**。 |
| 46 | `high_tip_30` | int8 | | `tip_pct ≥ 0.30` | 敏感性分析。 | **不能**。 |
| 47 | `pickup_date` | datetime | 2026-04-01 – 06-30 | 上车日期 | 聚类标准误和 day-block bootstrap 的单位。 | **不能**。 |
| 48 | `split` | str | train 395,144；val 127,349；test 274,496 | 按上车日期 | train = 4/1–5/15，val = 5/16–5/31，test = 6 月。 | **不能**。 |

### 3.5 v2 新加的列（6 列）

| # | 列名 | 类型 | 取值 / 比例 | 公式 | 含义 | 能当特征？ |
| --- | --- | --- | --- | --- | --- | --- |
| 49 | `is_rush_hour` | int8 | 约 21.1% 为 1 | 周一–周五 且 上车小时 16–19 | TLC $2.50 高峰附加费的时间窗（16:00–19:59，按上车时间）。**节假日不排除**，见第 5 节。 | 可以。 |
| 50 | `is_night` | int8 | 28.1% 为 1 | 上车小时 ≥ 20 或 ≤ 5 | TLC $1 夜间附加费的时间窗（20:00–5:59）。和 `is_rush_hour` 不会同时为 1。 | 可以。 |
| 51 | `tip_pct_bill` | float64 | 0 < x ≤ 0.896，中位数 0.200 | `tip_amount / (total_amount − tip_amount)` | 小费占**账单**的比例。 | **不能**。 |
| 52 | `high_tip_bill` | int8 | 约 16.4% 为 1 | `tip_amount > round(0.20 × bill, 2) + 0.015` | 账单分母下的主目标候选：小费比账单 20% 多出一分钱以上。 | **不能**。 |
| 53 | `high_tip_bill_25` | int8 | 约 4.8% 为 1 | `tip_amount > round(0.25 × bill, 2) + 0.015` | 敏感性分析。 | **不能**。 |
| 54 | `high_tip_bill_30` | int8 | 约 1.0% 为 1 | `tip_amount > round(0.30 × bill, 2) + 0.015` | 敏感性分析。 | **不能**。 |

（`bill = round(total_amount − tip_amount, 2)`；所有行 bill > 0，且 bill ≥ fare。）

---

## 4. 清洗规则回顾（v1 定的，v2 不改）

| 规则 | 删除行数 |
| --- | --- |
| 1. `tip_amount > 0` | 74,765 |
| 2. `fare_amount > 0` | 2 |
| 3. 上车时间在 2026-04-01 到 06-30 之间 | 见 `01_data_pipeline.ipynb` 清洗漏斗表 |
| 4. 时长 1–180 分钟 | 同上 |
| 5. 0 < 里程 ≤ 100 英里 | 同上 |
| 6. 上下车区域不是 264/265（未知） | 同上 |
| 7. `RatecodeID ≠ 99` | 同上 |
| 8. `tip_amount / fare_amount ≤ 1` | 1,205 |

抽样：每月从信用卡行程中按 `hash(file_row_number, 5450)` 取 30 万行（共 90 万），清洗后 796,989 行。

---

## 5. v2 运行结果核对（2026-11-16，Daisy 上传的 52 列版本）

| 检查 | 结果 |
| --- | --- |
| 行数 | 796,989 ✓ |
| 原来 48 列 | 列名、类型和 v1 一致；`tip_pct`、`high_tip`、`split` 日期范围、`payment_type` 等不变量都成立 ✓ |
| `is_night` | 和"上车小时 20–5"完全一致 ✓；Vendor 2 夜间 `extra` 87% 是 $1.00 ✓ |
| `is_rush_hour` 和计价器对照 | Vendor 2 高峰行程 98.4% 有 `extra ≥ 2.5`，非高峰白天约 6.6% ✓ |
| `tip_pct_bill` | 公式核对一致；bill ≤ 0 的 0 行；> 1 的 0 行；中位数 0.200 ✓ |
| `high_tip_bill` | 和"多一分钱"规则 100% 一致 ✓ |
| **节假日** | 5/25 和 6/19 的 16–19 点，Vendor 2 有 **约 90%** 行程被收了 $2.50 高峰费（普通工作日约 98%），而这两天 16 点前、20 点后只有约 10%。也就是说 **2026 年计价器在这两个节假日照样收高峰费**。原版本把这 3,489 行标成了"非高峰"，和乘客实际看到的账单不符，所以最新 notebook 改为**不排除节假日**。 |
| 列数 | 52 列：是加 `_25`/`_30` 之前的版本，需要用最新 notebook 重跑得到 54 列。 |
