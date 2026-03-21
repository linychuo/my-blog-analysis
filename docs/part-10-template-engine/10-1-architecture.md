# 10-1 模板引擎架构

> zig-handlebars 模板引擎设计与项目实例分析

---

## 1. 模板引擎概览

### 1.1 什么是 zig-handlebars

`zig-handlebars` 是 my-blog 项目使用的模板引擎，它是 Handlebars 模板语言的 Zig 实现。

**核心特性**：

| 特性 | 说明 |
|------|------|
| 变量插值 | `{{ variable }}` |
| 未转义 HTML | `{{{ html_content }}}` |
| 内联块定义 | `{{#*inline "name"}}...{{/inline}}` |
| 模板继承 | `{{~> (parent)~}}` |
| 部分模板 | `{{> partial_name }}` |

### 1.2 项目中的模板引擎

```
my-blog/
├── templates/
│   ├── layout.hbs      # 主布局模板（父模板）
│   ├── index.hbs       # 首页模板
│   ├── post.hbs        # 文章页模板
│   ├── about.hbs       # 关于页模板
│   └── tag.hbs         # 标签页模板
└── src/
    └── Blogger.zig     # 使用 TemplateEngine
```

---

## 2. TemplateEngine 架构

### 2.1 初始化

```zig
// Blogger.zig
pub const Blogger = struct {
    template_engine: TemplateEngine,
    
    pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger {
        const engine = TemplateEngine.init(allocator, "templates");
        return .{
            .allocator = allocator,
            .posts_dir = posts_dir,
            .dest_dir = dest_dir,
            .template_engine = engine,
        };
    }
};
```

**初始化参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `allocator` | `Allocator` | 内存分配器 |
| `"templates"` | `[]const u8` | 模板目录路径 |

### 2.2 渲染流程

```zig
// 渲染模板的基本流程
fn renderTemplate(self: *Blogger) !void {
    // [1] 初始化上下文
    var ctx = Context.init(self.allocator);
    defer ctx.deinit();
    
    // [2] 设置变量
    try ctx.set("title", "My Blog");
    try ctx.set("content", html_content);
    
    // [3] 渲染模板
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);
    
    // [4] 设置页面内容并渲染布局
    self.template_engine.setPageContent(page_html);
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);
}
```

**渲染流程图**：

```
Context.init()
    │
    ▼
设置变量 (ctx.set)
    │
    ▼
render("post.hbs", &ctx)
    │
    ▼
解析模板语法
    │
    ├─ 替换 {{variable}}
    ├─ 渲染 {{{html}}}
    └─ 处理内联块
    │
    ▼
返回渲染结果
    │
    ▼
setPageContent() 设置内容
    │
    ▼
render("layout.hbs", &ctx)
    │
    ▼
继承布局模板
    │
    ▼
返回最终 HTML
```

---

## 3. 模板文件结构

### 3.1 模板层次

```
layout.hbs (父模板/主布局)
    │
    ├─ index.hbs (子模板)
    ├─ post.hbs (子模板)
    ├─ about.hbs (子模板)
    └─ tag.hbs (子模板)
```

### 3.2 layout.hbs - 主布局

```handlebars
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{title}}</title>
    <link rel="stylesheet" href="/style.css">
</head>
<body>
    <header>
        <h1><a href="/">{{siteTitle}}</a></h1>
        <nav>
            <a href="/">Home</a>
            <a href="/about.html">About</a>
        </nav>
    </header>

    <main>
        {{~> page}}
    </main>

    <aside>
        {{~> sidebar}}
    </aside>

    <footer>
        <p>Generated with <a href="https://ziglang.org">Zig</a> · © {{year}}</p>
    </footer>
</body>
</html>
```

**布局元素**：

| 元素 | 说明 |
|------|------|
| `{{title}}` | 页面标题 |
| `{{siteTitle}}` | 站点标题 |
| `{{~> page}}` | 主内容块（子模板定义） |
| `{{~> sidebar}}` | 侧边栏块（子模板定义） |
| `{{year}}` | 版权年份 |

### 3.3 子模板模式

所有子模板都使用相同的模式：

