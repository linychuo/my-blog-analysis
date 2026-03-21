# 1.3 功能特性清单

> my-blog 完整功能列表和详细说明

---

## 目录

- [核心功能](#核心功能)
- [Markdown 支持](#markdown-支持)
- [模板系统](#模板系统)
- [样式与主题](#样式与主题)
- [生成特性](#生成特性)
- [开发体验](#开发体验)
- [功能对比表](#功能对比表)

---

## 核心功能

### ✅ 已完成功能

| 功能 | 状态 | 说明 |
|------|------|------|
| Markdown 解析 | ✅ | 完整 CommonMark + 扩展 |
| HTML 生成 | ✅ | 静态 HTML 文件输出 |
| 模板系统 | ✅ | Handlebars 风格 |
| 标签系统 | ✅ | 自动生成标签页面 |
| 日期 URL | ✅ | /YYYY/MM/DD/slug.html |
| 主题切换 | ✅ | 暗色/亮色自动切换 |
| 响应式设计 | ✅ | 移动端适配 |
| 代码高亮 | ✅ | highlight.js 集成 |
| 数学公式 | ✅ | KaTeX 渲染 |
| 关于页面 | ✅ | 自动生成 about.html |

### 🔄 计划中功能

| 功能 | 优先级 | 说明 |
|------|--------|------|
| RSS/Atom 订阅 | 高 | XML feed 生成 |
| 搜索功能 | 中 | 客户端全文搜索 |
| 阅读时间估算 | 低 | 文章阅读时间显示 |
| 相关文章 | 低 | 基于标签推荐 |
| sitemap.xml | 中 | SEO 优化 |
| 分页 | 中 | 大列表分页 |

---

## Markdown 支持

### 块级元素

| 元素 | 语法 | 渲染效果 |
|------|------|----------|
| **标题** | `# H1` ~ `###### H6` | `<h1>` ~ `<h6>` |
| **段落** | 空行分隔 | `<p>` |
| **代码块** | <code>```lang</code> | `<pre><code>` |
| **表格** | `\| col \|` | `<table>` |
| **引用** | `> text` | `<blockquote>` |
| **列表** | `- item` / `1. item` | `<ul>` / `<ol>` |
| **数学块** | `$$...$$` | `<pre class="math-block">` |
| **水平线** | `---` / `***` | `<hr>` |

### 内联元素

| 元素 | 语法 | 渲染效果 |
|------|------|----------|
| **粗体** | `**text**` | `<strong>` |
| **斜体** | `*text*` | `<em>` |
| **删除线** | `~~text~~` | `<del>` |
| **行内代码** | `` `code` `` | `<code>` |
| **链接** | `[text](url)` | `<a href>` |
| **图片** | `![alt](src)` | `<img>` |

### 扩展语法

| 扩展 | 语法 | 示例 |
|------|------|------|
| **换行** | 行尾加 `\` | `line1\` |
| **数学公式** | `$$...$$` | `$$E=mc^2$$` |
| **表格** | 标准 GFM 表格 | 见下方示例 |

### Markdown 示例

```markdown
---
title: "文章标题"
date_time: 2026-03-21 10:00:00
tags: zig blog
---

# 标题

这是**粗体**和*斜体*。

## 代码块

```zig
pub fn main() void {
    std.debug.print("Hello\n", .{});
}
```

## 表格

| 列 1 | 列 2 |
|------|------|
| A    | B    |

## 数学公式

$$\int_0^1 x^2 dx = \frac{1}{3}$$

## 换行

第一行\
第二行
```

---

## 模板系统

### 模板文件

| 模板 | 用途 | 路径 |
|------|------|------|
| **layout.hbs** | 主布局 | templates/layout.hbs |
| **index.hbs** | 首页 | templates/index.hbs |
| **post.hbs** | 文章页 | templates/post.hbs |
| **about.hbs** | 关于页 | templates/about.hbs |
| **tag.hbs** | 标签页 | templates/tag.hbs |

### 模板变量

#### layout.hbs 可用变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `{{page_title}}` | string | 页面标题前缀 |
| `{{year}}` | string | 当前年份 |
| `{{page}}` | block | 页面内容块 |
| `{{sidebar}}` | block | 侧边栏内容块 |

#### post.hbs 可用变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `{{title}}` | string | 文章标题 |
| `{{date_time}}` | string | 发布日期 |
| `{{content}}` | HTML | 文章 HTML 内容 |
| `{{tags}}` | HTML | 标签链接列表 |

#### tag.hbs 可用变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `{{tag_name}}` | string | 标签名称 |
| `{{post_count}}` | string | 文章数量 |
| `{{posts}}` | HTML | 文章列表 HTML |

### 模板语法示例

```handlebars
{! 定义内联块 !}
{{#*inline "page"}}
<div class="posts-grid">{{{posts}}}</div>
{{/inline}}

{! 调用父模板 !}
{{~> (parent)~}}
```

---

## 样式与主题

### 主题系统

| 特性 | 说明 |
|------|------|
| **自动检测** | 根据时间（6-18 点）自动切换 |
| **手动切换** | 点击导航栏按钮切换 |
| **持久化** | localStorage 保存偏好 |
| **代码主题** | highlight.js 主题同步切换 |

### CSS 变量

```css
:root {
    /* 颜色 */
    --bg-primary: #0d1117;
    --text-primary: #f0f6fc;
    --accent: #58a6ff;
    
    /* 字体大小 */
    --font-xs: 0.75rem;
    --font-base: 1rem;
    --font-4xl: 2.5rem;
}
```

### 响应式断点

| 断点 | 宽度 | 布局变化 |
|------|------|----------|
| **桌面** | > 768px | 完整布局 |
| **平板** | 768px | 调整边距 |
| **手机** | < 768px | 单列布局 |

### 组件样式

| 组件 | 特性 |
|------|------|
| 导航栏 | 固定顶部，毛玻璃效果 |
| 文章卡片 | 悬停动画，阴影效果 |
| 标签 | 圆角，悬停变色 |
| 代码块 | 语法高亮，圆角 |
| 表格 | 斑马纹，边框 |
| 引用块 | 左侧边框，缩进 |
| 回到顶部 | 滚动显示，平滑动画 |

---

## 生成特性

### 输出结构

```
zig-out/blog/
├── index.html              # 首页
├── about.html              # 关于页
├── style.css               # 样式表
├── imgs/                   # 图片资源
├── tags/                   # 标签页面
│   ├── zig.html
│   ├── blog.html
│   └── ...
└── YYYY/                   # 日期目录
    └── MM/
        └── DD/
            └── slug.html   # 文章页面
```

### URL 规则

| 页面类型 | URL 模式 | 示例 |
|----------|----------|------|
| 首页 | `/` | index.html |
| 关于 | `/about.html` | about.html |
| 文章 | `/YYYY/MM/DD/slug.html` | /2026/03/21/post.html |
| 标签 | `/tags/name.html` | /tags/zig.html |

### 生成统计

| 指标 | 数值 |
|------|------|
| 文章加载 | ~50ms/篇 |
| HTML 生成 | ~100ms/篇 |
| 标签页生成 | ~10ms/标签 |
| 静态文件复制 | ~50ms |

---

## 开发体验

### 命令行工具

```bash
# 基本用法
zig build run

# 自定义路径
zig build run -- --posts ./my-posts --output ./public

# 帮助信息
zig build run -- --help
```

### CLI 选项

| 选项 | 简写 | 默认值 | 说明 |
|------|------|--------|------|
| `--posts` | `-p` | posts | 文章源目录 |
| `--output` | `-o` | build | 输出目录 |
| `--help` | `-h` | - | 显示帮助 |

### 构建模式

| 模式 | 命令 | 用途 |
|------|------|------|
| Debug | `zig build run` | 开发调试 |
| Release | `zig build -Doptimize=ReleaseFast run` | 生产环境 |

### 错误提示

```
Generating blog...
  Posts directory: posts
  Output directory: build
  Copied: style.css
  Loaded: my-post.markdown
  Generated: index.html
  Generated: tags/zig.html
Blog generation complete!
```

---

## 功能对比表

### 完整功能矩阵

| 功能类别 | 功能 | 状态 | 优先级 |
|----------|------|------|--------|
| **内容** | Markdown 解析 | ✅ | 高 |
| **内容** | frontmatter 解析 | ✅ | 高 |
| **内容** | 标签系统 | ✅ | 高 |
| **内容** | 日期 URL | ✅ | 高 |
| **内容** | 关于页面 | ✅ | 中 |
| **模板** | 模板引擎 | ✅ | 高 |
| **模板** | 模板继承 | ✅ | 中 |
| **模板** | 部分模板 | ✅ | 低 |
| **样式** | 响应式设计 | ✅ | 高 |
| **样式** | 主题切换 | ✅ | 高 |
| **样式** | 代码高亮 | ✅ | 高 |
| **样式** | 数学公式 | ✅ | 中 |
| **样式** | 字体优化 | ✅ | 中 |
| **性能** | 快速生成 | ✅ | 高 |
| **性能** | 内存优化 | ✅ | 中 |
| **SEO** | 语义化 HTML | ✅ | 中 |
| **SEO** | sitemap.xml | ⚪ | 中 |
| **SEO** | RSS/Atom | ⚪ | 高 |
| **扩展** | 搜索功能 | ⚪ | 中 |
| **扩展** | 阅读时间 | ⚪ | 低 |
| **扩展** | 相关文章 | ⚪ | 低 |
| **扩展** | 分页 | ⚪ | 中 |

**图例**: ✅ 已完成 | ⚪ 计划中

---

## 小结

my-blog 已实现完整的静态博客生成器核心功能：

| 方面 | 完成度 |
|------|--------|
| Markdown 解析 | 100% |
| 模板系统 | 100% |
| 样式主题 | 100% |
| 标签系统 | 100% |
| SEO 优化 | 60% |
| 扩展功能 | 20% |

---

## 相关阅读

- [项目背景](01-1-project-background.md) - 项目起源
- [技术栈](01-2-tech-stack.md) - 技术选型
- [项目结构](01-4-project-structure.md) - 目录结构
- [编写博客](../part-13-extension-guides/13-1-writing-posts.md) - 文章格式指南
