# 05-1 函数基础

> Zig 语言的函数定义与调用机制

---

## 1. 函数定义基础

### 1.1 基本语法

Zig 函数使用 `fn` 关键字定义：

```zig
fn functionName(param1: Type1, param2: Type2) ReturnType {
    // 函数体
}
```

**项目实例** - `Blogger.zig` 中的排序函数：

```zig
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

### 1.2 返回类型

#### 无返回值

```zig
fn printHelp() void {
    std.debug.print("Usage: my-blog [OPTIONS]\n", .{});
}
```

**项目实例** - `main.zig` 中的帮助函数：

```zig
fn printHelp() void {
    std.debug.print(
        \\my-blog - A static blog generator written in Zig
        \\
        \\Usage: my-blog [OPTIONS]
        \\
        \\Options:
        \\  -p, --posts     Posts source directory (default: posts)
        \\  -o, --output    Output directory (default: zig-out/blog)
        \\  -h, --help      Show this help message
        \\
    , .{});
}
```

#### 有返回值

```zig
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

---

## 2. 错误返回类型

### 2.1 错误联合类型

使用 `!T` 表示可能返回错误的函数：

```zig
fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // 可能返回 error.InvalidFormat, error.MissingTitle 等
}
```

**项目实例** - `Post.parse()` 的完整定义：

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    var lines = std.mem.splitScalar(u8, content, '\n');
    const first_line = lines.next() orelse return error.InvalidFormat;
    
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
    }

    var title: ?[]const u8 = null;
    var date_time: ?[]const u8 = null;
    var tags: ?[]const u8 = null;
    var frontmatter_end: usize = 1;

    while (lines.next()) |line| {
        const trimmed = std.mem.trim(u8, line, " \r\n\t");
        if (std.mem.eql(u8, trimmed, "---")) {
            frontmatter_end += 1;
            break;
        }
        if (std.mem.startsWith(u8, trimmed, "title:")) {
            title = try extractValue(allocator, trimmed, "title:");
        } else if (std.mem.startsWith(u8, trimmed, "date_time:")) {
            date_time = try extractValue(allocator, trimmed, "date_time:");
        } else if (std.mem.startsWith(u8, trimmed, "tags:")) {
            tags = try extractValue(allocator, trimmed, "tags:");
        }
        frontmatter_end += 1;
    }

    const content_start = content_offset_for_line(content, frontmatter_end);
    return Post{
        .title = title orelse return error.MissingTitle,
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",
        .content = try allocator.dupe(u8, content[content_start..]),
        .filename = try allocator.dupe(u8, filename),
    };
}
```

### 2.2 可能的错误类型

从函数签名可以看出可能的错误：

```zig
// Post.parse 可能返回的错误：
// - error.InvalidFormat (frontmatter 格式错误)
// - error.MissingTitle (缺少 title 字段)
// - error.MissingDateTime (缺少 date_time 字段)
// - error.OutOfMemory (内存分配失败)
```

---

## 3. 函数可见性

### 3.1 公共函数 (pub)

使用 `pub` 关键字使函数对外可见：

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

**项目中的公共函数**：

| 函数 | 位置 | 用途 |
|------|------|------|
| `main()` | `main.zig` | 程序入口点 |
| `new()` | `Blogger` | 创建 Blogger 实例 |
| `generate()` | `Blogger` | 生成博客站点 |
| `parse()` | `Post` | 解析 Markdown 文件 |
| `deinit()` | `Post` | 释放 Post 内存 |

### 3.2 私有函数 (默认)

不使用 `pub` 的函数默认为私有，只能在模块内访问：

```zig
// 私有函数 - 只能在 Blogger.zig 内调用
fn copyStaticFiles(self: *Blogger) !void {
    // ...
}

fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    // ...
}

fn generatePostPage(self: *Blogger, post: Post) !void {
    // ...
}
```

**项目中的私有函数**：

| 函数 | 所属 | 用途 |
|------|------|------|
| `copyStaticFiles` | `Blogger` | 复制静态文件 |
| `loadPosts` | `Blogger` | 加载文章列表 |
| `generatePostPage` | `Blogger` | 生成文章页面 |
| `generateIndex` | `Blogger` | 生成首页 |
| `generateAbout` | `Blogger` | 生成关于页面 |
| `generateTagPages` | `Blogger` | 生成标签页面 |
| `writeHtmlFileWithDate` | `Blogger` | 写入 HTML 文件 |
| `extractValue` | `Post` | 提取 frontmatter 值 |
| `content_offset_for_line` | `Post` | 计算内容偏移 |

---

## 4. 函数调用

### 4.1 基本调用

```zig
// 调用公共函数
var blogger = Blogger.new(allocator, posts_dir, dest_dir);
try blogger.generate();

