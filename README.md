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
| `photographer` | `/skill photographer` | 用户描述场景，Agent 以角色身份输出文案 |
| `learn` | `/skill learn` | 读取新素材，提炼知识写入 wiki，重写角色画像 |
| `analyze` | `/skill analyze` | 拆解一段文案的结构、技巧、审美、世界观 |

所有 Skill 的完整指令在 [`prompts/`](prompts/) 目录。

---

## Agent 的记忆（Wiki）

角色由七个维度构成，所有维度持续被 `/skill learn` 喂养和演化：

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

## 两条链路

### 链路一：学习（`/skill learn`）

```
用户或定时触发 /skill learn
        ↓
Agent 读取 schema/update-rules.md（唯一准则）
        ↓
通过 MCP 拉取小红书高赞帖子 → 写入 raw/（只追加）
        ↓
逐帖提炼候选条目 → 五项 Gate 过筛
        ↓
通过的条目合并到 wiki/（查重、升级置信度、标矛盾）
        ↓
重写 wiki/persona.md → 更新 state/last_run.json
```

**raw/ 是事实层，不可修改；wiki/ 是解释层，持续演化。**

### 链路二：创作（`/skill photographer`）

```
用户 /skill photographer 描述场景
        ↓
Agent 读取 prompts/photographer.md
        ↓
按需检索 wiki/（persona → 按场景按需读子文件）
        ↓
以角色身份输出文案（标题 × 3 / 正文 / 标签 / 角色自述）
```

创作链路**只读 wiki，不修改任何文件**。

---

## 仓库结构

```
Self-Media-Agent/
├── prompts/                          # Agent 的动作指令（Skills）
│   ├── photographer.md               # /skill photographer — 创作
│   ├── learn.md                      # /skill learn — 学习
│   └── analyze.md                    # /skill analyze — 分析
│
├── wiki/                             # Agent 的记忆（持续演化）
│   ├── persona.md                    # 角色画像主入口
│   ├── worldview.md
│   ├── growth.md
│   ├── references.md
│   ├── aesthetics/{composition,light,color}.md
│   ├── skills/{copywriting,storytelling}.md
│   └── knowledge/{photography,culture}.md
│
├── schema/                           # Agent 的行为宪法（人类维护）
│   ├── update-rules.md               # 学习行为唯一准则
│   ├── dimensions.md                 # 七维度定义
│   └── quality-bar.md                # 入库质量标准
│
├── raw/                              # 事实层（只追加，永不修改）
│   └── YYYY-MM-DD-<post-id>.md
│
└── state/
    └── last_run.json                 # Agent 的运行游标
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
| 看角色当前画像 | [`wiki/persona.md`](wiki/persona.md) |
| 理解维度边界 | [`schema/dimensions.md`](schema/dimensions.md) |
| 理解入库标准 | [`schema/quality-bar.md`](schema/quality-bar.md) |

---

**当前状态**：仓库骨架已就位，`raw/` 为空，`wiki/` 为占位模板。运行第一次 `/skill learn` 后，角色开始积累人格。
