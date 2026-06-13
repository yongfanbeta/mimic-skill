# MIMIC-IV 诊断与合并症查询

## 目录

1. [查询患者 ICD 诊断](#查询患者-icd-诊断)
2. [Charlson/Deyo 合并症映射](#charlsondeyo-合并症映射)
3. [SQL 模板](#sql-模板)
4. [Python 模板](#python-模板)

---

## 查询患者 ICD 诊断

```sql
SELECT
    d.subject_id, d.hadm_id,
    d.seq_num,
    d.icd_code,
    d.icd_version,
    dd.long_title
FROM diagnoses_icd d
LEFT JOIN d_icd_diagnoses dd
    ON d.icd_code = dd.icd_code AND d.icd_version = dd.icd_version
WHERE d.hadm_id = :hadm_id
ORDER BY d.seq_num;
```

---

## Charlson/Deyo 合并症映射

### ICD-9 编码

| 合并症 | ICD-9 编码 |
|---------|----------|
| 心肌梧死 (MI) | 410.x, 412.x |
| 充血性心力衰竭 (CHF) | 428.x |
| 外周血管病 (PVD) | 443.9, 441.x, 785.4, V43.4 |
| 脑血管病 (CVD) | 430-438.x |
| 痴呆 (Dementia) | 290.x |
| 慢性肺病 (COPD) | 490-505.x, 506.4, 493.x |
| 结缔组织病 (Rheumatic) | 710.0, 710.1, 710.4, 714.0-714.2, 714.8, 725 |
| 消化性溃疡 (PUD) | 531-534.x |
| 轻度肝病 (Mild Liver) | 571.2, 571.4-571.6, 070.22, 070.23, 070.32, 070.33, 070.44, 070.54, 070.6, 070.9, 570.x, 571.0, 571.1, 571.3, 571.8, 571.9, V42.7 |
| 中重度肝病 (Mod-Severe Liver) | 572.2-572.8 |
| 糖尿病无并发症 (DM w/o Comp) | 250.0-250.3, 250.8, 250.9 |
| 糖尿病有并发症 (DM w/ Comp) | 250.4-250.7 |
| 偏瘫 (Hemiplegia) | 334.1, 342.x, 344.0-344.6 |
| 肾病 (Mild-Moderate) | 582.x, 583.0-583.7, 585.x, 586.x, 588.0, V42.0, V45.1, V56 |
| 肾病 (Severe/ESRD) | 585.5, 585.6, V45.11, V56.0, V56.31, V56.32 |
| 肿瘤无转移 (Cancer w/o Metastasis) | 140-172.x, 174-195.8, 200-208.x |
| 肿瘤有转移 (Cancer w/ Metastasis) | 196-199.x |
| AIDS | 042-044.x |

### ICD-10 编码

| 合并症 | ICD-10 编码 |
|---------|----------|
| 心肌梧死 (MI) | I21.x, I22.x, I25.2 |
| 充血性心力衰竭 (CHF) | I50.x, I09.9, I11.0, I13.0, I13.2, I25.5, I42.x |
| 外周血管病 (PVD) | I73.1, I73.8, I73.9, I77.1, I78, I79.0, I79.2, K55.1, K55.8, K55.9, Z95.8, Z95.9 |
| 脑血管病 (CVD) | G45.x, I60-I69.x |
| 痴呆 (Dementia) | F00.x-F03.x, F05.1, G30.x, G31.1 |
| 慢性肺病 (COPD) | J44.x, J47, J67.x, J68.4, J70.1, J70.3 |
| 结缔组织病 (Rheumatic) | M05.x, M06.x, M32.x, M33.x, M34.x, M35.1, M35.3 |
| 消化性溃疡 (PUD) | K25.x-K28.x |
| 轻度肝病 (Mild Liver) | B18, K73.x, K74.x, K70.0-K70.3, K70.9, K71.3-K71.5, K71.7, K71.9, K76.0, K76.2-K76.4, K76.8, K76.9, Z94.4 |
| 中重度肝病 (Mod-Severe Liver) | I85.0, I85.9, I86.4, I98.2, K70.4, K71.1, K72.x, K76.5, K76.6, K76.7 |
| 糖尿病无并发症 (DM w/o Comp) | E10.x, E11.x, E12.x, E13.x, E14.x |
| 糖尿病有并发症 (DM w/ Comp) | E10.2-E10.8 (excl E10.10), E11.2-E11.8, E12.2-E12.8, E13.2-E13.8, E14.2-E14.8 |
| 偏瘫 (Hemiplegia) | G81.x, G82.x, G04.1 |
| 肾病 (CKD) | N18.1-N18.6, N18.9, N19, Z49.0-Z49.2, Z94.0, Z99.2 |
| 肿瘤无转移 (Cancer w/o Metastasis) | C00-C26.x, C30-C34.x, C37-C41.x, C43-C58.x, C60-C76.x, C81-C88.x |
| 肿瘤有转移 (Cancer w/ Metastasis) | C77-C80.x |
| AIDS | B20-B24 |

---

## SQL 模板

### 提取合并症（ICD-9）

```sql
WITH charlson_icd9 AS (
    SELECT 'MI' AS comorbidity, icd_code FROM (VALUES
        ('410'), ('412')
    ) AS t(icd_code) UNION ALL
    SELECT 'CHF', icd_code FROM (VALUES ('428')) AS t(icd_code) UNION ALL
    SELECT 'PVD', icd_code FROM (VALUES ('4439'), ('441'), ('7854'), ('V434')) AS t(icd_code) UNION ALL
    SELECT 'CVD', icd_code FROM (VALUES
        ('430'), ('431'), ('432'), ('433'), ('434'), ('435'), ('436'), ('437'), ('438')
    ) AS t(icd_code) UNION ALL
    SELECT 'Dementia', icd_code FROM (VALUES ('290')) AS t(icd_code) UNION ALL
    SELECT 'COPD', icd_code FROM (VALUES
        ('490'), ('491'), ('492'), ('493'), ('494'), ('495'), ('496'), ('500'), ('501'),
        ('502'), ('503'), ('504'), ('505'), ('5064')
    ) AS t(icd_code) UNION ALL
    SELECT 'Rheumatic', icd_code FROM (VALUES
        ('7100'), ('7101'), ('7104'), ('7140'), ('7141'), ('7142'), ('7148'), ('725')
    ) AS t(icd_code) UNION ALL
    SELECT 'PUD', icd_code FROM (VALUES ('531'), ('532'), ('533'), ('534')) AS t(icd_code) UNION ALL
    SELECT 'MildLiver', icd_code FROM (VALUES
        ('5712'), ('5714'), ('5715'), ('5716'), ('5718'), ('5719'),
        ('07022'), ('07023'), ('07032'), ('07033'), ('07044'), ('07054')
    ) AS t(icd_code) UNION ALL
    SELECT 'SevereLiver', icd_code FROM (VALUES ('5722'), ('5723'), ('5724'), ('5728')) AS t(icd_code) UNION ALL
    SELECT 'DM_noComp', icd_code FROM (VALUES ('2500'), ('2501'), ('2502'), ('2503'), ('2508'), ('2509')) AS t(icd_code) UNION ALL
    SELECT 'DM_Comp', icd_code FROM (VALUES ('2504'), ('2505'), ('2506'), ('2507')) AS t(icd_code) UNION ALL
    SELECT 'Hemiplegia', icd_code FROM (VALUES
        ('3341'), ('3420'), ('3421'), ('3422'), ('3423'), ('3424'), ('3425'), ('3426'),
        ('3429'), ('3440'), ('3441'), ('3442'), ('3443'), ('3444'), ('3445'), ('3446')
    ) AS t(icd_code) UNION ALL
    SELECT 'Renal', icd_code FROM (VALUES
        ('582'), ('5830'), ('5831'), ('5832'), ('5834'), ('5836'), ('5837'),
        ('585'), ('586'), ('5880'), ('V420'), ('V451'), ('V56')
    ) AS t(icd_code) UNION ALL
    SELECT 'Cancer', icd_code FROM (VALUES
        ('140'), ('141'), ('142'), ('143'), ('144'), ('145'), ('146'), ('147'),
        ('148'), ('149'), ('150'), ('151'), ('152'), ('153'), ('154'), ('155'),
        ('156'), ('157'), ('158'), ('159'), ('160'), ('161'), ('162'), ('163'),
        ('164'), ('165'), ('166'), ('167'), ('168'), ('169'), ('170'), ('171'),
        ('172'), ('174'), ('175'), ('176'), ('177'), ('178'), ('179'), ('180'),
        ('181'), ('182'), ('183'), ('184'), ('185'), ('186'), ('187'), ('188'),
        ('189'), ('190'), ('191'), ('192'), ('193'), ('194'), ('195'), ('200'),
        ('201'), ('202'), ('203'), ('204'), ('205'), ('206'), ('207'), ('208')
    ) AS t(icd_code) UNION ALL
    SELECT 'MetastaticCancer', icd_code FROM (VALUES ('196'), ('197'), ('198'), ('199')) AS t(icd_code) UNION ALL
    SELECT 'AIDS', icd_code FROM (VALUES ('042'), ('043'), ('044')) AS t(icd_code)
)
SELECT
    d.subject_id, d.hadm_id,
    c.comorbidity,
    COUNT(DISTINCT d.icd_code) AS n_codes
FROM diagnoses_icd d
JOIN charlson_icd9 c
    ON d.icd_code LIKE c.icd_code || '%'
WHERE d.icd_version = 9
GROUP BY d.subject_id, d.hadm_id, c.comorbidity
ORDER BY d.subject_id, d.hadm_id;
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

def get_comorbidities(hadm_id):
    """Get Charlson comorbidities for a hospital admission."""
    query = """
    SELECT d.icd_code, d.icd_version, dd.long_title
    FROM diagnoses_icd d
    LEFT JOIN d_icd_diagnoses dd
        ON d.icd_code = dd.icd_code AND d.icd_version = dd.icd_version
    WHERE d.hadm_id = %(hadm_id)s
    ORDER BY d.seq_num
    """
    return pd.read_sql_query(query, conn, params={"hadm_id": hadm_id})

# 使用示例
comorbidities = get_comorbidities(20000003)
print(comorbidities)
conn.close()
```
