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
- 登录有效期约 7–30 天，过期后运行 `~/xiaohongshu-login` 重新扫码
