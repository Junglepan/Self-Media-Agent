# Self-Media-Agent

> 一个持续进化的**摄影师角色知识库**。
> 核心理念来自 Karpathy 的 LLM Wiki 思想：**原始数据不是知识，提炼后的 Wiki 才是**。

---

## 它是什么

Self-Media-Agent 是一个 GitHub 仓库，同时是三样东西：

1. **知识库** — 存储从小红书高赞摄影帖子中提炼的角色知识
2. **角色载体** — 一位有人格、有审美、有世界观的摄影师
3. **创作工具** — 用户描述场景，角色输出文案

---

## 两条独立链路

### 链路一：自动学习（后台）

Cloud Routine 每天定时运行：

```
小红书 MCP Connector → 高赞摄影帖子 → raw/（只追加）
                                      ↓
                              按 schema/ 规则提炼
                                      ↓
                                 写入 wiki/
                                      ↓
                          重写 wiki/persona.md
                                      ↓
                        更新 state/last_run.json
```

Agent 每次运行的唯一准则：`schema/update-rules.md`。

### 链路二：按需创作（用户触发）

```
用户 /skill photographer 描述场景
        ↓
Claude 读取 prompts/photographer.md
        ↓
按需检索 wiki/ 中的条目
        ↓
以角色身份输出文案
```

创作链路**只读** wiki/，不修改任何文件。

---

## 角色由七个维度构成

| 维度 | 文件 | 作用 |
|------|------|------|
| 人格 | `wiki/persona.md` | 综合摘要（主入口） |
| 审美 | `wiki/aesthetics/` | 构图 / 光线 / 色彩 |
| 技能 | `wiki/skills/` | 文案 / 叙事 |
| 学识 | `wiki/knowledge/` | 摄影理论 / 人文 |
| 世界观 | `wiki/worldview.md` | 拍摄动机与价值判断 |
| 成长轨迹 | `wiki/growth.md` | 演变历史 |
| 参照系 | `wiki/references.md` | 师承、同道、反对面 |

详见 [`schema/dimensions.md`](schema/dimensions.md)。

---

## 仓库结构

```
Self-Media-Agent/
├── raw/                              # 原始数据层（只追加，永不修改）
│   └── YYYY-MM-DD-<post-id>.md
│
├── wiki/                             # 角色知识层（LLM 持续维护）
│   ├── persona.md
│   ├── worldview.md
│   ├── growth.md
│   ├── references.md
│   ├── aesthetics/{composition,light,color}.md
│   ├── skills/{copywriting,storytelling}.md
│   └── knowledge/{photography,culture}.md
│
├── schema/                           # 角色构造手册（规则层，不常变）
│   ├── update-rules.md               # 学习规则：如何提炼、如何写入
│   ├── dimensions.md                 # 七维度定义与文件模板
│   └── quality-bar.md                # 入库质量标准
│
├── state/
│   └── last_run.json                 # 运行状态（游标、统计）
│
└── prompts/
    └── photographer.md               # Claude Code Skill 入口
```

---

## 六条铁律

1. **raw/ 只追加**，永不修改
2. **wiki/ 的每个条目必须有例句**，纯抽象描述不入库
3. 置信度用星级标注：⭐=1次 ⭐⭐=2–3次 ⭐⭐⭐=4次以上
4. 发现矛盾保留两种，标注 `[待收敛]`，不强行统一
5. 禁止把原帖文案直接复制进 wiki
6. `schema/update-rules.md` 是学习行为的**唯一准则**，每次运行必读

---

## 快速导航

- 想理解**学习行为**如何展开 → [`schema/update-rules.md`](schema/update-rules.md)
- 想理解**七维度**的边界 → [`schema/dimensions.md`](schema/dimensions.md)
- 想理解**条目入库的标准** → [`schema/quality-bar.md`](schema/quality-bar.md)
- 想**使用角色创作** → [`prompts/photographer.md`](prompts/photographer.md)
- 想看角色的**当前画像** → [`wiki/persona.md`](wiki/persona.md)

---

## 状态

当前处于**初始化阶段**：仓库骨架已就位，`raw/` 为空，`wiki/` 为占位模板。
首轮 Run Loop 执行后，角色才开始积累人格。
