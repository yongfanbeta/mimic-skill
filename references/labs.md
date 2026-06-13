# MIMIC-IV 生化指标查询

## 目录

1. [常用 itemid 映射](#常用-itemid-映射)
2. [SQL 模板：ICU 第一天实验室检查](#sql-模板icu-第一天实验室检查)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)

---

## 常用 itemid 映射

### 血液学

| 指标 | itemid | 说明 |
|------|--------|------|
| 白细胞 (WBC) | 51300 / 51301 | 10^3/uL |
| 血红蛋白 (Hemoglobin) | 51222 | g/dL |
| 血小板 (Platelet) | 51265 | 10^3/uL |
| 红细胞比容 (Hematocrit) | 51221 | % |

### 肾功能

| 指标 | itemid | 说明 |
|------|--------|------|
| 肌酐 (Creatinine) | 50912 | mg/dL |
| BUN | 51006 | mg/dL |

### 电解质

| 指标 | itemid | 说明 |
|------|--------|------|
| 钠 (Sodium) | 50983 / 52455 | mEq/L |
| 钾 (Potassium) | 50971 / 52546 | mEq/L |
| 氯 (Chloride) | 50902 / 52533 | mEq/L |
| 碳酸氢根 (Bicarbonate) | 50882 | mEq/L |
| 钙 (Calcium) | 50893 | mg/dL |
| 镁 (Magnesium) | 50960 | mg/dL |
| 磷 (Phosphorus) | 50970 | mg/dL |

### 肝功能

| 指标 | itemid | 说明 |
|------|--------|------|
| 总胆红素 (Total Bilirubin) | 50885 | mg/dL |
| ALT (SGPT) | 50861 | IU/L |
| AST (SGOT) | 50878 | IU/L |
| 白蛋白 (Albumin) | 50862 | g/dL |

### 凝血

| 指标 | itemid | 说明 |
|------|--------|------|
| PT | 51237 | 秒 |
| aPTT | 51274 | 秒 |

### 血气 + 其他

| 指标 | itemid | 说明 |
|------|--------|------|
| pH | 50821 | |
| PaO2 | 50821 | mmHg |
| PaCO2 | 50818 | mmHg |
| 乳酸 (Lactate) | 50813 | mmol/L |
| 葡萄糖 (Glucose) | 50931 / 50809 | mg/dL |
| CRP | 50889 | mg/L |
| 降钙素原 (PCT) | 50911 | ng/mL |
| 肌钙蛋白 T (Troponin T) | 51003 | ng/mL |

> 查询 itemid 完整列表: `SELECT itemid, label, fluid, category FROM d_labitems WHERE label ILIKE '%creatinine%';`

---

## SQL 模板：ICU 第一天实验室检查

```sql
WITH lab_config AS (
    SELECT 50912 AS itemid, 'Creatinine' AS label UNION ALL
    SELECT 51006, 'BUN' UNION ALL
    SELECT 50983, 'Sodium' UNION ALL
    SELECT 50971, 'Potassium' UNION ALL
    SELECT 50902, 'Chloride' UNION ALL
    SELECT 50882, 'Bicarbonate' UNION ALL
    SELECT 51300, 'WBC' UNION ALL
    SELECT 51222, 'Hemoglobin' UNION ALL
    SELECT 51265, 'Platelet' UNION ALL
    SELECT 50931, 'Glucose' UNION ALL
    SELECT 50813, 'Lactate' UNION ALL
    SELECT 50885, 'TotalBilirubin' UNION ALL
    SELECT 50861, 'ALT' UNION ALL
    SELECT 50878, 'AST' UNION ALL
    SELECT 50862, 'Albumin' UNION ALL
    SELECT 51274, 'aPTT'
)
SELECT
    i.stay_id, i.subject_id, i.hadm_id,
    lc.label AS lab_name, le.charttime, le.valuenum, le.valueuom,
    le.ref_range_lower, le.ref_range_upper
FROM icustays i
JOIN labevents le ON i.hadm_id = le.hadm_id
JOIN lab_config lc ON le.itemid = lc.itemid
WHERE le.charttime >= i.intime
  AND le.charttime < i.intime + INTERVAL '1 day'
  AND le.valuenum IS NOT NULL
ORDER BY i.stay_id, lc.label, le.charttime;
```

### 宽表格式

```sql
SELECT
    i.stay_id, i.subject_id, i.hadm_id,
    MAX(CASE WHEN le.itemid = 50912 THEN le.valuenum END) AS creatinine_last,
    MAX(CASE WHEN le.itemid = 51006 THEN le.valuenum END) AS bun_last,
    MAX(CASE WHEN le.itemid = 50983 THEN le.valuenum END) AS sodium_last,
    MAX(CASE WHEN le.itemid = 50971 THEN le.valuenum END) AS potassium_last,
    MAX(CASE WHEN le.itemid = 50902 THEN le.valuenum END) AS chloride_last,
    MAX(CASE WHEN le.itemid = 50882 THEN le.valuenum END) AS bicarbonate_last,
    MAX(CASE WHEN le.itemid = 51300 THEN le.valuenum END) AS wbc_last,
    MAX(CASE WHEN le.itemid = 51222 THEN le.valuenum END) AS hemoglobin_last,
    MAX(CASE WHEN le.itemid = 51265 THEN le.valuenum END) AS platelet_last,
    MAX(CASE WHEN le.itemid = 50885 THEN le.valuenum END) AS bilirubin_last,
    MAX(CASE WHEN le.itemid = 50861 THEN le.valuenum END) AS alt_last,
    MAX(CASE WHEN le.itemid = 50878 THEN le.valuenum END) AS ast_last,
    MAX(CASE WHEN le.itemid = 50862 THEN le.valuenum END) AS albumin_last,
    MAX(CASE WHEN le.itemid = 50931 THEN le.valuenum END) AS glucose_last,
    MAX(CASE WHEN le.itemid = 50813 THEN le.valuenum END) AS lactate_last,
    MAX(CASE WHEN le.itemid = 51274 THEN le.valuenum END) AS aptt_last
FROM icustays i
JOIN labevents le ON i.hadm_id = le.hadm_id
WHERE le.charttime >= i.intime
  AND le.charttime < i.intime + INTERVAL '1 day'
  AND le.itemid IN (50912,51006,50983,50971,50902,50882,51300,51222,
                    51265,50885,50861,50878,50862,50931,50813,51274)
  AND le.valuenum IS NOT NULL
GROUP BY i.stay_id, i.subject_id, i.hadm_id
ORDER BY stay_id;
```

---

## Python 模板

```python
import psycopg2
import pandas as pd

conn = psycopg2.connect(
    host="your_host", port=5432,
    dbname="mimiciv", user="your_user", password="your_password"
)

LAB_ITEMS = {
    50912: "Creatinine", 51006: "BUN", 50983: "Sodium",
    50971: "Potassium", 50902: "Chloride", 50882: "Bicarbonate",
    51300: "WBC", 51222: "Hemoglobin", 51265: "Platelet",
    50931: "Glucose", 50813: "Lactate", 50885: "TotalBilirubin",
    50861: "ALT", 50878: "AST", 50862: "Albumin", 51274: "aPTT",
}

def extract_first_day_labs(stay_ids=None, itemids=None):
    if itemids is None:
        itemids = list(LAB_ITEMS.keys())
    query = """
    SELECT i.stay_id, i.subject_id, i.hadm_id,
           le.itemid, le.charttime, le.valuenum, le.valueuom
    FROM icustays i
    JOIN labevents le ON i.hadm_id = le.hadm_id
    WHERE le.charttime >= i.intime
      AND le.charttime < i.intime + INTERVAL '1 day'
      AND le.itemid = ANY(%(itemids)s)
      AND le.valuenum IS NOT NULL
    """
    params = {"itemids": itemids}
    if stay_ids is not None:
        query += " AND i.stay_id = ANY(%(stay_ids)s)"
        params["stay_ids"] = stay_ids
    df = pd.read_sql_query(query, conn, params=params)
    df["lab_name"] = df["itemid"].map(LAB_ITEMS)
    return df

# 使用示例
labs = extract_first_day_labs()
print(f"Extracted {len(labs)} lab records")
conn.close()
```

---

## 注意事项

1. **labevents 通过 hadm_id 关联**（非 stay_id），需 JOIN icustays 获取时间范围
2. **同一指标可能有多个 itemid**（如不同标本类型），建议先用 d_labitems 确认
3. **异常值处理**：建议对 valuenum 做合理性检查
4. **建议建索引**: `CREATE INDEX ON labevents (hadm_id, itemid, charttime);`
5. SQL 宽表中 MAX() 并非按时间取最后一个值，若需精确控制请用窗口函数
