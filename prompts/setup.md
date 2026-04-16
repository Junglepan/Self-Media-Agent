---
name: setup
description: 检测本地环境，下载 xiaohongshu-mcp 二进制，注册 MCP 到 Claude Code，引导完成扫码登录。把一切能自动化的步骤都自动完成，只在必须人工介入时暂停等待。
---

# Setup — 本地环境初始化 Skill

> 运行一次，把所有依赖配置到位。
> 需要人工操作的步骤只有一个：扫码登录小红书。

---

## 执行前提

- 你在 Claude Code 本地终端运行此 Skill（不是 GitHub Actions）
- 有网络连接（需要下载二进制）
- 有小红书账号

---

## Step 1 — 检测操作系统和架构

运行以下命令确定平台：

```bash
uname -s   # 系统：Linux / Darwin（macOS）
uname -m   # 架构：x86_64 / arm64
```

根据结果确定要下载的二进制名称：

| uname -s | uname -m | 二进制 |
|----------|----------|--------|
| Darwin | arm64 | `xiaohongshu-mcp-darwin-arm64` |
| Darwin | x86_64 | `xiaohongshu-mcp-darwin-amd64` |
| Linux | x86_64 | `xiaohongshu-mcp-linux-amd64` |
| Linux | arm64 | `xiaohongshu-mcp-linux-arm64` |

---

## Step 2 — 检查是否已安装

```bash
ls ~/xiaohongshu-mcp 2>/dev/null && echo "exists" || echo "not found"
```

- 若已存在且版本正常 → 跳至 Step 4
- 若不存在 → 继续 Step 3

---

## Step 3 — 下载二进制

从 [github.com/xpzouying/xiaohongshu-mcp/releases/latest](https://github.com/xpzouying/xiaohongshu-mcp/releases/latest) 下载对应平台的二进制。

Agent 执行：

```bash
# 1. 获取最新 Release 的下载 URL
PLATFORM="<由 Step 1 确定>"
curl -sL "https://api.github.com/repos/xpzouying/xiaohongshu-mcp/releases/latest" \
  | grep "browser_download_url" \
  | grep "$PLATFORM" \
  | cut -d '"' -f 4

# 2. 下载到 Home 目录
curl -L "<下载URL>" -o ~/xiaohongshu-mcp
curl -L "<login下载URL>" -o ~/xiaohongshu-login

# 3. 授予执行权限
chmod +x ~/xiaohongshu-mcp ~/xiaohongshu-login
```

完成后确认：

```bash
~/xiaohongshu-mcp --version 2>/dev/null || echo "binary ready (no --version flag)"
```

---

## Step 4 — 扫码登录（唯一需要人工操作的步骤）

⚠️ **此步骤需要你亲自操作，Agent 无法代劳。**

告知用户：

```
────────────────────────────────────────────────
请在另一个终端窗口运行：

  ~/xiaohongshu-login

浏览器会自动打开，用手机小红书扫描二维码完成登录。
登录成功后回到这里，按回车继续。
────────────────────────────────────────────────
```

等待用户按回车确认后继续。

---

## Step 5 — 启动 MCP 服务

后台启动服务：

```bash
nohup ~/xiaohongshu-mcp > ~/xiaohongshu-mcp.log 2>&1 &
echo $! > ~/xiaohongshu-mcp.pid
echo "MCP service started, PID: $(cat ~/xiaohongshu-mcp.pid)"
```

等待 2 秒，确认服务就绪：

```bash
sleep 2
curl -s http://localhost:18060/mcp/health 2>/dev/null && echo "service up" || echo "checking..."
```

---

## Step 6 — 注册 MCP 到 Claude Code

```bash
claude mcp add --transport http xiaohongshu-mcp http://localhost:18060/mcp
```

若提示"already exists"则跳过（已注册）。

---

## Step 7 — 验证连接

调用 `check_login_status` 工具验证：
- 返回"已登录" → 配置成功
- 返回"未登录" → 提示用户重新运行 `~/xiaohongshu-login`
- 连接失败 → 检查 Step 5 的服务是否正常启动（`cat ~/xiaohongshu-mcp.log`）

---

## Step 8 — 汇报结果

```
Setup 完成
────────────────────────────────
平台：<Darwin arm64 / Linux x86_64 / ...>
二进制：~/xiaohongshu-mcp ✅
登录状态：已登录 ✅
MCP 注册：xiaohongshu-mcp @ localhost:18060 ✅

可以开始使用：
  /skill learn     — 开始学习
  /skill status    — 查看当前状态
  /skill seed      — 手动注入一篇帖子
────────────────────────────────
```

---

## 后续维护

| 场景 | 操作 |
|------|------|
| 重启电脑后需重新启动服务 | `~/xiaohongshu-mcp &` |
| 登录过期（约 7–30 天）| `~/xiaohongshu-login` 重新扫码 |
| 更新到新版本 | 重新运行 `/skill setup` |
| 查看服务日志 | `tail -f ~/xiaohongshu-mcp.log` |
