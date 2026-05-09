# Xiaoer's Arsenal — Claude Context

> **Future Claude / AI Agent reads this first**.
> 你是一个 Claude/AI agent,刚进入这个 project,需要 30 秒理解前因后果。读完这个文件你就懂了。

---

## What this project is (one paragraph)

Jane (小耳) 自己用的工具收藏宇宙: 浏览器按 ⌘D → Python watcher (FSEvents) 抓住书签 → Playwright 截图 + Gemini 分析 → 自动写到 Notion + 工具墙 + 自动 Vercel deploy。前端是 Next.js 16 + Three.js Verlet 撕扯入口,部署在 Vercel,数据源是 Notion。

**Domain**: 私人 AI 工具收藏与召回。Jane 每天在浏览器里发现 5–10 个工具,记不住名字 — 这套系统帮她"那个网站叫啥来着?"

---

## Where the code lives

| 子系统 | 路径 (实际) | 包内 symlink | 说明 |
|---|---|---|---|
| 抓取/分析后台 | `~/projects/website-capture/` | `code/capture/` | Python 守护进程 + capture pipeline |
| Next.js 工具墙 | `~/projects/xiaoer-tools-wall/` | `code/wall/` | 前端 + API + Notion 适配器 |
| 撕扯入口 skill | `~/.shared-skills/tearable-cloth/` | `code/skills/tearable-cloth/` | 已独立 skill,已上 GitHub 公开 |

**关键: 代码本体在原位,本包只是文档 + symlink。改代码请直接改原路径,不要在 symlink 里建新文件。**

---

## Where the secrets live

```
~/projects/xiaoer-tools-wall/.env.local
  NOTION_TOKEN          (Notion integration token)
  NOTION_DB_ID          (主数据库 ID)
  GEMINI_API_KEY        (Jane's master key, free tier user 用她的)
  REVALIDATE_SECRET     (POST /api/revalidate?secret=...)
  KV_REST_API_URL       (Upstash 限流)
  KV_REST_API_TOKEN
  MASTER_PASSWORD=8005  (前端 ⚙️ 设置可填,无限制)
```

> Notion token / API key 不要 commit 到 git。已在 .gitignore 防住。

---

## 入口点 (你最需要先看的几个文件)

按重要度排序:

1. **`code/wall/lib/notion.ts`** — Notion → Site type 的解析层。所有 Notion 字段在这定义。
2. **`code/wall/components/ToolsWall.tsx`** — 主前端组件 (~700 行)。包含网格 + sidebar 二级折叠 + AI chat + ⚙️ 设置弹窗。
3. **`code/wall/components/HomeShell.tsx`** — 多层 staging (cover → middle → wall) + sessionStorage 控制。
4. **`code/wall/app/api/chat/route.ts`** — AI 多轮对话 + 三层鉴权 (master / BYOK / free)。
5. **`code/capture/watcher.py`** — FSEvents 监听器,自动发现新浏览器。
6. **`code/capture/capture.py`** — 主 capture pipeline (~700 行)。
7. **`ARCHITECTURE.md`** (同目录) — 整体架构 + 数据流图。

---

## 常见任务 → 该改哪 / 该跑啥

| 任务 | 行动 |
|---|---|
| Jane 说"撕扯感不对" | 调 `code/wall/components/TearGate.tsx` 物理参数,或者引用 `code/skills/tearable-cloth/SKILL.md` 速查表 |
| Jane 说"工具墙图片空白" | 跑 `code/capture/backfill_covers.py` 重抓,然后 `vercel --prod --yes` |
| Jane 说"分类细化" | 改 `code/capture/reclassify.py` 的 TAXONOMY,跑一遍重分类 348 个 |
| Jane 说"刚才收藏没成功" | 看 `~/projects/website-capture/logs/watcher.log`,基本是 Chrome dialog 操作问题 |
| Jane 想加新浏览器 | **不用改代码** — watcher 自动 discover (`~/Library/Application Support/X/{Default,Profile *}/Bookmarks`) |
| Jane 说"AI 推不准" | 看 `code/wall/app/api/chat/route.ts` 的 systemPrompt + Notion 字段权重 |
| Jane 想归档某工具 | Notion 里 `archived: true` 即可,1 分钟 ISR 自动刷 |
| Jane 想改 cover/middle 视觉 | `code/wall/lib/paper-texture.ts` (canvas 2D 画) |
| Jane 想加新分类 | `code/wall/lib/types.ts` 的 `SUBCATEGORIES` + `code/capture/reclassify.py` TAXONOMY 同步 |

---

## 几个 gotcha

1. **macOS `SSL_CERT_FILE` 坑** — 几乎所有 bash 命令前先 `unset SSL_CERT_FILE`,否则 curl/Python TLS 会挂 (有个失效路径 `/tmp/apple-roots.pem`)。
2. **Notion S3 cover URL 1 小时过期** — 不能依赖,我们截下来存到 `public/covers/{id}.jpg`,跟 Vercel deploy 走静态。
3. **Vercel deploy 必须** — 写完本地 `public/covers/` 不会自动同步到 Vercel,需要 `vercel --prod --yes`。`capture.py` 末尾会自动触发,但调试时需要手动跑一次。
4. **lib/notion.ts 启动时缓存 cover IDs** — 加了新 cover 后必须 redeploy 才能让 Vercel function 看到。
5. **3 个 Chrome profile 中 Profile 3 是 Jane 主用的** ("Your Chrome"),Default 几乎不用。
6. **bb-browser 不稳** — 如果 daemon 断了重连麻烦。常规自动化优先用 Playwright 或 `agent-browser`。

---

## 关联 memory 文件 (~/.claude/memories/)

- `project_xiaoer_tools_wall_costguard.md` — 三层鉴权 + 限流 + 封面持久化 + 两层分类体系
- `browser-automation-playbook.md` — bb-browser 端口 19825 / OAuth 模板 / React form 坑
- `feedback_jane_no_english_commands.md` — Jane 用中文指挥
- `MEMORY.md` 索引里搜 "xiaoer-tools-wall" 看完整列表

---

## 重要原则 (Jane 反复强调过的)

1. **系统正常,不要每次让 Jane 来搞** — 任何问题先想"系统怎么不哑火",而不是手动 patch
2. **CLI 优先,别先写脚本** — 本机有 jq / glow / yt-dlp / http / xq / bat / dasel
3. **API key 不要问 Jane** — 先 source `~/.shared-skills/api-registry/.env`
4. **不熟悉的领域必先研究** — 走研究协议 (本地 KB → skill → 网络),不能用训练知识硬答
5. **测试要自己做完再报告** — Jane 反复说过"你不能每次都让我来帮你看",自己截图对比再交差

---

## 怎么 deploy / restart 常用命令

```bash
# 重 deploy 工具墙
cd ~/projects/xiaoer-tools-wall && unset SSL_CERT_FILE && vercel --prod --yes

# 强刷 ISR
SECRET=$(grep REVALIDATE_SECRET ~/projects/xiaoer-tools-wall/.env.local | cut -d= -f2)
curl -X POST "https://xiaoer-tools-wall.vercel.app/api/revalidate?secret=$SECRET"

# 重启 watcher (launchd 自动恢复)
pkill -f "watcher.py"

# 看 watcher log
tail -f ~/projects/website-capture/logs/watcher.log
```
