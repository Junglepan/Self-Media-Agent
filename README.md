# Self-Media-Agent

> 一个 **Agent-native** 的摄影师角色系统。
> Agent 就是运行时——Skills 是它的动作，Wiki 是它的记忆，Schema 是它的行为宪法。

---

## 核心理念

**原始数据不是知识，提炼后的 Wiki 才是。**（Karpathy LLM Wiki）

这个仓库里没有传统意义上的代码——没有解析器、没有 ETL 脚本、没有硬编码逻辑。
**Agent 本身就是执行引擎**。每一个动作都由 Agent 读取 Skill 文件后自主完成。

```
Skills  = 告诉 Agent "现在做什么"（动作）
Wiki    = 告诉 Agent "你知道什么"（记忆）
Schema  = 告诉 Agent "你是谁、怎么做事"（宪法）
```

---

## Agent 的能力（Skills）

| Skill | 触发方式 | 做什么 |
|-------|----------|--------|
| `photographer` | `/skill photographer` | 用户描述场景，角色输出文案 |
| `learn` | `/skill learn` 或定时调度 | 消化新素材，提炼知识写入 wiki |
| `analyze` | `/skill analyze` | 拆解一段文案的结构/技巧/审美 |
| `seed` | `/skill seed` | 手动注入一篇帖子，立即提炼 |
| `converge` | `/skill converge` | 处理 wiki 中的矛盾条目 |
| `audit` | `/skill audit` | Wiki 质量审计，出报告 |
| `status` | `/skill status` | 一眼看清角色当前积累状态 |

所有 Skill 的完整指令在 [`prompts/`](prompts/) 目录。

### Skill 之间的关系

```
定时 / 手动
    │
    ▼
/skill learn ──► 消化 raw/ → 更新 wiki/ → 重写 persona.md
    │                                          │
    │            /skill seed ──────────────────┘
    │            （手动注入单篇）
    │
    ├── 有 [待收敛]？ ──► /skill converge（收敛矛盾）
    ├── 质量有疑问？  ──► /skill audit（只出报告）
    └── 想看现状？   ──► /skill status（只读）

用户描述场景
    │
    ▼
/skill photographer ──► 检索 wiki/（只读）──► 输出文案
/skill analyze      ──► 检索 wiki/（只读）──► 输出分析
```

---

## Agent 的记忆（Wiki）

角色由七个维度构成，所有维度持续被 `/skill learn` 和 `/skill seed` 喂养和演化：

| 维度 | 文件 | 内容 |
|------|------|------|
| 人格 | `wiki/persona.md` | 综合摘要，每轮学习后重写 |
| 审美 | `wiki/aesthetics/` | 构图 / 光线 / 色彩偏好 |
| 技能 | `wiki/skills/` | 文案技巧 / 叙事结构 |
| 学识 | `wiki/knowledge/` | 摄影理论 / 人文联结 |
| 世界观 | `wiki/worldview.md` | 拍摄动机与价值判断 |
| 成长轨迹 | `wiki/growth.md` | 演变历史，每轮追加 |
| 参照系 | `wiki/references.md` | 师承 / 同道 / 反对面 |

> Wiki 里的每个条目都必须有原帖 evidence 支撑，并用星级标注置信度（⭐ / ⭐⭐ / ⭐⭐⭐）。

---

## Agent 的宪法（Schema）

| 文件 | 内容 |
|------|------|
| `schema/update-rules.md` | 学习流程的唯一准则（10 步 Run Loop + 7 条铁律） |
| `schema/dimensions.md` | 七维度边界定义 + 条目模板 + 归属速查 |
| `schema/quality-bar.md` | 五项入库 Gate 检查 + 通过率监控 |

Schema 由人类维护，Agent 不在常规运行中修改 schema/。

---

## 外部工具（MCP）

Agent 通过 MCP 访问外部系统。当前接入：

| Server | 规范 | 用途 |
|--------|------|------|
| 小红书 Connector | [`mcp/xiaohongshu.md`](mcp/xiaohongshu.md) | 拉取高赞摄影帖子 |

MCP 仅在 `/skill learn` 的 Step 2 中调用；创作/分析链路不依赖 MCP。
降级行为：MCP 不可用时，`/skill learn` 跳过拉取，仅重写 persona.md。

配置方式见 [`mcp/README.md`](mcp/README.md)。

---

## 调度

### 自动调度（GitHub Actions）

`.github/workflows/learn.yml` 每天 **UTC 02:00**（北京时间 10:00）自动触发 `/skill learn`。

### 手动调度

两种方式均可：

```bash
# 方式一：GitHub CLI
gh workflow run learn.yml

# 方式二：GitHub UI
# Actions → Learn — Daily Knowledge Distillation → Run workflow
```

也可以在 Claude Code 终端直接运行：

```bash
/skill learn
```

---

## 仓库结构

```
Self-Media-Agent/
├── prompts/                          # Agent 的动作指令（Skills）
│   ├── photographer.md               # 创作
│   ├── learn.md                      # 学习
│   ├── analyze.md                    # 分析
│   ├── seed.md                       # 手动注入
│   ├── converge.md                   # 矛盾收敛
│   ├── audit.md                      # 质量审计
│   └── status.md                     # 状态查看
│
├── wiki/                             # Agent 的记忆（持续演化）
│   ├── persona.md
│   ├── worldview.md
│   ├── growth.md
│   ├── references.md
│   ├── aesthetics/{composition,light,color}.md
│   ├── skills/{copywriting,storytelling}.md
│   └── knowledge/{photography,culture}.md
│
├── schema/                           # Agent 的行为宪法（人类维护）
│   ├── update-rules.md
│   ├── dimensions.md
│   └── quality-bar.md
│
├── mcp/                              # 外部工具接口规范
│   ├── README.md
│   └── xiaohongshu.md
│
├── raw/                              # 事实层（只追加，永不修改）
│   └── YYYY-MM-DD-<post-id>.md
│
├── state/
│   └── last_run.json                 # 运行游标
│
└── .github/workflows/
    └── learn.yml                     # 定时 + 手动调度
```

---

## 六条铁律

1. `raw/` **只追加**，永不修改
2. wiki/ 每个条目必须有 evidence 支撑
3. 置信度星级：⭐ 1次 / ⭐⭐ 2–3次 / ⭐⭐⭐ 4次+
4. 矛盾保留两种，标注 `[待收敛]`
5. 禁止把原帖文字直接复制进 wiki
6. `schema/update-rules.md` 是学习行为的**唯一准则**

---

## 快速导航

| 目的 | 看这里 |
|------|--------|
| 开始创作 | [`prompts/photographer.md`](prompts/photographer.md) |
| 理解学习流程 | [`prompts/learn.md`](prompts/learn.md) → [`schema/update-rules.md`](schema/update-rules.md) |
| 手动注入帖子 | [`prompts/seed.md`](prompts/seed.md) |
| 查看当前状态 | `/skill status` |
| 配置 MCP | [`mcp/README.md`](mcp/README.md) |
| 看角色当前画像 | [`wiki/persona.md`](wiki/persona.md) |

---

**当前状态**：仓库骨架已就位，`raw/` 为空，`wiki/` 为占位模板。运行第一次 `/skill learn` 或 `/skill seed` 后，角色开始积累人格。
