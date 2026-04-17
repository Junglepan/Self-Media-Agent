# CLAUDE.md — Claude Code 在本仓库的工作指引

> 精简指引。深入细节请读 [`README.md`](README.md)、[`schema/update-rules.md`](schema/update-rules.md)、[`schema/dimensions.md`](schema/dimensions.md)。

---

## 项目性质

**Agent-native 摄影师角色系统**——零代码，全 Markdown。
- `prompts/` = Agent 的"动作"（9 个 skill）
- `wiki/`    = Agent 的"记忆"（7 维度）
- `schema/`  = Agent 的"宪法"（人类维护）
- `raw/`     = 事实层（append-only）
- `state/`   = 运行状态（指针 + 历史流水 + 已发布索引）
- `mcp/`     = 外部工具规范

**重要**：本仓库**不要引入代码**（.py / .js / .ts 等）。所有逻辑由 Agent 读取 skill + schema 后自主执行。

---

## 操作边界

| 目录/文件 | Claude Code 可以做什么 |
|-----------|--------------------|
| `raw/**/*.md` | **只追加**，永不修改已存在文件（铁律 #1） |
| `wiki/**/*.md` | 通过 `/skill learn` / `/skill converge` / `/skill seed` 修改；`/skill photographer` 等创作/分析链路**只读** |
| `schema/*.md` | **Agent 运行时禁止修改**（Run Loop 铁律 #6）；人类用户显式要求修改 schema 时需 review |
| `state/last_run.json` | 顶层指针可覆盖；`runs[]` 只 append，不改历史 |
| `state/published.jsonl` | append-only；harvest 可"整文件重写保持行序 + 仅更新目标行指定字段" |
| `prompts/*.md` | 修改需明确用户请求；单改会影响所有后续 /skill 行为 |
| `.github/workflows/*.yml` | 修改前询问（涉及 CI/CD） |

---

## 触发 skill

用户输入 `/skill <name>`（或直接描述意图）时，Agent 读取对应 `prompts/<name>.md` 执行。

| Skill | 作用 | 读写面 |
|-------|------|------|
| `photographer` | 创作文案 | 只读 wiki |
| `publish` | 发布到小红书 | append `state/published.jsonl` |
| `harvest` | 回流自己帖子互动数据 | append `raw/harvest-*.md`，更新 `published.jsonl` |
| `learn` | 学习提炼（10 步 Run Loop） | `raw/`（append）+ `wiki/` + `state/last_run.json`（append run） |
| `seed` | 手动注入单帖 | 同 learn |
| `converge` | 处理 `[待收敛]` | 修改 `wiki/` + `persona.md` |
| `audit` | 质量审计 | 只读 |
| `status` | 系统状态速查 | 只读 |
| `analyze` | 拆解文案结构 | 只读 |

---

## 铁律速记

1. `raw/` 只追加；2. wiki 条目必须有 evidence；3. 置信度 ⭐/⭐⭐/⭐⭐⭐ 依 evidence 数量；4. 矛盾保留 `[待收敛]`，不强行统一；5. 禁止把原帖文字直接复制进 wiki（抽象再写）；6. `schema/update-rules.md` 是学习行为**唯一准则**，Run Loop 中禁止修改 schema/。

---

## 常见路径

| 需求 | 看这里 |
|------|------|
| 角色当前画像 | `wiki/persona.md` |
| 学习规则（Run Loop 10 步 / Five Gates） | `schema/update-rules.md` |
| 维度归属速查 | `schema/dimensions.md` 末尾 |
| 小红书 MCP 工具清单 | `mcp/xiaohongshu.md` |
| 最近一轮结果 | `state/last_run.json` 的 `runs[]` 末尾 |
| 已发布帖子 | `state/published.jsonl`（一行一个 JSON） |
| 定时任务 | `.github/workflows/learn.yml`（周一 UTC 02:00 跑 audit） |

---

## 协作约束

- **raw/ 目录结构**：`raw/<feed_id>.md` + `raw/images/<feed_id>-<n>.webp`（扁平结构，feed_id 全局唯一）
- **feed_id 去重**：学习链路 Step 2 会跳过已存在的 feed_id（见 update-rules）
- **self_post 标注**：harvest 回流的 raw 带 `source: self_post`；后续 learn 提炼时 evidence 标 `（自验证）`
- **语言**：默认中文回答（见全局 CLAUDE.md）
- **minimal diff**：修 schema / prompts 时保持小改动；大改动必须先与用户对齐

---

## 新功能检查清单

添加/修改 skill 时自检：
- [ ] 是否修改了 `raw/` 中已存在文件？（不允许）
- [ ] 是否引入了任何代码文件？（不允许）
- [ ] `prompts/` 里规则是否与 `schema/update-rules.md` 冲突？（schema 优先）
- [ ] 新 skill 是否需要 append `state/last_run.json` 的 `runs[]`？（seed/learn/converge/harvest 需要；只读 skill 不需要）
- [ ] README 与 `dimensions.md` 的维度定义是否同步？
