# browser-cdp

> Real Chrome browser automation for OpenClaw — access pages with full user login state, bypass anti-bot detection, perform interactive operations.

**中文说明请往下翻。**

## What is this?

`browser-cdp` is an OpenClaw skill that connects directly to your local Chrome browser via Chrome DevTools Protocol (CDP), giving AI agents the ability to:

- Access pages **with your full login state** (cookies, sessions)
- **Bypass anti-bot detection** that blocks static fetchers
- **Interact** with pages (click, fill forms, scroll, drag)
- **Extract dynamic content** from JavaScript-rendered pages
- **Take screenshots** at any point

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                     Chrome Browser                    │
│                  (with debugging port)               │
│              ws://localhost:9222/devtools/...        │
└──────────────────────┬───────────────────────────────┘
                       │ CDP (WebSocket)
┌──────────────────────▼───────────────────────────────┐
│                   CDP Proxy                          │
│  cdp-proxy.mjs — HTTP API on localhost:3456          │
│  - Bridges HTTP (from OpenClaw) ↔ WebSocket (to CDP) │
│  - Target/tab management                             │
│  - Mouse/keyboard simulation                          │
└──────────────────────┬───────────────────────────────┘
                       │ HTTP REST API
┌──────────────────────▼───────────────────────────────┐
│              OpenClaw AI Agent                       │
└──────────────────────────────────────────────────────┘
```

## Prerequisites

- **Node.js 22+** (uses native WebSocket)
- **Google Chrome** with remote debugging enabled

### Chrome Startup (macOS)

```bash
# Kill existing Chrome completely
pkill -9 "Google Chrome"

# Start Chrome with debugging port (use a separate profile)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run &
```

Verify with:
```bash
curl -s http://127.0.0.1:9222/json/version
```

## Installation

```bash
# Clone into OpenClaw skills directory
git clone https://github.com/0xcjl/browser-cdp.git \
  ~/.openclaw/skills/browser-cdp
```

## Start CDP Proxy

```bash
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &
sleep 3
curl -s http://localhost:3456/health
# {"status":"ok","connected":true,"sessions":0,"chromePort":9222}
```

## API Reference

```bash
# List all tabs
curl -s http://localhost:3456/targets

# Open URL in new tab
curl -s "http://localhost:3456/new?url=https://example.com"

# Execute JavaScript (read DOM / read properties / interact)
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" \
  -d 'document.title'

# JS click (fast, preferred for most interactions)
curl -s -X POST "http://localhost:3456/click?target=TARGET_ID" \
  -d 'button.submit'

# Real mouse click (triggers hover states, file dialogs, etc.)
curl -s -X POST "http://localhost:3456/clickAt?target=TARGET_ID" \
  -d '.upload-btn'

# Screenshot
curl -s "http://localhost:3456/screenshot?target=TARGET_ID&file=/tmp/shot.png"

# Scroll (trigger lazy loading)
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"

# Navigate
curl -s "http://localhost:3456/navigate?target=TARGET_ID&url=https://..."

# Close tab
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

## Tool Selection Strategy

| Scenario | Recommended | Why |
|----------|-------------|-----|
| Public pages (GitHub, Wikipedia, blogs) | `agent-reach` / WebFetch | Fast, low token, structured output |
| **Search results** (Bing/Google/YouTube) | **`browser-cdp`** | agent-reach gets blocked |
| **Login-gated content** | **`browser-cdp`** | agent-reach has no cookies |
| JavaScript-rendered pages | **`browser-cdp`** | Reads rendered DOM directly |
| Simple automation, isolated screenshots | `agent-browser` | No Chrome setup needed |
| Large-scale parallel scraping | `agent-reach` + parallel | browser-cdp gets rate-limited |

**Decision flow:**
```
1. Public content (GitHub/Wikipedia/blog) → agent-reach (fast, cheap)
2. Search results / anti-bot blocked → browser-cdp
3. browser-cdp still fails → agent-reach fallback + record in site-patterns
```

## Three-Layer Architecture

This tool is part of a three-layer web access strategy:

```
Layer 1 — agent-reach (Jina)
  → GitHub, Wikipedia, public blogs/articles
  → Speed: ~1-2s, token: minimal, structured output

Layer 2 — browser-cdp (CDP)
  → Search results (Google/Bing), YouTube, Twitter, login-gated pages
  → Bypasses anti-bot, reads rendered DOM, full login state

Layer 3 — agent-browser (OpenClaw isolated browser)
  → Simple automation, screenshots, isolated environment
  → No Chrome setup, doesn't affect your browser
```

## Known Limitations

- Chrome must use a **separate profile** (`/tmp/chrome-debug-profile`), not your daily browser profile
- Parallel operations on the same site may get rate-limited
- Node.js 22+ required (uses native WebSocket)
- On macOS, Chrome must be started with the **full path** (not `open -a`), or debugging port won't bind

## Site Patterns

When you encounter a new website, record its characteristics:

```bash
~/.openclaw/skills/browser-cdp/references/site-patterns/
```

Filename: `{domain}.md`

Example:
```markdown
# github.com

## Login State
- Logged-in: `document.querySelector('.header-body')` exists
- Login page: redirects to github.com/login

## Anti-bot
- Raw GitHub pages work fine via Jina
- API calls require authentication

## Notes
- Uses lazy loading for file trees
- Scroll to bottom to load more
```

## Usage Log

Track each use to optimize over time:

```bash
~/.openclaw/skills/browser-cdp/references/usage-log.md
```

Format:
```markdown
### YYYY-MM-DD | [Task Type] | [URL/Website] | [Result: success/fail/degrade] | [Notes]
```

