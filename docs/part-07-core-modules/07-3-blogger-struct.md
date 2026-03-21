# 07-3 Blogger 结构体

> 博客生成核心逻辑与页面渲染

---

## 1. Blogger 结构体概览

### 1.1 完整代码

```zig
pub const Blogger = struct {
    dest_dir: []const u8,
    posts_dir: []const u8,
    allocator: Allocator,
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

    pub fn generate(self: *Blogger) !void {
        std.debug.print("Generating blog...\n Posts directory: {s}\n Output directory: {s}\n", .{ self.posts_dir, self.dest_dir });
        try std.fs.cwd().makePath(self.dest_dir);

        // Copy static files
        try self.copyStaticFiles();

        var posts = std.ArrayList(Post){};
        defer {
            for (posts.items) |*post| post.deinit(self.allocator);
            posts.deinit(self.allocator);
        }
        try self.loadPosts(&posts);
        std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

        for (posts.items) |post| try self.generatePostPage(post);
        try self.generateIndex(posts.items);
        try self.generateAbout();
        try self.generateTagPages(posts.items);

        std.debug.print("Blog generation complete!\n", .{});
    }

    // ... 私有方法
};
```

### 1.2 结构体字段

| 字段 | 类型 | 大小 | 说明 |
|------|------|------|------|
| `dest_dir` | `[]const u8` | 16 字节 | 输出目录路径 |
| `posts_dir` | `[]const u8` | 16 字节 | 文章源目录路径 |
| `allocator` | `Allocator` | 16 字节 | 内存分配器 |
| `template_engine` | `TemplateEngine` | ~64 字节 | 模板引擎实例 |

**总大小**：约 112 字节（64 位系统）

### 1.3 方法分类

| 可见性 | 方法数 | 方法列表 |
|--------|--------|----------|
| `pub` | 2 | `new()`, `generate()` |
| `private` | 7 | `copyStaticFiles()`, `loadPosts()`, `generatePostPage()`, `writeHtmlFileWithDate()`, `generateIndex()`, `generateAbout()`, `generateTagPages()` |

---

## 2. 构造函数：new()

### 2.1 函数签名

```zig
pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger
```

**参数分析**：

| 参数 | 类型 | 用途 | 生命周期 |
|------|------|------|----------|
| `allocator` | `Allocator` | 内存分配器 | 由调用者管理 |
| `posts_dir` | `[]const u8` | 文章源目录 | 调用者保证有效 |
| `dest_dir` | `[]const u8` | 输出目录 | 调用者保证有效 |

### 2.2 初始化流程

```zig
pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger {
    const engine = TemplateEngine.init(allocator, "templates");
    return .{
        .allocator = allocator,
        .posts_dir = posts_dir,
        .dest_dir = dest_dir,
        .template_engine = engine,
    };
}
```

**初始化步骤**：

```
[1] 初始化模板引擎
    TemplateEngine.init(allocator, "templates")
    │
    ▼
[2] 构造 Blogger 实例
    .{
        .allocator = allocator,
        .posts_dir = posts_dir,
        .dest_dir = dest_dir,
        .template_engine = engine,
    }
    │
    ▼
[3] 返回实例
```

### 2.3 使用示例

```zig
// 在 main.zig 中
var blogger = Blogger.new(allocator, posts_dir, dest_dir);
try blogger.generate();
```

---

## 3. 主入口：generate()

### 3.1 函数签名

```zig
pub fn generate(self: *Blogger) !void
```

**返回类型**：`!void`（可能失败的无返回值函数）

### 3.2 完整流程

