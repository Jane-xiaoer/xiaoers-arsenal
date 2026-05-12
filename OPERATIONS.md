# Operations Runbook

> 怎么部署 / 排错 / 加新功能。Jane 自己 + 未来 Claude session 都用得上。

---

## 🚀 常规部署

### 完整重 deploy 工具墙 (代码改动后)

```bash
cd ~/projects/xiaoer-tools-wall
unset SSL_CERT_FILE
rm -rf .next                    # clean build cache
npm run build                   # local sanity check
vercel --prod --yes             # ~30-45s
```

### 强刷工具墙数据 (Notion 改了内容)

```bash
SECRET=$(grep REVALIDATE_SECRET ~/projects/xiaoer-tools-wall/.env.local | cut -d= -f2)
curl -X POST "https://xiaoer-tools-wall.vercel.app/api/revalidate?secret=$SECRET"
```

### 重启 watcher

```bash
pkill -f "watcher.py"            # launchd 10s 后自动恢复
# 验证
ps aux | grep watcher.py | grep -v grep
tail -20 ~/projects/website-capture/logs/watcher.log
```

---

## 🔍 常见问题排查

### Jane 收藏没成功 / 工具墙没出现新卡

```bash
# 1. 查 watcher 是否活着
ps aux | grep watcher.py | grep -v grep

# 2. 看最近日志
tail -30 ~/projects/website-capture/logs/watcher.log

# 3. 看 capture 日志
tail -30 ~/projects/website-capture/logs/capture.log

# 4. 看 deploy 日志
tail -30 ~/projects/website-capture/logs/deploy.log
```

**最常见原因**:
- **Chrome dialog 没确认** → `⌘D` 弹窗按 Esc 或切标签 = 取消收藏。watcher 看不到
- **新装的浏览器之前没收藏过** → Bookmarks 文件还没创建。第一次按 ⌘D 后下一秒 watcher 自动 discover

### 工具墙图片是空白的

**先看 deploy 是否失败** (2026-05-12 起,失败 8s 内会发 macOS 通知):

```bash
tail -30 ~/projects/website-capture/logs/deploy.log
```

如果 deploy 都失败 → Vercel token 出问题:
```bash
unset SSL_CERT_FILE
TOKEN=$(grep '^VERCEL_TOKEN=' ~/projects/xiaoer-tools-wall/.env.local | cut -d= -f2)
vercel whoami --token "$TOKEN"  # 应该返回 jane-xiaoer
```
失效 → 去 https://vercel.com/account/tokens 重新生成 Full Account 永不过期 token,替换两处 .env (xiaoer-tools-wall/.env.local 和 ~/.shared-skills/api-registry/.env)。**无需 `vercel login`**。

如果 deploy 都成功但 cover 还是缺,才需要 backfill:
```bash
unset SSL_CERT_FILE
cd ~/projects/website-capture
python3 backfill_covers.py
TOKEN=$(grep '^VERCEL_TOKEN=' ~/projects/xiaoer-tools-wall/.env.local | cut -d= -f2)
cd ~/projects/xiaoer-tools-wall && vercel --prod --yes --token "$TOKEN"
```

### 分类不准 / 同一类工具反复要手动改

2026-05-12 起,`feedback_collector.py` 每 30min 扫 Notion,自动学习 Jane 手工改的分类偏好。

看系统已学到啥:
```bash
wc -l ~/projects/website-capture/.classification_corrections.jsonl
tail -5 ~/projects/website-capture/.classification_corrections.jsonl | jq .
```

手动触发一次学习:
```bash
unset SSL_CERT_FILE
python3 ~/projects/website-capture/feedback_collector.py
```

如果是**整类工具**都分错 (例如"音乐生成全部错") → 不靠 fewshot,改源头 prompt:
- `~/projects/website-capture/capture.py` 的 `CATEGORY_HINTS` 多行字符串加边界示例
- 已有 `Suno / Udio / MusicGen / ElevenLabs / TTS → 🔊 声音` 这种范式可参照

### 健康自检 (主动诊断系统状态)

