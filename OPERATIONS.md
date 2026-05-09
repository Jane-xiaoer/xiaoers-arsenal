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

```bash
# 重新 backfill 所有 cover
unset SSL_CERT_FILE
cd ~/projects/website-capture
python3 backfill_covers.py

# 看结果 (有问题的 site)
cat /tmp/xiaoer-audit/missing-covers.json | python3 -m json.tool

# 重 deploy 让 Vercel 看到新 covers
cd ~/projects/xiaoer-tools-wall
vercel --prod --yes
```

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