```zig
pub fn generate(self: *Blogger) !void {
    // [1] 打印生成信息
    std.debug.print("Generating blog...\n Posts directory: {s}\n Output directory: {s}\n", .{ self.posts_dir, self.dest_dir });
    
    // [2] 创建输出目录
    try std.fs.cwd().makePath(self.dest_dir);

    // [3] 复制静态文件
    try self.copyStaticFiles();

    // [4] 加载并排序文章
    var posts = std.ArrayList(Post){};
    defer {
        for (posts.items) |*post| post.deinit(self.allocator);
        posts.deinit(self.allocator);
    }
    try self.loadPosts(&posts);
    std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

    // [5] 生成各种页面
    for (posts.items) |post| try self.generatePostPage(post);
    try self.generateIndex(posts.items);
    try self.generateAbout();
    try self.generateTagPages(posts.items);

    // [6] 完成提示
    std.debug.print("Blog generation complete!\n", .{});
}
```

### 3.3 生成流程图

```
generate()
    │
    ├─ [1] 打印配置信息
    │
    ├─ [2] 创建输出目录
    │
    ├─ [3] copyStaticFiles()
    │   └─ 复制 static/ 到输出目录
    │
    ├─ [4] 加载文章
    │   ├─ loadPosts() → ArrayList[Post]
    │   └─ sort() → 按日期降序排序
    │
    ├─ [5] 生成页面
    │   ├─ for 循环 → generatePostPage() (每篇文章)
    │   ├─ generateIndex() (首页)
    │   ├─ generateAbout() (关于页)
    │   └─ generateTagPages() (标签页)
    │
    └─ [6] 打印完成信息
```

### 3.4 内存管理

```zig
var posts = std.ArrayList(Post){};
defer {
    // 清理所有 Post 实例
    for (posts.items) |*post| post.deinit(self.allocator);
    // 清理 ArrayList 本身
    posts.deinit(self.allocator);
}
```

**defer 块执行时机**：

```
generate() 开始
    │
    ▼
posts 初始化
    │
    ▼
loadPosts() → 填充 posts
    │
    ▼
生成各种页面
    │
    ▼
generate() 返回前
    │
    ▼
defer 块执行 → 释放所有内存
```

---

## 4. 静态文件复制：copyStaticFiles()

### 4.1 函数签名

```zig
fn copyStaticFiles(self: *Blogger) !void
```

### 4.2 完整代码

```zig
fn copyStaticFiles(self: *Blogger) !void {
    const static_dir = "static";
    var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
        std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
        return;
    };
    defer dir.close();

    try std.fs.cwd().makePath(self.dest_dir);

    var walker = try dir.walk(self.allocator);
    defer walker.deinit();

    while (try walker.next()) |entry| {
        if (entry.kind == .file) {
            const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
            defer self.allocator.free(src_path);
            const dest_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, entry.path });
            defer self.allocator.free(dest_path);

            const dest_dir_path = std.fs.path.dirname(dest_path);
            if (dest_dir_path) |dir_path| {
                try std.fs.cwd().makePath(dir_path);
            }

            var src_file = try dir.openFile(entry.path, .{});
            defer src_file.close();
            const dest_file = try std.fs.cwd().createFile(dest_path, .{});
            defer dest_file.close();
            const content = try src_file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);
            try dest_file.writeAll(content);

            std.debug.print(" Copied: {s}\n", .{entry.path});
        }
    }
}
```

### 4.3 错误处理模式

```zig
var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
    std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
    return;  // 优雅降级：没有静态文件也能继续
};
```

**catch vs try**：

```zig
// try - 传播错误
try std.fs.cwd().makePath(self.dest_dir);

// catch - 本地处理
var dir = openDir() catch {
    // 处理错误
    return;
};
```

### 4.4 目录遍历

```zig
var walker = try dir.walk(self.allocator);
defer walker.deinit();

while (try walker.next()) |entry| {
    if (entry.kind == .file) {
        // 处理文件
    }
}
```

**walker 遍历结果**：

```
static/
├── style.css      → entry.path = "style.css"
└── imgs/
    └── logo.png   → entry.path = "imgs/logo.png"
```

---

## 5. 文章加载：loadPosts()

