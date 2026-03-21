# 10-4 模板详解

> 每个模板文件的详细分析与变量说明

---

## 1. 模板文件概览

| 模板 | 用途 | 输入变量 | 输出文件 |
|------|------|----------|----------|
| `layout.hbs` | 主布局 | title, year, page, sidebar | 所有 HTML 页面 |
| `index.hbs` | 首页 | posts | index.html |
| `post.hbs` | 文章页 | TITLE, date_time, tags, content | YYYY/MM/DD/post.html |
| `about.hbs` | 关于页 | content | about.html |
| `tag.hbs` | 标签页 | TAG_NAME, post_count, posts | tags/TAG.html |

---

## 2. layout.hbs - 主布局模板

### 2.1 完整代码

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

### 2.2 变量说明

| 变量 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `{{title}}` | 转义 | `ctx.set("title", ...)` | 页面标题 |
| `{{siteTitle}}` | 转义 | 未设置（可能硬编码） | 站点标题 |
| `{{year}}` | 转义 | `ctx.set("year", "2026")` | 版权年份 |
| `{{~> page}}` | 块 | 子模板定义 | 主内容 |
| `{{~> sidebar}}` | 块 | 子模板定义 | 侧边栏 |

### 2.3 设置位置

```zig
// generatePostPage() 中
try ctx.set("title", post.title);  // 文章标题
try ctx.set("year", "2026");       // 硬编码年份

// generateIndex() 中
try ctx.set("title", "");          // 首页标题为空
try ctx.set("year", "2026");

// generateAbout() 中
try ctx.set("title", "About");     // 关于页标题
try ctx.set("year", "2026");
```

### 2.4 生成的 HTML 结构

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Blog</title>
    <link rel="stylesheet" href="/style.css">
</head>
<body>
    <header>
        <h1><a href="/">My Blog</a></h1>
        <nav>
            <a href="/">Home</a>
            <a href="/about.html">About</a>
        </nav>
    </header>

    <main>
        <!-- 子模板的 page 块内容注入到这里 -->
    </main>

    <aside>
        <!-- 子模板的 sidebar 块内容注入到这里 -->
    </aside>

    <footer>
        <p>Generated with <a href="https://ziglang.org">Zig</a> · © 2026</p>
    </footer>
