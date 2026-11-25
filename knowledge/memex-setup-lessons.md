---
title: "Memex 接入 Cursor 的经验"
type: insight
summary: "Memex 的读取靠本地 MCP，写入靠 Cursor API Key 开 PR；OpenAI 密钥和 GitHub 网页登录都不能代替这两步。"
tags: [memex, cursor, github]
created: "2026-11-25"
edges:
  - path: /knowledge/knowledge-base-init.md
    label: about
    description: "初始化笔记记录接入已完成这一事实"
  - path: /knowledge/cursor-session-index.md
    label: source
    description: "这条经验来自 2026-11-25 的接入会话"
---

## Observation

仓库 `golmin/memex` 已经能在本地搜、读知识，但 `kb_add` 还不能自动开 PR。卡点不在仓库地址，而在两把不同的钥匙：Cursor Cloud Agent 要 `cursor_...`，GitHub 推送要 `gh` 登录。

## Evidence

- 本地服务是 `uv run memex`，MCP 地址 `http://localhost:8787/mcp`。`config.yaml` 里仓库是 `golmin/memex`，默认分支是 `master`，不是模板里的 `main`。
- `.env` 里如果填的是 `sk-` 开头的 OpenAI 密钥，`kb_add` 会返回 `Invalid CURSOR_API_KEY`。这把密钥只能放在 `OPENAI_API_KEY`，给可选的语义搜索用。
- 这台机器访问 `github.com` 网页会超时，`gh auth login` 的浏览器设备码会报 `unexpected EOF`。`api.github.com` 是通的，所以要用 classic token（`ghp_`，至少包含 `repo`、`read:org`、`workflow`）粘贴登录。
- `brew install gh` 会被阿里云瓶装源上缺失的 Portable Ruby 4.0.10 卡住。可以直接把官方 `gh` 二进制放到 `~/.local/bin`。
- 规则要放在 `~/.cursor/rules/`，并配一个 `~/.cursor/plugins/local/` 插件，不要只放在某个业务仓库里。改完后需要 Reload Window。

## Implications

以后用户要求把任务总结存进知识库时，先调 `kb_add`。没有 `cursor_...` 密钥就说明原因，再用已登录的 `gh` 按知识库格式直接提交 PR，不要假装 Cloud Agent 已经写好。