```bash
unset SSL_CERT_FILE
python3 -c "
import sys
sys.path.insert(0, '/Users/jane/projects/website-capture')
from watcher import health_check_once
health_check_once()
"
```
输出 `💚 健康自检全部通过` 或 `🚨 ... 异常详情`。系统每 24h 自跑一次,这里是手动触发。

### 某个工具想归档

不用动代码,Notion 里 `archived: true` 即可。1 分钟 ISR 自动刷,或手动 `/api/revalidate`。

### 某个工具 URL 录错了 / 死链

```bash
# 编辑 ~/projects/website-capture/url_corrections.py 的 CORRECTIONS 数组
# 然后跑
unset SSL_CERT_FILE
/Users/jane/.claude/skills.backup/webapp-testing/venv/bin/python3 \
  ~/projects/website-capture/url_corrections.py
```

### 撕扯感不对了 (太硬 / 太软 / 不撕)

参数都在 `~/projects/xiaoer-tools-wall/components/TearGate.tsx` 顶部。**速查表**: 看 `code/skills/tearable-cloth/SKILL.md` 的 `🎨 Tuning Cheat Sheet` 节。

---

## 🛠 加新功能

### 加新浏览器 (比如 Vivaldi)

**不用改代码**。watcher 自动 discover 任何 `~/Library/Application Support/X/{Default,Profile *,Guest Profile}/Bookmarks` 路径。

第一次在 Vivaldi 按 ⌘D 后,watcher 60s 内 (rediscover 间隔) 自动加入监控。立即生效:

```bash
pkill -f "watcher.py"  # 立即重新发现
```

### 加新分类 / 二级类

```
1. 改 ~/projects/xiaoer-tools-wall/lib/types.ts SUBCATEGORIES
2. 改 ~/projects/website-capture/reclassify.py TAXONOMY (保持同步!)
3. 跑全量重分:
   cd ~/projects/website-capture
   unset SSL_CERT_FILE
   python3 reclassify.py
4. cd ~/projects/xiaoer-tools-wall && vercel --prod --yes
```

### 改 cover / middle 视觉

`~/projects/xiaoer-tools-wall/lib/paper-texture.ts` — 全部 Canvas 2D。改完 vercel deploy 自动出。

### 改撕扯物理

`~/projects/xiaoer-tools-wall/components/TearGate.tsx` 顶部参数。改完按 OPERATIONS.md 顶部部署即可。

---

## 🔐 密钥 / 凭证

```
~/projects/xiaoer-tools-wall/.env.local      (本地 dev)
Vercel Project Settings → Environment Variables  (生产)

需要的 keys:
  NOTION_TOKEN           Notion integration token
  NOTION_DB_ID           主数据库 (database id)
  GEMINI_API_KEY         Jane's master Gemini key
  REVALIDATE_SECRET      ISR 刷新口令
  KV_REST_API_URL        Upstash Redis
  KV_REST_API_TOKEN
  MASTER_PASSWORD=8005   前端 ⚙️ 用
```

⚠️ **绝不 commit `.env.local`**,`.gitignore` 已处理。

---

## 📊 监控

- **Watcher 状态**: `ps aux | grep watcher.py`
- **Vercel 部署**: <https://vercel.com/jane-xiaoers-projects/xiaoer-tools-wall>
- **Notion DB**: <https://www.notion.so/0a1dacea18844fabb467e91eab858891>
- **Upstash Redis**: <https://console.upstash.com/redis>

---

## 🆘 紧急情况

### 工具墙完全挂了

```bash
# 1. 检查 Vercel 状态
curl -sI https://xiaoer-tools-wall.vercel.app | head -3

# 2. 看最近 deploy 状态
unset SSL_CERT_FILE
vercel ls --yes 2>&1 | head -5

# 3. rollback 到之前 deploy (如果新 deploy 出问题)
vercel rollback --yes
```

### Notion API 挂了 / token 过期

工具墙 ISR 缓存 60s,会显示之前缓存。Jane 大概不会立刻发现。
修 token 后 deploy 一次或 hit revalidate webhook 即可。

