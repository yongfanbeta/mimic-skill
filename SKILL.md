---
name: mimic-skill
description: "从 MIMIC-IV 重症监护数据库提取 ICU 患者数据（生命体征/实验室检查/诊断/合并症）。当用户提到 MIMIC、MIMIC-IV、MIMIC-IV、查询 MIMIC 数据、提取 ICU 数据、重症数据库查询、重症医学数据挖掘、MIMIC 数据分析、ICU 患者数据提取时使用此技能。支持 SQL 和 Python (psycopg2) 两种查询方式，需用户自行提供 PostgreSQL 连接参数。Do NOT use for eICU data (use eicu-skill instead).
---

# MIMIC-IV Data Extraction Skill

# MIMIC-IV Data Extraction Skill

Extract intensive care data from the MIMIC-IV database. Provides SQL templates and Python code; users handle database connections themselves.

## Connection Method

Users provide their own PostgreSQL connection parameters. This skill only provides query code.

## Important Changes in MIMIC-IV (vs MIMIC-III)

⚠️ **If you have used MIMIC-III, please note the following changes!**

### 1. Table Name Changes

| MIMIC-III (❌ Deprecated) | MIMIC-IV (✅ Correct Usage) | Description |
|---------------------------|--------------------------|------|
| `icustay_id` | `stay_id` | ICU stay ID |
| `inputevents_mv` | `inputevents` | Input events (merged mv and cv)|
| `inputevents_cv` | `inputevents` | Merged into `inputevents` |
| `procedureevents_mv` | `procedureevents` | Procedure events |
| `datetimeevents` | `datetimeevents` | New table (datetime events)|
| `ingredientevents` | `ingredientevents` | New table (drug ingredient events)|

### 2. New Tables (MIMIC-IV)

| Table Name | Description |
|--------|------|
| `ingredientevents` | Drug active ingredient events |
| `datetimeevents` | Datetime events |
| `emar` | Electronic medication administration record |
| `emar_detail` | Electronic medication administration detail |
| `poe` | Provider order entry records |
| `poe_detail` | Provider order entry detail |

### 3. New Modules (MIMIC-IV)

MIMIC-IV is now divided into **6 modules**:

| Module | Description | Main Tables |
|------|------|--------|
| **hosp** | Hospital-level data | patients, admissions, labevents, diagnoses_icd, etc. |
| **icu** | ICU-level data | icustays, chartevents, inputevents, etc. |
| **ed** | Emergency department data | edstays, edcharting, etc. |
| **cxr** | Chest X-ray metadata | cxr_records, cxr_paths, etc. |
| **note** | Clinical notes | discharges, echos, etc. |
| **ecg** | ECG data | ecg_records, ecg_paths, etc. |

### 4. Field Name Conventions

✅ **Correct** (lowercase):
```sql
SELECT ce.subject_id, ce.stay_id, ce.itemid, ce.charttime
FROM chartevents ce
WHERE ce.stay_id = 100001
```

❌ **Incorrect** (camelCase or old field names):
```sql
SELECT ce.subjectId, ce.icustay_id  -- Field does not exist!
FROM chartevents ce
```

---

## Supported Query Types

| Query Type | Reference File |
|---------|---------|
| Vital Signs | references/vital_signs.md |
| Laboratory Tests | references/labs.md |
| Diagnoses and Comorbidities | references/diagnoses.md |
| Database Schema | references/schema.md |
| Common Query Templates | references/common_queries.md |

## Workflow

1. Confirm the query type needed by the user
2. Read the corresponding references file to get SQL/Python templates
3. Adjust query conditions based on specific user requirements
4. Provide both SQL and Python implementations
5. Explain how to customize parameters (e.g., time range, itemid, etc.)

## Key Conventions