</body>
</html>
```

---

## 3. index.hbs - 首页模板

### 3.1 完整代码

```handlebars
{{#*inline "page"}}
{{{posts}}}
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

### 3.2 变量说明

| 变量 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `{{{posts}}}` | 未转义 | `generateIndex()` 生成 | 文章列表 HTML |

### 3.3 posts 变量生成

```zig
// generateIndex() 中
var posts_html = std.ArrayList(u8){};
defer posts_html.deinit(self.allocator);

for (posts) |post| {
    // 构建文章链接
    var filename = std.ArrayList(u8){};
    // ... 构建 YYYY/MM/DD/filename.html 路径

    // 构建标签 HTML
    var tags_html = std.ArrayList(u8){};
    var tag_iter = std.mem.tokenizeScalar(u8, post.tags, ' ');
    while (tag_iter.next()) |tag| {
        try tags_html.appendSlice(self.allocator, 
            "<a href=\"/tags/" ++ tag ++ ".html\" class=\"tag-link\">" ++ tag ++ "</a>");
    }

    // 构建文章条目
    try posts_html.appendSlice(self.allocator, "<article class=\"post-preview\">\n");
    try posts_html.appendSlice(self.allocator, 
        "<h2><a href=\"" ++ filename.items ++ "\">" ++ post.title ++ "</a></h2>\n");
    try posts_html.appendSlice(self.allocator, 
        "<div class=\"post-meta\">" ++ post.date_time ++ "</div>\n");
    try posts_html.appendSlice(self.allocator, 
        "<div class=\"post-tags\">" ++ tags_html.items ++ "</div>\n");
    try posts_html.appendSlice(self.allocator, "</article>\n");
}

try ctx.set("posts", posts_html.items);
```

### 3.4 生成的 HTML

```html
<!-- 注入到 layout.hbs 的 <main> -->
<main>
    <article class="post-preview">
        <h2><a href="2026/03/21/my-post.html">My Post</a></h2>
        <div class="post-meta">2026-03-21</div>
        <div class="post-tags">
            <a href="/tags/zig.html" class="tag-link">zig</a>
            <a href="/tags/blog.html" class="tag-link">blog</a>
        </div>
    </article>
    
    <article class="post-preview">
        <!-- 更多文章... -->
    </article>
</main>
```

---

## 4. post.hbs - 文章页模板

### 4.1 完整代码

```handlebars
{{TITLE}}

{{date_time}}
{{{tags}}}
{{{content}}}
```

### 4.2 变量说明

| 变量 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `{{TITLE}}` | 转义 | `ctx.set("title", post.title)` | 文章标题 |
| `{{date_time}}` | 转义 | `ctx.set("date_time", post.date_time)` | 发布日期 |
| `{{{tags}}}` | 未转义 | `generatePostPage()` 生成 | 标签链接 HTML |
| `{{{content}}}` | 未转义 | `markdown.toHtml()` | Markdown 转换的 HTML |

### 4.3 变量生成流程

```zig
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] Markdown → HTML
    const html_content = try markdown.toHtml(self.allocator, post.content);
    // html_content = "<p>...</p>\n<h2>...</h2>\n..."

    // [2] 设置上下文
    var ctx = Context.init(self.allocator);
    try ctx.set("title", post.title);       // "My First Post"
    try ctx.set("date_time", post.date_time); // "2026-03-21"
    try ctx.set("content", html_content);   // "<p>...</p>"

    // [3] 生成标签 HTML
    var tags_html = std.ArrayList(u8){};
    var tag_iter = std.mem.tokenizeScalar(u8, post.tags, ' ');
    while (tag_iter.next()) |tag| {
        try tags_html.appendSlice(self.allocator, 
            "<a href=\"/tags/" ++ tag ++ ".html\" class=\"tag-link\">" ++ tag ++ "</a>");
    }
    try ctx.set("tags", tags_html.items);
    // tags = "<a href=\"/tags/zig\">zig</a> <a href=\"/tags/blog\">blog</a>"

    // [4] 渲染 post.hbs
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    // page_html = "My First Post\n\n2026-03-21\n<a>zig</a> <a>blog</a>\n<p>...</p>"
}
```

### 4.4 生成的 HTML

```html
<!-- post.hbs 渲染结果 -->
My First Post

2026-03-21
<a href="/tags/zig.html" class="tag-link">zig</a>
<a href="/tags/blog.html" class="tag-link">blog</a>
<h1>Article Title</h1>
<p>Article content...</p>

<!-- 然后注入到 layout.hbs -->
<main>
    上述内容
</main>
```

---

## 5. about.hbs - 关于页模板

### 5.1 完整代码

```handlebars
{{#*inline "page"}}
{{{content}}}
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

### 5.2 变量说明

| 变量 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `{{{content}}}` | 未转义 | `markdown.toHtml()` | about.markdown 的 HTML |

### 5.3 生成流程

```zig
fn generateAbout(self: *Blogger) !void {
    // [1] 加载 about.markdown
    const about_path = try std.fs.path.join(self.allocator, &.{ 
        self.posts_dir, "about.markdown" 
    });
    defer self.allocator.free(about_path);

    const file = std.fs.cwd().openFile(about_path, .{}) catch {
        std.debug.print(" Skipped: about.markdown not found\n", .{});
        return;
    };
    defer file.close();

    const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
    defer self.allocator.free(content);

    // [2] Markdown → HTML (about.markdown 没有 frontmatter)
    const html_content = try markdown.toHtml(self.allocator, content);
    defer self.allocator.free(html_content);

    // [3] 设置上下文
    var ctx = Context.init(self.allocator);
    try ctx.set("title", "About");
    try ctx.set("content", html_content);
    try ctx.set("year", "2026");

    // [4] 渲染 about.hbs
    const page_html = try self.template_engine.render("about.hbs", &ctx);
    defer self.allocator.free(page_html);

    // [5] 渲染 layout.hbs
    self.template_engine.setPageContent(page_html);
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);

    // [6] 写入 about.html
    const output_path = try std.fs.path.join(self.allocator, &.{ 
        self.dest_dir, "about.html" 
    });
    defer self.allocator.free(output_path);
    const out_file = try std.fs.cwd().createFile(output_path, .{});
    defer out_file.close();
    try out_file.writeAll(html);
}
```

### 5.4 生成的 HTML

```html
<!-- about.hbs 渲染结果 -->
<h1>About Me</h1>
<p>I am a software developer...</p>

<!-- 注入到 layout.hbs -->
<main>
    <h1>About Me</h1>
    <p>I am a software developer...</p>
</main>
```

---

## 6. tag.hbs - 标签页模板

### 6.1 完整代码

```handlebars
{{#*inline "page"}}

POSTS TAGGED WITH "{{TAG_NAME}}"

{{post_count}} post(s) found

{{{posts}}}

{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

### 6.2 变量说明

| 变量 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `{{TAG_NAME}}` | 转义 | `generateTagPages()` | 标签名称 |
| `{{post_count}}` | 转义 | `std.fmt.bufPrint()` | 文章数量 |
| `{{{posts}}}` | 未转义 | `generateTagPages()` 生成 | 该标签下的文章列表 HTML |

### 6.3 变量生成

```zig
fn generateTagPages(self: *Blogger, posts: []const Post) !void {
    // [1] 构建标签映射
    var tag_map = std.StringHashMap(std.ArrayListUnmanaged(Post)).init(self.allocator);
    defer tag_map.deinit();

    for (posts) |post| {
        var tag_iter = std.mem.tokenizeScalar(u8, post.tags, ' ');
        while (tag_iter.next()) |tag| {
            const gop = try tag_map.getOrPut(tag);
            if (!gop.found_existing) {
                gop.value_ptr.* = .{};
            }
            try gop.value_ptr.append(self.allocator, post);
        }
    }

    // [2] 为每个标签生成页面
    var it = tag_map.iterator();
    while (it.next()) |entry| {
        const tag_name = entry.key_ptr.*;  // "zig"
        const tag_posts = entry.value_ptr.*;  // [post1, post2, ...]

        // [3] 生成文章列表 HTML
        var posts_html = std.ArrayList(u8){};
        for (tag_posts.items) |post| {
            // ... 构建文章条目（与 index.hbs 类似）
        }

        // [4] 设置上下文
        var ctx = Context.init(self.allocator);
        
        var post_count_buf: [16]u8 = undefined;
        const post_count_str = std.fmt.bufPrint(&post_count_buf, "{d}", .{
            tag_posts.items.len
        }) catch "0";
        // post_count_str = "3"

        var page_title_buf: [256]u8 = undefined;
        const page_title_str = std.fmt.bufPrint(&page_title_buf, 
            "Posts tagged with \"{s}\" - ", .{tag_name}) catch "";
        // page_title_str = "Posts tagged with \"zig\" - "

        try ctx.set("tag_name", tag_name);
        try ctx.set("post_count", post_count_str);
        try ctx.set("posts", posts_html.items);
        try ctx.set("page_title", page_title_str);
        try ctx.set("year", "2026");

        // [5] 渲染 tag.hbs
        const page_html = try self.template_engine.render("tag.hbs", &ctx);
        defer self.allocator.free(page_html);

        // [6] 渲染 layout.hbs
        self.template_engine.setPageContent(page_html);
        const html = try self.template_engine.render("layout.hbs", &ctx);
        defer self.allocator.free(html);

        // [7] 写入 tags/TAG_NAME.html
        // ...
    }
}
```

### 6.4 生成的 HTML

```html
<!-- tag.hbs 渲染结果 -->
POSTS TAGGED WITH "zig"

3 post(s) found

<article class="post-preview">
    <h2><a href="2026/03/21/post1.html">Post 1</a></h2>
    <div class="post-meta">2026-03-21</div>
    <div class="post-tags">
        <a href="/tags/zig.html" class="tag-link">zig</a>
    </div>
</article>

<article class="post-preview">
    <!-- 更多文章... -->
</article>

<!-- 注入到 layout.hbs -->
<main>
    上述内容
</main>
```

---

## 7. 模板变量汇总表

### 7.1 所有变量及来源

| 变量名 | 模板 | 类型 | 设置位置 | 说明 |
|--------|------|------|----------|------|
| `title` | layout | 转义 | 所有生成函数 | 页面标题 |
| `siteTitle` | layout | 转义 | 未设置 | 站点标题 |
| `year` | layout | 转义 | 所有生成函数 | 年份 |
| `page` | layout | 块 | 子模板定义 | 主内容块 |
| `sidebar` | layout | 块 | 子模板定义 | 侧边栏块 |
| `posts` | index/tag | 未转义 | `generateIndex/TagPages()` | 文章列表 HTML |
| `TITLE` | post | 转义 | `generatePostPage()` | 文章标题 |
| `date_time` | post | 转义 | `generatePostPage()` | 发布日期 |
| `tags` | post | 未转义 | `generatePostPage()` | 标签 HTML |
| `content` | post/about | 未转义 | `markdown.toHtml()` | Markdown HTML |
| `TAG_NAME` | tag | 转义 | `generateTagPages()` | 标签名称 |
| `post_count` | tag | 转义 | `std.fmt.bufPrint()` | 文章数量 |
| `page_title` | layout | 转义 | `generatePostPage()` | 完整页面标题 |

### 7.2 转义 vs 未转义

| 转义 `{{}}` | 未转义 `{{{{}}}}` |
|-------------|------------------|
| `title` | `content` |
| `TITLE` | `tags` |
| `date_time` | `posts` |
| `year` | |
| `TAG_NAME` | |
| `post_count` | |

---

## 8. 模板渲染流程图

```
生成博客页面
    │
    ▼
[1] 准备数据
    │
    ├─ 加载 Post
    ├─ Markdown → HTML
    └─ 构建标签 HTML
    │
    ▼
[2] 设置 Context
    │
    ├─ ctx.set("title", ...)
    ├─ ctx.set("content", ...)
    └─ ctx.set("tags", ...)
    │
    ▼
[3] 渲染子模板
    │
    ├─ render("post.hbs", &ctx)
    │   │
    │   ├─ {{TITLE}} → "My Post"
    │   ├─ {{{content}}} → "<p>...</p>"
    │   └─ {{{tags}}} → "<a>zig</a>"
    │
    ▼
[4] 设置页面内容
    │
    └─ setPageContent(page_html)
    │
    ▼
[5] 渲染父模板
    │
    └─ render("layout.hbs", &ctx)
        │
        ├─ {{title}} → "My Post"
        ├─ {{~> page}} → post.hbs 内容
        └─ {{year}} → "2026"
    │
    ▼
[6] 写入文件
    │
    └─ createFile(path)
        └─ writeAll(html)
```

---

## 9. 注意事项

### 9.1 变量命名一致性

```zig
// 项目中存在不一致
ctx.set("title", ...);    // 小写
ctx.set("TITLE", ...);    // 大写
ctx.set("TAG_NAME", ...); // 蛇形大写

// 建议统一为蛇形小写
ctx.set("tag_name", ...);
```

### 9.2 HTML 转义

```zig
// 用户输入必须转义
{{title}}  // 安全

// 生成的 HTML 使用未转义
{{{content}}}  // 确保来源可信（markdown.toHtml 生成）
```

### 9.3 内存管理

```zig
// 模板渲染结果需要释放
const html = try template_engine.render("layout.hbs", &ctx);
defer self.allocator.free(html);
```

---

## 相关文档

- [10-1 架构设计](./10-1-architecture.md) - 模板引擎架构
- [10-2 模板语法](./10-2-template-syntax.md) - Handlebars 语法
- [10-3 模板继承](./10-3-inheritance.md) - 继承机制
- [07-3 Blogger 结构体](../part-07-core-modules/07-3-blogger-struct.md) - 渲染代码详解