```handlebars
{{#*inline "page"}}
<!-- 主内容 -->
{{/inline}}

{{#*inline "sidebar"}}
<!-- 侧边栏内容（可选） -->
{{/inline}}

{{~> (parent)~}}
```

**语法解析**：

| 语法 | 说明 |
|------|------|
| `{{#*inline "name"}}` | 定义名为 "name" 的内联块 |
| `{{/inline}}` | 结束内联块定义 |
| `{{~> (parent)~}}` | 继承父模板（layout.hbs） |
| `~` | 修剪空白（可选） |

---

## 4. 各模板功能分析

### 4.1 index.hbs - 首页

```handlebars
{{#*inline "page"}}
{{{posts}}}
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

**特点**：

| 特点 | 说明 |
|------|------|
| `{{{posts}}}` | 渲染所有文章的 HTML 列表 |
| 空 sidebar | 首页不需要侧边栏 |
| 三重括号 | 渲染未转义的 HTML |

### 4.2 post.hbs - 文章页

```handlebars
{{TITLE}}

{{date_time}}
{{{tags}}}
{{{content}}}
```

**特点**：

| 变量 | 类型 | 说明 |
|------|------|------|
| `{{TITLE}}` | 转义 | 文章标题 |
| `{{date_time}}` | 转义 | 发布日期 |
| `{{{tags}}}` | 未转义 | 标签链接 HTML |
| `{{{content}}}` | 未转义 | Markdown 转换的 HTML |

**注意**：post.hbs 是最简单的模板，直接输出变量，依赖 layout.hbs 提供结构。

### 4.3 about.hbs - 关于页

```handlebars
{{#*inline "page"}}
{{{content}}}
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

**特点**：

| 特点 | 说明 |
|------|------|
| `{{{content}}}` | 渲染 about.markdown 的 HTML |
| 无 frontmatter | about.markdown 不需要 frontmatter |
| 空 sidebar | 关于页不需要侧边栏 |

### 4.4 tag.hbs - 标签页

```handlebars
{{#*inline "page"}}

POSTS TAGGED WITH "{{TAG_NAME}}"

{{post_count}} post(s) found

{{{posts}}}

{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

**特点**：

| 变量 | 说明 |
|------|------|
| `{{TAG_NAME}}` | 当前标签名称 |
| `{{post_count}}` | 该标签下的文章数量 |
| `{{{posts}}}` | 该标签下所有文章的 HTML 列表 |

---

## 5. 模板渲染实例

### 5.1 generatePostPage() 完整流程

```zig
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] Markdown → HTML
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);

    // [2] 初始化上下文
    var ctx = Context.init(self.allocator);
    defer ctx.deinit();
    try ctx.set("title", post.title);
    try ctx.set("date_time", post.date_time);
    try ctx.set("content", html_content);

    // [3] 构建标签 HTML
    var tags_html = std.ArrayList(u8){};
    defer tags_html.deinit(self.allocator);
    var tag_iter = std.mem.tokenizeScalar(u8, post.tags, ' ');
    while (tag_iter.next()) |tag| {
        try tags_html.appendSlice(self.allocator, 
            "<a href=\"/tags/" ++ tag ++ ".html\" class=\"tag-link\">" ++ tag ++ "</a>");
    }
    try ctx.set("tags", tags_html.items);

    // [4] 渲染 post.hbs
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);

    // [5] 设置页面标题
    var full_title = std.ArrayList(u8){};
    defer full_title.deinit(self.allocator);
    try full_title.appendSlice(self.allocator, post.title);
    try full_title.appendSlice(self.allocator, " - ");
    try ctx.set("page_title", full_title.items);
    try ctx.set("year", "2026");

    // [6] 渲染 layout.hbs
    self.template_engine.setPageContent(page_html);
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);

    // [7] 写入文件
    try self.writeHtmlFileWithDate(post.filename, post.date_time, html);
}
```

### 5.2 上下文变量传递

```
Context 变量设置:
    │
    ├─ title = "My First Post"
    ├─ date_time = "2026-03-21"
    ├─ content = "<p>HTML content...</p>"
    ├─ tags = "<a href=\"/tags/zig\">zig</a>"
    ├─ page_title = "My First Post - "
    └─ year = "2026"
    │
    ▼
