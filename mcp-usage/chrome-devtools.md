# Chrome DevTools MCP 使用指南（WSL → Windows Chrome）

让 WSL 下的 Claude Code（或任意 MCP 客户端）控制 Windows 端**已登录**的 Chrome，保留所有 Cookie / 扩展 / 标签页。

---

## 工作原理

| 组件 | 位置 | 作用 |
| --- | --- | --- |
| Chrome | Windows | 用户日常浏览器，保留登录态 |
| `chrome://inspect/#remote-debugging` | Windows Chrome 内 | 启动 WebSocket 调试服务（端口随机，如 65281） |
| `DevToolsActivePort` 文件 | `%LOCALAPPDATA%\Google\Chrome\User Data\` | 自动写入 `<port>\n/devtools/browser/<uuid>` |
| `chrome-devtools-mcp` (`--autoConnect`) | WSL | 读 Win 端 `DevToolsActivePort`，建立 WebSocket 连接 |
| WSL Mirrored Networking | `.wslconfig` | 让 WSL 能用 `127.0.0.1` 访问 Win 的 localhost |

> ⚠️ 不要用 `taskkill chrome.exe` + `--remote-debugging-port=9222` 重启 Chrome 的老办法 —— 会丢登录态、干扰用户、还要绕 host 校验。

---

## 一次性环境准备

### 1. 确认 WSL 网络模式为 Mirrored

`C:\Users\<你的用户名>\.wslconfig`：

```ini
[wsl2]
networkingMode=Mirrored
```

修改后 Windows PowerShell 执行 `wsl --shutdown` 重启 WSL。

### 2. 确认 Chrome ≥ 144（`--autoConnect` 要求）

```bash
cmd.exe /c "wmic datafile where name='C:\\\\Program Files\\\\Google\\\\Chrome\\\\Application\\\\chrome.exe' get Version /value"
```

### 3. Windows Chrome 启用远程调试开关

地址栏访问：

```
chrome://inspect/#remote-debugging
```

勾选 **"Allow remote debugging for this browser instance"**。该设置持久化，下次启动 Chrome 自动生效。

### 4. WSL 安装并注册 MCP

```bash
claude mcp add chrome-devtools --scope user -- \
  npx -y chrome-devtools-mcp@latest \
  --autoConnect \
  --userDataDir "/mnt/c/Users/<你的用户名>/AppData/Local/Google/Chrome/User Data"
```

验证：

```bash
claude mcp list | grep chrome-devtools
# 应显示: chrome-devtools: ... - ✓ Connected
```

注册后**重启 Claude Code 会话**，新 MCP 工具才会加载到对话中。

---

## 日常使用

启用后无需任何额外操作。在 Claude Code 里直接说：

- "打开知乎搜索 XXX"
- "帮我在小红书发一篇 XXX"
- "把当前 GitHub PR 页面截图"
- "提取这个购物车的所有商品价格"

Claude 会调用 `mcp__chrome-devtools__*` 系列工具操作你已登录的 Chrome。

---

## 排查清单

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `claude mcp list` 显示 ✗ Failed | Chrome 没启用远程调试 / 没运行 | 检查 `chrome://inspect/#remote-debugging` 是否勾选；确认 Chrome 进程在 |
| WSL 里 `curl 127.0.0.1:<port>/json` 返回连不上 | 没开 Mirrored 网络模式 | 改 `.wslconfig` 加 `networkingMode=Mirrored`，`wsl --shutdown` 重启 |
| `curl 127.0.0.1:<port>/json/version` 返回 404 | **正常** | chrome://inspect 模式只开 WebSocket，不开 HTTP discovery API。`--autoConnect` 直接走 WebSocket，不用管 |
| `curl` 走了系统代理报错 502 | `http_proxy` / `no_proxy` 拦截 | 加 `--noproxy '*'` 或 `env -u http_proxy curl ...`；MCP 内部有自己的连接逻辑通常不受影响 |
| 工具搜不到 `mcp__chrome-devtools__*` | MCP 注册后没重启会话 | `/exit` 后重新进入 Claude Code |
| Chrome 重启后 UUID 变了 | WebSocket endpoint 的 UUID 每次随机 | `--autoConnect` 自动从 `DevToolsActivePort` 文件读最新值，不需要改配置 |
| 工具调用报 `Network.enable timed out` | Chrome 同时开了较多标签页（重扩展 / 大页面 / 卡死脚本），目标 tab 对 CDP 命令无响应 | 关闭多余标签页，保留少量必要 tab 后重新调用 MCP 工具即可。**不需要**重启 Chrome 也不需要重启 MCP；不要反复重试同一个调用 |

---

## 验证端口与文件

```bash
# Win 端监听
cmd.exe /c "netstat -ano | findstr LISTENING | findstr :<port>"

# 读取自动发现文件
cat "/mnt/c/Users/<你的用户名>/AppData/Local/Google/Chrome/User Data/DevToolsActivePort"
# 输出:
#   65281
#   /devtools/browser/<uuid>
```

---

## 与 Playwright MCP 的取舍

| 场景 | 推荐 |
| --- | --- |
| 操作用户已登录的 Chrome（社交、电商、内网系统） | **chrome-devtools-mcp** |
| 全新干净环境跑测试 / 抓数据 | Playwright MCP（自启浏览器） |
| 跨浏览器自动化（Firefox / WebKit） | Playwright MCP |
| 性能 trace、网络请求详细分析 | **chrome-devtools-mcp**（有 Performance / Network 类工具） |

两者可以共存，按场景调用。

---

## 参考

- 项目主页: https://github.com/ChromeDevTools/chrome-devtools-mcp
- npm 包: `chrome-devtools-mcp`（Google 官方维护）
- 完整 CLI 选项: `npx chrome-devtools-mcp@latest --help`
