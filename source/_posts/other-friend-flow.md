---
title: FriendFlow 友链互助插件
date: 2026-04-27T16:00:00.000Z
categories: 其他
tags:
toc: true
excerpt: "为 Kratos : Rebirth 主题添加 FriendFlow 友链互助系统的 Hexo 插件"
author: Rimuru
cover: cover.png
---

## 概述

**FriendFlow** 是一个基于 Cloudflare Workers 的友链互助系统。它自动抓取友链博客的最新文章，以 REST API 的形式提供数据，让友链页面从静态链接列表升级为动态内容聚合站。

**FriendFlow Hexo 插件**是一个标签插件（Tag Plugin），在友链页面中嵌入友链联盟卡片，自动展示友链博客的最新文章。只需配置 API 地址，在页面中写入 `{% friendflow %}` 即可完成接入。

> **项目地址**：[https://github.com/AinzRimuru/friend-flow](https://github.com/AinzRimuru/friend-flow)
> **在线体验**：[https://friend.rimuru.work](https://friend.rimuru.work)

## 功能特色

| 特性 | 说明 |
|------|------|
| 一行代码接入 | 页面中写入 `{% friendflow %}` 即可，无需手动编写 JS |
| 自动抓取文章 | 从友链博客的 Atom/RSS Feed 获取最近文章 |
| 智能缓存 | 5 分钟 TTL 缓存，10 秒抓取超时 |
| 容错降级 | 连续失败 3 次冷却 1 小时，30 次停止刷新，凌晨自动重试 |
| 图片代理 | Worker 重定向透传图片资源，跨域零障碍 |
| 主题适配 | 自动适配 Kratos-Rebirth 亮色/暗色模式，其他主题提供合理回退 |
| 随机排列 | 每次加载随机打乱友链顺序 |
| 独立命名空间 | 全部 CSS 使用 `ff-` 前缀，不与主题样式冲突 |

## 效果展示

<!-- 截图：友链卡片整体效果 -->

![FriendFlow 友链卡片效果展示](effect-overview.png)

插件会在友链页面渲染一组卡片，每个卡片包含：

- 友链博客的头像、名称和描述
- 该博客最新发布的文章列表（标题 + 日期）
- 每次加载时随机排列顺序

<!-- 截图：单个卡片的详细结构 -->

![单个友链卡片结构](effect-card-detail.png)

<!-- 截图：暗色模式下的效果（如适用） -->

![暗色模式效果](effect-dark-mode.png)

## 快速开始

### 前置条件

- Hexo 博客已正常运行
- FriendFlow 实例已部署（参考 [FriendFlow 项目文档](https://github.com/AinzRimuru/friend-flow)）

### 三步接入

**1. 添加插件文件** → **2. 配置 `_config.yml`** → **3. 页面中使用标签**

详细步骤见下文。

## 安装步骤

### 文件结构

在博客中添加如下文件结构：

```
scripts/
└── friend-flow-tag.js            # 插件入口，注册 Hexo 标签
source/_data/
└── friend-flow/
    ├── style.css                  # 卡片样式
    └── template.js                # 数据拉取与渲染逻辑
```

- `scripts/` 是 Hexo 的约定目录，放在里面的 JS 文件会在 `hexo generate` / `hexo server` 时自动加载
- 样式和模板放在 `source/_data/friend-flow/` 下，避免被 Hexo 当作脚本执行

### style.css — 卡片样式

样式使用 CSS 自定义属性实现双层变量映射，自动适配 Kratos-Rebirth 主题的亮色/暗色模式，同时为非该主题的博客提供合理的回退值：

```css
.ff-container {
  margin: 16px 0;
  overflow: hidden;
  width: 100%;
  box-sizing: border-box;

  /* 变量映射：优先使用 Kratos-Rebirth 主题变量，回退到内置默认值 */
  --ff-bg: var(--kr-theme-card-bg, #fff);
  --ff-text: var(--kr-theme-text, #000);
  --ff-text-alt: var(--kr-theme-text-alt, #666);
  --ff-link-hover: var(--kr-theme-link-hover, #6ec3f5);
  --ff-border: var(--kr-theme-border, #eaecef);
  --ff-muted: var(--kr-theme-text-alt, #999);
}

.ff-container .ff-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 12px;
}

/* 卡片 */
.ff-container .ff-card {
  background: var(--ff-bg);
  border-radius: 0;
  overflow: hidden;
  min-width: 0;
  max-width: 100%;
  box-sizing: border-box;
  transition: all 0.3s ease-in-out;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.ff-container .ff-card:hover {
  box-shadow: 0 8px 15px rgba(146, 146, 146, 0.39);
}

/* 卡片头部 - 头像 + 名称 */
.ff-container .ff-card-header {
  display: flex;
  align-items: center;
  padding: 10px 12px;
  gap: 10px;
}

.ff-container .ff-card-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  object-fit: cover;
  flex-shrink: 0;
}

.ff-container .ff-card-info {
  flex: 1;
  min-width: 0;
  overflow: hidden;
}

.ff-container .ff-card-name {
  display: block;
  font-size: 14px;
  font-weight: 600;
  color: var(--ff-text);
  text-decoration: none;
}

.ff-container .ff-card-name:hover {
  color: var(--ff-link-hover);
}

.ff-container .ff-card-desc {
  font-size: 12px;
  color: var(--ff-text-alt);
  line-height: 1.3;
  margin-top: 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 文章列表 */
.ff-container .ff-card-articles {
  border-top: 1px solid var(--ff-border);
  padding: 6px 12px 8px;
}

.ff-container .ff-article-item {
  display: flex;
  align-items: baseline;
  padding: 2px 0;
  text-decoration: none;
  color: var(--ff-text-alt);
  font-size: 13px;
  line-height: 1.5;
}

.ff-container .ff-article-item:hover {
  color: var(--ff-link-hover);
}

.ff-container .ff-article-title {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  margin-right: 8px;
}

.ff-container .ff-article-time {
  font-size: 11px;
  color: var(--ff-muted);
  flex-shrink: 0;
  white-space: nowrap;
}

/* 加载状态 */
.ff-container .ff-spinner {
  display: inline-block;
  width: 24px;
  height: 24px;
  border: 3px solid var(--ff-border);
  border-top-color: var(--ff-text-alt);
  border-radius: 50%;
  animation: ff-spin 0.8s linear infinite;
}

@keyframes ff-spin {
  to { transform: rotate(360deg); }
}

.ff-container .ff-loading {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 120px;
}

.ff-container .ff-error {
  text-align: center;
  padding: 40px 0;
  color: var(--ff-muted);
}
```

### template.js — 数据拉取与渲染

一段自执行函数（IIFE），接收运行时参数（容器 ID、API 地址、排除 URL、文章上限），负责 `fetch` → `render` 的完整流程：

```js
(function (id, api, self, max) {
  var c = document.getElementById(id);
  if (!c) return;

  function esc(s) {
    if (!s) return "";
    var d = document.createElement("div");
    d.textContent = s;
    return d.innerHTML;
  }

  function resIcon(i) {
    if (!i) return "";
    if (i.indexOf("http") === 0) return i;
    return api + i;
  }

  function fmtDate(s) {
    try {
      return new Date(s).toLocaleDateString("zh-CN", {
        year: "numeric",
        month: "2-digit",
        day: "2-digit",
      });
    } catch (e) {
      return s;
    }
  }

  function render(data) {
    var list = data.slice().sort(function () {
      return Math.random() - 0.5;
    });

    var h = '<div class="ff-grid">';

    for (var i = 0; i < list.length; i++) {
      var f = list[i];
      var ic = resIcon(f.icon);
      var arts = f.recentArticles || [];

      h += '<div class="ff-card"><div class="ff-card-header">';
      h +=
        '<a target="_blank" href="' +
        esc(f.url) +
        '" rel="noopener">' +
        '<img class="ff-card-avatar" src="' +
        esc(ic) +
        '" alt="' +
        esc(f.name) +
        '" loading="lazy"/></a>';
      h +=
        '<div class="ff-card-info">' +
        '<a class="ff-card-name" target="_blank" href="' +
        esc(f.url) +
        '" rel="noopener">' +
        esc(f.name) +
        "</a>";
      h +=
        '<div class="ff-card-desc">' + esc(f.description) + "</div></div></div>";

      if (arts.length > 0) {
        h += '<div class="ff-card-articles">';
        var n = Math.min(arts.length, max);
        for (var j = 0; j < n; j++) {
          var a = arts[j];
          var t = a.publishTime ? fmtDate(a.publishTime) : "";
          h +=
            '<a class="ff-article-item" target="_blank" href="' +
            esc(a.url) +
            '" rel="noopener">' +
            '<span class="ff-article-title">' +
            esc(a.title) +
            "</span>";
          if (t) h += '<span class="ff-article-time">' + t + "</span>";
          h += "</a>";
        }
        h += "</div>";
      }

      h += "</div>";
    }

    h += "</div>";
    c.innerHTML = h;
  }

  var url = api + "/api/friend-links";
  if (self) url += "?exclude=" + encodeURIComponent(self);

  fetch(url)
    .then(function (r) {
      if (!r.ok) throw new Error("HTTP " + r.status);
      return r.json();
    })
    .then(render)
    .catch(function (e) {
      console.error("Friend Flow:", e);
      c.innerHTML =
        '<div class="ff-error">' + (e.message || "Fetch failed") + "</div>";
    });
});
```

### friend-flow-tag.js — 插件入口

插件入口非常精简——读取 CSS/JS 文件，注入运行时参数，输出内联 HTML：

```js
'use strict';

var fs = require('fs');
var path = require('path');

var DIR = path.join(hexo.base_dir, 'source', '_data', 'friend-flow');
var CSS = fs.readFileSync(path.join(DIR, 'style.css'), 'utf8').trim();
var JS_TEMPLATE = fs.readFileSync(path.join(DIR, 'template.js'), 'utf8').trim();

hexo.extend.tag.register('friendflow', function () {
  var config = hexo.config.friend_flow || {};
  var apiBase = (config.api_base || '').replace(/\/+$/, '');
  var selfUrl = (config.self_url || '').replace(/\/+$/, '');
  var maxArticles = config.articles_per_card || 5;

  if (!apiBase) {
    return '<div style="text-align:center;color:#999;padding:40px 0">' +
      'Friend Flow: please set friend_flow.api_base in _config.yml</div>';
  }

  var id = 'ff-' + Math.random().toString(36).substr(2, 8);

  // 将模板 IIFE 转为立即执行，注入运行时参数
  var js = JS_TEMPLATE
    .replace(/^\(function\s*/, '(function ')
    .replace(/\}\);$/, '})("' + id + '","' + apiBase + '","' + selfUrl + '",' + maxArticles + ');');

  return '<style>' + CSS + '</style>' +
    '<div id="' + id + '" class="ff-container">' +
    '<div class="ff-loading"><span class="ff-spinner"></span></div>' +
    '</div>' +
    '<script>' + js + '</script>';
});
```

## 配置说明

在博客根目录的 `_config.yml` 中添加以下配置：

```yaml
# Friend Flow 友链互助系统
# Docs: https://github.com/AinzRimuru/friend-flow
friend_flow:
  api_base: https://friend.rimuru.work   # Friend Flow API 地址（必填）
  self_url: https://blog.example.com      # 你的博客地址（选填）
  articles_per_card: 5                    # 每张卡片最多显示的文章数（选填，默认 5）
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `api_base` | 是 | — | FriendFlow 实例地址 |
| `self_url` | 否 | — | 你自己的博客地址，API 调用时通过 `exclude` 参数排除，避免展示自己的文章 |
| `articles_per_card` | 否 | `5` | 每个友链卡片最多展示的文章数量 |

## 使用方法

在友链页面的 Markdown 中插入标签：

```markdown
---
title: 好伙伴们
---

## 友链联盟

{% friendflow %}
```

运行 `hexo generate` 或 `hexo server`，访问该页面即可看到效果。

<!-- 截图：在编辑器中写入标签的示意 -->

![在页面中使用 friendflow 标签](usage-tag-insert.png)

## API 参考

FriendFlow 提供的 REST API 端点如下。

### 获取全部友链

```http
GET /api/friend-links
```

返回所有友链及其最近 10 篇文章。

### 排除指定站点

```http
GET /api/friend-links?exclude=https://blog.example.com
```

支持多次传参：

```http
GET /api/friend-links?exclude=https://a.com&exclude=https://b.com
```

### 随机选取友链

```http
GET /api/friend-links?limit=5&exclude=https://blog.example.com
```

### 返回数据格式

```json
[
  {
    "name": "博客名称",
    "url": "https://example.com",
    "description": "博客描述",
    "icon": "/images/icons/blog.png",
    "lastFetchTime": "2026-04-24T12:00:00Z",
    "fetchStatus": "ok",
    "recentArticles": [
      {
        "title": "文章标题",
        "url": "https://example.com/post-1",
        "publishTime": "2026-04-20"
      }
    ]
  }
]
```

### 图片代理

```http
GET /images/icons/blog.png
```

返回 302 重定向到图片的实际地址，无需额外配置。

## 架构说明

### 工作原理

```
┌─────────────┐     抓取 Atom/RSS     ┌──────────────────┐
│  友链博客 A  │ ◄────────────────── │                  │
│  友链博客 B  │ ◄────────────────── │  Friend Flow     │
│  友链博客 C  │ ◄────────────────── │  (CF Workers+D1) │
└─────────────┘                      │                  │
                                     └────────┬─────────┘
                                              │
                              ┌────────────────┼────────────────┐
                              ▼                ▼                ▼
                        友链展示页面      REST API          图片代理
                     friend.rimuru.work  /api/...         /images/...
```

后端基于 **Cloudflare Workers + Hono**，数据存储在 **Cloudflare D1**。配置文件和图标资源托管在 GitHub Pages 上，Worker 启动时自动拉取。

每当有请求到达时，Worker 会检查缓存是否过期（TTL 5 分钟），过期则并发刷新最多 10 个站点的 Feed 数据。对于持续抓取失败的站点，系统逐步降级：3 次失败冷却 1 小时，30 次失败停止请求时刷新，等待每天凌晨的定时任务重试。

### 构建时读取，运行时拉取

```
hexo generate / hexo server
  │
  ├─ 加载 scripts/friend-flow-tag.js
  │    └─ fs.readFileSync 读取 source/_data/friend-flow/ 下的 style.css + template.js
  │    └─ 注册 {% friendflow %} 标签
  │
  ├─ 渲染友链页面
  │    └─ {% friendflow %} → 输出 HTML 容器 + 内联 CSS + 内联 JS
  │
  └─ 浏览器加载页面
       └─ JS 执行 fetch(api_base/api/friend-links?exclude=self_url)
            └─ 渲染友链卡片（随机排列）
```

- CSS/JS 文件在构建时通过 `fs.readFileSync` 读入并内联到 HTML 中，不产生额外的 HTTP 请求
- 友链数据在用户访问页面时实时拉取，保证内容始终最新

### 技术栈

| 组件 | 技术 |
|------|------|
| 后端 | Cloudflare Workers + Hono |
| 数据库 | Cloudflare D1 |
| 前端 | React + Vite + TailwindCSS |
| Hexo 插件 | 原生 JS（无构建依赖） |
| 配置托管 | GitHub Pages（YAML + 图标） |

## 主题适配

### 适配机制

样式通过 CSS 自定义属性的双层映射实现主题适配：

```
--kr-theme-card-bg (主题变量，亮色 #ffffffcc / 暗色 #282c34dd)
       ↓
--ff-bg (插件变量) → 回退值 #fff
       ↓
.ff-card { background: var(--ff-bg, #fff) }
```

### 设计原则

- **独立性** — 不引入主题 SCSS 文件，不依赖主题 class，全部使用 `ff-` 命名空间
- **兼容性** — Kratos-Rebirth 主题下自动跟随亮色/暗色模式切换；其他主题下使用回退默认值
- **可覆盖** — 其他主题的开发者可以通过覆盖 `--ff-*` 变量来自定义配色

### 自定义配色

如果你的主题不使用 Kratos-Rebirth 变量，可以在主题的 CSS 中覆盖插件变量：

```css
:root {
  --ff-bg: #1a1a2e;
  --ff-text: #e0e0e0;
  --ff-text-alt: #a0a0a0;
  --ff-link-hover: #7f5af0;
  --ff-border: #2a2a4a;
  --ff-muted: #666;
}
```

### 与主题自带友链共存

Kratos-Rebirth 主题自带 `{% linklist %}` 标签，通过本地 YAML 文件维护友链。两者可以同时使用：

```markdown
## 常驻友链

{% linklist friends random %}

---

## 友链联盟

{% friendflow %}
```

| | `{% linklist %}` | `{% friendflow %}` |
|---|---|---|
| 数据来源 | 本地 `source/_data/linklist.yml` | FriendFlow API |
| 展示内容 | 名称 + 描述 | 名称 + 描述 + 最新文章 |
| 维护方式 | 手动编辑 YAML | FriendFlow 自动抓取 |
| 适用场景 | 固定的核心友链 | 友链联盟，内容动态更新 |

## 常见问题

### 页面显示"please set friend_flow.api_base in _config.yml"

检查 `_config.yml` 中是否正确添加了 `friend_flow.api_base` 配置项，确保缩进格式正确。

### 友链卡片不显示

1. 打开浏览器开发者工具（F12），检查 Console 是否有报错
2. 检查 Network 面板中 `/api/friend-links` 请求是否正常返回
3. 确认 FriendFlow 实例正常运行

### 样式与主题冲突

插件使用 `ff-` 命名空间，通常不会与主题样式冲突。如遇到异常，可通过覆盖 `--ff-*` CSS 变量调整。

### 如何加入友链联盟

如果你的博客也想加入友链联盟（让其他人的 FriendFlow 展示你的文章），需要在 FriendFlow 的配置仓库中添加你的博客信息。具体方式参考 [FriendFlow 项目文档](https://github.com/AinzRimuru/friend-flow)。

---

**项目地址**：[https://github.com/AinzRimuru/friend-flow](https://github.com/AinzRimuru/friend-flow)

**在线体验**：[https://friend.rimuru.work](https://friend.rimuru.work)

**效果预览**：[https://blog.rimuru.work/friends/](https://blog.rimuru.work/friends/)