### 5.1 函数签名

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void
```

### 5.2 完整代码

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    var dir = try std.fs.cwd().openDir(self.posts_dir, .{ .iterate = true });
    defer dir.close();

    var walker = try dir.walk(self.allocator);
    defer walker.deinit();

    while (try walker.next()) |entry| {
        if (entry.kind == .file and std.mem.endsWith(u8, entry.path, ".markdown")) {
            // 排除 about.markdown
            if (std.mem.eql(u8, std.fs.path.basename(entry.path), "about.markdown")) {
                continue;
            }

            const file = try dir.openFile(entry.path, .{});
            defer file.close();
            const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);

            const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| {
                std.debug.print(" Skipped ({s}): {s}\n", .{ @errorName(err), entry.path });
                continue;
            };
            try posts.append(self.allocator, post);
            std.debug.print(" Loaded: {s}\n", .{entry.path});
        }
    }
}
```

### 5.3 文件过滤逻辑

```zig
if (entry.kind == .file and std.mem.endsWith(u8, entry.path, ".markdown")) {
    // 排除 about.markdown
    if (std.mem.eql(u8, std.fs.path.basename(entry.path), "about.markdown")) {
        continue;
    }
    // 处理文章文件
}
```

**过滤条件**：

```
目录遍历结果
    │
    ├─ 是目录？ → 跳过
    ├─ 不是 .markdown？ → 跳过
    ├─ 是 about.markdown？ → 跳过（单独处理）
    └─ 是普通文章？ → 加载
```

### 5.4 错误恢复

```zig
const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| {
    std.debug.print(" Skipped ({s}): {s}\n", .{ @errorName(err), entry.path });
    continue;  // 跳过坏文章
};
```

**错误处理策略**：

```
Post.parse() 失败
    │
    ├─ 捕获错误
    │
    ├─ 打印错误信息（包括错误类型和文件路径）
    │
    └─ continue → 继续处理下一篇文章
```

---

## 6. 文章页面生成：generatePostPage()

### 6.1 函数签名

```zig
fn generatePostPage(self: *Blogger, post: Post) !void
```

### 6.2 关键步骤

```zig
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] Markdown → HTML
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);

    // [2] 初始化模板上下文
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
        try tags_html.appendSlice(self.allocator, "<a href=\"/tags/" ++ tag ++ ".html\" class=\"tag-link\">" ++ tag ++ "</a>");
    }
    try ctx.set("tags", tags_html.items);

    // [4] 渲染 post.hbs
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);

    // [5] 设置页面标题和年份
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

    // [7] 写入文件（日期目录结构）
    try self.writeHtmlFileWithDate(post.filename, post.date_time, html);
}
```

### 6.3 模板渲染流程

```
post.hbs 渲染
    │
    ├─ 上下文变量
    │   ├─ title
    │   ├─ date_time
    │   ├─ content (HTML)
    │   └─ tags (HTML)
    │
    ▼
post.hbs → page_html
    │
    ▼
layout.hbs 渲染
    │
    ├─ 继承 post.hbs 内容
    ├─ 添加页眉页脚
    └─ 添加完整 HTML 结构
    │
    ▼
最终 HTML
```

---

## 7. 日期目录结构：writeHtmlFileWithDate()

### 7.1 函数签名

```zig
fn writeHtmlFileWithDate(self: *Blogger, markdown_filename: []const u8, date_time: []const u8, html: []const u8) !void
```

### 7.2 日期解析

```zig
// 输入格式："2026-03-21" 或 "2026-3-5"
const year = date_time[0..4];  // "2026"

// 解析月份（处理前导零）
var month_end = pos;
while (month_end < date_time.len and date_time[month_end] != '-') : (month_end += 1) {}
const month = if (month_end - pos == 1) blk: {
    month_buf[0] = '0';
    month_buf[1] = date_time[pos];
    break :blk month_buf[0..2];
} else date_time[pos..month_end];

// 解析日期（处理前导零）
// 类似逻辑...
```

