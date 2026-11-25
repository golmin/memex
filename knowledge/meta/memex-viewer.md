---
title: "知识库页面和 MCP 不是同一个端口"
type: note
summary: "8787 只提供 MCP，打开根路径是 404。图形页面由 viewer 目录在 8788 提供，正文 HTML 要在生成数据时渲染出来。"
tags: [memex, 知识库]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/meta/knowledge-base-init.md
    label: about
    description: "知识库怎么接入，和页面怎么打开是两件事"
---

## 问题

本地已经跑起 Memex，浏览器打开 `http://127.0.0.1:8787/` 是 404。在仓库根目录起静态服务，看到的是文件列表和原始 Markdown。详情页标题和关联都有，正文是空的。

## 卡点

8787 上的进程只提供 MCP，路径在 `/mcp`。根路径没有页面，所以 404 是这个进程的正常结果。

在仓库根目录开的静态服务，打开的是磁盘上的文件。带列表和关系图的页面在 `viewer/`，它读的是已经生成的 `data.json`，不负责把 Markdown 现转成 HTML。

详情页能显示标题和边，说明 JSON 里有元数据。正文空白，是生成时没有 Markdown 渲染库，`body_html` 被写成了空字符串。浏览器若缓存了那一版 JSON，换上新文件之后仍会看到空白。

## 解决思路

图形页面从 `viewer` 目录起静态服务，地址是 `http://127.0.0.1:8788/`。生成 `data.json` 前装上带 mistune 的 viewer 依赖。没有 mistune 时，生成脚本把正文转义放进 `<pre>`，避免再写出空的 `body_html`。

条目直接改 `knowledge/` 下的文件。云端写入要 `cursor_` 开头的密钥，`sk-` 密钥不能走那条写入。

## 关键位置

- `uv run memex` 监听 8787 的 `/mcp`
- 页面 `viewer/index.html`，数据 `viewer/data.json`，生成脚本 `viewer/build.py`
- 条目目录 `knowledge/{业务域}/{主题}/`
- 接入说明：[知识库初始化](/knowledge/meta/knowledge-base-init.md)

## 以后怎么用

看不到正文时，先确认端口是 8788、工作目录是 `viewer`，再重新生成 `data.json` 并强制刷新。新增条目后要重新生成数据；静态服务读的是磁盘上的 JSON，不用为了新文件重启。