## Troubleshooting

**Chrome debugging port not opening?**
```bash
# macOS requires full path — "open -a" does NOT pass arguments correctly
pkill -9 "Google Chrome"; sleep 2
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile &
```

**Proxy connection fails?**
```bash
# Check Chrome is responding
curl -s http://127.0.0.1:9222/json/version

# Restart proxy
pkill -f cdp-proxy.mjs
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &
```

## Origin

This skill was adapted from [eze-is/web-access](https://github.com/eze-is/web-access) (MIT license), redesigned for OpenClaw's skill format and tool selection philosophy. A bug in the original (`require()` used in ES module context) was [reported](https://github.com/eze-is/web-access/issues/10) and fixed in this version.

## License

MIT — same as the original project.

---

---

# browser-cdp

> 面向 OpenClaw 的真实浏览器自动化工具 —— 以用户登录态访问页面、绕过反爬检测、执行交互操作。

## 是什么？

`browser-cdp` 是一个 OpenClaw skill，通过 Chrome DevTools Protocol (CDP) 直连你本地的 Chrome 浏览器，让 AI Agent 能够：

- **携带完整登录态**访问页面（cookies、sessions）
- **绕过反爬检测**（静态抓取被拦截的页面）
- **执行交互操作**（点击、填表、滚动、拖拽）
- **提取动态内容**（JavaScript 渲染后的 DOM）
- **随时截图**

## 架构

```
┌──────────────────────────────────────────────────────┐
│                     Chrome 浏览器                    │
│               （已开启远程调试端口）                  │
│              ws://localhost:9222/devtools/...        │
└──────────────────────┬───────────────────────────────┘
                       │ CDP (WebSocket)
┌──────────────────────▼───────────────────────────────┐
│                   CDP Proxy                          │
│  cdp-proxy.mjs — HTTP API (localhost:3456)           │
│  - HTTP (来自 OpenClaw) ↔ WebSocket (到 CDP) 桥接   │
│  - 目标/标签页管理                                    │
│  - 鼠标/键盘模拟                                      │
└──────────────────────┬───────────────────────────────┘
                       │ HTTP REST API
┌──────────────────────▼───────────────────────────────┐
│              OpenClaw AI Agent                       │
└──────────────────────────────────────────────────────┘
```

## 前置条件

- **Node.js 22+**
- **Google Chrome** 已开启远程调试

### Chrome 启动（macOS）

```bash
# 先彻底退出 Chrome
pkill -9 "Google Chrome"

# 用独立 profile 启动（调试端口必须用完整路径）
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run &
```

验证：
```bash
curl -s http://127.0.0.1:9222/json/version
```

## 安装

```bash
git clone https://github.com/0xcjl/browser-cdp.git \
  ~/.openclaw/skills/browser-cdp
```

## 启动 CDP Proxy

```bash
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &
sleep 3
curl -s http://localhost:3456/health
# {"status":"ok","connected":true,"sessions":0,"chromePort":9222}
```

## API 参考

```bash
# 列出所有标签页
curl -s http://localhost:3456/targets

# 新建标签（后台打开）
curl -s "http://localhost:3456/new?url=https://example.com"

# 执行 JS（读 DOM / 读属性 / 操控元素）
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" \
  -d 'document.title'

# JS 点击（快速，推荐优先）
curl -s -X POST "http://localhost:3456/click?target=TARGET_ID" \
  -d 'button.submit'

# 真实鼠标点击（触发 hover、文件对话框等）
curl -s -X POST "http://localhost:3456/clickAt?target=TARGET_ID" \
  -d '.upload-btn'

# 截图
curl -s "http://localhost:3456/screenshot?target=TARGET_ID&file=/tmp/shot.png"

# 滚动（触发懒加载）
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"

# 导航到新 URL
curl -s "http://localhost:3456/navigate?target=TARGET_ID&url=https://..."

# 关闭标签
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

## 工具选择策略（三层分工）

| 场景 | 推荐工具 | 原因 |
|------|---------|------|
| 公开页面（GitHub、Wikipedia、博客） | `agent-reach` / WebFetch | 速度快、token 少、结构化 |
| **搜索结果页**（Bing/Google/YouTube） | **`browser-cdp`** | agent-reach 被反爬拦截 |
| **登录态私有内容** | **`browser-cdp`** | agent-reach 无 cookies |
| JavaScript 渲染页 | **`browser-cdp`** | 直接读取渲染后 DOM |
| 简单自动化、截图验证 | `agent-browser` | 无需配置 Chrome |
| 大规模并行采集 | `agent-reach` + 并行 | browser-cdp 易被限速 |

**判断顺序：**
```
1. 公开内容 → agent-reach（快、便宜）
2. 搜索结果 / 被反爬拦截 → browser-cdp
3. 仍失败 → agent-reach 备选 + 记录到 site-patterns
```

## 已知局限

- Chrome 必须使用**独立 profile**（`/tmp/chrome-debug-profile`），不能共用日常浏览器
- 同一站点的多个标签页并行操作容易被限速
- Node.js 22+（使用原生 WebSocket）
- macOS 上必须用**完整路径**启动 Chrome（`open -a` 无法正确传递参数）

## 起源

本 skill 从 [eze-is/web-access](https://github.com/eze-is/web-access)（MIT 协议）提取并针对 OpenClaw 重新设计。原项目有一个 bug（在 ES module 中使用 `require()`）已 [报告给作者](https://github.com/eze-is/web-access/issues/10)，本版本已修复。

## License

MIT