- MIMIC-IV uses `stay_id` as the ICU stay identifier (**not MIMIC-III's `icustay_id`**)
- `anchor_age` is truncated (patients >89 are all set to 91)
- `chartevents` is linked via `stay_id`, `labevents` via `hadm_id`
- `itemid` comes from `d_items` (vital signs) and `d_labitems` (lab tests)
- **Time representation**: MIMIC-IV uses absolute timestamps (`TIMESTAMP` type), time functions can be used directly

---

## Common Query Scenarios

### 1. Extract First Day Vital Signs in ICU

```sql
SELECT 
    ce.subject_id,
    ce.stay_id,
    ce.itemid,
    di.label,
    ce.charttime,
    ce.valuenum,
    ce.valueuom
FROM chartevents ce
INNER JOIN icustays ic ON ce.stay_id = ic.stay_id
INNER JOIN d_items di ON ce.itemid = di.itemid
WHERE ce.charttime >= ic.intime
  AND ce.charttime < ic.intime + INTERVAL '1 day'
  AND ce.itemid IN (220045, 220050, 220051, 220052, 223762, 220210, 220277, 223835)
  AND ce.valuenum IS NOT NULL
ORDER BY ce.stay_id, ce.charttime;
```

### 2. Extract First Day Laboratory Tests in ICU

```sql
SELECT 
    le.subject_id,
    le.hadm_id,
    le.itemid,
    dli.label,
    le.charttime,
    le.valuenum,
    le.valueuom
FROM labevents le
INNER JOIN admissions a ON le.hadm_id = a.hadm_id
INNER JOIN d_labitems dli ON le.itemid = dli.itemid
WHERE le.charttime >= a.admittime
  AND le.charttime < a.admittime + INTERVAL '1 day'
  AND le.itemid IN (50912, 51006, 50983, 50971, 50902, 50882)
  AND le.valuenum IS NOT NULL
ORDER BY le.subject_id, le.charttime;
```

### 3. Extract Patient Diagnoses (ICD Codes)

```sql
SELECT 
    d.subject_id,
    d.hadm_id,
    d.seq_num,
    d.icd_code,
    d.icd_version,
    did.long_title
FROM diagnoses_icd d
LEFT JOIN d_icd_diagnoses did
    ON d.icd_code = did.icd_code AND d.icd_version = did.icd_version
WHERE d.hadm_id = %(hadm_id)s
ORDER BY d.seq_num;
```

---

## Notes

### 1. Table and Field Name Case Sensitivity

- **All lowercase**: All table and field names in MIMIC-IV are lowercase
- **Use double quotes**: If you must use uppercase or camelCase names, wrap them in double quotes (not recommended)

### 2. Time Handling

- **Absolute timestamps**: All time fields are `TIMESTAMP` type (UTC timezone)
- **Time functions**: You can directly use functions like `EXTRACT`, `DATE_PART`, `INTERVAL`
- **Timezone conversion**: Use `AT TIME ZONE` for local time conversion

```sql
-- Convert to local time (e.g., Eastern Time)
SELECT charttime AT TIME ZONE 'America/New_York'
FROM chartevents
WHERE stay_id = 100001;
```

### 3. Large Table Query Optimization

- The `chartevents` table has **hundreds of millions of rows**; when querying, be sure to:
  - Filter by `stay_id`
  - Limit the `charttime` range
  - Add a `LIMIT` clause (for testing)
  - Use `EXPLAIN` to analyze the query plan

### 4. Numeric vs Text Results

- `valuenum`: Numeric results (for calculations)
- `value`: Text results (for qualitative results, e.g., "Positive", "Negative")

```sql
-- Correct: Use valuenum for numeric results
SELECT AVG(valuenum) AS avg_creatinine
FROM labevents
WHERE itemid = 50912 AND valuenum IS NOT NULL;

-- Correct: Use value for text results
SELECT COUNT(*) AS num_positive
FROM labevents
WHERE itemid = 500061 AND value = 'Positive';
```

---

## Reference Links

- **MIMIC-IV Official Documentation**: https://mimic.mit.edu/docs/IV/
- **MIMIC-IV Data Introduction**: https://physionet.org/content/mimiciv/3.1/
- **MIMIC-IV-ED Data Introduction**: https://physionet.org/content/mimic-iv-ed/2.0/
- **MIMIC-IV-Note Data Introduction**: https://physionet.org/content/mimic-iv-note/2.0/
- **MIMIC-CXR Data Introduction**: https://physionet.org/content/mimic-cxr/2.1/
- **Access Application**: https://physionet.org/register/
- **MIMIC Code Repository**: https://github.com/MIT-LCP/mimic-code

---

**Last Updated**: 2026-05-20  
**Updated by**: 悟空（基于 MIMIC-IV 官方文档修正）