// 调用私有函数（在模块内）
try self.copyStaticFiles();
try self.loadPosts(&posts);
```

### 4.2 方法调用语法

Zig 支持两种调用方式：

```zig
// 方式 1：显式传递 self
Blogger.generate(&blogger);

// 方式 2：方法调用语法（推荐）
blogger.generate();
```

**项目实例** - 同一函数的两种调用：

```zig
// 在 generate() 函数内调用其他方法
pub fn generate(self: *Blogger) !void {
    // 方法调用语法
    try self.copyStaticFiles();
    try self.loadPosts(&posts);
    
    // 等价于
    try copyStaticFiles(self);
    try loadPosts(self, &posts);
}
```

---

## 5. 函数作为值

### 5.1 函数指针

函数可以作为参数传递：

```zig
// std.mem.sort 需要比较函数
std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

// sortPostsByDateDesc 的签名必须匹配 std.mem.sort 的要求
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

### 5.2 比较函数模式

Zig 标准库常用 `context` 参数模式：

```zig
// 通用排序函数签名
fn sort(comptime T: type, items: []T, context: anytype, comptime lessThan: fn (@TypeOf(context), T, T) bool) void

// 项目中的使用
std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

// lessThan 函数接收 context 和两个元素
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    // 返回 true 如果 a 应该排在 b 前面
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

---

## 6. 完整函数分析

### 6.1 Blogger.new() - 构造函数

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

**特点分析**：

| 方面 | 说明 |
|------|------|
| 参数 | 3 个：allocator, posts_dir, dest_dir |
| 返回 | `Blogger` 结构体实例 |
| 错误 | 无（不返回错误） |
| 可见性 | `pub` - 外部可调用 |
| 内存 | 不分配内存，只初始化 |

### 6.2 Blogger.generate() - 主入口

```zig
pub fn generate(self: *Blogger) !void {
    std.debug.print("Generating blog...\n Posts directory: {s}\n Output directory: {s}\n", .{ self.posts_dir, self.dest_dir });
    try std.fs.cwd().makePath(self.dest_dir);

    // 复制静态文件
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
```

**特点分析**：

| 方面 | 说明 |
|------|------|
| 参数 | 1 个：`self` 指针 |
| 返回 | `!void` - 可能失败 |
| 可见性 | `pub` - 主要 API |
| 职责 | 协调整个生成流程 |
| 内存管理 | 使用 `defer` 清理临时资源 |

---

## 7. 函数命名约定

### 7.1 动词 + 名词模式

| 模式 | 项目实例 |
|------|----------|
| `loadXxx` | `loadPosts()` |
| `generateXxx` | `generateIndex()`, `generateAbout()`, `generateTagPages()` |
| `writeXxx` | `writeHtmlFileWithDate()` |
| `copyXxx` | `copyStaticFiles()` |

### 7.2 生命周期方法

| 方法 | 用途 |
|------|------|
| `new()` | 构造函数 |
| `init()` | 初始化（项目中使用 `TemplateEngine.init()`） |
| `deinit()` | 析构/清理（`Post.deinit()`） |
| `parse()` | 解析/创建（`Post.parse()`） |

---

## 8. 注意事项

### 8.1 参数传递方式

```zig
// 值传递 - 复制
fn process(post: Post) void { }

// 指针传递 - 可修改
fn modify(post: *Post) void { }

// 切片传递 - 引用语义
fn process(content: []const u8) void { }
```

### 8.2 错误传播

```zig
// 使用 try 传播错误
fn caller() !void {
    try someFallibleFunction();
}

// 使用 catch 处理错误
fn caller() void {
    someFallibleFunction() catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };
}
```

### 8.3 defer 在函数中的使用

```zig
fn processFile(allocator: Allocator) !void {
    var buffer = try allocator.alloc(u8, 1024);
    defer allocator.free(buffer);  // 函数返回前释放
    
    // 使用 buffer
    // ...
}  // defer 在此执行
```

---

## 相关文档

- [03-3 可选类型与错误处理](../part-03-zig-basics/03-3-optional-and-error.md) - 错误联合类型
- [05-2 参数与返回值](./05-2-parameters-and-return.md) - 深入参数传递
- [08-1 内存分配器](../part-08-memory-management/08-1-allocators.md) - Allocator 详解
