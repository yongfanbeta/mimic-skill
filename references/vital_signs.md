# MIMIC-IV 生命体征查询

## 目录

1. [常用 itemid 映射](#常用-itemid-映射)
2. [SQL 模板：ICU 第一天生命体征](#sql-模板icu-第一天生命体征)
3. [Python 模板：ICU 第一天生命体征](#python-模板icu-第一天生命体征)
4. [处理重复记录](#处理重复记录)
5. [注意事项](#注意事项)

---

## 常用 itemid 映射

| 指标 | itemid | 单位 | 说明 |
|------|--------|------|------|
| 心率 (Heart Rate) | 220045 | bpm | |
| 收缩压 (SBP) | 220050 | mmHg | |
| 舒张压 (DBP) | 220051 | mmHg | |
| 平均动脉压 (MAP) | 220052 / 225312 | mmHg | 两者取其一即可 |
| 体温 (华氏) | 223761 | \u00b0F | |
| 体温 (摄氏) | 223762 | \u00b0C | |
| 呼吸频率 (RR) | 220210 / 224690 | breaths/min | |
| 氧饱和度 (SpO2) | 220277 | % | |
| FiO2 (%) | 223835 | % | 0-100 范围 |
| FiO2 (0-1) | 2981 / 3420 / 3422 | fraction | 0-1 范围 |
| CVP | 220675 | mmHg | |

> 查询 itemid 完整列表：`SELECT * FROM d_items WHERE category = 'Heart Rate'` 或按 label 搜索。

---

## SQL 模板：ICU 第一天生命体征

提取 ICU 入住后 24 小时内的生命体征，按小时聚合取均值。

```sql
WITH vitals_config AS (
    SELECT 220045 AS itemid, 'HeartRate' AS label UNION ALL
    SELECT 220050, 'SBP' UNION ALL
    SELECT 220051, 'DBP' UNION ALL
    SELECT 220052, 'MAP' UNION ALL
    SELECT 223762, 'Temperature_C' UNION ALL
    SELECT 220210, 'RespRate' UNION ALL
    SELECT 220277, 'SpO2' UNION ALL
    SELECT 223835, 'FiO2'
)
SELECT
    i.stay_id,
    i.subject_id,
    i.hadm_id,
    vc.label AS vital_name,
    EXTRACT(EPOCH FROM (ce.charttime - i.intime)) / 3600 AS hours_since_icu,
    -- 第一天均值
    AVG(ce.valuenum) AS mean_value,
    -- 第一天最值
    MIN(ce.valuenum) AS min_value,
    MAX(ce.valuenum) AS max_value,
    -- 记录数
    COUNT(*) AS n_records
FROM icustays i
JOIN chartevents ce
    ON i.stay_id = ce.stay_id
JOIN vitals_config vc
    ON ce.itemid = vc.itemid
WHERE ce.charttime >= i.intime
  AND ce.charttime < i.intime + INTERVAL '1 day'
  AND ce.valuenum IS NOT NULL
  AND ce.valuenum > 0        -- 过滤异常值
  AND ce.valuenum < 300      -- 过滤不合理数值
GROUP BY i.stay_id, i.subject_id, i.hadm_id, vc.label, hours_since_icu
ORDER BY i.stay_id, vc.label, hours_since_icu;
```

### 简化版：每个指标取第一天均值（宽表格式）

```sql
WITH first_day_vitals AS (
    SELECT
        i.stay_id,
        i.subject_id,
        i.hadm_id,
        AVG(CASE WHEN ce.itemid = 220045 THEN ce.valuenum END) AS heart_rate_mean,
        MIN(CASE WHEN ce.itemid = 220045 THEN ce.valuenum END) AS heart_rate_min,
        MAX(CASE WHEN ce.itemid = 220045 THEN ce.valuenum END) AS heart_rate_max,
        AVG(CASE WHEN ce.itemid IN (220050, 220051) THEN ce.valuenum END) AS sbp_mean,
        AVG(CASE WHEN ce.itemid IN (220050, 220051) THEN ce.valuenum END) AS dbp_mean,
        AVG(CASE WHEN ce.itemid IN (220052, 225312) THEN ce.valuenum END) AS map_mean,
        AVG(CASE WHEN ce.itemid = 223762 THEN ce.valuenum END) AS temperature_mean,
        AVG(CASE WHEN ce.itemid = 220210 THEN ce.valuenum END) AS resp_rate_mean,
        AVG(CASE WHEN ce.itemid = 220277 THEN ce.valuenum END) AS spo2_mean,
        AVG(CASE WHEN ce.itemid = 223835 THEN ce.valuenum END) AS fio2_mean
    FROM icustays i
    JOIN chartevents ce
        ON i.stay_id = ce.stay_id
    WHERE ce.charttime >= i.intime
      AND ce.charttime < i.intime + INTERVAL '1 day'
      AND ce.itemid IN (220045, 220050, 220051, 220052, 225312,
                        223762, 220210, 220277, 223835)
      AND ce.valuenum IS NOT NULL
      AND ce.valuenum > 0
      AND ce.valuenum < 300
    GROUP BY i.stay_id, i.subject_id, i.hadm_id
)
SELECT * FROM first_day_vitals
ORDER BY stay_id;
```

---

## Python 模板：ICU 第一天生命体征

```python
import psycopg2
import pandas as pd

# 用户自行填写连接参数
conn = psycopg2.connect(
    host="your_host",
    port=5432,
    dbname="mimiciv",
    user="your_user",
    password="your_password"
)

VITAL_ITEMS = {
    220045: "HeartRate",
    220050: "SBP",
    220051: "DBP",
    220052: "MAP",
    223762: "Temperature_C",
    220210: "RespRate",
    220277: "SpO2",
    223835: "FiO2",
}

def extract_first_day_vitals(stay_ids=None, itemids=None):
    \"\"\"
    提取 ICU 第一天生命体征。

    Parameters
    ----------
    stay_ids : list, optional
        指定 stay_id 列表，默认提取全部
    itemids : list, optional
        指定 itemid 列表，默认使用 VITAL_ITEMS 全部

    Returns
    -------
    pd.DataFrame
        每行一条记录，包含 stay_id, itemid, charttime, valuenum
    \"\"\"
    if itemids is None:
        itemids = list(VITAL_ITEMS.keys())

    query = \"\"\"
    SELECT
        i.stay_id,
        i.subject_id,
        ce.itemid,
        ce.charttime,
        ce.valuenum,
        ce.valueuom
    FROM icustays i
    JOIN chartevents ce ON i.stay_id = ce.stay_id
    WHERE ce.charttime >= i.intime
      AND ce.charttime < i.intime + INTERVAL '1 day'
      AND ce.itemid = ANY(%(itemids)s)
      AND ce.valuenum IS NOT NULL
      AND ce.valuenum > 0 AND ce.valuenum < 300
    \"\"\"

    params = {"itemids": itemids}
    if stay_ids is not None:
        query += " AND i.stay_id = ANY(%(stay_ids)s)"
        params["stay_ids"] = stay_ids

    df = pd.read_sql_query(query, conn, params=params)
    df["vital_name"] = df["itemid"].map(VITAL_ITEMS)
    return df

def aggregate_vitals_wide(df):
    \"\"\"
    将长格式生命体征数据聚合为宽表（每个指标取第一天均值/最小值/最大值）。

    Parameters
    ----------
    df : pd.DataFrame
        extract_first_day_vitals 的输出

    Returns
    -------
    pd.DataFrame
        宽表格式，每个指标一行统计
    \"\"\"
    agg = df.groupby(["stay_id", "subject_id", "vital_name"]).agg(
        mean_value=("valuenum", "mean"),
        min_value=("valuenum", "min"),
        max_value=("valuenum", "max"),
        n_records=("valuenum", "count")
    ).reset_index()
    return agg

# 使用示例
if __name__ == "__main__":
    vitals = extract_first_day_vitals()
    print(f"Extracted {len(vitals)} records")
    agg = aggregate_vitals_wide(vitals)
    print(agg.head(10))

conn.close()
```

---

## 处理重复记录

同一时段内可能有多个相同 itemid 的记录（如不同护士同时记录）。处理策略：

| 策略 | SQL 实现 | 适用场景 |
|------|---------|---------|
| 取第一个 | `DISTINCT ON (stay_id, itemid, hour)` | 只需要一个代表值 |
| 取均值 | `AVG(valuenum)` | 需要平滑处理 |
| 取最值 | `MIN/MAX(valuenum)` | 关注极端值（如最高体温） |
| 取最后一个 | 使用窗口函数 `ROW_NUMBER() ... ORDER BY charttime DESC` | 关注最新记录 |

按小时聚合示例：
```sql
-- 按小时取最后一个值
WITH hourly AS (
    SELECT
        stay_id, itemid,
        EXTRACT(EPOCH FROM (charttime - intime))::int / 3600 AS hour,
        valuenum,
        ROW_NUMBER() OVER (
            PARTITION BY stay_id, itemid,
            EXTRACT(EPOCH FROM (charttime - intime))::int / 3600
            ORDER BY charttime DESC
        ) AS rn
    FROM icustays i
    JOIN chartevents ce ON i.stay_id = ce.stay_id
    WHERE ce.charttime >= i.intime
      AND ce.charttime < i.intime + INTERVAL '1 day'
      AND ce.valuenum IS NOT NULL
)
SELECT stay_id, itemid, hour, valuenum
FROM hourly WHERE rn = 1;
```

---

## 注意事项

1. **chartevents 表数据量巨大**（数亿行），查询时务必加时间范围和 itemid 限制
2. **valuenum 过滤**：`valuenum > 0 AND valuenum < 300` 可过滤大部分异常值，但某些指标（如 FiO2=100）的阈值需要调整
3. **华氏/摄氏温度**：itemid 223761 是华氏度，223762 是摄氏度，注意区分
4. **FiO2 单位不统一**：itemid 223835 为百分比(%)，2981/3420/3422 为小数(0-1)
5. **有创/无创血压**：通常取有创血压（更准确），无创血压作为补充
6. **建议建索引**：`CREATE INDEX ON chartevents (stay_id, itemid, charttime);`
