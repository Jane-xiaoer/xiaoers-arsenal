# Architecture

## 全局数据流

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  USER                                         │
│  在任意 Chromium 浏览器 (Chrome/Brave/Edge/Tabbit/Doubao/Comet/Quark/Dia)     │
│  按 ⌘D                                                                        │
└──────────────────────┬───────────────────────────────────────────────────────┘
                       │ Bookmark 文件被 Chrome 改写
                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  watcher.py  (Python · launchd 守护)                                          │
│  · FSEvents (watchdog) 实时监听 ~/Library/Application Support/                │
│  · auto-discover: 任何 Chromium fork 第一次按 ⌘D 自动发现                    │
│  · 比对 date_added vs last_processed → 识别新增 URL                           │
│  · 调 capture.py URL                                                          │
└──────────────────────┬───────────────────────────────────────────────────────┘
                       │ subprocess.Popen
                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  capture.py  (Python · 主 pipeline)                                           │
│                                                                               │
│  1. fetch_page(url) → HTML / og:image (反爬时跳过)                           │
│  2. Playwright headless 截图 → JPG bytes                                      │
│  3. analyze() → Gemini 2.5 Flash 多模态:                                      │
│       · name / headline / category / subcategory                              │
│       · capabilities / scenarios / tags / search_keywords                     │
│       · alternatives (replaces / similar / pairs)                             │
│       · 反爬时 fallback → capture_with_search.py                              │
│         (Gemini + Google Search grounding)                                    │
│  4. 写 Notion page (cover = file_upload)                                      │
│  5. 写 Obsidian vault (markdown + 截图)                                       │
│  6. 写本地 ~/projects/xiaoer-tools-wall/public/covers/{id}.jpg                │
│  7. POST /api/revalidate  → ISR 刷新                                          │
│  8. subprocess vercel --prod --yes  → 上传新 cover 到 CDN                     │
└──────────────────────┬───────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Notion DB              │    │  Obsidian Vault  │    │  Vercel CDN       │
│  (主存储)                │    │  (本地备份)       │    │  (覆盖图静态)      │
└──────┬──────────────────┘    └──────────────────┘    └──────┬───────────┘
       │ Notion API                                            │
       ▼                                                       │
