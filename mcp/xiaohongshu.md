# MCP — 小红书 Connector

> 基于 [xpzouying/xiaohongshu-mcp](https://github.com/xpzouying/xiaohongshu-mcp)。
> 本地 HTTP 服务，通过 Playwright 驱动浏览器与小红书交互。

---

## 架构与连接

```
Claude Code ──HTTP──► xiaohongshu-mcp (localhost:18060) ──Playwright──► 小红书
```

**启动服务：**
```bash
~/xiaohongshu-login   # 首次：扫码登录
~/xiaohongshu-mcp     # 后续：启动服务（默认无头模式）
```

**注册到 Claude Code（执行一次）：**
```bash
claude mcp add --transport http xiaohongshu-mcp http://localhost:18060/mcp
```

也可使用 Docker：`docker compose up -d`（需克隆仓库）。

---

## 工具：学习链路（`/skill learn`）

### `check_login_status`
Step 2 开始前验证。返回未登录 → 跳过拉取，降级至 Step 9。

### `search_feeds`

| 参数 | 类型 | 说明 |
|------|------|------|
| `keyword` | string（必填）| 搜索词 |
| `sort_by` | string | 学习链路用 `"最多点赞"` |
| `note_type` | string | 用 `"图文"` 过滤视频 |
| `publish_time` | string | `"不限"` / `"一周内"` / `"半年内"` |

返回：`feed_id`、`xsec_token`、`title`、`content`（可能截断）、`author`、点赞/收藏/评论数

### `get_feed_detail`

| 参数 | 类型 | 说明 |
|------|------|------|
| `feed_id` | string（必填）| 从 search_feeds 结果取 |
| `xsec_token` | string（必填）| 从 search_feeds 结果取 |
| `load_all_comments` | boolean | 学习链路传 `false` |

content 截断时调用，返回完整正文和互动数据。

---

## 工具：发布链路（`/skill publish`）

### `publish_content`

| 参数 | 类型 | 限制 |
|------|------|------|
| `title` | string（必填）| ≤20 字 |
| `content` | string（必填）| ≤1000 字 |
| `images` | string[]（必填）| 本地绝对路径或 HTTP 链接 |
| `tags` | string[] | 话题标签 |
| `schedule_at` | string | ISO8601，省略则立即发布 |
| `visibility` | string | 默认公开 |

**返回值**：若 MCP 返回 `post_id` / `url`，由 `/skill publish` 记录到 `state/published.jsonl`；未返回则留 null，由 `/skill harvest` 要求用户补全。

---

## 工具：反馈回流链路（`/skill harvest`）

### `get_feed_detail`（复用）

对"自己已发布"的帖子拉取互动数据：点赞/收藏/分享/评论数 + 前 10 条一级评论。

### `user_profile`（可选 / 容错路径）

| 参数 | 类型 | 说明 |
|------|------|------|
| `user_id` 或主页 URL | string | 用户主页定位 |

用途：当 `published.jsonl` 中 post_id 全部缺失、用户又记不清链接时，可列出主页全部笔记让用户手动挑选匹配。**不作为常规路径**。

---

## 服务器实际工具全量清单（v2.0.0）

当前 xiaohongshu-mcp v2.0.0 实际暴露 13 个工具。本仓库 skill 已使用的以 ✅ 标记；其他保留能力但暂未在 skill 中调用。

| 工具 | 用途 | 本仓库调用方 |
|------|------|-----------|
| `check_login_status` | 登录状态检查 | ✅ learn / publish / harvest |
| `get_login_qrcode` | 获取登录二维码 | setup 流程之外（首次登录用独立 CLI）|
| `delete_cookies` | 重置登录态 | — |
| `search_feeds` | 关键词搜索 | ✅ learn |
| `get_feed_detail` | 帖子详情 + 互动数据 + 评论 | ✅ learn / harvest |
| `list_feeds` | 首页 feeds | — |
| `user_profile` | 用户主页 + 其笔记 | ⚠️ harvest 容错路径 |
| `publish_content` | 图文发布 | ✅ publish |
| `publish_with_video` | 视频发布 | — |
| `like_feed` | 点赞/取消点赞 | — |
| `favorite_feed` | 收藏/取消收藏 | — |
| `post_comment_to_feed` | 发表评论 | — |
| `reply_comment_in_feed` | 回复评论 | — |

**原则**：能写操作（like / favorite / comment / reply）暂不纳入任何 skill——角色的互动由用户本人完成，Agent 不代劳。

---

## 搜索关键词轮转

每轮 `/skill learn` 从以下列表按 `state/last_run.json` 的 `search_keyword_index` 取一个关键词：

```
摄影  街拍  胶片摄影  人文摄影  光影摄影  城市摄影
风景摄影  纪实摄影  构图技巧  摄影文案  摄影日记  摄影故事
```

---

## 调用约束

- 每轮最多 3 次 search 调用，最多拉取 50 篇
- 两次调用间隔 ≥2 秒
- 抓取冷却：两轮实际抓取之间间隔必须 ≥3 分钟（使用 `state/last_run.json.mcp_fetch_last_at` 判断）
- 登录有效期约 7–30 天，过期后运行 `~/xiaohongshu-login` 重新扫码