### Gemini API 用完额度

free tier 用户报 429 → 弹设置让用 master 8005 或 BYOK。Jane 用 master 不受影响。

### 数据全没了 (灾难恢复)

- Notion: 有 30 天 trash 可恢复
- 本地 covers/: 跟 Vercel deploy 走,Vercel 保留 deploy 历史
- Obsidian backup: 在 `~/Documents/Obsidian Vault/工具收藏/` 有完整 markdown 备份
- 三处都挂的概率极低

---

## 👣 Friends Wall (脚印墙)

### Notion DB

- DB ID: `35b1c99d-3c96-81e7-b600-df5f4bd55189`
- Data Source ID: `35b1c99d-3c96-8198-bc80-000bf7f5c6c7`
- Title: 脚印墙 / Friends Wall
- Parent: 工具收藏库 root page (`2411c99d-3c96-8000-81a0-cc09b7f46d5f`)
- Fields: `Name` (title) · `Color` (select: mint/cream/cherry/sky/lime) · `Message` (rich_text) · `Fingerprint` (rich_text)

DB ID 同时存于:
- `~/projects/xiaoer-tools-wall/.env.local` → `FOOTPRINTS_DB_ID` / `FOOTPRINTS_DATA_SOURCE_ID`
- Vercel project env → `FOOTPRINTS_DB_ID` / `FOOTPRINTS_DATA_SOURCE_ID`
- `~/.shared-skills/api-registry/.env` → 同上

### Manual DB rebuild (if Notion DB ever gets nuked)

```bash
unset SSL_CERT_FILE
TOKEN=$(grep NOTION_TOKEN ~/projects/xiaoer-tools-wall/.env.local | cut -d= -f2)
PARENT="2411c99d-3c96-8000-81a0-cc09b7f46d5f"  # 工具收藏库 root page

curl -s -X POST "https://api.notion.com/v1/databases" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{
    "parent": {"type":"page_id","page_id":"'"$PARENT"'"},
    "icon": {"type":"emoji","emoji":"👣"},
    "title": [{"type":"text","text":{"content":"脚印墙 / Friends Wall"}}],
    "properties": {
      "Name": {"title":{}},
      "Color": {"select":{"options":[{"name":"mint","color":"green"},{"name":"cream","color":"yellow"},{"name":"cherry","color":"red"},{"name":"sky","color":"blue"},{"name":"lime","color":"default"}]}},
      "Message": {"rich_text":{}},
      "Fingerprint": {"rich_text":{}}
    }
  }' | jq '.id'

# get the new data_sources[0].id
curl -s "https://api.notion.com/v1/databases/<NEW_DB_ID>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Notion-Version: 2025-09-03" | jq '.data_sources[0].id'

# update env vars (.env.local + Vercel project env) and redeploy
```

### API contract

- `GET /api/footprints` → `{ ok, footprints: [{id,name,color,message,createdAt,isMine}] }`
  - Sorted oldest-first (founder #1 = first POST ever).
  - `isMine` is a server-side join (sha256 of `IP|xfp` matches stored Fingerprint).
  - `Fingerprint` itself never leaves the server.
- `POST /api/footprints` body: `{ name?, color?, message? }` → `{ ok, action: "created"|"updated", id }`
  - One footprint per fingerprint (upsert).
  - 5 POSTs/day per fingerprint anti-spam (uses same Upstash limiter as `/api/chat`).
  - Limits: name ≤ 16 chars, message ≤ 50 chars.
  - Empty name → `路过的朋友`; invalid color → `mint`.

### Tier system (rendered client-side)

- Tier 1 (rank 1-10):  80px · founder gold glow · `★ FOUNDER #N` label
- Tier 2 (rank 11-50): 60px · deep palette
- Tier 3 (rank 51-200): 45px · pastel · name on hover
- Tier 4 (rank 201+):  30px · light pastel · name on hover

Layout = concentric annular rings around 法老 anchor (lime cross at center). Each print rotated ±30° with deterministic-pseudo-random placement (so positions stay stable across reloads).
