---
layout: default
title: "在 Claude Desktop 中配置 TinyFish MCP"
date: 2026-05-07 14:00:00 +0800
categories: [AI, 工具]
tags: [claude, mcp, tinyfish, mcp-remote, ai-tools]
description: "记录在 Claude Desktop 中通过 mcp-remote 接入 TinyFish Web Agent 的配置写法，以及与官方推荐写法的区别。"
---

[TinyFish](https://agent.tinyfish.ai) 是一个云端 Web Agent 服务，通过 MCP（Model Context Protocol）协议暴露工具，让 Claude 能够代替你搜索网页、抓取内容、执行多步骤浏览器自动化操作。本文记录如何在 Claude Desktop 中将它接入，重点说明 `mcp-remote` 的配置写法。

---

## 🐟 TinyFish 提供哪些能力？

连接后，Claude 可以调用以下工具：

- **`fetch_content`** — 在真实浏览器中渲染页面并提取干净的文本内容，支持 JS 渲染页面
- **`search`** — 搜索网页，返回带标题、摘要和 URL 的结构化结果
- **`run_web_automation`** — 给定 URL 和自然语言目标，执行多步骤网页自动化（点击、填表、导航等）
- **`run_web_automation_async`** — 同上，但立即返回 `run_id`，适合长时任务
- **`create_browser_session`** — 创建远端 Chrome 会话，返回 CDP 连接信息，供 Playwright / Puppeteer 直接控制

---

## 🔌 两种配置写法

TinyFish MCP 服务器地址为 `https://agent.tinyfish.ai/mcp`，支持 OAuth 2.1 认证。  
Claude Desktop 有两种方式接入远端 MCP 服务器：

### 官方推荐写法（新版 Claude Desktop）

较新版本的 Claude Desktop 已原生支持在配置中直接填写 `url` 字段，由客户端自行处理 OAuth 流程：

```json
{
  "mcpServers": {
    "tinyfish": {
      "url": "https://agent.tinyfish.ai/mcp"
    }
  }
}
```

首次启动后，Claude Desktop 会自动打开浏览器完成 OAuth 授权。

---

### mcp-remote 代理写法（兼容性更强）

部分版本的 Claude Desktop 不支持直接填 `url` 字段（`claude_desktop_config.json` 只理解 `command` / `args` 形式的本地 stdio 进程）。这时需要借助 **`mcp-remote`**，在本地起一个轻量代理进程，将 stdio 转发给远端 HTTP MCP 服务器，同时处理 OAuth 认证。

**我目前使用的配置：**

```json
{
  "mcpServers": {
    "agent-tinyfish-ai": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://agent.tinyfish.ai/mcp"
      ]
    }
  }
}
```

参数说明：

| 字段 | 值 | 说明 |
|------|----|------|
| `command` | `npx` | 用 npx 直接运行，无需全局安装 |
| `-y` | — | 遇到确认提示自动回答 yes |
| `mcp-remote@latest` | — | 始终使用最新版代理包 |
| 末尾 URL | `https://agent.tinyfish.ai/mcp` | TinyFish MCP 服务端地址 |

> **前置条件**：本机需已安装 Node.js（建议 LTS 版本）。`mcp-remote` 包由 `npx` 自动拉取，不需要手动 `npm install`。

---

## 📝 完整配置文件示例

配置文件位于：

| 系统 | 路径 |
|------|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

```json
{
  "mcpServers": {
    "agent-tinyfish-ai": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://agent.tinyfish.ai/mcp"
      ]
    }
  }
}
```

保存后**完全退出**（不是最小化）Claude Desktop，再重新打开，首次启动时会弹出浏览器完成 OAuth 授权。  
授权成功后，聊天框底部会出现 🔨 工具图标，点击可验证 TinyFish 的工具是否已加载。

---

## 🧪 简单测试

授权完成后，在对话中输入：

```
帮我搜索一下"mcp-remote 是什么"
```

Claude 会调用 TinyFish 的 `search` 工具，返回带来源链接的搜索结果，说明接入成功。

---

## 💡 两种写法如何选择？

- 如果你的 Claude Desktop 版本较新，优先试 `url` 写法——更简洁，不依赖 Node.js。
- 如果 `url` 写法不生效（工具图标没出现），换成 `mcp-remote` 写法，兼容性更广。
- 两种写法的 key 名称可以自定义，不影响功能，但建议取个有辨识度的名字（如 `agent-tinyfish-ai`）方便日后排查。

---

## 参考

- [TinyFish MCP Integration 文档](https://docs.tinyfish.ai/mcp-integration)
- [mcp-remote（npm）](https://www.npmjs.com/package/mcp-remote)
- [Anthropic MCP 文档](https://docs.anthropic.com)

---

**完**
