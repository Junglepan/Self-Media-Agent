# MCP Spec — 小红书 Connector

> 本文件定义 Agent 对小红书 MCP Server **期望的工具接口**。
> 具体的 Server 实现可替换，但工具名、参数结构、返回格式必须符合本规范，否则 `/skill learn` 的 Step 2 将无法正确调用。

---

## Server 标识

```json
{
  "serverName": "xiaohongshu",
  "requiredVersion": ">=1.0.0"
}
```

---

## 工具列表

### 1. `xhs_search_posts`

按关键词搜索高赞帖子。`/skill learn` 的主要拉取工具。

**参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `keywords` | `string[]` | 是 | 搜索关键词列表，如 `["摄影", "街拍", "胶片"]` |
| `sort` | `"popular" \| "latest"` | 否 | 排序方式，默认 `"popular"` |
| `min_likes` | `number` | 否 | 最低点赞数，默认 `500` |
| `limit` | `number` | 否 | 返回数量上限，默认 `20`，最大 `50` |
| `after_cursor` | `string \| null` | 否 | 游标，用于分页（从 `state/last_run.json` 中取） |

**返回**：

```json
{
  "posts": [
    {
      "post_id": "string",
      "title": "string",
      "content": "string",
      "author": "string",
      "likes": 1234,
      "posted_at": "2026-04-10T08:30:00Z",
      "url": "https://www.xiaohongshu.com/explore/<post_id>",
      "images": [
        {
          "url": "string",
          "description": "string | null"
        }
      ],
      "tags": ["string"]
    }
  ],
  "next_cursor": "string | null",
  "has_more": true
}
```

**错误码**：

| 代码 | 含义 | Agent 处理 |
|------|------|-----------|
| `AUTH_FAILED` | Cookie 失效 | 跳过 Step 2–3，降级运行 |
| `RATE_LIMITED` | 请求过频 | 等待后重试一次，若仍失败则降级 |
| `NO_RESULTS` | 无新结果 | 正常情况，记录 growth.md 后继续 |

---

### 2. `xhs_get_post`

获取单篇帖子的完整内容。用于拉取后补充详情（当 search 返回的 content 被截断时）。

**参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `post_id` | `string` | 是 | 帖子 ID |

**返回**：与 `xhs_search_posts` 的单条 `post` 对象格式相同，content 为完整正文。

**错误码**：

| 代码 | 含义 | Agent 处理 |
|------|------|-----------|
| `POST_NOT_FOUND` | 帖子已删除 | 跳过该帖，记录 skip |
| `POST_PRIVATE` | 帖子已设为私密 | 跳过该帖 |

---

## 调用约束

| 约束 | 值 | 说明 |
|------|----|------|
| 每次 Learn 最大搜索请求数 | 3 次 | 避免触发风控 |
| 每次 Learn 最大获取帖子数 | 50 篇 | 对应 `xhs_search_posts` 的 `limit` 最大值 |
| 最小调用间隔 | 2 秒 | 两次 MCP 调用之间等待 |
| Cookie 轮换提醒 | 连续 3 次 `AUTH_FAILED` | 在 growth.md 记录"需要更新 Cookie" |

---

## 搜索策略（`/skill learn` Step 2 的推荐做法）

Agent 应按以下关键词组合轮流搜索，每轮 Learn 不必全部覆盖，**轮转使用**以保持素材多样性：

```
摄影          街拍          胶片摄影
人文摄影      光影摄影      城市摄影
风景摄影      纪实摄影      构图技巧
摄影文案      摄影日记      摄影故事
```

每轮 Learn 从 `state/last_run.json` 的 `search_keyword_index` 取当前轮到的关键词（若无此字段，从第一个开始），执行后更新该索引。

---

## 过滤规则（在调用返回后、写入 raw/ 前执行）

Agent 在收到帖子列表后，自行判断过滤，以下情况**直接丢弃**：

- 含 `#广告`、`#合作`、`#品牌合作人` 标签
- 标题或正文含"领取优惠"、"扫码"、"私信报名"等商业话术
- 内容明显为转载或搬运（正文 < 50 字且全为图片说明）
- 帖子发布时间早于 `state/last_run.json` 的 `cursor`（已处理过的时间区间）

---

## 配置示例（`.claude/settings.json`）

```json
{
  "mcpServers": {
    "xiaohongshu": {
      "command": "npx",
      "args": ["-y", "mcp-server-xiaohongshu"],
      "env": {
        "XHS_COOKIE": "<从浏览器开发者工具获取的 cookie 字符串>",
        "XHS_USER_AGENT": "<可选，模拟的 User-Agent>"
      }
    }
  }
}
```

> Cookie 获取方式：登录小红书网页版 → 开发者工具 → Network → 任意请求 → 复制 `Cookie` 请求头值。
> Cookie 有效期约 7–30 天，过期后需手动更新。

---

## 本地测试

在运行 `/skill learn` 前，可先验证 MCP 连接：

```bash
# 在 Claude Code 中调用
xhs_search_posts(keywords=["摄影"], limit=3)
```

若返回帖子列表则连接正常；若返回 AUTH_FAILED 则需更新 Cookie。
