---
name: browser-cdp
description: >
  Real Chrome browser automation via CDP Proxy — access pages with full user login state,
  bypass anti-bot detection, perform interactive operations (click/fill/scroll), extract
  dynamic JavaScript-rendered content, take screenshots.
  Triggers (satisfy ANY one):
  - Target URL is a search results page (Bing/Google/YouTube search)
  - Static fetch (agent-reach/WebFetch) is blocked by anti-bot (captcha/intercept/empty)
  - Need to read logged-in user's private content
  - YouTube, Twitter/X, Xiaohongshu, WeChat public accounts, etc.
  - Task involves "click", "fill form", "scroll", "drag"
  - Need screenshot or dynamic-rendered page capture
metadata:
  author: adapted from eze-is/web-access (MIT licensed)
  version: "1.0.0"
---

<!--
  language: en | zh
  default: en
-->
## What is browser-cdp?

browser-cdp connects directly to your local Chrome via Chrome DevTools Protocol (CDP), giving the AI agent:

- **Full login state** — your cookies and sessions are carried through
- **Anti-bot bypass** — pages that block static fetchers (search results, video platforms)
- **Interactive operations** — click, fill forms, scroll, drag, file upload
- **Dynamic content extraction** — read JavaScript-rendered DOM
- **Screenshots** — capture any page at any point

## Architecture

```
Chrome (remote-debugging-port=9222)
    ↓ CDP WebSocket
CDP Proxy (cdp-proxy.mjs) — HTTP API on localhost:3456
    ↓ HTTP REST
OpenClaw AI Agent
```

## Setup

### 1. Start Chrome with debugging port

```bash
# macOS — must use full binary path (not `open -a`)
pkill -9 "Google Chrome"; sleep 2
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run &
```

Verify:
```bash
curl -s http://127.0.0.1:9222/json/version
```

### 2. Start CDP Proxy

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

# Execute JavaScript
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" \
  -d 'document.title'

# JS click (fast, preferred)
curl -s -X POST "http://localhost:3456/click?target=TARGET_ID" \
  -d 'button.submit'

# Real mouse click
curl -s -X POST "http://localhost:3456/clickAt?target=TARGET_ID" \
  -d '.upload-btn'

# Screenshot
curl -s "http://localhost:3456/screenshot?target=TARGET_ID&file=/tmp/shot.png"

# Scroll (lazy loading)
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"

# Navigate
curl -s "http://localhost:3456/navigate?target=TARGET_ID&url=https://..."

# Close tab
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

## Tool Selection: Three-Layer Strategy

| Scenario | Use | Reason |
|----------|------|--------|
| Public pages (GitHub, Wikipedia, blogs) | `agent-reach` | Fast, low token, structured |
| **Search results** (Bing/Google/YouTube) | **`browser-cdp`** | agent-reach blocked |
| **Login-gated content** | **`browser-cdp`** | No cookies in agent-reach |
| JS-rendered pages | **`browser-cdp`** | Reads rendered DOM |
| Simple automation, isolated screenshots | `agent-browser` | No Chrome setup |
| Large-scale parallel scraping | `agent-reach` + parallel | browser-cdp gets rate-limited |

**Decision flow:**
```
Public content → agent-reach (fast, cheap)
Search results / blocked → browser-cdp
Still fails → agent-reach fallback + record in site-patterns
```

## Known Limitations

- Chrome must use a **separate profile** (`/tmp/chrome-debug-profile`)
- Same-site parallel tabs may get rate-limited
- Node.js 22+ required (native WebSocket)
- macOS: use **full binary path** to start Chrome, not `open -a`

## Site Patterns & Usage Log

```bash
~/.openclaw/skills/browser-cdp/references/site-patterns/   # per-domain experience
~/.openclaw/skills/browser-cdp/references/usage-log.md    # per-use tracking
```

## Origin

Adapted from [eze-is/web-access](https://github.com/eze-is/web-access) (MIT) for OpenClaw.
A bug in the original (`require()` in ES module, [reported here](https://github.com/eze-is/web-access/issues/10)) is fixed in this version.

---

## 🀄 中文说明

> **Language: English version above / 中文说明见下**
## browser-cdp 是什么？

browser-cdp 通过 Chrome DevTools Protocol (CDP) 直连你本地的 Chrome，让 AI Agent 能够：

- **携带完整登录态** — 你的 cookies 和 sessions 全部保留
- **绕过反爬检测** — 搜索结果页、视频平台等静态抓取被拦截的页面
- **执行交互操作** — 点击、填表、滚动、拖拽、文件上传
- **提取动态内容** — 读取 JavaScript 渲染后的 DOM
- **随时截图** — 任意时间点截取页面

## 架构

```
Chrome (remote-debugging-port=9222)
    ↓ CDP WebSocket
CDP Proxy (cdp-proxy.mjs) — HTTP API (localhost:3456)
    ↓ HTTP REST
OpenClaw AI Agent
```

## 安装配置

### 1. 启动 Chrome（macOS）

```bash
# 必须用完整二进制路径，不能用 open -a（参数传递有问题）
pkill -9 "Google Chrome"; sleep 2
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run &
```

验证：
```bash
curl -s http://127.0.0.1:9222/json/version
```

### 2. 启动 CDP Proxy

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

# 新建标签
curl -s "http://localhost:3456/new?url=https://example.com"

# 执行 JS（读 DOM / 读属性）
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

# 滚动（触发懒加载）
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"

# 导航
curl -s "http://localhost:3456/navigate?target=TARGET_ID&url=https://..."

# 关闭标签
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

## 工具选择：三层分工

| 场景 | 工具 | 原因 |
|------|------|------|
| 公开页面（GitHub、Wikipedia、博客） | `agent-reach` | 速度快、token 少、结构化 |
| **搜索结果页**（Bing/Google/YouTube） | **`browser-cdp`** | agent-reach 被反爬拦截 |
| **登录态私有内容** | **`browser-cdp`** | agent-reach 无 cookies |
| JavaScript 渲染页 | **`browser-cdp`** | 直接读渲染后 DOM |
| 简单自动化、隔离截图 | `agent-browser` | 无需配置 Chrome |
| 大规模并行采集 | `agent-reach` + 并行 | browser-cdp 易被限速 |

**判断顺序：**
```
公开内容 → agent-reach（快、便宜）
搜索结果 / 被反爬 → browser-cdp
仍失败 → agent-reach 备选 + 记录 site-patterns
```

## 已知局限

- Chrome 必须使用**独立 profile**（`/tmp/chrome-debug-profile`）
- 同一站点并行标签页容易被限速
- Node.js 22+（使用原生 WebSocket）
- macOS 必须用**完整路径**启动 Chrome

## 站点经验 & 使用日志

```bash
~/.openclaw/skills/browser-cdp/references/site-patterns/   # 按域名积累经验
~/.openclaw/skills/browser-cdp/references/usage-log.md    # 每次使用记录
```

## 起源

从 [eze-is/web-access](https://github.com/eze-is/web-access)（MIT）提取，针对 OpenClaw 重新设计。
原项目有个 bug（在 ES module 中使用 `require()`），已 [报告给作者](https://github.com/eze-is/web-access/issues/10)，本版本已修复。
