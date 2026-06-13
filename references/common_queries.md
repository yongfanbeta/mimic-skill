# MIMIC-IV 常用查询模板

## 目录

1. [患者基本信息](#1-患者基本信息)
2. [SOFA 评分](#2-sofa-评分)
3. [机械通气信息](#3-机械通气信息)
4. [血管活性药物](#4-血管活性药物)
5. [AKI 分期 (KDIGO)](#5-aki-分期-kdigo)
6. [Sepsis-3 患者筛选](#6-sepsis-3-患者筛选)

---

## 1. 患者基本信息

```sql
SELECT
    p.subject_id,
    p.gender,
    p.anchor_age,
    p.dod AS date_of_death,
    a.hadm_id,
    a.admittime,
    a.dischtime,
    a.deathtime,
    a.admission_type,
    a.admission_location,
    a.discharge_location,
    a.ethnicity,
    a.hospital_expire_flag,
    i.stay_id,
    i.intime AS icu_intime,
    i.outtime AS icu_outtime,
    i.los AS icu_los,
    i.first_careunit,
    i.last_careunit
FROM patients p
JOIN admissions a ON p.subject_id = a.subject_id
JOIN icustays i ON a.hadm_id = i.hadm_id
ORDER BY p.subject_id, a.admittime;
```

---

## 2. SOFA 评分

SOFA = Sequential Organ Failure Assessment，ICU 第一天最高分。

```sql
WITH vitals AS (
    -- 心血管得分 (MAP < 70 mmHg)
    SELECT stay_id,
        CASE
            WHEN AVG(valuenum) >= 70 THEN 0
            WHEN AVG(valuenum) < 70 THEN 1
            WHEN AVG(valuenum) < 65 THEN 2
            WHEN AVG(valuenum) < 50 THEN 3
            ELSE 0
        END AS cv_score
    FROM icustays i
    JOIN chartevents ce ON i.stay_id = ce.stay_id
    WHERE ce.itemid IN (220052, 225312)
      AND ce.charttime >= i.intime
      AND ce.charttime < i.intime + INTERVAL '1 day'
      AND ce.valuenum IS NOT NULL AND ce.valuenum > 0
    GROUP BY stay_id
),
resp AS (
    -- 呼吸得分 (PaO2/FiO2 比)
    SELECT stay_id,
        CASE
            WHEN MAX(pao2_fio2) >= 400 THEN 0
            WHEN MAX(pao2_fio2) < 400 THEN 1
            WHEN MAX(pao2_fio2) < 300 THEN 2
            WHEN MAX(pao2_fio2) < 200 THEN 3
            WHEN MAX(pao2_fio2) < 100 THEN 4
            ELSE 0
        END AS resp_score
    FROM (
        SELECT i.stay_id,
            -- 简化: 直接用 SpO2/FiO2 估算
            MAX(ce1.valuenum) / MAX(ce2.valuenum) AS pao2_fio2
        FROM icustays i
        JOIN chartevents ce1 ON i.stay_id = ce1.stay_id AND ce1.itemid = 220277  -- SpO2
        JOIN chartevents ce2 ON i.stay_id = ce2.stay_id AND ce2.itemid = 223835  -- FiO2
        WHERE ce1.charttime >= i.intime AND ce1.charttime < i.intime + INTERVAL '1 day'
          AND ce2.charttime >= i.intime AND ce2.charttime < i.intime + INTERVAL '1 day'
          AND ce1.valuenum > 0 AND ce2.valuenum > 0
        GROUP BY i.stay_id
    ) sub
    GROUP BY stay_id
),
coag AS (
    -- 凝血得分 (血小板)
    SELECT stay_id,
        CASE
            WHEN MIN(valuenum) >= 150 THEN 0
            WHEN MIN(valuenum) < 150 THEN 1
            WHEN MIN(valuenum) < 100 THEN 2
            WHEN MIN(valuenum) < 50 THEN 3
            WHEN MIN(valuenum) < 20 THEN 4
            ELSE 0
        END AS coag_score
    FROM icustays i
    JOIN labevents le ON i.hadm_id = le.hadm_id
    WHERE le.itemid = 51265  -- Platelet
      AND le.charttime >= i.intime
      AND le.charttime < i.intime + INTERVAL '1 day'
      AND le.valuenum IS NOT NULL
    GROUP BY stay_id
),
liver AS (
    -- 肝脏得分 (胆红素 mg/dL)
    SELECT stay_id,
        CASE
            WHEN MAX(valuenum) < 1.2 THEN 0
            WHEN MAX(valuenum) < 2.0 THEN 1
            WHEN MAX(valuenum) < 6.0 THEN 2
            WHEN MAX(valuenum) < 12.0 THEN 3
            WHEN MAX(valuenum) >= 12.0 THEN 4
            ELSE 0
        END AS liver_score
    FROM icustays i
    JOIN labevents le ON i.hadm_id = le.hadm_id
    WHERE le.itemid = 50885  -- Total Bilirubin
      AND le.charttime >= i.intime
      AND le.charttime < i.intime + INTERVAL '1 day'
      AND le.valuenum IS NOT NULL
    GROUP BY stay_id
),
neuro AS (
    -- 神经得分 (GCS)
    SELECT stay_id,
        CASE
            WHEN MAX(valuenum) >= 15 THEN 0
            WHEN MAX(valuenum) < 15 THEN 1
            WHEN MAX(valuenum) < 13 THEN 2
            WHEN MAX(valuename) IN ('No Response', 'No Response-Endotracheal') THEN 3
            WHEN MAX(valuenum) < 6 THEN 4
            ELSE 0
        END AS neuro_score
    FROM icustays i
    JOIN chartevents ce ON i.stay_id = ce.stay_id
    WHERE ce.itemid = 198  -- GCS Motor
      AND ce.charttime >= i.intime
      AND ce.charttime < i.intime + INTERVAL '1 day'
      AND ce.valuenum IS NOT NULL
    GROUP BY stay_id
),
renal AS (
    -- 肾脏得分 (肌酐或 尿量)
    SELECT stay_id,
        CASE
            WHEN MAX(valuenum) < 1.2 THEN 0
            WHEN MAX(valuenum) < 2.0 THEN 1
            WHEN MAX(valuenum) < 3.5 THEN 2
            WHEN MAX(valuenum) < 5.0 THEN 3
            WHEN MAX(valuenum) >= 5.0 THEN 4
            ELSE 0
        END AS renal_score
    FROM icustays i
    JOIN labevents le ON i.hadm_id = le.hadm_id
    WHERE le.itemid = 50912  -- Creatinine
      AND le.charttime >= i.intime
      AND le.charttime < i.intime + INTERVAL '1 day'
      AND le.valuenum IS NOT NULL
    GROUP BY stay_id
)
SELECT
    i.stay_id, i.subject_id,
    COALESCE(v.cv_score, 0) AS sofa_cardiovascular,
    COALESCE(r.resp_score, 0) AS sofa_respiratory,
    COALESCE(c.coag_score, 0) AS sofa_coagulation,
    COALESCE(l.liver_score, 0) AS sofa_liver,
    COALESCE(n.neuro_score, 0) AS sofa_neurological,
    COALESCE(ren.renal_score, 0) AS sofa_renal,
    COALESCE(v.cv_score, 0) + COALESCE(r.resp_score, 0) +
    COALESCE(c.coag_score, 0) + COALESCE(l.liver_score, 0) +
    COALESCE(n.neuro_score, 0) + COALESCE(ren.renal_score, 0) AS sofa_total
FROM icustays i
LEFT JOIN vitals v ON i.stay_id = v.stay_id
LEFT JOIN resp r ON i.stay_id = r.stay_id
LEFT JOIN coag c ON i.stay_id = c.stay_id
LEFT JOIN liver l ON i.stay_id = l.stay_id
LEFT JOIN neuro n ON i.stay_id = n.stay_id
LEFT JOIN renal ren ON i.stay_id = ren.stay_id
ORDER BY i.stay_id;
```

---

## 3. 机械通气信息

```sql
-- 查找机械通气的 itemid
SELECT itemid, label, category
FROM d_items
WHERE label ILIKE '%vent%'
   OR label ILIKE '%mechanical%'
   OR label ILIKE '%respiratory support%'
ORDER BY itemid;

-- 常用机械通气相关 itemid
-- 223835 FiO2, 224696 Ventilator Mode, 227009 O2 Delivery Device
-- 224700 Exhaled Tidal Volume, 224685 Respiratory Rate (Set)
-- 224687 Peak Insp. Pressure, 224684 Plateau Pressure

-- 提取机械通气信息
SELECT
    i.stay_id, i.subject_id,
    ce.charttime,
    ce.itemid, di.label, ce.valuenum, ce.value, ce.valueuom
FROM icustays i
JOIN chartevents ce ON i.stay_id = ce.stay_id
JOIN d_items di ON ce.itemid = di.itemid
WHERE ce.itemid IN (
    223835,   -- FiO2
    224696,   -- Ventilator Mode
    227009,   -- O2 Delivery Device
    224700,   -- Exhaled Tidal Volume
    224685,   -- Respiratory Rate (Set)
    224687,   -- Peak Insp. Pressure
    224684    -- Plateau Pressure
)
AND ce.charttime >= i.intime
AND ce.charttime < i.intime + INTERVAL '1 day'
ORDER BY i.stay_id, ce.charttime;
```

---

## 4. 血管活性药物

```sql
-- 常用血管活性药物 (MIMIC-IV 使用 inputevents 表)
WITH vasopressors AS (
    SELECT
        i.stay_id, i.subject_id,
        ie.starttime,
        d.label,
        ie.rate, ie.rateuom
    FROM icustays i
    JOIN inputevents ie ON i.stay_id = ie.stay_id
    JOIN d_items d ON ie.itemid = d.itemid
    WHERE ie.starttime >= i.intime
      AND ie.starttime < i.intime + INTERVAL '1 day'
      AND (
          d.label ILIKE '%norepinephrine%'
          OR d.label ILIKE '%epinephrine%'
          OR d.label ILIKE '%vasopressin%'
          OR d.label ILIKE '%dopamine%'
          OR d.label ILIKE '%dobutamine%'
          OR d.label ILIKE '%phenylephrine%'
          OR d.label ILIKE '%milrinone%'
      )
      AND ie.rate IS NOT NULL
      AND ie.rate > 0
)
SELECT * FROM vasopressors
ORDER BY stay_id, starttime;
```

**注意**: MIMIC-IV 中 `inputevents_mv` 和 `inputevents_cv` 已合并为 `inputevents` 表。

---

## 5. AKI 分期 (KDIGO)

```sql
-- 基于肌酐变化判断 AKI (KDIGO 标准)
-- 需要与基线肌酐对比
WITH baseline_cr AS (
    -- 取住院前 7 天到入院前的最低肌酐作为基线
    SELECT
        le.subject_id, le.hadm_id,
        MIN(le.valuenum) AS baseline_cr
    FROM labevents le
    WHERE le.itemid = 50912  -- Creatinine
      AND le.charttime >= ALL(
          SELECT admittime - INTERVAL '7 days' - INTERVAL '365 days'
          FROM admissions a WHERE a.hadm_id = le.hadm_id
      )
      AND le.charttime < (
          SELECT admittime FROM admissions a WHERE a.hadm_id = le.hadm_id
      )
      AND le.valuenum IS NOT NULL
    GROUP BY le.subject_id, le.hadm_id
),
icu_cr AS (
    SELECT
        i.stay_id, i.subject_id, i.hadm_id, i.intime,
        le.charttime,
        le.valuenum AS creatinine,
        ROW_NUMBER() OVER (PARTITION BY i.stay_id ORDER BY le.valuenum DESC) AS max_row
    FROM icustays i
    JOIN labevents le ON i.hadm_id = le.hadm_id
    WHERE le.itemid = 50912
      AND le.charttime >= i.intime
      AND le.charttime < i.intime + INTERVAL '7 days'
      AND le.valuenum IS NOT NULL
)
SELECT
    icr.stay_id, icr.subject_id, icr.hadm_id,
    COALESCE(bl.baseline_cr, 1.0) AS baseline_cr,
    icr.creatinine AS max_icu_cr,
    CASE
        WHEN icr.creatinine >= COALESCE(bl.baseline_cr, 1.0) * 3 THEN 3
        WHEN icr.creatinine >= COALESCE(bl.baseline_cr, 1.0) * 2 THEN 2
        WHEN icr.creatinine >= COALESCE(bl.baseline_cr, 1.0) * 1.5 THEN 1
        ELSE 0
    END AS aki_stage,
    CASE
        WHEN icr.creatinine >= 4.0 THEN 3
        ELSE 0
    END AS aki_stage_abs
FROM icu_cr icr
LEFT JOIN baseline_cr bl
    ON icr.subject_id = bl.subject_id AND icr.hadm_id = bl.hadm_id
WHERE icr.max_row = 1;
```

---

## 6. Sepsis-3 患者筛选

Sepsis-3 定义：感染 + SOFA 评分变化 >= 2 分。

简化筛选（基于猛性感染 + 器官功能障碍）:

```sql
WITH suspicion AS (
    -- 感染疑似: 使用血培养 + 抗生素
    SELECT DISTINCT i.stay_id
    FROM icustays i
    JOIN microbiologyevents me ON i.subject_id = me.subject_id
    WHERE me.charttime >= i.intime
      AND me.charttime < i.intime + INTERVAL '1 day'
      AND me.org_name IS NOT NULL
      AND me.org_name != ''
),
organ_dysfunction AS (
    -- 器官功能障碍: 使用简化标准
    SELECT DISTINCT i.stay_id
    FROM icustays i
    WHERE EXISTS (
        -- 肌酐 >= 2.0
        SELECT 1 FROM labevents le
        WHERE le.hadm_id = i.hadm_id AND le.itemid = 50912
          AND le.charttime >= i.intime AND le.charttime < i.intime + INTERVAL '1 day'
          AND le.valuenum >= 2.0
        UNION ALL
        -- 胆红素 >= 2.0
        SELECT 1 FROM labevents le
        WHERE le.hadm_id = i.hadm_id AND le.itemid = 50885
          AND le.charttime >= i.intime AND le.charttime < i.intime + INTERVAL '1 day'
          AND le.valuenum >= 2.0
        UNION ALL
        -- 平均动脉压 < 70
        SELECT 1 FROM chartevents ce
        WHERE ce.stay_id = i.stay_id AND ce.itemid IN (220052, 225312)
          AND ce.charttime >= i.intime AND ce.charttime < i.intime + INTERVAL '1 day'
          AND ce.valuenum < 70
        UNION ALL
        -- 乳酸 > 2.0
        SELECT 1 FROM labevents le
        WHERE le.hadm_id = i.hadm_id AND le.itemid = 50813
          AND le.charttime >= i.intime AND le.charttime < i.intime + INTERVAL '1 day'
          AND le.valuenum > 2.0
        UNION ALL
        -- 尿量 < 0.5 mL/kg/h (6h)
        SELECT 1 FROM outputevents oe
        WHERE oe.stay_id = i.stay_id
          AND oe.charttime >= i.intime AND oe.charttime < i.intime + INTERVAL '1 day'
          AND oe.valuenum < 500
    )
)
SELECT
    i.stay_id, i.subject_id, i.hadm_id, i.intime, i.outtime
FROM icustays i
WHERE i.stay_id IN (SELECT stay_id FROM suspicion)
  AND i.stay_id IN (SELECT stay_id FROM organ_dysfunction);
```

> 注: 这是简化版本。MIT LCP 提供了更完整的 Sepsis-3 SQL 实现，建议参考: https://github.com/MIT-LCP/mimic-code/tree/main/concepts/sepsis
