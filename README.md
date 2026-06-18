# MIMIC-Skill

从 MIMIC-IV 重症监护数据库提取 ICU 患者数据的 OpenClaw Skill / Claude Code Plugin。

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

### Claude Code 用户

**方法1：从统一市场安装（推荐）**

```bash
# 步骤1：添加 OAMD 插件市场（包含 mimic-skill 和 eicu-skill）
/plugin marketplace add https://github.com/yongfanbeta/OAMD-marketplace

# 步骤2：安装 mimic-skill
/plugin install mimic-skill

# 步骤3：重载插件系统
/reload-plugins
```

**方法2：从 GitHub 仓库直接安装（单个仓库）**

```bash
# 添加单个仓库作为 marketplace
/plugin marketplace add https://github.com/yongfanbeta/mimic-skill

# 安装插件
/plugin install mimic-skill
```

**方法3：本地开发模式（测试用）**

```bash
# 克隆仓库到本地
git clone https://github.com/yongfanbeta/mimic-skill.git ~/.claude/plugins/mimic-skill

# 启动 Claude Code 并加载插件
claude --plugin-dir ~/.claude/plugins/mimic-skill
```

### Codex 用户

**Codex App：**
1. 点击侧边栏的 **Plugins**
2. 搜索 `mimic-skill`
3. 点击 **+** 安装

**Codex CLI：**
```bash
# 搜索插件
/plugins

# 搜索 mimic-skill
mimic-skill

# 选择 Install Plugin
```

### Cursor 用户

在 Cursor Agent 聊天中输入：
```bash
/add-plugin mimic-skill
```

或者：
1. 打开插件市场
2. 搜索 `mimic-skill`
3. 点击安装

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
   - 新表：`ingredients`, `datetimeevents`, `emar` 等
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
- [访问申请](https://physionet.org/register/)
- [MIMIC Code 仓库](https://github.com/MIT-LCP/mimic-code)
- [eICU-Code 官方仓库](https://github.com/MIT-LCP/eicu-code)

## 📦 发布文件

- **GitHub Releases**: [v1.0.0](https://github.com/yongfanbeta/mimic-skill/releases/tag/v1.0.0)
- **ClawHub**: [mimic-skill](https://clawhub.ai/skills/mimic-skill)

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📄 许可证

MIT License
