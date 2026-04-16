# 本地环境配置指南

> 最快路径：在 Claude Code 终端运行 `/skill setup`，Agent 会自动完成大部分步骤。
> 本文档是手动配置的备查参考。

---

## 依赖一览

| 依赖 | 用途 | 是否必须 |
|------|------|----------|
| Claude Code | 运行所有 Skill 的宿主环境 | ✅ 必须 |
| xiaohongshu-mcp | 小红书数据读取 + 发布 | `/skill learn` 和 `/skill publish` 必须 |
| Git | 版本控制 | ✅ 必须 |
| 小红书账号 | 登录授权 | `/skill learn` 和 `/skill publish` 必须 |

创作链路（`/skill photographer` / `/skill analyze`）**不依赖**小红书 MCP，可以随时使用。

---

## 快速配置（推荐）

```bash
# 在 Claude Code 终端运行，Agent 自动完成环境检测、下载、注册
/skill setup
```

---

## 手动配置步骤

### 1. 下载 xiaohongshu-mcp 二进制

前往 [github.com/xpzouying/xiaohongshu-mcp/releases](https://github.com/xpzouying/xiaohongshu-mcp/releases/latest) 下载对应平台版本：

| 系统 | 架构 | 文件名 |
|------|------|--------|
| macOS | Apple Silicon (M1/M2/M3) | `xiaohongshu-mcp-darwin-arm64` |
| macOS | Intel | `xiaohongshu-mcp-darwin-amd64` |
| Linux | x86_64 | `xiaohongshu-mcp-linux-amd64` |
| Linux | ARM64 | `xiaohongshu-mcp-linux-arm64` |

同时下载 `xiaohongshu-login`（同页面，对应同一平台）。

```bash
# 下载后授予执行权限
chmod +x ~/xiaohongshu-mcp ~/xiaohongshu-login
```

或使用命令行一键下载（以 macOS ARM64 为例）：

```bash
REPO="xpzouying/xiaohongshu-mcp"
PLATFORM="darwin-arm64"   # 按实际平台修改

# 获取最新版本下载链接
BASE_URL=$(curl -sL "https://api.github.com/repos/${REPO}/releases/latest" \
  | grep "browser_download_url" \
  | grep "$PLATFORM" \
  | head -2 \
  | cut -d '"' -f 4)

# 下载
curl -L $(echo "$BASE_URL" | grep "xiaohongshu-mcp") -o ~/xiaohongshu-mcp
curl -L $(echo "$BASE_URL" | grep "xiaohongshu-login") -o ~/xiaohongshu-login
chmod +x ~/xiaohongshu-mcp ~/xiaohongshu-login
```

### 2. 首次登录（扫码）

```bash
~/xiaohongshu-login
```

浏览器会自动打开，用手机小红书扫描二维码。登录状态保存在本地，有效期约 7–30 天。

### 3. 启动 MCP 服务

```bash
# 前台运行（调试时用）
~/xiaohongshu-mcp

# 后台运行
nohup ~/xiaohongshu-mcp > ~/xiaohongshu-mcp.log 2>&1 &
```

服务默认监听 `http://localhost:18060/mcp`。

### 4. 注册到 Claude Code

```bash
claude mcp add --transport http xiaohongshu-mcp http://localhost:18060/mcp
```

执行一次即永久生效。

### 5. 验证

在 Claude Code 终端运行：

```bash
/skill status
```

输出中出现"已登录 ✅"即配置成功。

---

## 日常维护

### 重启后恢复服务

xiaohongshu-mcp 不会自动随系统启动，重启后需手动重新启动服务：

```bash
~/xiaohongshu-mcp &
```

**macOS 可设置开机自动启动（可选）：**

```bash
# 创建 LaunchAgent plist
cat > ~/Library/LaunchAgents/com.xiaohongshu-mcp.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.xiaohongshu-mcp</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/xiaohongshu-mcp</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/xiaohongshu-mcp.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/xiaohongshu-mcp.log</string>
</dict>
</plist>
EOF

# 替换 YOUR_USERNAME 后加载
launchctl load ~/Library/LaunchAgents/com.xiaohongshu-mcp.plist
```

### 登录过期后重新扫码

```bash
~/xiaohongshu-login
```

### 查看服务日志

```bash
tail -f ~/xiaohongshu-mcp.log
```

### 更新到新版本

```bash
# 停止旧服务
kill $(cat ~/xiaohongshu-mcp.pid) 2>/dev/null

# 重新运行 setup
/skill setup
```

---

## 不依赖 MCP 的功能

以下 Skill 不需要小红书 MCP，随时可用：

| Skill | 用途 |
|-------|------|
| `/skill photographer` | 生成文案 |
| `/skill analyze` | 分析文案 |
| `/skill status` | 查看 wiki 积累状态 |
| `/skill audit` | Wiki 质量审计 |
| `/skill converge` | 处理矛盾条目 |

只有 `/skill learn` 和 `/skill publish` 需要 MCP 服务在线。
