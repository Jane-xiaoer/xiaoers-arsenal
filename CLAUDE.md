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
| Jane 说"工具墙图片空白" | **先看 deploy.log** 看 token/build 是否挂 → 健康自检会发通知,不会再 36h 没人知道 |
| Jane 说"分类细化" | 改 `code/capture/reclassify.py` 的 `TAXONOMY` (3 处同步,见下) |
| Jane 说"刚才收藏没成功" | 看 `~/projects/website-capture/logs/watcher.log`,搜 `⏭ 跳过重复` 或 Chrome dialog 问题 |
| Jane 想加新浏览器 | **不用改代码** — watcher 自动 discover (`~/Library/Application Support/X/{Default,Profile *}/Bookmarks`) |
| Jane 说"AI 推不准" | 看 `code/wall/app/api/chat/route.ts` 的 systemPrompt + Notion 字段权重 |
| Jane 说"分类不准" | **优先**改 `code/capture/capture.py` 的 `CATEGORY_HINTS` 加边界示例;同时 feedback_collector 会自学她手工改的偏好 |
| Jane 想归档某工具 | Notion 里 `archived: true` 即可,1 分钟 ISR 自动刷 |
| Jane 想改 cover/middle 视觉 | `code/wall/lib/paper-texture.ts` (canvas 2D 画) |
| Jane 想加新分类 | **3 处同步**:`code/capture/capture.py` 的 `TAXONOMY` + `code/capture/reclassify.py` 的 `TAXONOMY` + `code/wall/lib/types.ts` 的 `SUBCATEGORIES` |
| Vercel token 出问题 | 改 `.env.local` 的 `VERCEL_TOKEN`,不用 `vercel login`。详见无人值守章节 |

---

## 无人值守机制 (2026-05-12 改造) ★

系统现在能自己照顾自己,Jane 不用再为日常运维介入。背景:之前几次故障是因为 Jane 不知道后台失败,所以加了三层 ABC:

### A. 永久 Vercel token (替代 `vercel login`)
- `VERCEL_TOKEN` (Full Account, 永不过期) 在 `~/projects/xiaoer-tools-wall/.env.local` 和 `~/.shared-skills/api-registry/.env`
- `capture.py` deploy 加 `--token $VERCEL_TOKEN`,**不依赖 auth.json**
- 历史教训:auth.json 被清空过一次,36h Jane 不知道。改 token 模式后这个故障模式消失

### B. 分类自学反馈 (in-context learning)
- 每次首次分类写到 `~/projects/website-capture/.classification_log.jsonl`
- `feedback_collector.py` 每 30min 扫 Notion 看哪条卡的 Category/Subcategory 被 Jane 改过
- 改过的写到 `.classification_corrections.jsonl`,下次 capture.py 自动注入 Gemini prompt 作 fewshot
- **Jane 改一次,系统记一辈子,类似工具下次自动分对**

### B+. 分类二审 agent (capture 内联,实时质检)
- `capture.py:classification_audit()` — 一审输出后立刻接一个 Gemini call 当「挑刺专家」
- 可推翻一级+二级,必须给 verdict (`ok` / `swap_sub` / `swap_both`),禁止 uncertain
- 注入 Jane 历史更正作 fewshot,二审会越来越懂她
- 修改全部写 `.classification_audit.jsonl` 供 Jane 后期分析「哪类最易错」
- 实测: Bruno Simon 个人作品集 (一审 🎨 视觉/3D 动效) → 二审正确推翻成 🌟 灵感/设计灵感
- Subcategory 字段已经从 rich_text 改成 **select 类型**,Jane 在 Notion 表格视图直接下拉点选

### B 任务延伸: cover 同色检测
- `capture.py:is_uniform_image()` PIL 稀疏采样,主色占比 >85% 即拒
- 同色 → 等 4s 让 canvas/webgl 稳定后重抓
- 仍同色 → 用 og:image 兜底,本地 covers/ 不覆盖原图,发 macOS 通知

### C. 健康自检 + 失败告警 (macOS notification)
- **Deploy 监控**:`capture.py` 每次 deploy 启动一个 monitor 子进程,8s 内 vercel 进程异常退出 → 立刻 `osascript display notification "🚨 capture deploy: ..."`
- **每日自检**:`watcher.py:health_check_once()` 由 healthcheck_loop 每 24h 跑一次:
  - vercel whoami 验证 token
  - deploy.log 近期 Error 计数
  - 工具墙 sites API HTTP 状态
  - 任一异常发通知

### 调试入口
| 现象 | 看哪 |
|---|---|
| Jane 说"刚才好像没成功" | `tail logs/watcher.log` 搜 `📌` 找最近触发,搜 `⏭ 跳过重复` 看是否被 dedup gate 拦 |
| 工具墙图没刷 | `tail logs/deploy.log` 看最近 deploy 是否成功 |
| 系统学到啥分类偏好 | `wc -l ~/projects/website-capture/.classification_corrections.jsonl` |
| 想立刻跑一次自检 | `python3 -c "import sys;sys.path.insert(0,'/Users/jane/projects/website-capture');from watcher import health_check_once;health_check_once()"` |
| 想立刻跑一次反馈学习 | `python3 ~/projects/website-capture/feedback_collector.py` |
| 二审改过了哪几条 / 哪类最易错 | `tail ~/projects/website-capture/.classification_audit.jsonl \| jq` |
| 工具墙卡片是黑图 / 同色屏 | `capture.py:is_uniform_image()` 已自动拒;若 Notion 上仍是黑,大概率 og:image 也没救,Jane 手动改 cover |

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
