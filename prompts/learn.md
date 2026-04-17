---
name: learn
description: 运行一轮学习流程——从 raw/ 中读取新素材，提炼知识写入 wiki/，重写 persona.md，更新运行状态。这是角色成长的唯一合法入口。
---

# Learn — 学习链路 Skill 入口

> 你将执行一轮完整的**知识提炼流程**，让角色从新素材中成长。
> 这是角色进化的唯一合法入口。所有步骤由你作为 Agent 自主完成——不依赖任何外部代码。

---

## 准备：读取上下文（必做，不可跳过）

在开始任何动作前，先读取以下文件让自己"回到系统语境"：

1. `schema/update-rules.md` — **学习行为的唯一准则（含 Run Loop、铁律、Quality Gates）**
2. `schema/dimensions.md` — 七维度边界，判断条目归属时参照
3. `state/last_run.json` — 取出 `cursor` 和 `search_keyword_index`
4. `wiki/persona.md` — 让自己"回到角色"

读完后在内部简短确认：**"我是谁、上次跑到哪里、本次要做什么"**。

---

## 执行：按 update-rules.md 的 10 步运行

> 完整步骤定义在 `schema/update-rules.md`，本文件是触发入口，不重复规则。
> 执行中遇到规则不清晰的情况，以 `schema/update-rules.md` 为准。

### 你在每一步的行为：

**Step 1–3（素材获取）**
- 读取 `state/last_run.json`，取出 `cursor` 和 `search_keyword_index`
- 调用 `check_login_status` 确认小红书 MCP 可用
  - 返回未登录/服务不可达 → 跳至 Step 9，在 growth.md 记录"MCP 不可用"
- 调用 `search_feeds(keyword=<当前关键词>, sort_by="最多点赞", note_type="图文")`
  - 当前关键词由 `search_keyword_index` 决定（见 `mcp/xiaohongshu.md` 关键词列表）
  - 若 content 字段被截断，对该帖追加调用 `get_feed_detail(feed_id, xsec_token, load_all_comments=false)`
- 过滤：**feed_id 已存在于 raw/ → 丢弃（去重）** / 广告/软文/非摄影/发布时间早于 cursor → 丢弃
- 保留的帖子写入 `raw/<feed_id>.md`（格式见 `schema/update-rules.md` Step 3）
- `search_keyword_index` 加一（轮转），更新至 state（Step 10 统一写回）

**Step 4–5（逐帖提炼 + 质量过筛）**
- 对每篇新帖，按七维度产出候选条目
- 对每个候选，在内部过一遍五项 Gate（定义在 `schema/update-rules.md` Step 5）
- **明确记录**：哪些通过、哪些丢弃，以及原因

**Step 6–7（合并到 wiki）**
- 通过质量关的条目，查重后写入对应的 wiki 文件
- 维度文件内部按置信度降序排列

**Step 8（更新辅助记录）**
- `wiki/growth.md` 追加本轮摘要（不超过 5 行）
- `wiki/references.md` 若出现新参照源，追加条目

**Step 9（重写 persona.md）**
- 基于最新的六个子文件，重写 `wiki/persona.md`
- 长度控制在 200–400 行，语气：第三人称、克制

**Step 10（写回状态）**
- 更新 `state/last_run.json` 顶层指针（cursor / last_run_at / search_keyword_index）
- 在 `runs[]` 末尾 append 本轮的 run 记录（含本轮 stats），**不覆盖历史**

---

## 运行结束后，向用户汇报

用简洁格式汇报本轮结果，例如：

```
本轮 Learn 完成
────────────────────────────────
拉取素材：23 篇（过滤 8 篇广告/非摄影）
保留 raw/：15 篇
候选条目：31 个
入库：9 个（通过率 29%）
  ├─ 新增 ⭐：7 个
  └─ 升级：2 个（⭐ → ⭐⭐）
[待收敛] 新增：1 个（worldview.md — 记录vs创造）
persona.md 已重写（287 行）
cursor 更新至：2026-04-15T18:00:00Z
────────────────────────────────
```

---

## 铁律提醒（执行中随时自检）

- ❌ 不改 raw/ 下任何已存在文件
- ❌ 不把原文直接粘贴进 wiki/（抽象，再写入）
- ❌ 不写无 evidence 的条目
- ❌ 不修改 schema/ 下任何文件
- ❌ 不把 persona.md 写成条目堆砌
