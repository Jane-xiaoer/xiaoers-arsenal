# Roadmap

> 我想加什么。优先级从上到下。

---

## 🟢 短期 (1–2 周)

- [ ] **死链清理巡检** — 每周自动跑一次 `url_corrections.py` 的"探活"模式,检测 cover=null 或 URL 200 + content-zero 的 site,通知我归档
- [ ] **导出页 / 公开 portfolio** — 工具墙顶部加 "/分享给朋友" 按钮,生成不带 AI chat 的纯展示链接 (供朋友看,不消耗我的 Gemini 额度)
- [ ] **微信文章一键引用** — 写公众号时,选中工具能直接生成 Markdown 引用 (含截图 + 简介)
- [ ] **Safari 支持** — 现在 watcher 只盯 Chromium 系。Safari 用 Bookmarks.plist,要单独写 plistlib 解析

---

## 🟡 中期 (1–2 月)

- [ ] **Visual Search** — 给我的截图建 CLIP embedding,可以"截一张图,找类似设计的工具"
- [ ] **跨设备同步** — 现在 watcher 只在 Mac 上跑。手机端我也想能"⌘D"(分享 → 我的 webhook → capture pipeline)
- [ ] **工具相似度图谱** — `alternatives.similarTo` 字段画成 force graph,看哪些工具是"同类",找替代品
- [ ] **使用频率染色** — 卡片 hover 时显示"上次访问 X 天前",颜色随时间淡化,促使我清理冷工具

---

## 🟠 长期 / 探索

- [ ] **多语言版本** — 英文版工具墙(给海外朋友分享),Notion 字段双语
- [ ] **Skill 市场化** — 把 capture pipeline 也沉淀成 Claude Code skill (`tool-curator-skill`),别人 fork 一份就能跑自己的私人工具墙
- [ ] **从工具墙生成"我的工具流"博客** — 每周自动选 3 个本周新加 + 我备注最长的,生成一篇推荐 (公众号 + X)
- [ ] **API 化** — 把 fetchAllSites() 暴露成 GraphQL,我做其他 side project 时能直接调用我的工具库

---

## 🔵 想法收集 (待评估)

- [ ] 撕扯入口能不能做"季节限定" — 春节红包风、圣诞礼盒风,每月换一个
- [ ] 能不能让 AI 帮我**主动发现** — 看我的浏览历史,推荐"你最近看了 X、Y、Z,可能感兴趣的工具是 W"
- [ ] Notion DB 太大后,会不会需要**分库** — 比如"工作工具" / "灵感站点" / "个人玩具"
- [ ] 能不能让访客"匿名留言" — 看到喜欢的工具留个 emoji,我能感知热度

---

## ❌ 不做 (明确放弃)

- ❌ **浏览器扩展** — 故意不做,要保持纯后台守护的清洁性 + 跨浏览器普适性
- ❌ **移动 App** — 我自己用 Mac,移动端体验不是重点
- ❌ **多用户 / SaaS 化** — 这是私人系统,不卖给别人。要分享我会沉淀成 skill / 模板让别人 fork
- ❌ **大模型微调** — Gemini 2.5 Flash 的通用分类能力够用,投入产出不值得
