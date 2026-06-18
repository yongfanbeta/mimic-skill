# MIMIC-Skill

从 MIMIC-IV 重症监护数据库提取 ICU 患者数据的 OpenClaw Skill。

提供 SQL 和 Python (psycopg2) 查询模板，支持提取生命体征、实验室检查、诊断、合并症等数据。

## 🚀 一键安装（推荐）

### OpenClaw / QClaw 用户

```bash
# 方法1：通过 skillhub 一键安装（推荐）
skillhub install mimic-skill

# 方法2：通过 ClawHub 安装
openclaw skill install https://clawhub.ai/skills/mimic-skill

# 方法3：本地安装（已下载 .skill 文件）
openclaw skill install ./mimic-skill.skill
```

### Claude Code / Cursor / Windsurf 用户

```bash
# 下载 .skill 文件
curl -L -o mimic-skill.skill \
  https://github.com/yongfanbeta/mimic-skill/releases/download/v1.0.0/mimic-skill.skill

# 安装（假设你的智能体支持 openclaw CLI）
openclaw skill install mimic-skill.skill
```

### 手动安装（所有智能体通用）

1. 从 [Releases](https://github.com/yongfanbeta/mimic-skill/releases) 下载 `mimic-skill.skill`
2. 将文件重命名为 `mimic-skill.zip` 并解压
3. 将解压后的 `mimic-skill/` 文件夹复制到智能体的 skills 目录（例如 `~/.qclaw/skills/`）
4. 重启智能体

## 📖 使用方法

```python
# 示例：查询 MIMIC-IV 数据
请帮我提取 MIMIC-IV 数据库中第一个 ICU 停留的生命体征数据
```

## 🔍 支持的查询类型

| 查询类型 | 说明 |
|---------|------|
| 生命体征 | 心率、血压、体温、呼吸频率等 |
| 实验室检查 | 肌酐、胆红素、血常规、电解质等 |
| 诊断与合并症 | ICD 编码、APACHE IV 评分等 |
| 数据库 Schema | MIMIC-IV 表结构和字段说明 |

## ⚠️ 重要提示

1. **MIMIC-IV 变化**（vs MIMIC-III）：
   - 表名变化：`icustay_id` → `stay_id`
   - 新表：`ingredientevents`, `datetimeevents`, `emar` 等
   - 模块化：分为 hosp、icu、ed、cxr、note、ecg 6 个模块

2. **数据库连接**：
   - 本 Skill 只提供查询代码模板
   - 用户需自行提供 PostgreSQL 连接参数

3. **字段命名**：
   - 全部小写（例如：`stay_id`, `charttime`）
   - 不要用驼峰命名（例如：`stayId`, `chartTime`）

## 📚 参考资料

- [MIMIC-IV 官方文档](https://mimic.mit.edu/docs/IV/)
- [MIMIC-IV 数据介绍](https://physionet.org/content/mimiciv/3.1/)
- [MIMIC Code 仓库](https://github.com/MIT-LCP/mimic-code)
- [访问申请](https://physionet.org/register/)

## 📦 发布文件

- **GitHub Releases**: [v1.0.0](https://github.com/yongfanbeta/mimic-skill/releases/tag/v1.0.0)
- **ClawHub**: [mimic-skill](https://clawhub.ai/skills/mimic-skill)

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📄 许可证

MIT License
