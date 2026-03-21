# 1.4 项目结构说明

> 详细了解 my-blog 项目的目录结构和文件组织

---

## 目录

- [整体结构](#整体结构)
- [源代码目录](#源代码目录)
- [模板目录](#模板目录)
- [静态资源](#静态资源)
- [依赖库](#依赖库)
- [构建配置](#构建配置)
- [输出目录](#输出目录)
- [文件命名规范](#文件命名规范)

---

## 整体结构

```
my-blog/
├── src/                      # Zig 源代码
├── templates/                # Handlebars 模板
├── static/                   # 静态资源
├── posts/                    # Markdown 文章
├── libs/                     # 依赖库
├── zig-out/                  # 构建输出
├── build.zig                 # 构建配置
├── build.zig.zon             # 包清单
└── QWEN.md                   # 项目说明
```

---

## 源代码目录

### src/

```
src/
├── main.zig                  # 应用程序入口
└── Blogger.zig               # 博客生成核心逻辑
```

### main.zig

**职责**: 应用程序入口点，处理 CLI 参数

**代码结构**:

```zig
const std = @import("std");
const markdown = @import("zig-markdown");
const Blogger = @import("Blogger.zig").Blogger;

pub fn main() !void {
    // 1. 初始化内存分配器
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // 2. 解析命令行参数
    var args = try std.process.argsWithAllocator(allocator);
    defer args.deinit();

    // 3. 创建 Blogger 并生成
    var blogger = Blogger.new(allocator, posts_dir, dest_dir);
    try blogger.generate();
}
```

**关键函数**:

| 函数 | 职责 |
|------|------|
| `main()` | 程序入口 |
| `printHelp()` | 打印帮助信息 |

### Blogger.zig

**职责**: 博客生成核心逻辑

**结构**:

```zig
pub const Post = struct {
    // 文章数据结构
    title: []const u8,
    date_time: []const u8,
    tags: []const u8,
    content: []const u8,
    filename: []const u8,
    
    pub fn parse(...) !Post { ... }
    pub fn deinit(...) void { ... }
};

pub const Blogger = struct {
    // 博客生成器
    dest_dir: []const u8,
    posts_dir: []const u8,
    template_engine: TemplateEngine,
    
    pub fn generate() !void { ... }
    pub fn loadPosts() !void { ... }
    pub fn generateIndex() !void { ... }
    pub fn generatePostPage() !void { ... }
    pub fn generateTagPages() !void { ... }
    pub fn generateAbout() !void { ... }
};
```

**核心方法**:

| 方法 | 职责 |
|------|------|
| `new()` | 构造函数 |
| `generate()` | 主生成入口 |
| `loadPosts()` | 加载文章 |
| `generateIndex()` | 生成首页 |
| `generatePostPage()` | 生成文章页 |
| `generateTagPages()` | 生成标签页 |
| `generateAbout()` | 生成关于页 |
| `copyStaticFiles()` | 复制静态文件 |

---

## 模板目录

### templates/

```
templates/
├── layout.hbs                # 主布局模板
├── index.hbs                 # 首页模板
├── post.hbs                  # 文章页模板
├── about.hbs                 # 关于页模板
└── tag.hbs                   # 标签页模板
```

### 模板继承关系

```
┌─────────────────────────────────────────┐
│            layout.hbs (父模板)           │
│  ┌─────────────────────────────────┐    │
│  │  {{~> page}}     {{~> sidebar}} │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
           ▲                  ▲
           │                  │
    ┌──────┴──────┐    ┌──────┴──────┐
    │ index.hbs   │    │ post.hbs    │
    │ about.hbs   │    │ tag.hbs     │
    └─────────────┘    └─────────────┘
```

### layout.hbs

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>{{page_title}}YongchaoLi's Blog</title>
    <!-- 头部资源 -->
</head>
<body>
    <nav class="navbar">...</nav>
    <div class="main-container">
        <main class="content-area">{{~> page}}</main>
        <aside class="sidebar">{{~> sidebar}}</aside>
    </div>
    <footer class="footer">...</footer>
</body>
</html>
```

### index.hbs

```handlebars
{{#*inline "page"}}
<div class="posts-grid">{{{posts}}}</div>
{{/inline}}
{{#*inline "sidebar"}}{{/inline}}
{{~> (parent)~}}
```

---

## 静态资源

### static/

```
static/
├── style.css                 # 主样式表
└── imgs/                     # 图片资源
    ├── favicon.ico
    └── *.jpg, *.png
```

### style.css 结构

```
style.css
├── CSS 变量定义
├── 基础样式 (reset, body)
├── 导航栏样式
├── 主容器布局
├── 文章卡片样式
├── 代码块样式
├── 表格样式
├── 引用块样式
├── 标签样式
├── 响应式断点
└── 动画效果
```

---

## 依赖库

### libs/

```
libs/
├── zig-markdown/             # Markdown 解析器
│   ├── src/
│   │   ├── markdown.zig      # 主入口
│   │   ├── parser.zig        # 解析状态
│   │   ├── html_writer.zig   # HTML 写入
│   │   ├── inline.zig        # 内联处理
│   │   └── blocks/
│   │       └── table.zig     # 表格处理
│   ├── build.zig
│   └── README.md
│
└── zig-handlebars/           # 模板引擎
    ├── src/
    │   ├── root.zig          # 模块导出
    │   ├── engine.zig        # 引擎核心
    │   ├── context.zig       # 上下文
    │   ├── partials.zig      # 部分模板
    │   └── error.zig         # 错误定义
    ├── build.zig
    └── README.md
```

---

## 构建配置

### 根目录文件

| 文件 | 用途 |
|------|------|
| **build.zig** | Zig 构建脚本 |
| **build.zig.zon** | 包清单（依赖、版本） |
| **QWEN.md** | 项目说明文档 |

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // 1. 目标和优化配置
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // 2. 导入依赖
    const zig_markdown = b.dependency("zig_markdown", ...);
    const zig_handlebars = b.dependency("zig_handlebars", ...);

    // 3. 创建可执行文件
    const exe = b.addExecutable(.{
        .name = "my-blog",
        .root_module = mod,
    });

    // 4. 安装和运行步骤
    b.installArtifact(exe);
    
    const run_step = b.step("run", "Run the app");
    const run_cmd = b.addRunArtifact(exe);
    run_step.dependOn(&run_cmd.step);
}
```

### build.zig.zon

```zig
.{
    .name = .my_blog,
    .version = "0.1.0",
    .minimum_zig_version = "0.15.2",
    .dependencies = .{
        .zig_markdown = .{ .path = "libs/zig-markdown" },
        .zig_handlebars = .{ .path = "libs/zig-handlebars" },
    },
    .paths = .{
        "build.zig", "build.zig.zon", "src",
        "templates", "static", "libs/zig-markdown",
        "libs/zig-handlebars",
    },
}
```

---

## 输出目录

### zig-out/blog/

```
zig-out/blog/
├── index.html              # 首页
├── about.html              # 关于页
├── style.css               # 样式表（复制）
├── imgs/                   # 图片（复制）
│   └── *.jpg, *.png
├── tags/                   # 标签页面
│   ├── zig.html
│   ├── blog.html
│   └── ...
└── YYYY/                   # 日期目录
    └── MM/
        └── DD/
            └── slug.html   # 文章页面
```

### 生成示例

```
zig-out/blog/
├── index.html
├── about.html
├── style.css
├── tags/
│   ├── zig.html           # 标签：zig
│   ├── blog.html          # 标签：blog
│   └── rust.html          # 标签：rust
└── 2026/
    └── 03/
        └── 21/
            ├── my-first-post.html
            └── zig-intro.html
```

---

## 文章目录

### posts/

```
posts/
├── about.markdown          # 关于页面内容
├── my-first-post.markdown  # 文章 1
├── zig-intro.markdown      # 文章 2
└── ...
```

### 文章格式

```markdown
---
title: "文章标题"
date_time: 2026-03-21 10:00:00
tags: zig blog tutorial
---

# 正文内容

这是文章内容...
```

---

## 文件命名规范

### Zig 源代码

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 主文件 | `main.zig` | main.zig |
| 模块 | `PascalCase.zig` | Blogger.zig |
| 测试 | `*_test.zig` | parser_test.zig |

### 模板文件

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 布局 | `layout.hbs` | layout.hbs |
| 页面 | `page-name.hbs` | index.hbs, post.hbs |

### 文档文件

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 文档 | `kebab-case.md` | project-structure.md |
| 带序号 | `NNN-feature.md` | 01-1-project-background.md |

### Markdown 文章

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 文章 | `kebab-case.markdown` | my-first-post.markdown |
| 关于 | `about.markdown` | about.markdown |

---

## 目录大小参考

```
my-blog/
├── src/                    ~600 行
├── templates/              ~70 行
├── static/                 ~450 行 CSS
├── libs/zig-markdown/      ~800 行
├── libs/zig-handlebars/    ~300 行
└── posts/                  N 篇文章
```

---

## 小结

my-blog 的目录结构设计遵循以下原则：

| 原则 | 体现 |
|------|------|
| **清晰分离** | 源代码、模板、静态资源分离 |
| **模块化** | 依赖库独立目录 |
| **可扩展** | 易于添加新模板和功能 |
| **约定优于配置** | 固定目录名，减少配置 |

---

## 相关阅读

- [项目背景](01-1-project-background.md) - 项目起源
- [功能特性](01-3-features.md) - 完整功能
- [核心模块分析](../part-07-core-modules/07-1-main-entry.md) - 代码详解
- [构建系统](../part-06-build-system/06-1-build-zig-structure.md) - build.zig 详解
