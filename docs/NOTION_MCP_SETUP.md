# Notion MCP — 安装短指南

> 让 Claude Code (任意 session) 直接和 Arsenal 的 Notion DB 对话 — 不再每次让我手写 Python 调 Notion API。
>
> 适用场景: 想 "/帮我列出本周新增工具"、"/批量改 X 字段"、"/搜含某关键字的工具" 这种,直接对 Claude 自然语言说就行。

---

## 1 分钟安装

### a. 安装官方 Notion MCP

```bash
# Claude Code 自动管理,跑这一句即可
claude mcp add notion 'npx -y @notionhq/notion-mcp-server'
```

### b. 配置 token

把你的 Notion integration token 加到环境(用 capture pipeline 同一个 token 即可,**它已经有 DB 的访问权限**):

```bash
# 已经有了 (capture pipeline 一直在用)
grep NOTION_TOKEN ~/.shared-skills/api-registry/.env
```

把这个 token 给 Notion MCP 用:

```bash
# 在 Claude Code 的 MCP 配置里加 env var (settings.json)
# 或者直接在 shell 里 export 让 npx 看到:
export NOTION_TOKEN=$(grep NOTION_TOKEN ~/.shared-skills/api-registry/.env | cut -d= -f2)
```

### c. 验证

重启 Claude Code,在对话里输入:

> "用 notion mcp 给我列出 xiaoer arsenal 库里最新加的 5 个工具"

如果它能正确返回 — 装好了。

---

## 用得最多的几条命令

我自己最常用:

```
/帮我列出最近 7 天新加的工具
/找一下 Notion 里所有标 "🤖 AI 大模型 / 提示词工程" 的工具
/把 "Bolt" 这条工具的 myNotes 字段改成 ...
/批量给 ✍️ 文字写作 类别加一个 tag "中文友好"
```

---

## 跟 capture pipeline 的边界

| 谁干啥 | 工具 |
|---|---|
| **新增**(浏览器 ⌘D 触发的全自动写入) | `capture.py` (Python pipeline) |
| **修改/查询/批量操作** (临时让 Claude 直接动) | Notion MCP |

两个**互不冲突**,共用同一个 Notion token。

---

## 排错

**Notion MCP 调不动 / 报 401**:
检查 token 是否过期 + 检查 DB 是否对该 integration 开了"share"(在 Notion DB 右上角 share 里加 integration)。

**找不到 DB**:
DB ID 在 `~/.shared-skills/api-registry/.env` 的 `NOTION_DB_ID` 字段。直接告诉 Claude 这个 ID 让它用。

**官方文档**: <https://github.com/makenotion/notion-mcp-server>