**输出路径**：

```
输入：markdown_filename = "my-post.markdown"
      date_time = "2026-03-21"

输出：zig-out/blog/2026/03/21/my-post.html
                          │    │   │    └─ 文件名
                          │    │   └─ 日期
                          │    └─ 月份
                          └─ 年份
```

---

## 8. 首页生成：generateIndex()

### 8.1 函数签名

```zig
fn generateIndex(self: *Blogger, posts: []const Post) !void
```

### 8.2 文章列表 HTML 构建

```zig
for (posts) |post| {
    // 构建文章链接（日期目录结构）
    var filename = std.ArrayList(u8){};
    defer filename.deinit(self.allocator);
    
    // YYYY/MM/DD/
    if (post.date_time.len >= 8) {
        try filename.appendSlice(self.allocator, post.date_time[0..4]);
        try filename.appendSlice(self.allocator, "/");
        // 月份和日期解析（带前导零填充）
        // ...
    }
    
    // 文章 slug
    const base_len = post.filename.len;
    if (base_len >= 9 and std.mem.endsWith(u8, post.filename, ".markdown")) {
        try filename.appendSlice(self.allocator, post.filename[0 .. base_len - 9]);
    }
    try filename.appendSlice(self.allocator, ".html");
    
    // 构建文章条目 HTML
    try posts_html.appendSlice(self.allocator, "<article class=\"post-preview\">\n");
    try posts_html.appendSlice(self.allocator, "<h2><a href=\"" ++ filename.items ++ "\">" ++ post.title ++ "</a></h2>\n");
    try posts_html.appendSlice(self.allocator, "<div class=\"post-meta\">" ++ post.date_time ++ "</div>\n");
    try posts_html.appendSlice(self.allocator, "<div class=\"post-tags\">" ++ tags_html.items ++ "</div>\n");
    try posts_html.appendSlice(self.allocator, "</article>\n");
}
```

---

## 9. 标签页面：generateTagPages()

### 9.1 标签映射构建

```zig
var tag_map = std.StringHashMap(std.ArrayListUnmanaged(Post)).init(self.allocator);
defer {
    var it = tag_map.iterator();
    while (it.next()) |entry| {
        entry.value_ptr.deinit(self.allocator);
    }
    tag_map.deinit();
}

// 构建 tag → posts 映射
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
```

**HashMap 结构**：

```
tag_map: StringHashMap(ArrayListUnmanaged(Post))
    │
    ├─ "zig" → [post1, post2, post3]
    ├─ "blog" → [post1, post4]
    └─ "tutorial" → [post2, post3]
```

---

## 10. 排序函数：sortPostsByDateDesc

### 10.1 函数签名

```zig
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool
```

### 10.2 比较逻辑

```zig
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

**排序顺序**：

```
输入文章（无序）:
  - Post A: 2026-03-15
  - Post B: 2026-03-21
  - Post C: 2026-03-10

排序后（降序）:
  - Post B: 2026-03-21 (最新)
  - Post A: 2026-03-15
  - Post C: 2026-03-10 (最旧)
```

---

## 11. 设计模式总结

### 11.1 构造器模式

```zig
// 清晰的构造语义
var blogger = Blogger.new(allocator, posts_dir, dest_dir);
```

### 11.2 外观模式

```zig
// 单一入口点，隐藏复杂内部逻辑
try blogger.generate();
```

### 11.3 资源管理模式

```zig
// 统一的内存管理
defer {
    for (posts.items) |*post| post.deinit(self.allocator);
    posts.deinit(self.allocator);
}
```

---

## 相关文档

- [07-2 Post 结构体](./07-2-post-struct.md) - 文章数据结构
- [07-4 文件系统操作](./07-4-filesystem-ops.md) - std.fs API 使用
- [10-4 模板详解](../part-10-template-engine/10-4-templates-detail.md) - 模板文件分析
