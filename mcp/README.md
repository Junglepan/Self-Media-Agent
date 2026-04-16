# MCP — 外部工具接入层

> MCP（Model Context Protocol）是 Agent 访问外部系统的唯一合法入口。
> 本目录存放每个 MCP Server 的**接入规范**——Agent 在调用 MCP 工具前，应读取对应规范文件以了解工具边界和参数约定。

---

## 当前接入的 MCP Server

| Server | 规范文件 | 用途 |
|--------|----------|------|
| 小红书 Connector | [`xiaohongshu.md`](xiaohongshu.md) | 拉取高赞摄影帖子（学习链路专用） |

---

## MCP 在系统中的位置

```
/skill learn 触发
      ↓
Agent 读取 schema/update-rules.md
      ↓
Step 2：通过 MCP 拉取素材          ← mcp/xiaohongshu.md 描述了这里
      ↓
Step 3：写入 raw/
      ↓
...后续步骤...
```

MCP 工具**只在学习链路（`/skill learn`）的 Step 2 中使用**。创作链路（`/skill photographer`）和分析链路（`/skill analyze`）不调用任何 MCP 工具。

---

## 配置方式

MCP Server 通过 Claude Code 的 `~/.claude/settings.json`（本地）或 `.claude/settings.json`（仓库级）配置：

```json
{
  "mcpServers": {
    "xiaohongshu": {
      "command": "npx",
      "args": ["-y", "<xiaohongshu-mcp-server-package>"],
      "env": {
        "XHS_COOKIE": "<your-session-cookie>"
      }
    }
  }
}
```

> 具体的 Server 实现由外部维护，本仓库只定义**工具接口规范**（Agent 期望的工具名、参数、返回格式）。

---

## MCP 不可用时的降级行为

当 MCP 工具不可用时（无网络、凭据失效、Server 未启动），`/skill learn` 的行为：

1. 跳过 Step 2–3（不拉取新素材）
2. 直接执行 Step 9（基于现有 wiki 重写 persona.md）
3. 在 `state/last_run.json` 的 notes 中记录"MCP unavailable"
4. 在 `wiki/growth.md` 追加一条"本轮跳过素材拉取"

降级不报错，不中止，保持角色画像的持续更新。

---

## 添加新 MCP Server

1. 在本目录新增 `<server-name>.md`，按 `xiaohongshu.md` 的结构描述：工具列表、参数、返回格式、错误码、调用频率约束
2. 更新本文件的"当前接入"表格
3. 更新对应 Skill 文件，说明该 Server 在哪一步被调用
