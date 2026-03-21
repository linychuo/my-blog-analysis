# 1.2 技术栈选型

> 深入了解 my-blog 项目的技术选型和依赖关系

---

## 目录

- [核心技术栈](#核心技术栈)
- [依赖库详解](#依赖库详解)
- [外部 CDN 资源](#外部-cdn-资源)
- [技术选型原因](#技术选型原因)

---

## 核心技术栈

### 语言与工具

| 组件 | 版本 | 用途 |
|------|------|------|
| **Zig** | 0.15.2 | 主编程语言 |
| **Zig Build System** | 内置 | 构建配置 |
| **Git** | - | 版本控制 |

### 为什么选择 Zig 0.15.2？

```zig
// Zig 0.15.2 的关键特性
const std = @import("std");

// 1. 编译时执行
comptime {
    std.debug.print("编译时计算\n", .{});
}

// 2. 错误处理
fn riskyOperation() !u8 {
    return error.SomethingWentWrong;
}

// 3. 泛型编程
fn ArrayList(comptime T: type) type {
    return struct {
        items: []T,
        // ...
    };
}
```

---

## 依赖库详解

### 项目依赖结构

```
my-blog/
├── libs/
│   ├── zig-markdown/     # Markdown → HTML 转换器
│   └── zig-handlebars/   # 模板引擎
├── build.zig.zon         # 依赖配置
└── build.zig             # 构建脚本
```

### zig-markdown

**位置**: `libs/zig-markdown/`

**功能**: Markdown 到 HTML 转换器

**支持的特性**:

| 特性 | 语法 | 示例 |
|------|------|------|
| 标题 | `#` ~ `######` | `# H1` |
| 粗体 | `**text**` | `**bold**` |
| 斜体 | `*text*` | `*italic*` |
| 删除线 | `~~text~~` | `~~deleted~~` |
| 行内代码 | `` `code` `` | `` `SELECT` `` |
| 代码块 | `\`\`\`lang` | `\`\`\`zig` |
| 表格 | `\| col \|` | `\| A \| B \|` |
| 引用 | `> text` | `> quote` |
| 链接 | `[text](url)` | `[link](url)` |
| 图片 | `![alt](src)` | `![img](a.png)` |
| 数学公式 | `$$...$$` | `$$E=mc^2$$` |
| 换行 | `\` (行尾) | `line1\` |

**核心 API**:

```zig
const markdown = @import("zig-markdown");

// 转换 Markdown 为 HTML
const html = try markdown.toHtml(allocator, markdown_text);
defer allocator.free(html);
```

**架构**:

```
┌─────────────────────────────────────────┐
│              markdown.zig               │
│            (toHtml 入口函数)             │
└───────────────────┬─────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
┌──────────────┐ ┌──────────┐ ┌───────────┐
│  parser.zig  │ │ inline.zig│ │ blocks/   │
│  (状态管理)   │ │(内联处理) │ │ table.zig │
└──────────────┘ └──────────┘ └───────────┘
```

### zig-handlebars

**位置**: `libs/zig-handlebars/`

**功能**: Handlebars 风格的模板引擎

**模板语法**:

```handlebars
{! 变量（HTML 转义）!}
{{ title }}

{! 未转义的 HTML !}
{{{ content }}}

{! 定义内联块 !}
{{#*inline "page"}}
  <div>{{{content}}}</div>
{{/inline}}

{! 模板继承 !}
{{~> (parent)~}}
```

**核心 API**:

```zig
const TemplateEngine = @import("zig-handlebars").TemplateEngine;
const Context = @import("zig-handlebars").Context;

// 初始化引擎
const engine = TemplateEngine.init(allocator, "templates");
defer engine.deinit();

// 创建上下文
var ctx = Context.init(allocator);
defer ctx.deinit();
try ctx.set("title", "My Post");

// 渲染模板
const html = try engine.render("post.hbs", &ctx);
```

**模块结构**:

```
zig-handlebars/
├── src/
│   ├── root.zig          # 模块导出
│   ├── engine.zig        # 模板引擎核心
│   ├── context.zig       # 上下文管理
│   ├── partials.zig      # 部分模板
│   ├── error.zig         # 错误定义
│   └── html_escape.zig   # HTML 转义
└── templates/
    └── test_partial.hbs  # 测试模板
```

---

## 外部 CDN 资源

### 前端资源（通过 CDN 加载）

| 资源 | 版本 | 用途 | URL |
|------|------|------|-----|
| **Inter Font** | - | 正文字体 | fonts.googleapis.com |
| **JetBrains Mono** | - | 代码字体 | fonts.googleapis.com |
| **highlight.js** | 11.9.0 | 代码高亮 | cdnjs.cloudflare.com |
| **KaTeX** | 0.12.0 | 数学公式 | cdn.jsdelivr.net |

### layout.hbs 中的引用

```html
<!-- 字体 -->
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500;600&display=swap" rel="stylesheet">

<!-- 代码高亮 -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css">
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js"></script>

<!-- 数学公式 -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.css">
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.js"></script>
```

### 为什么使用 CDN？

| 优势 | 说明 |
|------|------|
| **缓存共享** | 用户可能已缓存这些资源 |
| **加载速度** | CDN 节点就近访问 |
| **减少体积** | 无需打包到项目 |
| **自动更新** | 可使用最新版 |

---

## 技术选型原因

### 为什么不用现成的 Markdown 库？

**选项考虑过**:
- [x] cmark (C)
- [x] pulldown-cmark (Rust)
- [x] commonmark.js (JavaScript)

**选择自定义实现的原因**:

1. **学习目的** - 深入理解 Markdown 解析原理
2. **零依赖** - 无需绑定 C 库
3. **定制化** - 可添加特殊语法（如数学公式）
4. **代码量小** - 约 1000 行，易于维护

### 为什么不用现有模板引擎？

**考虑因素**:

| 引擎 | 问题 |
|------|------|
| **Terminator** | 过于复杂 |
| **Ziggy** | 功能过多 |
| **自定义** | 刚好满足需求 |

**自定义模板引擎优势**:

```
1. 代码量少（~300 行）
2. 学习 Handlebars 设计思想
3. 完全控制渲染逻辑
4. 无多余功能
```

### 为什么选择这些字体？

| 字体 | 用途 | 原因 |
|------|------|------|
| **Inter** | 正文 | 清晰、现代、开源 |
| **JetBrains Mono** | 代码 | 专为编程设计、连字支持 |

---

## 依赖关系图

```
                    ┌─────────────────┐
                    │    main.zig     │
                    │  (应用程序入口)  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
     ┌────────────────┐ ┌─────────┐ ┌──────────────┐
     │  Blogger.zig   │ │  std    │ │  第三方依赖   │
     │  (核心逻辑)    │ │  标准库  │ │              │
     └───────┬────────┘ └─────────┘ └──────┬───────┘
             │                             │
             │              ┌──────────────┼──────────────┐
             │              │              │              │
             ▼              ▼              ▼              ▼
    ┌────────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ zig-markdown   │ │zig-      │ │ highlight│ │  KaTeX   │
    │ (Markdown 解析) │ │handlebars│ │  .js     │ │ (数学公式)│
    │                │ │(模板引擎) │ │(代码高亮) │ │          │
    └────────────────┘ └──────────┘ └──────────┘ └──────────┘
```

---

## build.zig.zon 配置

```zig
.{
    .name = .my_blog,
    .version = "0.1.0",
    .minimum_zig_version = "0.15.2",
    .dependencies = .{
        .zig_markdown = .{
            .path = "libs/zig-markdown",
        },
        .zig_handlebars = .{
            .path = "libs/zig-handlebars",
        },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
        "templates",
        "static",
        "libs/zig-markdown",
        "libs/zig-handlebars",
    },
}
```

---

## 小结

my-blog 的技术栈设计原则：

| 原则 | 体现 |
|------|------|
| **简洁** | 最小依赖，自定义核心组件 |
| **性能** | Zig 编译型语言，无运行时 |
| **可控** | 关键组件自研，完全掌握 |
| **现代** | CDN 资源，最新前端技术 |

---

## 相关阅读

- [项目背景](01-1-project-background.md) - 项目起源
- [功能特性](01-3-features.md) - 完整功能列表
- [构建系统](../part-06-build-system/06-1-build-zig-structure.md) - build.zig 详解
