# browser-cdp

<!--
Language: **English** (default) | [中文](#中文说明)
Click the language link to jump to your section.
-->

## English

> Real Chrome browser automation for OpenClaw — access pages with full user login state, bypass anti-bot detection, perform interactive operations.

### What is this?

`browser-cdp` connects directly to your local Chrome via Chrome DevTools Protocol (CDP), giving AI agents the ability to:

- **Access pages with your full login state** (cookies, sessions)
- **Bypass anti-bot detection** that blocks static fetchers
- **Interact** with pages (click, fill forms, scroll, drag)
- **Extract dynamic content** from JavaScript-rendered pages
- **Take screenshots** at any point

### Architecture

```
┌─────────────────────────────────────────────────────┐
│         Chrome (remote-debugging-port=9222)          │
│            ws://localhost:9222/devtools/...          │
└──────────────────────┬──────────────────────────────┘
                       │ CDP (WebSocket)
┌──────────────────────▼──────────────────────────────┐
│                  CDP Proxy                            │
│  cdp-proxy.mjs — HTTP API on localhost:3456          │
│  Bridges: HTTP (OpenClaw) ↔ WebSocket (Chrome CDP)  │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP REST API
┌──────────────────────▼──────────────────────────────┐
│              OpenClaw AI Agent                       │
└─────────────────────────────────────────────────────┘
```

### Prerequisites

- **Node.js 22+** (uses native WebSocket)
- **Google Chrome** with remote debugging enabled

### Chrome Startup (macOS)

```bash
# Kill existing Chrome completely
pkill -9 "Google Chrome"

# Start Chrome with debugging port (use full binary path — NOT `open -a`)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run &
```

Verify:
```bash
curl -s http://127.0.0.1:9222/json/version
```

### Installation

```bash
git clone https://github.com/0xcjl/browser-cdp.git \
  ~/.openclaw/skills/browser-cdp
```

### Start CDP Proxy

```bash
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &
sleep 3
curl -s http://localhost:3456/health
# {"status":"ok","connected":true,"sessions":0,"chromePort":9222}
```

### API Reference

```bash
# List all tabs
curl -s http://localhost:3456/targets

# Open URL in new tab
curl -s "http://localhost:3456/new?url=https://example.com"

# Execute JavaScript (read DOM / read properties)
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" \
  -d 'document.title'

# JS click (fast, preferred for most interactions)
curl -s -X POST "http://localhost:3456/click?target=TARGET_ID" \
  -d 'button.submit'

# Real mouse click (triggers hover states, file dialogs)
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

### Tool Selection Strategy

| Scenario | Recommended | Why |
|----------|-------------|-----|
| Public pages (GitHub, Wikipedia, blogs) | `agent-reach` / WebFetch | Fast, low token, structured |
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

### Three-Layer Architecture

This tool is part of a three-layer web access strategy:

```
Layer 1 — agent-reach (Jina)
  → GitHub, Wikipedia, public blogs/articles
  → ~1-2s, minimal token, structured output

Layer 2 — browser-cdp (CDP)
  → Search results (Google/Bing), YouTube, Twitter, login-gated pages
  → Bypasses anti-bot, reads rendered DOM, full login state

Layer 3 — agent-browser (OpenClaw isolated browser)
  → Simple automation, screenshots, isolated environment
  → No Chrome setup, doesn't affect your browser
```

### Known Limitations

- Chrome must use a **separate profile** (`/tmp/chrome-debug-profile`), not your daily profile
- Parallel operations on the same site may get rate-limited
- Node.js 22+ required (uses native WebSocket)
- On macOS, Chrome must be started with the **full binary path** (`open -a` does NOT pass args)

### Site Patterns

Record experience with new websites:

```bash
~/.openclaw/skills/browser-cdp/references/site-patterns/
```

Filename: `{domain}.md`

Example (`github.com.md`):
```markdown
# github.com

## Login State
- Logged-in: `document.querySelector('.header-body')` exists
- Redirects to login page if not authenticated

## Anti-bot
- Raw GitHub pages work fine via Jina
- API calls require authentication

## Notes
- Lazy loads file trees — scroll to bottom
```

### Usage Log

Track each use for continuous optimization:

```bash
~/.openclaw/skills/browser-cdp/references/usage-log.md
```

### Troubleshooting

**Chrome debugging port won't open?**
```bash
# macOS requires full path — "open -a" does NOT pass arguments
pkill -9 "Google Chrome"; sleep 2
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile &
```

**Proxy connection fails?**
```bash
curl -s http://127.0.0.1:9222/json/version  # confirm Chrome responds
pkill -f cdp-proxy.mjs
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &
```

### Origin

Adapted from [eze-is/web-access](https://github.com/eze-is/web-access) (MIT license), redesigned for OpenClaw's skill format and tool selection philosophy. A bug in the original (`require()` in ES module context) was [reported](https://github.com/eze-is/web-access/issues/10) and fixed in this version.

---

## 🀄 中文说明

> **English version above / 英文说明往上翻**

### 是什么？

`browser-cdp` 通过 Chrome DevTools Protocol (CDP) 直连你本地的 Chrome，让 AI Agent 能够：

- **携带完整登录态** — cookies 和 sessions 全部保留
- **绕过反爬检测** — 搜索结果页、视频平台等静态抓取被拦截的页面
- **执行交互操作** — 点击、填表、滚动、拖拽、文件上传
- **提取动态内容** — 读取 JavaScript 渲染后的 DOM
- **随时截图** — 任意时间点截取页面

### 架构

```
Chrome (remote-debugging-port=9222)
    ↓ CDP WebSocket
CDP Proxy (cdp-proxy.mjs) — HTTP API (localhost:3456)
    ↓ HTTP REST
OpenClaw AI Agent
```

### 安装配置

```bash
git clone https://github.com/0xcjl/browser-cdp.git \
  ~/.openclaw/skills/browser-cdp
```

### 启动 Chrome（macOS）

```bash
# 必须用完整二进制路径，不能用 open -a
pkill -9 "Google Chrome"; sleep 2
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run &
```

### 启动 CDP Proxy

```bash
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &
sleep 3
curl -s http://localhost:3456/health
# {"status":"ok","connected":true,"sessions":0,"chromePort":9222}
```

### API 参考

```bash
# 列出所有标签页
curl -s http://localhost:3456/targets

# 新建标签
curl -s "http://localhost:3456/new?url=https://example.com"

# 执行 JS
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" \
  -d 'document.title'

# JS 点击（快速，推荐）
curl -s -X POST "http://localhost:3456/click?target=TARGET_ID" \
  -d 'button.submit'

# 真实鼠标点击
curl -s -X POST "http://localhost:3456/clickAt?target=TARGET_ID" \
  -d '.upload-btn'

# 截图
curl -s "http://localhost:3456/screenshot?target=TARGET_ID&file=/tmp/shot.png"

# 滚动
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"

# 导航
curl -s "http://localhost:3456/navigate?target=TARGET_ID&url=https://..."

# 关闭标签
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

### 工具选择：三层分工

| 场景 | 工具 | 原因 |
|------|------|------|
| 公开页面（GitHub、Wikipedia、博客） | `agent-reach` | 速度快、token 少、结构化 |
| **搜索结果页**（Bing/Google/YouTube） | **`browser-cdp`** | agent-reach 被反爬拦截 |
| **登录态私有内容** | **`browser-cdp`** | agent-reach 无 cookies |
| JavaScript 渲染页 | **`browser-cdp`** | 直接读渲染后 DOM |
| 简单自动化、隔离截图 | `agent-browser` | 无需配置 Chrome |
| 大规模并行采集 | `agent-reach` + 并行 | browser-cdp 易被限速 |

### 已知局限

- Chrome 必须使用**独立 profile**（`/tmp/chrome-debug-profile`）
- 同一站点并行标签页容易被限速
- Node.js 22+（使用原生 WebSocket）
- macOS 必须用**完整路径**启动 Chrome

### License

MIT — same as [eze-is/web-access](https://github.com/eze-is/web-access).
TABEND

echo "Tabbed README prepared, lines: $(wc -l < /tmp/readme-tabbed.md)"