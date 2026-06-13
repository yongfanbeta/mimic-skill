---
name: mimic-skill
description: 从 MIMIC-IV 重症监护数据库中提取数据的专用技能。当用户提到 MIMIC、MIMIC-IV、查询 MIMIC 数据、提取 ICU 患者数据（生命体征/实验室检查/诊断/合并症）且涉及 MIMIC 数据库时使用此技能。支持 SQL 和 Python (psycopg2) 两种查询方式，数据通过 PostgreSQL 连接。
---

# MIMIC-IV 数据提取技能

从 MIMIC-IV 数据库中提取重症监护数据。提供 SQL 模板和 Python 代码，用户自行处理数据库连接。

## 连接方式

用户提供自己的 PostgreSQL 连接参数。本技能只提供查询代码。

## MIMIC-IV 重要变更（vs MIMIC-III）

⚠️ **如果使用过 MIMIC-III，请务必注意以下变更！**

### 1. 表名变更

| MIMIC-III (❌ 已废弃) | MIMIC-IV (✅ 正确使用) | 说明 |
|---------------------------|--------------------------|------|
| `icustay_id` | `stay_id` | ICU 住院 ID |
| `inputevents_mv` | `inputevents` | 输入事件（合并了 mv 和 cv）|
| `inputevents_cv` | `inputevents` | 已合并到 `inputevents` |
| `procedureevents_mv` | `procedureevents` | 操作事件 |
| `datetimeevents` | `datetimeevents` | 新增表（日期时间事件）|
| `ingredientevents` | `ingredientevents` | 新增表（药物成分事件）|

### 2. 新增表（MIMIC-IV）

| 表名 | 说明 |
|--------|------|
| `ingredientevents` | 药物活性成分事件 |
| `datetimeevents` | 日期时间事件 |
| `emar` | 电子药历管理记录 |
| `emar_detail` | 电子药历详细记录 |
| `poe` | 医嘱记录（Provider Order Entry）|
| `poe_detail` | 医嘱详细记录 |

### 3. 新增模块（MIMIC-IV）

MIMIC-IV 现在分为 **6 个模块**：

| 模块 | 说明 | 主要表 |
|------|------|--------|
| **hosp** | 医院级数据 | patients, admissions, labevents, diagnoses_icd, etc. |
| **icu** | ICU 级数据 | icustays, chartevents, inputevents, etc. |
| **ed** | 急诊科数据 | edstays, edcharting, etc. |
| **cxr** | 胸片元数据 | cxr_records, cxr_paths, etc. |
| **note** | 临床笔记 | discharges, echos, etc. |
| **ecg** | 心电图数据 | ecg_records, ecg_paths, etc. |

### 4. 字段名规范

✅ **正确** (小写):
```sql
SELECT ce.subject_id, ce.stay_id, ce.itemid, ce.charttime
FROM chartevents ce
WHERE ce.stay_id = 100001
```

❌ **错误** (camelCase 或旧字段名):
```sql
SELECT ce.subjectId, ce.icustay_id  -- 字段不存在！
FROM chartevents ce
```

---

## 支持的查询类型

| 查询类型 | 参考文件 |
|---------|---------|
| 生命体征 | references/vital_signs.md |
| 生化指标 | references/labs.md |
| 诊断与合并症 | references/diagnoses.md |
| 数据库 Schema | references/schema.md |
| 常用查询模板 | references/common_queries.md |

## 工作流程

1. 确认用户需要的查询类型
2. 读取对应的 references 文件获取 SQL/Python 模板
3. 根据用户具体需求调整查询条件
4. 同时提供 SQL 和 Python 两种实现
5. 说明如何自定义参数（如时间范围、itemid 等）

## 关键约定

- MIMIC-IV 使用 `stay_id` 作为 ICU 住院标识（**非 MIMIC-III 的 `icustay_id`**）
- `anchor_age` 截断处理（>89 统一为 91）
- `chartevents` 通过 `stay_id` 关联，`labevents` 通过 `hadm_id` 关联
- `itemid` 来自 `d_items`（生命体征）和 `d_labitems`（实验室检查）
- **时间表示**: MIMIC-IV 使用绝对时间戳（`TIMESTAMP` 类型），可直接使用时间函数

---

## 常用查询场景

### 1. 提取 ICU 第一天生命体征

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

### 2. 提取 ICU 第一天实验室检查

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

### 3. 提取患者诊断（ICD 编码）

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

## 注意事项

### 1. 表名和字段名大小写

- **全部小写**: MIMIC-IV 所有表名和字段名均为小写
- **使用双引号**: 如果必须使用大写或驼峰命名，需用双引号括起来（不推荐）

### 2. 时间处理

- **绝对时间戳**: 所有时间字段均为 `TIMESTAMP` 类型（UTC 时区）
- **时间函数**: 可直接使用 `EXTRACT`, `DATE_PART`, `INTERVAL` 等函数
- **时区转换**: 如需本地时间，使用 `AT TIME ZONE`

```sql
-- 转换为本地时间（如东部时间）
SELECT charttime AT TIME ZONE 'America/New_York'
FROM chartevents
WHERE stay_id = 100001;
```

### 3. 大表查询优化

- `chartevents` 表有 **数亿行**，查询时务必：
  - 使用 `stay_id` 过滤
  - 限定 `charttime` 范围
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

### 4. 数值型 vs 文本型结果

- `valuenum`: 数值型结果（用于计算）
- `value`: 文本型结果（用于定性结果，如 "Positive", "Negative"）

```sql
-- 正确：数值型结果用 valuenum
SELECT AVG(valuenum) AS avg_creatinine
FROM labevents
WHERE itemid = 50912 AND valuenum IS NOT NULL;

-- 正确：文本型结果用 value
SELECT COUNT(*) AS num_positive
FROM labevents
WHERE itemid = 500061 AND value = 'Positive';
```

---

## 参考链接

- **MIMIC-IV 官方文档**: https://mimic.mit.edu/docs/IV/
- **MIMIC-IV 数据介绍**: https://physionet.org/content/mimiciv/3.1/
- **MIMIC-IV-ED 数据介绍**: https://physionet.org/content/mimic-iv-ed/2.0/
- **MIMIC-IV-Note 数据介绍**: https://physionet.org/content/mimic-iv-note/2.0/
- **MIMIC-CXR 数据介绍**: https://physionet.org/content/mimic-cxr/2.1/
- **访问申请**: https://physionet.org/register/
- **MIMIC Code Repository**: https://github.com/MIT-LCP/mimic-code

---

**最后更新**: 2026-05-20  
**更新人**: 悟空（基于 MIMIC-IV 官方文档修正）
