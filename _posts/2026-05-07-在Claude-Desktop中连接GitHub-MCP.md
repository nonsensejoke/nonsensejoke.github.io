---
layout: default
title: "在 Claude Desktop 中连接 GitHub MCP"
date: 2026-05-07 12:00:00 +0800
categories: [AI, 工具]
tags: [claude, mcp, github, docker, ai-tools]
description: "手把手配置 GitHub MCP Server，让 Claude Desktop 能直接读写你的 GitHub 仓库、Issue 和 PR。"
---

**Model Context Protocol（MCP）** 是 Anthropic 推出的开放标准，允许 AI 助手通过统一接口连接外部工具和数据源。本文介绍如何在 Claude Desktop 中接入 GitHub 官方 MCP Server，让 Claude 直接操作你的 GitHub 仓库。

---

## 🗂️ 两种接入方式

GitHub MCP Server 有两种运行形式，各有适用场景：

| | 本地 Docker | 远程 HTTP（mcp-remote） |
|--|------------|------------------------|
| 运行位置 | 你的电脑 | GitHub 云端服务器 |
| 需要 Docker | ✅ 是 | ❌ 否（需要 Node.js） |
| 版本更新 | 手动拉取新镜像 | 自动（GitHub 托管） |
| 网络要求 | 无 | 需要能访问 `api.githubcopilot.com` |
| 配置难度 | 低 | 低 |

两种方式功能完全相同，选一种即可。

---

## 🧩 能做什么？

连接 GitHub MCP 后，你可以用自然语言让 Claude：

- 列出仓库文件和目录内容
- 查看、创建、修改 Issue 和 Pull Request
- 搜索代码和仓库
- 读取 Commit 记录
- 自动化 CI/CD 和 Workflow 相关操作

---

## ✅ 前置条件

- 已安装 [Claude Desktop](https://claude.ai/download)
- 已安装 [Docker Desktop](https://www.docker.com/products/docker-desktop/) 并确保其正在运行
- 拥有 GitHub 账号，且能生成 Personal Access Token（PAT）

---

## 🔑 第一步：生成 GitHub Personal Access Token

1. 前往 [github.com/settings/personal-access-tokens/new](https://github.com/settings/personal-access-tokens/new)
2. 填写 Token 名称（如 `claude-mcp`）
3. 按需勾选权限，推荐至少勾选：
   - `repo`（仓库读写）
   - `read:org`（读取组织信息）
4. 点击 **Generate token**，**立即复制**保存（离开页面后将无法再次查看）

---

## 📝 第二步：编辑配置文件

Claude Desktop 通过 `claude_desktop_config.json` 加载 MCP Server。

**配置文件位置：**

| 系统 | 路径 |
|------|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

在 Claude Desktop 中也可以通过 **Settings → Developer → Edit Config** 直接打开。

---

在现有 JSON 中新增 `mcpServers` 字段（注意不要破坏原有 JSON 结构），根据你选择的方式二选一：

### 方式一：本地 Docker（推荐，最稳定）

需要提前安装并启动 Docker Desktop。

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm", "-e",
        "GITHUB_PERSONAL_ACCESS_TOKEN",
        "ghcr.io/github/github-mcp-server"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_你的Token"
      }
    }
  },
  "preferences": {
    ...原有内容保持不变...
  }
}
```

> ⚠️ 2025 年 4 月起，旧的 npm 包 `@modelcontextprotocol/server-github` 已停止维护，请使用上方 Docker 镜像。

---

### 方式二：远程 HTTP（无需 Docker，需要 Node.js）

`claude_desktop_config.json` 本身只支持本地 stdio 通信，无法直接填写远程 HTTP 地址。  
解决方案是使用 **`mcp-remote`** ——一个轻量级 Node.js 代理，由 Claude Desktop 在本地启动，负责将 stdio 转发给远端的 GitHub MCP 服务器。

**前置条件**：安装 [Node.js](https://nodejs.org/)（建议 LTS 版本），`mcp-remote` 会由 `npx` 自动拉取，无需手动安装。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://api.githubcopilot.com/mcp/",
        "--transport",
        "http-only",
        "--header",
        "Authorization: Bearer ghp_你的Token"
      ]
    }
  },
  "preferences": {
    ...原有内容保持不变...
  }
}
```

两种方式将 `ghp_你的Token` 替换为第一步中生成的真实 PAT 即可。

---

## 🔄 第三步：重启 Claude Desktop

**完全退出**（不是最小化）后重新打开：

- macOS：`Cmd+Q` 退出，或菜单栏 → Quit Claude
- Windows：右键系统托盘图标 → Quit

重启后，在聊天框底部应出现一个 **🔨 工具图标**，点击可查看已加载的 MCP 工具列表。

---

## 🧪 第四步：测试连接

在对话框中输入：

```
帮我查看我的 GitHub 账号信息
```

Claude 会调用 `get_me` 工具，返回类似：

```
用户名：nonsensejoke
昵称：黑猫白猫
公开仓库：177 个
```

连接成功！接下来可以尝试：

```
读取我的 nonsensejoke.github.io 仓库里 _posts 目录下的文件列表
```

---

## 🛠️ 常见问题

### 🔌 工具图标没有出现？

- 确认 Docker Desktop 正在运行（首次使用会自动拉取镜像，需要等待）
- 检查 JSON 格式是否正确（缺少逗号、括号不匹配都会导致静默失败）
- 重启是否是真正退出而非最小化

### 🔒 提示权限不足？

检查 PAT 是否包含操作所需的 scope。例如读取私有仓库需要 `repo`，读取组织成员需要 `read:org`。

### 🐢 首次响应很慢？

- **Docker 方式**：第一次运行时需要从 `ghcr.io` 拉取镜像（约 50MB），之后缓存在本地，启动速度正常。
- **mcp-remote 方式**：首次运行 `npx mcp-remote` 时会自动下载依赖包，同样只需等待一次。

### ❓ mcp-remote 方式：提示找不到 npx？

Claude Desktop 启动时使用的是精简 PATH，可能找不到 `npx`。需要在配置中使用**完整路径**：

- macOS / Linux：将 `"command": "npx"` 改为 `"command": "/usr/local/bin/npx"`（通过 `which npx` 查找实际路径）
- Windows：改为 `"command": "C:\\Program Files\\nodejs\\npx.cmd"`

---

## 📌 小结

| 步骤 | 操作 |
|------|------|
| 1 | 生成 GitHub PAT |
| 2 | 选择 Docker 或 mcp-remote 方式，编辑 `claude_desktop_config.json` |
| 3 | 完全退出并重启 Claude Desktop |
| 4 | 与 Claude 对话，验证工具图标出现 |

整个流程不超过 10 分钟，配置完成后 Claude 就能直接读写你的 GitHub 数据，无需在浏览器和 AI 之间来回切换。

---

## 参考

- [github/github-mcp-server](https://github.com/github/github-mcp-server)
- [Anthropic MCP 文档](https://docs.anthropic.com)
- [GitHub Personal Access Tokens](https://github.com/settings/tokens)

---

**完**