post.hbs 渲染:
    │
    ├─ {{TITLE}} → "My First Post"
    ├─ {{date_time}} → "2026-03-21"
    ├─ {{{tags}}} → "<a>zig</a>"
    └─ {{{content}}} → "<p>HTML content...</p>"
    │
    ▼
layout.hbs 渲染:
    │
    ├─ {{title}} → "My First Post - "
    └─ {{year}} → "2026"
    │
    ▼
最终 HTML 输出
```

---

## 6. 模板引擎 API

### 6.1 TemplateEngine 方法

| 方法 | 签名 | 说明 |
|------|------|------|
| `init` | `fn init(allocator, dir) TemplateEngine` | 初始化引擎 |
| `render` | `fn render(name, ctx) ![]const u8` | 渲染模板 |
| `setPageContent` | `fn setPageContent(content) void` | 设置页面内容 |

### 6.2 Context 方法

| 方法 | 签名 | 说明 |
|------|------|------|
| `init` | `fn init(allocator) Context` | 初始化上下文 |
| `deinit` | `fn deinit() void` | 清理资源 |
| `set` | `fn set(key, value) !void` | 设置变量 |

---

## 7. 模板变量来源

### 7.1 变量生成位置

| 变量 | 生成位置 | 说明 |
|------|----------|------|
| `title` | `generatePostPage()` | Post.title |
| `date_time` | `generatePostPage()` | Post.date_time |
| `content` | `markdown.toHtml()` | Markdown 转换 |
| `tags` | `generatePostPage()` | 标签链接 HTML |
| `posts` | `generateIndex()` / `generateTagPages()` | 文章列表 HTML |
| `page_title` | `generatePostPage()` | 完整页面标题 |
| `year` | 硬编码 | "2026" |
| `TAG_NAME` | `generateTagPages()` | 标签名称 |
| `post_count` | `generateTagPages()` | 文章数量 |

### 7.2 变量类型

| 变量 | 转义 | 用途 |
|------|------|------|
| `title`, `date_time` | 转义 `{{}}` | 纯文本 |
| `content`, `tags`, `posts` | 未转义 `{{{{}}}}` | HTML 内容 |

---

## 8. 模板设计模式

### 8.1 布局模式 (Layout Pattern)

```
layout.hbs (父模板)
    │
    ├─ 定义整体结构
    ├─ 定义占位块 (page, sidebar)
    └─ 提供公共元素 (header, footer, nav)
```

### 8.2 内联块模式 (Inline Block Pattern)

```handlebars
{{#*inline "page"}}
<!-- 定义 page 块内容 -->
{{/inline}}

{{#*inline "sidebar"}}
<!-- 定义 sidebar 块内容 -->
{{/inline}}
```

### 8.3 继承模式 (Inheritance Pattern)

```handlebars
{{~> (parent)~}}
<!-- 继承父模板，注入定义的块 -->
```

---

## 9. 注意事项

### 9.1 转义与未转义

```handlebars
<!-- 转义输出（安全） -->
{{title}}  <!-- "Hello" → &quot;Hello&quot; -->

<!-- 未转义输出（危险但必要） -->
{{{content}}}  <!-- "<p>Hello</p>" → <p>Hello</p> -->
```

### 9.2 空白修剪

```handlebars
{{~> page}}   <!-- 修剪左侧空白 -->
{{> page~}}   <!-- 修剪右侧空白 -->
{{~> page~}}  <!-- 修剪两侧空白 -->
```

### 9.3 模板命名

```zig
// 模板文件名必须匹配
template_engine.render("post.hbs", &ctx);
//                      ^^^^^^^^^^
//                      templates/post.hbs
```

---

## 相关文档

- [10-2 模板语法](./10-2-template-syntax.md) - Handlebars 语法详解
- [10-3 模板继承](./10-3-inheritance.md) - 继承机制深入
- [10-4 模板详解](./10-4-templates-detail.md) - 每个模板详细分析