┌──────────────────────────────────────────────────────────────┴───────────────┐
│  Next.js 工具墙  (https://xiaoer-tools-wall.vercel.app)                       │
│                                                                               │
│  ┌─ HomeShell ─────────────────────────────────────────────┐                │
│  │  Stage = cover / middle / wall                          │                │
│  │  ├── Cover TearGate (z=100, 黑色"那个网站叫啥来着?")     │                │
│  │  ├── Middle TearGate (z=99, 米色 "小耳的弹药库")         │                │
│  │  └── ToolsWall (z=0, 卡片墙)                            │                │
│  │       ├── Hero 首屏: AI 多轮对话                         │                │
│  │       ├── Sidebar: 12 一级 × 二级折叠                   │                │
│  │       └── Grid: 348 张卡片,3D hover                     │                │
│  └─────────────────────────────────────────────────────────┘                │
│                                                                               │
│  /api/sites          ISR=60s · fetchAllSites() from Notion                   │
│  /api/chat           三层鉴权 + Upstash Redis 限流 + Gemini multi-turn       │
│  /api/revalidate     接收 capture.py 的刷新通知                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 子系统职责

### 1. 抓取层 `code/capture/`

| 文件 | 职责 |
|---|---|
| `watcher.py` | FSEvents 监听 + 自动发现浏览器 + 调用 capture |
| `capture.py` | 主 pipeline (700+ 行) Playwright/Gemini/Notion/Obsidian/Vercel |
| `capture_with_search.py` | 反爬站兜底 (Gemini + Google Search grounding) |
| `backfill_covers.py` | 批量从 Notion 拉所有 cover URL → 下载到 `public/covers/` |
| `refetch_covers.py` | og:image 抓取 (代理) — 失败 site 兜底 |
| `refetch_via_proxy.py` / `refetch_via_playwright.py` | 进一步兜底 |
| `reclassify.py` | Gemini 批量重分类 (12 一级 × 二级) → PATCH Notion |
| `url_corrections.py` | URL 录错/死链批量修复脚本 |

**部署方式**: `~/Library/LaunchAgents/com.jane.bookmark-watcher.plist` → launchd 开机自启 + crash 自动重启

### 2. 展示层 `code/wall/`

| 路径 | 职责 |
|---|---|
| `app/page.tsx` | 入口,fetchAllSites() server-render → HomeShell |
| `app/api/sites/route.ts` | 透传 Notion 数据 (ISR=60s) |
| `app/api/chat/route.ts` | AI 推荐 + 三层鉴权 (master 8005 / BYOK / free) |
| `app/api/revalidate/route.ts` | webhook 接 capture.py 刷新通知 |
| `lib/notion.ts` | Notion → Site type 映射,扫描本地 covers 缓存 ID |
| `lib/types.ts` | Site type + CATEGORIES + SUBCATEGORIES 常量 (client-safe) |
| `lib/ratelimit.ts` | IP+xfp 限流 (Upstash REST API) |
| `lib/paper-texture.ts` | Canvas 2D 画第一/二层纸面 |
| `components/HomeShell.tsx` | 多层 staging + sessionStorage |
| `components/TearGate.tsx` | Three.js Verlet 撕扯 (~540 行) |
| `components/ToolsWall.tsx` | 主 UI (~700 行) |
| `components/SettingsPanel.tsx` | ⚙️ 弹窗 (master / BYOK 配置) |
| `middleware.ts` | xfp cookie 指纹 |
| `public/covers/{id}.jpg` | 静态 CDN |

**部署**: Vercel · Production at `https://xiaoer-tools-wall.vercel.app` · 自动 build & ISR

### 3. 沉淀层 `code/skills/`

| Skill | 状态 |
|---|---|
| `tearable-cloth` | ✅ 已独立 [GitHub repo](https://github.com/Jane-xiaoer/claude-skill-tearable-cloth) · MIT |

---

## 关键技术决策

### 为什么 FSEvents 而不是 poll?
- poll 3s 间隔: 每秒读 10 个 JSON 文件,99.9% 时间无变化 → 浪费
- FSEvents: 内核 push,几毫秒延迟,平时 0 CPU
- 结果: 用户 ⌘D 几乎瞬时触发 capture

### 为什么 cover 存本地 + Vercel deploy?
- Notion 给的 S3 cover URL **1 小时过期** (`X-Amz-Expires=3600`)
- 不能让前端缓存 → 否则 1h 后图全 broken
- 解法: capture 时同时写到 `public/covers/{id}.jpg` → Vercel deploy 上传 → CDN 永久静态

### 为什么三层鉴权?
- 工具墙公开 → AI chat 用 Gemini API,Jane 出钱
- master `8005` → Jane 自己用,无限
- BYOK → 用户填自己的 Gemini key,他自己出钱
- free (默认) → IP+xfp 指纹限流 3 次/天 (Upstash Redis)

### 为什么 Three.js + Verlet 撕扯?
- 比 CSS 假动效更有"仪式感"
- 撕扯 = 用户主动参与的微交互,比 fade in/out 更有记忆点
- 反编译 pushmatrix 提炼参数,用 Web Worker-less 单线程实现 (够快,90×56 grid 60fps)

### 为什么两层 cover/middle?
- Layer 1 (黑底"那个网站叫啥来着?") = 引发好奇 / 关联工具墙的核心痛点
- Layer 2 (米色 "小耳的弹药库" + 343 件) = 揭示品牌
- 撕开 1 层立刻看到 2 层 (z-index 同时挂载),无 1 帧 flash

### 为什么 sessionStorage 而不是 localStorage?
- 这是仪式,不是工具
- 关 tab 再开 → 再撕一次,这是 feature 不是 bug
- 同 tab 切换不烦人

---

## 性能 / 成本

- watcher: idle ~ 0% CPU (FSEvents,无 poll)
- capture pipeline: 单次 ~ 30s (Playwright 10s + Gemini 15s + Vercel deploy 30s 后台)
- Vercel: free tier · 月度 build minutes 用不完 (Jane 一周收 5-10 个新工具)
- Gemini: 免费 1500 req/day,Jane 自用远低于
- Upstash Redis: free tier 10K req/day,限流计数远低于
- Notion: API 免费,容量上限 2000 blocks/page (一个工具一个 page,无压力)

---

## 安全 / 隐私

- Notion token / Gemini key / 限流 token: 全部在 `~/projects/xiaoer-tools-wall/.env.local`,git ignore
- 工具墙公开,但写入 Notion 的能力被 token 锁死,游客只能读
- AI chat 三层鉴权防白嫖
- 浏览器书签数据本地处理,**不上传任何第三方** (除了 Gemini 拿到 URL + screenshot 做分析)
