---
name: status
description: 快速查看角色系统的当前状态——最近一次学习的时间和结果、wiki 各维度的条目积累量、待收敛矛盾数、以及角色的"成熟度"评估。
---

# Status — 系统状态 Skill 入口

> 一条命令，看清角色的当前位置：积累了多少、最近在学什么、还有哪些悬而未决。

---

## 执行步骤

### Step 1 — 读取运行状态
读取 `state/last_run.json`，提取：
- 顶层 `last_run_at`、`cursor`、`search_keyword_index`
- `runs[]` 末尾最近一条的 `stats` 作为"最近一轮数字"
- 可按 `type` 统计 seed / learn / converge / harvest 各自累计轮数

### Step 2 — 统计 wiki 条目
读取 wiki/ 下所有文件，统计：
- 每个维度文件的条目数量
- 各置信度分布（⭐ / ⭐⭐ / ⭐⭐⭐ / `[待收敛]`）
- 总条目数

### Step 3 — 读取成长轨迹
读取 `wiki/growth.md` 的最近 3 条时间线条目，提取核心变化。

### Step 4 — 读取角色定位
读取 `wiki/persona.md` 的"一句话定位"和"当前正在演化的议题"部分。

---

## 输出格式

```
Self-Media-Agent 系统状态
════════════════════════════════════════
角色定位：<persona.md 的一句话定位，若为空则显示"尚未积累">

────────── 最近一次学习 ──────────
时间：YYYY-MM-DD HH:MM UTC（N 天前）
拉取：N 篇  →  保留：N 篇  →  入库：N 条
cursor：<值，或 null>

────────── Wiki 积累 ──────────
                    ⭐  ⭐⭐  ⭐⭐⭐  矛盾
aesthetics/composition  0   0    0    0
aesthetics/light        0   0    0    0
aesthetics/color        0   0    0    0
skills/copywriting      0   0    0    0
skills/storytelling     0   0    0    0
knowledge/photography   0   0    0    0
knowledge/culture       0   0    0    0
worldview               0   0    0    0
references              0   0    0    0
────────────────────────────────
总计                    0   0    0    0

────────── 近期成长 ──────────
• <growth.md 最近 3 条摘要>

────────── 待处理 ──────────
[待收敛] 矛盾：N 个（运行 /skill converge 处理）
[⭐ 孤立条目]：N 个（运行 /skill audit 了解详情）

────────── 健康度 ──────────
<一句简短评估，如：>
"知识积累充足，⭐⭐⭐条目占比健康，有 2 个矛盾待处理。"
或
"角色尚在起步阶段，建议运行更多 /skill learn 积累素材。"
════════════════════════════════════════
```

---

## 适用场景

- 长时间没看这个仓库，想快速"找回感觉"
- 在运行 `/skill photographer` 前，想知道角色目前的积累程度
- 决定是否需要立刻运行 `/skill converge` 或 `/skill audit`
- 在 GitHub Actions 日志中快速确认最近一次 learn 是否成功

---

## 说明

- Status 是**纯读取操作**，不修改任何文件
- 若 `state/last_run.json` 的 `last_run_at` 为 null，说明从未运行过学习流程
