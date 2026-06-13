# mimic-skill 🏥

> OpenClaw Skill for querying the **MIMIC-IV** clinical database

A comprehensive reference skill that provides SQL and Python (psycopg2) query templates for extracting data from the [MIMIC-IV](https://mimic.mit.edu/docs/IV/) critical care database. Designed for researchers working with ICU patient data including vital signs, laboratory results, diagnoses, and comorbidities.

## ✨ Features

- **MIMIC-IV vs MIMIC-III migration guide** — table name changes, new modules, field naming conventions
- **Ready-to-use SQL & Python templates** — just plug in your connection params
- **6 reference documents** covering the most common query scenarios
- **Performance tips** for querying the billion-row `chartevents` table

## 📁 Structure

```
mimic-skill/
├── SKILL.md                        # Core skill definition & main documentation
├── references/
│   ├── schema.md                   # Full database schema (6 modules: hosp, icu, ed, cxr, note, ecg)
│   ├── vital_signs.md              # Vital signs extraction (Heart Rate, BP, SpO2, etc.)
│   ├── labs.md                     # Laboratory results (Creatinine, BUN, Lactate, etc.)
│   ├── diagnoses.md                # ICD diagnoses & comorbidities
│   └── common_queries.md           # Common queries: SOFA, ventilation, vasopressors, AKI, Sepsis-3
└── scripts/                        # (placeholder for future automation scripts)
```

## 🚀 Quick Start

### 1. Install as an OpenClaw Skill

```bash
# Via SkillHub
openclaw skill install mimic-skill

# Or clone directly into your skills directory
git clone https://github.com/YOUR_USERNAME/mimic-skill.git ~/.qclaw/skills/mimic-skill
```

### 2. Query Example — ICU First Day Vital Signs

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

### 3. Python Example

```python
import psycopg2
import pandas as pd

conn = psycopg2.connect(
    host='localhost', port=5432,
    database='mimiciv', user='your_user', password='your_pass'
)

df = pd.read_sql("""
    SELECT subject_id, stay_id, charttime, valuenum
    FROM chartevents
    WHERE stay_id = %(stay_id)s
      AND itemid IN (220045, 220050)
      AND valuenum IS NOT NULL
""", conn, params={'stay_id': 100001})
```

## ⚠️ Key Conventions

| Concept | MIMIC-IV | MIMIC-III (deprecated) |
|---------|----------|------------------------|
| ICU stay ID | `stay_id` | `icustay_id` |
| Input events | `inputevents` (merged) | `inputevents_mv` / `inputevents_cv` |
| Field naming | lowercase, snake_case | mixed case |
| Time representation | Absolute timestamps (UTC) | Absolute timestamps (UTC) |

- **`anchor_age`**: truncated at 89 (patients >89 show as 91)
- **`chartevents`** has hundreds of millions of rows — always filter by `stay_id` and time range
- Use `valuenum` for numeric results, `value` for text results

## 📋 Supported Query Types

| Type | Reference File | Examples |
|------|---------------|----------|
| Database Schema | `references/schema.md` | All 6 modules, table relationships |
| Vital Signs | `references/vital_signs.md` | HR, BP, SpO2, Temperature, RR |
| Lab Results | `references/labs.md` | Creatinine, BUN, Lactate, WBC, etc. |
| Diagnoses | `references/diagnoses.md` | ICD codes, comorbidity extraction |
| Common Queries | `references/common_queries.md` | SOFA, ventilation, vasopressors, AKI, Sepsis-3 |

## 🔗 References

- [MIMIC-IV Documentation](https://mimic.mit.edu/docs/IV/)
- [MIMIC-IV on PhysioNet](https://physionet.org/content/mimiciv/3.1/)
- [MIMIC Code Repository](https://github.com/MIT-LCP/mimic-code)
- [Data Access Application](https://physionet.org/register/)

## 📄 License

MIT License — see [LICENSE](LICENSE) file.

## 🙏 Acknowledgments

- [MIT Laboratory for Computational Physiology](https://lcp.mit.edu/) for maintaining MIMIC-IV
- The [mimic-code](https://github.com/MIT-LCP/mimic-code) community for shared query concepts
