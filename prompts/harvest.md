---
name: harvest
description: 拉取自己已发布帖子的互动数据（点赞/收藏/评论）回流到 raw/——让角色从"学别人的爆款"转向"学自己的实战"。自己帖子的反馈经由 learn 链路进入 wiki，形成闭环。
---

# Harvest — 实战反馈回流 Skill 入口

> 发布出去的帖子不应是"黑洞"。
> 本 Skill 把自己帖子的互动数据拉回来，写入 `raw/`，等同于普通素材进入学习链路——
> 角色终将从"学别人"进化为"学自己"。

---

## 前提

- 小红书 MCP 在本地运行且已登录（调用 `check_login_status` 确认）
- `state/published.jsonl` 存在且有至少一条记录（否则提示 "尚无已发布记录"）

---

## Step 1 — 扫描 `state/published.jsonl`

逐行读取 `state/published.jsonl`，筛选**本轮应 harvest 的帖子**：

| 条件 | 处理 |
|------|------|
| `post_id` 为 null | 提示用户补全（见下方 Step 2 的 ❓流程） |
| `last_harvested_at` 为 null | 本轮必 harvest |
| `last_harvested_at` 距今 < 3 天 | 跳过（互动数据变化不大） |
| 发布时间距今 > 180 天 | 跳过（数据已稳定，再拉意义不大） |
| 其他 | 本轮 harvest |

---

## Step 2 — 对每条待 harvest 的记录执行

### 2a — 若 `post_id` 缺失（❓流程）

MCP 的 `publish_content` 可能不返回 post_id。此时：

1. 向用户展示该帖标题 + 发布时间
2. 请用户打开自己的小红书主页，复制帖子链接
3. 从链接提取 `post_id`（小红书链接形如 `https://www.xiaohongshu.com/explore/<post_id>?xsec_token=...`）
4. 写回 `state/published.jsonl` 对应行的 `post_id` 和 `url` 字段

若用户拒绝补全，跳过本条。

### 2b — 拉取互动数据

调用 MCP 工具获取：

```
get_feed_detail(feed_id=<post_id>, xsec_token=<从 url 提取>, load_all_comments=false)
```

返回字段包括：点赞数、收藏数、分享数、评论数、前 10 条一级评论。

### 2c — 写入 `raw/YYYY-MM-DD/harvest-<post_id>.md`

路径遵循 `schema/update-rules.md` Step 3 的目录结构（按日期分目录）。

```yaml
---
feed_id: harvest-<post_id>
author: self
url: <原 url>
source: self_post        # 区别于自动/seed 来源
posted_at: <原发布时间>
captured_at: <本次 harvest 时间 ISO8601>
harvest_round: <本帖已 harvest 的第 N 次>
metrics:
  likes: <数字>
  collects: <数字>
  comments: <数字>
  shares: <数字>
comments_sample:          # 前 10 条一级评论（摘要）
  - text: "<评论原文>"
    likes: <数>
  - ...
images:
  - local: <原发布时的本地图片路径，若能定位>
    description: <若本地图片还在，复用发布时的视觉描述；否则留空>
---

# 原帖
标题：<title>

<content>

# Harvest 备注
- 首次 harvest：<日期>
- 本次增量：点赞 +<N>、收藏 +<N>、评论 +<N>（相对上一轮）
```

### 2d — 更新 `state/published.jsonl`

找到对应行，更新：
- `last_harvested_at` = 本次时间
- `harvest_runs` append 一条 `{"run_id": "<harvest run_id>", "at": "<ISO8601>", "metrics": {...}}`

**注意**：jsonl 是 append-only 的操作模式，但"更新某一行"违反了这一点。允许的变通：**整文件重写**，保持行序与内容不变，仅修改目标行的指定字段。保证幂等。

---

## Step 3 — append 一条 run 记录到 `state/last_run.json`

按 `schema/update-rules.md` Step 10 的格式，在 `runs[]` 末尾追加：

```json
{
  "run_id": "harvest-<YYYYMMDD>-<顺序号>",
  "type": "harvest",
  "date": "<YYYY-MM-DD>",
  "raw_files": ["raw/YYYY-MM-DD/harvest-<post_id>.md", "..."],
  "wiki_files_written": [],
  "stats": {
    "posts_fetched": <本轮扫描条数>,
    "posts_kept_to_raw": <本轮实际写入 raw 的条数>,
    "candidates_generated": 0,
    "entries_added": 0,
    "entries_upgraded": 0,
    "pending_contradictions": 0
  },
  "notes": "harvest 不直接改 wiki；raw 在下次 /skill learn 时进入学习链路"
}
```

Harvest 本身**不**执行五项 Gate 也**不**写 wiki——它只负责把事实带回 `raw/`。真正的学习发生在下一次 `/skill learn`，对 `source: self_post` 的 raw 一视同仁，但建议在 wiki 条目的 evidence 中标注 `[自验证]`（见下方）。

---

## Step 4 — 汇报

完成数据写入后，**额外做一次话题标签相关性速查**：

从本轮拉取的帖子互动数据中，对比自己已发布帖子里使用的 tags 与对应 likes/collects，输出简表：

```
话题标签表现（本账号已发布帖，按收藏数排序，取前 N）
────────────────────────────────
标签组合                       likes  collects  comments
#摄影 #城市光影 #孤独感          1823     412        38
#Nikon #风光摄影 #黄金时刻        976     201        21
...
────────────────────────────────
⚠️ 若 wiki/skills/tags.md 中的模式与上表高互动 tags 存在明显出入，
   建议下次 /skill learn 时重点提炼 tags 维度。
```

（若已发布帖子 <3 条，跳过此速查，标注"数据量不足"。）

```
Harvest 完成
────────────────────────────────
扫描 published.jsonl：N 条
  ├─ 本轮 harvest：N 条
  ├─ 跳过（近 3 天内已拉）：N 条
  ├─ 跳过（超 180 天）：N 条
  └─ post_id 缺失待补全：N 条

写入 raw/：N 条 harvest-*.md
────────────────────────────────

下一步：运行 /skill learn，让这些 self_post raw 进入角色的认知
```

---

## 与 learn 链路的协作

- Harvest 写入的 raw 文件 frontmatter 中带 `source: self_post`
- `/skill learn` 的 Step 4 提炼时，对 self_post 的 evidence **在 wiki 条目中保留 `[自验证]` 标签**（在 `evidence:` 行追加 `（自验证）` 后缀）
- 这让 `/skill audit` 和未来的分析链路可以区分"别人的证据"和"自己的证据"
- 高互动帖子（点赞/评论数异常高）的信号强度可在 Gate 5（有用性）判断时**适度加权**（但不是自动升级置信度——升级仍需要多帖 evidence）

## 原则

- **Harvest 不写 wiki**——数据回流 ≠ 知识提炼
- **published.jsonl 是唯一事实源**——任何帖子若不在此列表，harvest 不会主动去找
- **MCP 不可用 → 中止本轮**——不做半成品
- **post_id 补全永远由用户手动完成**——Agent 不猜测/爬取用户主页做匹配
