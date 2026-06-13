# mimic-skill 🏥

> OpenClaw Skill for querying the **MIMIC-IV** critical care database

An [OpenClaw](https://github.com/openclaw/openclaw) skill that helps AI agents query the [MIMIC-IV](https://mimic.mit.edu/docs/IV/) clinical database. When a user asks for ICU patient data — vital signs, lab results, diagnoses, comorbidities — the agent reads the skill's reference docs and generates ready-to-run SQL or Python code tailored to the request.

## How It Works

```
User asks: "提取 MIMIC-IV 中 AMI 患者第一天的生命体征和肌酐"
        ↓
Agent loads mimic-skill → reads relevant reference files
        ↓
Agent generates custom SQL/Python code with correct table names, itemid mappings, and filters
        ↓
User runs the code against their own PostgreSQL database
```

**This skill does NOT connect to any database.** It provides the knowledge (schema, field mappings, query patterns) so that an AI agent can write correct queries. Users run the queries themselves against their own MIMIC-IV instance.

## What It Generates

Depending on the user's request, the agent will produce:

| Output | When | Example |
|--------|------|---------|
| **SQL queries** | User wants to run directly in psql / pgAdmin / DBeaver | `SELECT ... FROM chartevents WHERE stay_id = ...` |
| **Python scripts** | User wants programmatic access via psycopg2 + pandas | `extract_vital_signs(patient_ids=[...])` |
| **Data extraction pipeline** | Complex multi-step requests | Combined scripts that join vitals + labs + diagnoses into a flat table |

## Installation

### Option 1: Via SkillHub (recommended)

```bash
openclaw skill install mimic-skill
```

### Option 2: Clone into skills directory

```bash
git clone https://github.com/yongfanbeta/mimic-skill.git ~/.qclaw/skills/mimic-skill
```

After installation, the skill is automatically available in your OpenClaw agent. No additional configuration needed.

## File Structure

```
mimic-skill/
├── SKILL.md                        # Core skill definition & usage guide
├── README.md                       # This file
├── LICENSE                         # MIT
└── references/
    ├── schema.md                   # Full database schema (6 modules: hosp, icu, ed, cxr, note, ecg)
    ├── vital_signs.md              # Vital signs: HR, BP, SpO2, Temperature, RR + itemid mappings
    ├── labs.md                     # Lab results: Creatinine, BUN, Lactate, WBC, etc. + itemid mappings
    ├── diagnoses.md                # ICD diagnoses & comorbidity extraction
    └── common_queries.md           # SOFA score, mechanical ventilation, vasopressors, AKI staging, Sepsis-3
```

## What's Inside

### Key References

| File | What It Covers |
|------|---------------|
| `schema.md` | All 6 MIMIC-IV modules, table relationships, field definitions |
| `vital_signs.md` | `chartevents` itemid → vital sign mapping, 1st/24h/entire-stay extraction templates |
| `labs.md` | `labevents` itemid → lab test mapping, common lab panels |
| `diagnoses.md` | ICD-9/10 code lookup, comorbidity (Charlson/Elixhauser) extraction |
| `common_queries.md` | Pre-built concepts: SOFA, ventilation status, vasopressors, KDIGO AKI, Sepsis-3 |

### MIMIC-III → MIMIC-IV Migration

The skill includes a built-in migration guide covering:
- `icustay_id` → `stay_id`
- `inputevents_mv/cv` → `inputevents` (merged)
- New tables: `ingredientevents`, `datetimeevents`, `emar`, `poe`
- New modules: `ed`, `cxr`, `note`, `ecg`
- Field naming: all lowercase, snake_case

## Usage Examples

After installing, just talk to your OpenClaw agent naturally:

| What You Say | What The Agent Does |
|-------------|-------------------|
| "帮我从 MIMIC-IV 提取 AMI 患者第一天的生命体征" | Reads `vital_signs.md`, generates SQL with correct itemid filters |
| "MIMIC 里怎么算 SOFA 评分？" | Reads `common_queries.md`, outputs SOFA calculation query |
| "我要提取肌酐和 BUN 的趋势数据" | Reads `labs.md`, generates Python script with itemid mapping |
| "MIMIC-III 的 icustay_id 在 IV 里叫什么？" | Answers from migration guide: `stay_id` |
| "帮我筛 Sepsis-3 患者" | Reads `common_queries.md`, generates Sepsis-3 identification query |

## Important Notes

- **No database access**: This skill only provides query knowledge. You need your own MIMIC-IV PostgreSQL instance.
- **Access required**: Apply for MIMIC-IV access at [PhysioNet](https://physionet.org/register/).
- **Big table warning**: `chartevents` has hundreds of millions of rows. Always filter by `stay_id` + time range.
- **Age truncation**: `anchor_age > 89` is set to 91 (per MIMIC policy).

## References

- [MIMIC-IV Documentation](https://mimic.mit.edu/docs/IV/)
- [MIMIC-IV on PhysioNet](https://physionet.org/content/mimiciv/3.1/)
- [MIMIC Code Repository](https://github.com/MIT-LCP/mimic-code)

## License

MIT
