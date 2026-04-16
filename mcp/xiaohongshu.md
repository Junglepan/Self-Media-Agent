# MCP Spec — 小红书 Connector

> 基于 [xpzouying/xiaohongshu-mcp](https://github.com/xpzouying/xiaohongshu-mcp)。
> 这是一个**本地 HTTP 服务**，不是 npm 包——需要先在本机启动服务，再通过 HTTP 与 Claude Code 连接。

---

## 架构

```
Claude Code（Agent）
      │  HTTP
      ▼
xiaohongshu-mcp 本地服务
http://localhost:18060/mcp
      │  Playwright（浏览器自动化）
      ▼
小红书网页
```

---

## 安装与启动

### 方式一：下载预编译二进制（推荐）

```bash
# 从 GitHub Releases 下载对应平台的二进制
# macOS ARM64 / Intel / Windows / Linux 均有提供

# 首次运行：扫码登录（会打开浏览器）
./xiaohongshu-login

# 后续运行：无头模式启动服务
./xiaohongshu-mcp
# 或带界面调试：
./xiaohongshu-mcp -headless=false
```

### 方式二：Docker（最简单）

```bash
docker compose up -d
```

服务启动后监听 `http://localhost:18060/mcp`。

---

## 连接 Claude Code

```bash
claude mcp add --transport http xiaohongshu-mcp http://localhost:18060/mcp
```

执行一次即可，配置会保存到 Claude Code 的全局设置中。

验证连接：在 Claude Code 中调用 `check_login_status`，返回已登录状态即成功。

---

## 学习链路使用的工具

`/skill learn` 的 Step 2 仅使用以下三个工具：

### `check_login_status`

**用途**：Step 2 开始前验证登录状态，不可用则降级。

**参数**：无

**返回**：登录状态（已登录 / 未登录 / 已过期）

---

### `search_feeds`

**用途**：拉取高赞摄影帖子的主要工具。

**参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `keyword` | `string` | 是 | 搜索关键词 |
| `sort_by` | `string` | 否 | 默认 `"综合"`，学习链路用 `"最多点赞"` |
| `note_type` | `string` | 否 | 默认 `"不限"`，用 `"图文"` 过滤图文帖 |
| `publish_time` | `string` | 否 | `"不限"` / `"一天内"` / `"一周内"` / `"半年内"` |

**学习链路推荐调用**：
```
search_feeds(keyword="<当前关键词>", sort_by="最多点赞", note_type="图文")
```

**返回字段（每条帖子）**：
- `feed_id`、`xsec_token` — 后续调用 `get_feed_detail` 用
- `title`、`content`（可能截断）、`author`
- 点赞/收藏/评论数（用于过滤低质量帖）

---

### `get_feed_detail`

**用途**：拉取单篇帖子的完整正文（当 `search_feeds` 返回的 content 被截断时调用）。

**参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `feed_id` | `string` | 是 | 从 `search_feeds` 结果中取 |
| `xsec_token` | `string` | 是 | 从 `search_feeds` 结果中取 |
| `load_all_comments` | `boolean` | 否 | 学习链路无需评论，传 `false` |

**返回**：完整正文、用户信息、点赞/收藏/分享/评论数。

---

## 发布工具（`/skill publish` 使用）

### `publish_content`

**用途**：发布图文帖子到小红书。由 `/skill publish` 调用，学习链路不使用。

**参数**：

| 参数 | 类型 | 必填 | 限制 |
|------|------|------|------|
| `title` | `string` | 是 | 最多 20 字 |
| `content` | `string` | 是 | 最多 1000 字 |
| `images` | `string[]` | 是 | HTTP/HTTPS 链接或本地绝对路径（推荐本地） |
| `tags` | `string[]` | 否 | 话题标签，推荐填写 |
| `schedule_at` | `string` | 否 | 定时发布，ISO8601 |
| `visibility` | `string` | 否 | 公开 / 仅自己 |

---

## 搜索关键词轮转策略

每次 `/skill learn` 从以下列表取一个关键词（轮转），避免重复搜索：

```
摄影    街拍    胶片摄影    人文摄影
光影摄影  城市摄影  风景摄影  纪实摄影
构图技巧  摄影文案  摄影日记  摄影故事
```

当前轮到的关键词索引存在 `state/last_run.json` 的 `search_keyword_index` 字段。

---

## 调用约束

| 约束 | 值 |
|------|----|
| 每次 Learn 最大 search 调用次数 | 3 次 |
| 每次 Learn 最大拉取帖子数 | 50 篇 |
| 两次 MCP 调用最小间隔 | 2 秒 |

---

## MCP 不可用的处理

- `check_login_status` 返回未登录 → 跳过 Step 2–3，执行 Step 9
- 服务未启动（连接失败） → 同上，在 growth.md 记录"MCP 不可用"
- 不阻塞整轮运行，persona.md 照常更新
