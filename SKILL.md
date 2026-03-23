---
name: browser-cdp
description: >
  Real Chrome browser automation via CDP Proxy — access pages with full user login state,
  bypass anti-bot detection, perform interactive operations (click/fill/scroll), extract
  dynamic JavaScript-rendered content, take screenshots.
  Triggers (满足任一即触发):
  - 目标 URL 是搜索结果页（Bing/Google/YouTube 搜索页）
  - 静态抓取（agent-reach/WebFetch）被反爬拦截（验证码/拦截页/空内容）
  - 需要读取已登录用户的私有内容
  - YouTube、Twitter/X、小红书、微信公众号等平台内容
  - 任务涉及"点击"、"填表"、"滚动加载"、"拖拽"
  - 需要截图、截取动态渲染页面
metadata:
  author: adapted from eze-is/web-access (MIT licensed)
  version: "1.0.0"
---

# Browser CDP — Real Chrome with Login State

## 核心价值

| 能力 | 说明 |
|------|------|
| **真实 Chrome** | 直连用户日常 Chrome，携带完整登录态 |
| **反爬绕过** | 搜索结果页（Google/Bing）、YouTube 等不被 static fetcher 支持的站 |
| **交互操作** | 点击、填表、滚动、文件上传 — 任何浏览器能做的事 |
| **截图** | 任意时间点截图，动态页面截屏 |

## 前置条件

```bash
# Chrome 必须开启远程调试端口（profile 建议用独立的）
pkill -9 "Google Chrome"; sleep 2
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run &

# 确认就绪
curl -s http://127.0.0.1:9222/json/version  # 应返回 JSON
```

## 启动 CDP Proxy

```bash
# 启动（常驻进程）
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &

# 验证
sleep 3
curl -s http://localhost:3456/health  # {"connected":true}
```

## API 参考

```bash
# 列出所有标签页
curl -s http://localhost:3456/targets

# 新建标签（后台打开 URL）
curl -s "http://localhost:3456/new?url=https://example.com"

# 执行 JS（读 DOM / 读属性 / 操控元素）
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" \
  -d 'document.title'

# JS 点击（快速，推荐优先使用）
curl -s -X POST "http://localhost:3456/click?target=TARGET_ID" \
  -d 'button.submit'

# 真实鼠标点击（触发 hover/文件对话框等）
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

## 工具选择策略（三层分工，已验证）

**目标：降低 token 消耗 + 提高成功率**

| 场景 | 首选 | 原因 |
|------|------|------|
| GitHub、Wikipedia、公开博客/文章 | agent-reach / WebFetch | 速度快、token 少、结构化 |
| **搜索结果页**（Bing/Google/YouTube 搜索） | **browser-cdp** | agent-reach 被反爬拦截 |
| **登录态私有内容** | **browser-cdp** | agent-reach 无 cookie |
| 动态 JS 渲染页 | **browser-cdp** | 直接读渲染后 DOM |
| 简单自动化、截图验证 | agent-browser | 隔离环境，无需配置 Chrome |
| 大规模并行数据采集 | agent-reach + 并行 | browser-cdp 易被限速 |

**判断顺序：**
```
1. 公开内容（GitHub/Wikipedia/博客）→ agent-reach（快、便宜）
2. 搜索结果页 / 被反爬拦截 → browser-cdp
3. browser-cdp 仍失败 → agent-reach 备选 + 记录到 site-patterns
```

## 浏览哲学

**像人一样思考：带着目标进入，边看边判断。**

```
① 明确目标 — 什么算完成？需要什么信息/操作？
② 选最直接的起点 — 一次成功最好，不成功则调整
③ 每步结果都是证据 — 方向错了立即换，不在一个方式上反复重试
④ 完成后停止 — 不为"完整"浪费代价
```

**程序化 vs GUI 交互：**
- 程序化（eval DOM、构造 URL）：快、准，但容易被反爬
- GUI 交互（点击、滚动）：像人一样，确定性高，但步骤多

程序化受阻 → 立即换 GUI 交互。

## 站点经验积累

访问新网站时，将站点特征记录到参考目录：
```bash
~/.openclaw/skills/browser-cdp/references/site-patterns/
```
按域名创建 `.md` 文件，记录：
- URL 模式（哪些路径需要登录）
- 反爬机制（检测方式、绕过方法）
- 已知陷阱（懒加载、无限滚动等）

## 故障排除

**Chrome 端口不开？**
```bash
# macOS 必须用完整路径启动，open -a 有参数传递问题
pkill -9 "Google Chrome"; sleep 2
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile &
```

**Proxy 连接失败？**
```bash
curl -s http://127.0.0.1:9222/json/version  # 确认 Chrome 在响应
pkill -f cdp-proxy.mjs
node ~/.openclaw/skills/browser-cdp/scripts/cdp-proxy.mjs &
```

## 已知局限

- Chrome 须为独立 profile（`/tmp/chrome-debug-profile`），不能共享日常 Chrome 数据
- 并行操作受限：同一站点的多个标签页容易被限速
- Node.js 22+ 才能运行（使用原生 WebSocket）
