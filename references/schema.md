# MIMIC-IV 数据库 Schema 参考

> **数据来源**: [MIMIC-IV 官方文档](https://mimic.mit.edu/docs/IV/)  
> **数据库**: MIMIC-IV (v3.1)  
> **总模块数**: 6 个（hosp, icu, ed, cxr, note, ecg）

---

## 目录

1. [数据库概览](#数据库概览)
2. [hosp 模块（医院级数据）](#hosp-模块医院级数据)
3. [icu 模块（ICU 级数据）](#icu-模块icu-级数据)
4. [ed 模块（急诊科数据）](#ed-模块急诊科数据)
5. [cxr 模块（胸片数据）](#cxr-模块胸片数据)
6. [note 模块（临床笔记）](#note-模块临床笔记)
7. [ecg 模块（心电图数据）](#ecg-模块心电图数据)
8. [表间关系](#表间关系)
9. [重要说明](#重要说明)

---

## 数据库概览

MIMIC-IV 分为 6 个模块：

| 模块 | 说明 | 主要表 |
|------|------|--------|
| **hosp** | 医院级数据 | patients, admissions, labevents, diagnoses_icd, etc. |
| **icu** | ICU 级数据 | icustays, chartevents, inputevents, etc. |
| **ed** | 急诊科数据 | edstays, edcharting, etc. |
| **cxr** | 胸片元数据 | cxr_records, cxr_paths, etc. |
| **note** | 临床笔记 | discharges, echos, etc. |
| **ecg** | 心电图数据 | ecg_records, ecg_paths, etc. |

**重要变更（vs MIMIC-III）**:
- ❌ `inputevents_mv` → ✅ `inputevents`
- ❌ `procedureevents_mv` → ✅ `procedureevents`
- ✅ 新增 `ingredientevents` 表
- ✅ 新增 `datetimeevents` 表
- ✅ 新增 `emar`, `emar_detail` 表（电子药历）
- ✅ 新增 `ed` 模块（急诊科数据）

---

## hosp 模块（医院级数据）

### patients

患者人口学信息，每人一行。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (PK) |
| gender | VARCHAR | 性别 (M/F) |
| anchor_age | INTEGER | 锚定年龄（>89 统一为 91）|
| anchor_year | INTEGER | 锚定年份 |
| anchor_year_group | VARCHAR | 年份分组（用于去标识化）|
| dod | TIMESTAMP | 死亡时间（如有）|

**重要说明**:
- `anchor_age`: 患者的年龄在特定年份（`anchor_year`）的值。如果年龄 > 89，统一设为 91（去标识化）。
- `anchor_year`: 不是实际年份，而是相对年份（如 2110, 2111, etc.）。
- `anchor_year_group`: 年份分组（如 "2001 - 2005", "2006 - 2008" 等）。

---

### admissions

每次住院一行。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (PK) |
| admittime | TIMESTAMP | 入院时间 |
| dischtime | TIMESTAMP | 出院时间 |
| deathtime | TIMESTAMP | 死亡时间（如有）|
| admission_type | VARCHAR | 入院类型（EMERGENCY/OBSERVATION/ELECTIVE/ etc.）|
| admission_location | VARCHAR | 入院来源 |
| discharge_location | VARCHAR | 出院去向 |
| insurance | VARCHAR | 保险类型 |
| language | VARCHAR | 语言 |
| marital_status | VARCHAR | 婚姻状态 |
| ethnicity | VARCHAR | 种族 |
| hospital_expire_flag | INTEGER | 院内死亡标志（1=死亡）|
| edregtime | TIMESTAMP | 急诊登记时间（如有）|
| edouttime | TIMESTAMP | 急诊离开时间（如有）|

---

### transfers

每次科室转移一行。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| transfer_id | INTEGER | 转移 ID (PK) |
| eventtype | VARCHAR | 事件类型（admit/transfer/discharge）|
| careunit | VARCHAR | 护理单元 |
| intime | TIMESTAMP | 转入时间 |
| outtime | TIMESTAMP | 转出时间 |

---

### labevents

实验室检查结果。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| itemid | INTEGER | 项目 ID (FK → d_labitems) |
| charttime | TIMESTAMP | 标本采集时间 |
| storetime | TIMESTAMP | 存储时间 |
| value | VARCHAR | 结果值（文本）|
| valuenum | DOUBLE | 结果值（数值）|
| valueuom | VARCHAR | 单位 |
| ref_range_lower | DOUBLE | 参考值下限 |
| ref_range_upper | DOUBLE | 参考值上限 |
| flag | VARCHAR | 异常标志 |
| priority | VARCHAR | 优先级（STAT/ROUTINE）|
| comments | VARCHAR | 备注 |

**注意**: `valuenum` 是数值型结果，`value` 是文本型结果（用于定性结果）。

---

### d_labitems

实验室项目字典。

| 字段 | 类型 | 说明 |
|------|------|------|
| itemid | INTEGER | 项目 ID (PK) |
| label | VARCHAR | 项目名称 |
| fluid | VARCHAR | 体液类型 |
| category | VARCHAR | 类别 |
| loinc_code | VARCHAR | LOINC 编码 |

---

### diagnoses_icd

ICD 诊断编码。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| seq_num | INTEGER | 诊断序号（1=主诊断）|
| icd_code | VARCHAR | ICD 编码 |
| icd_version | INTEGER | ICD 版本（9 或 10）|

---

### d_icd_diagnoses

ICD 诊断字典。

| 字段 | 类型 | 说明 |
|------|------|------|
| icd_code | VARCHAR | ICD 编码 (PK，与 icd_version 组合) |
| icd_version | INTEGER | ICD 版本 (PK) |
| long_title | VARCHAR | 完整名称 |
| short_title | VARCHAR | 简称 |

---

### procedures_icd

ICD 操作编码。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| seq_num | INTEGER | 操作序号 |
| icd_code | VARCHAR | ICD 编码 |
| icd_version | INTEGER | ICD 版本（9 或 10）|

---

### d_icd_procedures

ICD 操作字典。

| 字段 | 类型 | 说明 |
|------|------|------|
| icd_code | VARCHAR | ICD 编码 (PK，与 icd_version 组合) |
| icd_version | INTEGER | ICD 版本 (PK) |
| long_title | VARCHAR | 完整名称 |
| short_title | VARCHAR | 简称 |

---

### prescriptions

药物处方。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| pharmacy_id | INTEGER | 药房 ID |
| poe_id | INTEGER | 医嘱 ID |
| poe_seq | INTEGER | 医嘱序号 |
| order_type | VARCHAR | 医嘱类型 |
| drug | VARCHAR | 药物名称 |
| drug_type | VARCHAR | 药物类型（MAIN/ADJUNCT）|
| formulary_drug_cd | VARCHAR | 药典代码 |
| route | VARCHAR | 给药途径 |
| drug_dose | VARCHAR | 剂量 |
| drug_dose_unit | VARCHAR | 剂量单位 |
| form_unit_disp | VARCHAR | 配方单位 |
| form_value_disp | DOUBLE | 配方数值 |
| doses_per_24_hrs | DOUBLE | 每 24 小时剂量次数 |
| duration | DOUBLE | 持续时间 |
| duration_interval | VARCHAR | 持续时间间隔 |
| starttime | TIMESTAMP | 开始时间 |
| stoptime | TIMESTAMP | 停止时间 |

---

### pharmacy

药房记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| pharmacy_id | INTEGER | 药房 ID (PK) |
| poe_id | INTEGER | 医嘱 ID |
| med_routes | VARCHAR | 给药途径 |
| med_durations | VARCHAR | 持续时间 |
| med_dosages | VARCHAR | 剂量 |
| medication | VARCHAR | 药物名称 |
| ndc | VARCHAR | NDC 代码 |
| proc_type | VARCHAR | 流程类型 |
| status | VARCHAR | 状态 |
| starttime | TIMESTAMP | 开始时间 |
| stoptime | TIMESTAMP | 停止时间 |

---

### emar

电子药历管理记录（Electronic Medication Administration Record）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| emar_id | VARCHAR | EMAR ID (PK) |
| emar_seq | VARCHAR | EMAR 序号 |
| poe_id | INTEGER | 医嘱 ID |
| pharmacy_id | INTEGER | 药房 ID |
| charttime | TIMESTAMP | 记录时间 |
| medication | VARCHAR | 药物名称 |
| eventtype | VARCHAR | 事件类型 |
| scheduletime | TIMESTAMP | 计划时间 |
| storetime | TIMESTAMP | 存储时间 |

---

### emar_detail

电子药历详细记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| emar_id | VARCHAR | EMAR ID (FK → emar) |
| emar_seq | VARCHAR | EMAR 序号 (FK → emar) |
| parent_field | VARCHAR | 父字段 |
| parent_field_ordinal | INTEGER | 父字段序号 |
| field | VARCHAR | 字段名 |
| value | VARCHAR | 值 |

---

### poe

医嘱记录（Provider Order Entry）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| poe_id | INTEGER | 医嘱 ID (PK) |
| poe_seq | INTEGER | 医嘱序号 |
| order_type | VARCHAR | 医嘱类型 |
| order_subtype | VARCHAR | 医嘱子类型 |
| order_priority | VARCHAR | 医嘱优先级 |
| order_status | VARCHAR | 医嘱状态 |
| transaction_type | VARCHAR | 交易类型 |
| discontinue | VARCHAR | 停止标志 |
| discontinue_time | TIMESTAMP | 停止时间 |
| ordertime | TIMESTAMP | 医嘱时间 |
| storetime | TIMESTAMP | 存储时间 |

---

### poe_detail

医嘱详细记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| poe_id | INTEGER | 医嘱 ID (FK → poe) |
| poe_seq | INTEGER | 医嘱序号 (FK → poe) |
| parent_field | VARCHAR | 父字段 |
| parent_field_ordinal | INTEGER | 父字段序号 |
| field | VARCHAR | 字段名 |
| value | VARCHAR | 值 |

---

### microbiologyevents

微生物培养结果。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| microspecimen_id | BIGINT | 标本 ID (PK) |
| charttime | TIMESTAMP | 采集时间 |
| chartdate | DATE | 采集日期 |
| spec_type | VARCHAR | 标本类型 ID |
| spec_type_desc | VARCHAR | 标本类型描述 |
| order_id | INTEGER | 医嘱 ID |
| test_seq | INTEGER | 测试序号 |
| storetime | TIMESTAMP | 存储时间 |
| org_itemid | INTEGER | 微生物 ID |
| org_name | VARCHAR | 微生物名称 |
| isolate_num | INTEGER | 分离株编号 |
| ab_itemid | INTEGER | 抗生素 ID |
| ab_name | VARCHAR | 抗生素名称 |
| dilution_text | VARCHAR | 稀释度文本 |
| dilution_comparison | VARCHAR | 稀释比较符 |
| dilution_value | DOUBLE | 稀释数值 |
| interpretation | VARCHAR | 敏感度解释 |

---

### hcpcsevents

HCPCS（医疗保健通用程序编码系统）事件。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| hcpcs_cd | VARCHAR | HCPCS 代码 |
| seq_num | INTEGER | 序号 |
| chartdate | DATE | 日期 |
| charttime | TIMESTAMP | 时间 |

---

### d_hcpcs

HCPCS 字典。

| 字段 | 类型 | 说明 |
|------|------|------|
| hcpcs_cd | VARCHAR | HCPCS 代码 (PK) |
| hcpcs_dscr | VARCHAR | HCPCS 描述 |

---

### drgcodes

诊断相关分组（DRG）代码。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| drg_code | VARCHAR | DRG 代码 |
| description | VARCHAR | DRG 描述 |
| drg_severity | INTEGER | DRG 严重程度 |
| drg_mortality | INTEGER | DRG 死亡率 |

---

### services

住院期间的服务（如医疗团队）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| trans_id | INTEGER | 转移 ID |
| prev_service | VARCHAR | 前服务 |
| curr_service | VARCHAR | 当前服务 |
| transftime | TIMESTAMP | 转移时间 |

---

### provider

提供者（医生）信息。

| 字段 | 类型 | 说明 |
|------|------|------|
| provider_id | VARCHAR | 提供者 ID (PK) |
| provider_title | VARCHAR | 标题 |
| provider_name | VARCHAR | 姓名 |
| provider_role | VARCHAR | 角色 |

---

### omr

在线医疗记录（Online Medical Record）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| seq_num | INTEGER | 序号 |
| chartdate | DATE | 日期 |
| charttime | TIMESTAMP | 时间 |
| column_id | VARCHAR | 列 ID |
| column_name | VARCHAR | 列名称 |
| result_name | VARCHAR | 结果名称 |
| result_value | VARCHAR | 结果值 |

---

## icu 模块（ICU 级数据）

### icustays

每次 ICU 住院一行。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | ICU 住院 ID (PK) |
| first_careunit | VARCHAR | 首个护理单元 |
| last_careunit | VARCHAR | 最后护理单元 |
| intime | TIMESTAMP | ICU 入住时间 |
| outtime | TIMESTAMP | ICU 离开时间 |
| los | DOUBLE | ICU 停留天数 |

**重要**: MIMIC-IV 使用 `stay_id` 作为 ICU 住院标识（MIMIC-III 使用 `icustay_id`）。

---

### d_items

生命体征项目字典。

| 字段 | 类型 | 说明 |
|------|------|------|
| itemid | INTEGER | 项目 ID (PK) |
| label | VARCHAR | 项目名称 |
| abbreviation | VARCHAR | 缩写 |
| category | VARCHAR | 类别 |
| unitname | VARCHAR | 单位 |
| param_type | VARCHAR | 参数类型（Numeric/Text/DateTime）|
| conceptid | INTEGER | OMOP CDM 概念 ID |

---

### chartevents

生命体征和护理记录（数据量最大）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | ICU 住院 ID (FK → icustays) |
| itemid | INTEGER | 项目 ID (FK → d_items) |
| charttime | TIMESTAMP | 记录时间 |
| storetime | TIMESTAMP | 存储时间 |
| cgid | INTEGER | 护理 ID (FK → caregiver) |
| value | VARCHAR | 文本值 |
| valuenum | DOUBLE | 数值 |
| valueuom | VARCHAR | 单位 |
| warning | INTEGER | 警告标志 |
| error | INTEGER | 错误标志 |
| resultstatus | VARCHAR | 结果状态 |
| stopped | VARCHAR | 停止标志 |

**注意**: `chartevents` 表通过 `stay_id` 关联 ICU 住院。

---

### datetimeevents

日期时间事件（MIMIC-IV 新增表）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | ICU 住院 ID (FK → icustays) |
| itemid | INTEGER | 项目 ID (FK → d_items) |
| cgid | INTEGER | 护理 ID (FK → caregiver) |
| charttime | TIMESTAMP | 记录时间 |
| storetime | TIMESTAMP | 存储时间 |
| value | VARCHAR | 值 |

---

### caregiver

护理提供者信息。

| 字段 | 类型 | 说明 |
|------|------|------|
| cgid | INTEGER | 护理 ID (PK) |
| label | VARCHAR | 标签 |
| description | VARCHAR | 描述 |
| isdoctor | VARCHAR | 是否医生 |

---

### inputevents

输入事件（MIMIC-IV 中合并了 MIMIC-III 的 `inputevents_mv` 和 `inputevents_cv`）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | ICU 住院 ID (FK → icustays) |
| itemid | INTEGER | 项目 ID (FK → d_items) |
| starttime | TIMESTAMP | 开始时间 |
| endtime | TIMESTAMP | 结束时间 |
| cgid | INTEGER | 护理 ID (FK → caregiver) |
| amount | DOUBLE | 数量 |
| amountuom | VARCHAR | 数量单位 |
| rate | DOUBLE | 速率 |
| rateuom | VARCHAR | 速率单位 |
| storetime | TIMESTAMP | 存储时间 |
| orderid | INTEGER | 医嘱 ID |
| linkorderid | INTEGER | 关联医嘱 ID |
| ordercategorydescription | VARCHAR | 医嘱类别 |
| secondaryordercategorydescription | VARCHAR | 次级医嘱类别 |
| ordercomponenttypedescription | VARCHAR | 组件类型 |
| isopenbag | INTEGER | 开放袋标志 |
| continueinnextdept | INTEGER | 跨科室延续标志 |
| cancelreason | VARCHAR | 取消原因 |
| statusdescription | VARCHAR | 状态描述 |
| comments_editdby | VARCHAR | 修改者 |
| comments_canceledby | VARCHAR | 取消者 |

**重要**: MIMIC-IV 中 `inputevents_mv` 和 `inputevents_cv` 合并为 `inputevents`。

---

### ingredientevents

成分事件（MIMIC-IV 新增表，记录药物的活性成分）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | ICU 住院 ID (FK → icustays) |
| itemid | INTEGER | 项目 ID (FK → d_items) |
| starttime | TIMESTAMP | 开始时间 |
| endtime | TIMESTAMP | 结束时间 |
| cgid | INTEGER | 护理 ID (FK → caregiver) |
| amount | DOUBLE | 数量 |
| amountuom | VARCHAR | 数量单位 |
| orderid | INTEGER | 医嘱 ID |
| linkorderid | INTEGER | 关联医嘱 ID |
| ordercategorydescription | VARCHAR | 医嘱类别 |
| statusdescription | VARCHAR | 状态描述 |
| storetime | TIMESTAMP | 存储时间 |

---

### outputevents

输出事件（尿量、引流量等）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | ICU 住院 ID (FK → icustays) |
| itemid | INTEGER | 项目 ID (FK → d_items) |
| charttime | TIMESTAMP | 记录时间 |
| cgid | INTEGER | 护理 ID (FK → caregiver) |
| value | DOUBLE | 数值 |
| valueuom | VARCHAR | 单位 |
| storetime | TIMESTAMP | 存储时间 |

---

### procedureevents

操作事件（MIMIC-IV 中合并了 MIMIC-III 的 `procedureevents_mv`）。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | ICU 住院 ID (FK → icustays) |
| itemid | INTEGER | 项目 ID (FK → d_items) |
| starttime | TIMESTAMP | 开始时间 |
| endtime | TIMESTAMP | 结束时间 |
| cgid | INTEGER | 护理 ID (FK → caregiver) |
| ordercategorydescription | VARCHAR | 医嘱类别 |
| ordercategory | VARCHAR | 医嘱类别代码 |
| secondaryordercategorydescription | VARCHAR | 次级类别 |
| type | VARCHAR | 类型 |
| storetime | TIMESTAMP | 存储时间 |

**重要**: MIMIC-IV 中 `procedureevents_mv` 改为 `procedureevents`。

---

## ed 模块（急诊科数据）

MIMIC-IV 新增模块，包含急诊科数据。

### edstays

每次急诊科停留一行。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | 急诊科停留 ID (PK) |
| intime | TIMESTAMP | 急诊科进入时间 |
| outtime | TIMESTAMP | 急诊科离开时间 |

---

### edcharting

急诊科记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| stay_id | INTEGER | 急诊科停留 ID (FK → edstays) |
| itemid | INTEGER | 项目 ID |
| charttime | TIMESTAMP | 记录时间 |
| value | VARCHAR | 值 |

---

## cxr 模块（胸片数据）

MIMIC-IV 新增模块，包含 MIMIC-CXR 的元数据。

### cxr_records

胸片记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| study_id | BIGINT | 研究 ID (PK) |
| dicom_id | BIGINT | DICOM ID |
| grid | VARCHAR | 网格 |
| studytime | TIMESTAMP | 研究时间 |
| patientid | VARCHAR | 患者 ID（DICOM）|
| studydate | DATE | 研究日期 |
| studyprocedure | VARCHAR | 研究流程 |
| seriesdescription | VARCHAR | 序列描述 |
| imagecount | INTEGER | 图像数量 |
| modality | VARCHAR | 模态 |
| sop_count | INTEGER | SOP 数量 |
| study_negativescore | INTEGER | 研究负分 |
| study_neutralscore | INTEGER | 研究中立分 |
| study_positivescore | INTEGER | 研究正分 |

---

### cxr_paths

胸片文件路径。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| study_id | BIGINT | 研究 ID (FK → cxr_records) |
| dicom_id | BIGINT | DICOM ID |
| path | VARCHAR | 文件路径 |

---

## note 模块（临床笔记）

MIMIC-IV 新增模块，包含去标识化的临床笔记。

### discharges

出院摘要。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| note_id | VARCHAR | 笔记 ID (PK) |
| charttime | TIMESTAMP | 记录时间 |
| text | TEXT | 笔记内容 |

---

### echos

超声心动图报告。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| hadm_id | INTEGER | 住院 ID (FK → admissions) |
| note_id | VARCHAR | 笔记 ID (PK) |
| charttime | TIMESTAMP | 记录时间 |
| text | TEXT | 笔记内容 |

---

## ecg 模块（心电图数据）

MIMIC-IV 新增模块，包含心电图数据。

### ecg_records

心电图记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| study_id | BIGINT | 研究 ID (PK) |
| ecgtime | TIMESTAMP | 心电图时间 |
| patientid | VARCHAR | 患者 ID（ECG 系统）|
| studydate | DATE | 研究日期 |
| studytime | VARCHAR | 研究时间 |
| waveform_count | INTEGER | 波形数量 |
| sample_base | INTEGER | 采样基线 |
| sample_count | INTEGER | 采样数量 |

---

### ecg_paths

心电图文件路径。

| 字段 | 类型 | 说明 |
|------|------|------|
| subject_id | INTEGER | 患者 ID (FK → patients) |
| study_id | BIGINT | 研究 ID (FK → ecg_records) |
| path | VARCHAR | 文件路径 |

---

## 表间关系

```
patients (subject_id) [PK]
  |
  +-- admissions (hadm_id) [PK]
  |     |
  |     +-- labevents → d_labitems
  |     +-- diagnoses_icd → d_icd_diagnoses
  |     +-- procedures_icd → d_icd_procedures
  |     +-- prescriptions
  |     +-- pharmacy
  |     +-- emar → emar_detail
  |     +-- poe → poe_detail
  |     +-- microbiologyevents
  |     +-- hcpcsevents → d_hcpcs
  |     +-- drgcodes
  |     +-- transfers
  |     +-- services
  |     +-- omr
  |     |
  |     +-- icustays (stay_id) [PK]
  |     |     |
  |     |     +-- chartevents → d_items
  |     |     +-- datetimeevents → d_items
  |     |     +-- inputevents → d_items
  |     |     +-- ingredientevents → d_items
  |     |     +-- outputevents → d_items
  |     |     +-- procedureevents → d_items
  |     |     +-- caregiver
  |     |
  |     +-- edstays (stay_id) [PK]
  |           |
  |           +-- edcharting
  |
  +-- cxr_records → cxr_paths
  +-- ecg_records → ecg_paths
  +-- discharges (note module)
  +-- echos (note module)
```

**关联路径**：
- `patients.subject_id` = `admissions.subject_id` (1:N，一个患者可以有多次住院)
- `admissions.hadm_id` = `icustays.hadm_id` (1:N，一次住院可有多次 ICU 入住)
- `icustays.stay_id` = `chartevents.stay_id` (1:N，一次 ICU 住院可有多个生命体征记录)
- `patients.subject_id` = `labevents.subject_id` (1:N，一个患者可有多个实验室检查)

---

## 重要说明

### 1. 时间表示

MIMIC-IV 使用 **绝对时间戳**（`TIMESTAMP` 类型），不同于 eICU 的偏移量表示法。

**优点**: 可以直接使用时间函数（如 `EXTRACT`, `DATE_PART`, `INTERVAL`）。
**缺点**: 需要进行时区处理（所有时间为 UTC）。

```sql
-- 提取 ICU 第一天的数据
SELECT *
FROM chartevents ce
INNER JOIN icustays ic ON ce.stay_id = ic.stay_id
WHERE ce.charttime BETWEEN ic.intime AND ic.intime + INTERVAL '24 hours'
```

---

### 2. 患者标识

| 标识 | 说明 | 使用场景 |
|------|------|----------|
| `subject_id` | 患者 ID | 跨多次住院标识同一患者 |
| `hadm_id` | 住院 ID | 标识一次住院 |
| `stay_id` | ICU 住院 ID | 标识一次 ICU 住院（MIMIC-IV 新增）|
| `edstay_id` | 急诊科停留 ID | 标识一次急诊科停留 |

**注意**: MIMIC-III 使用 `icustay_id`，MIMIC-IV 改为 `stay_id`。

---

### 3. 字段命名规范

✅ **正确** (小写):
```sql
SELECT ce.subject_id, ce.stay_id, ce.itemid, ce.charttime
FROM chartevents ce
WHERE ce.stay_id = 100001
```

❌ **错误** (camelCase):
```sql
SELECT ce.subjectId, ce.stayId  -- 字段不存在！
FROM chartevents ce
```

---

### 4. 数据类型

- 数值型: `INTEGER`, `BIGINT`, `DOUBLE`, `NUMERIC`
- 字符型: `VARCHAR`, `TEXT`
- 日期时间: `TIMESTAMP`, `DATE`
- 布尔型: `INTEGER` (0/1) 或 `VARCHAR` ('true'/'false')

---

### 5. 常用查询优化建议

- 使用 `subject_id`, `hadm_id`, `stay_id` 进行表连接
- 对时间范围查询使用 `charttime`, `intime`, `outtime` 等字段
- 大表（如 `chartevents`, `labevents`）查询时务必加 `LIMIT`
- 使用 `EXPLAIN` 分析查询计划
- 使用分区表（如 `chartevents` 按 `stay_id` 分区）

---

### 6. MIMIC-IV vs MIMIC-III 主要区别

| 项目 | MIMIC-III | MIMIC-IV |
|------|-----------|----------|
| ICU 住院 ID | `icustay_id` | `stay_id` |
| 输入事件表 | `inputevents_mv`, `inputevents_cv` | `inputevents` (合并) |
| 操作事件表 | `procedureevents_mv` | `procedureevents` |
| 新增表 | - | `ingredientevents`, `datetimeevents`, `emar`, `emar_detail`, `poe`, `poe_detail` |
| 新增模块 | - | `ed`, `cxr`, `note`, `ecg` |
| 时间表示 | 绝对时间戳 | 绝对时间戳（相同）|

---

## 参考链接

- **MIMIC-IV 官方文档**: https://mimic.mit.edu/docs/IV/
- **MIMIC-IV 数据介绍**: https://physionet.org/content/mimiciv/3.1/
- **MIMIC-IV-ED 数据介绍**: https://physionet.org/content/mimic-iv-ed/2.0/
- **MIMIC-IV-Note 数据介绍**: https://physionet.org/content/mimic-iv-note/2.0/
- **MIMIC-CXR 数据介绍**: https://physionet.org/content/mimic-cxr/2.1/
- **访问申请**: https://physionet.org/register/

---

**最后更新**: 2026-05-20  
**更新人**: 悟空（基于 MIMIC-IV 官方文档修正）
