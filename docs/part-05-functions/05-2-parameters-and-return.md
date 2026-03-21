# 05-2 参数与返回值

> Zig 语言的参数传递机制与返回值模式

---

## 1. 参数传递方式

### 1.1 值传递

对于小类型，Zig 使用值传递（复制）：

```zig
// usize, bool, 小结构体 - 值传递
fn process(count: usize, enabled: bool) void {
    // count 和 enabled 是原始值的副本
}
```

**项目实例** - 比较函数使用值传递：

```zig
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    // a 和 b 是 Post 的副本
    // Post 包含 5 个切片（40 字节），复制成本较低
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

### 1.2 指针传递

对于大类型或需要修改的情况，使用指针：

```zig
// 修改原始值
fn modify(post: *Post) void {
    post.title = "New Title";  // 修改原始数据
}

// 避免大结构体复制
fn process(blogger: *const Blogger) void {
    // 只读访问，不复制
}
```

**项目实例** - `Post.deinit()` 需要修改权限：

```zig
pub fn deinit(self: *Post, allocator: Allocator) void {
    allocator.free(self.title);
    allocator.free(self.date_time);
    allocator.free(self.tags);
    allocator.free(self.content);
    allocator.free(self.filename);
}

// 调用
post.deinit(allocator);  // self 是 *Post
```

### 1.3 切片传递

切片是引用语义（指针 + 长度）：

```zig
// 切片传递 - 不复制底层数据
fn process(content: []const u8) void {
    // content 指向原始数据
}
```

**项目实例** - 大量使用切片参数：

```zig
pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger {
    // posts_dir 和 dest_dir 是切片，传递的是引用
    return .{
        .allocator = allocator,
        .posts_dir = posts_dir,  // 不复制字符串
        .dest_dir = dest_dir,
        .template_engine = engine,
    };
}
```

---

## 2. 返回值类型

### 2.1 简单返回值

```zig
fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

**项目实例** - 返回布尔值：

```zig
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

### 2.2 返回结构体

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

**注意**：使用 `.{} `语法初始化并返回结构体。

### 2.3 返回错误联合

```zig
// 可能失败的函数
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // 可能返回 error.InvalidFormat, error.MissingTitle 等
}
```

---

## 3. 内存所有权与参数

### 3.1 分配器参数模式

需要分配内存的函数通常接收 `allocator` 参数：

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    return Post{
        .title = try allocator.dupe(u8, title_value),      // 分配内存
        .content = try allocator.dupe(u8, content_value),  // 分配内存
        // ...
    };
}
```

**调用者责任**：

```zig
const post = try Post.parse(allocator, content, filename);
defer post.deinit(allocator);  // 调用者负责释放
```

### 3.2 内存所有权转移

```zig
// 函数返回分配的内存
fn createTitle(allocator: Allocator) ![]const u8 {
    return try allocator.dupe(u8, "My Blog Title");
}

// 调用者获得所有权
const title = try createTitle(allocator);
defer allocator.free(title);  // 调用者释放
```

**项目中的模式**：

```zig
// Post.parse 返回的 Post 拥有所有字段的所有权
const post = try Post.parse(allocator, content, filename);
// post.title, post.content 等都是分配的内存
// 必须调用 deinit 释放
defer post.deinit(allocator);
```

---

## 4. 可选类型参数

### 4.1 可选参数模式

Zig 不支持默认参数，但可以使用可选类型：

```zig
fn process(value: ?u32) void {
    const v = value orelse 0;  // 提供默认值
    // 使用 v
}
```

**项目实例** - `Post` 字段处理：

```zig
var title: ?[]const u8 = null;
var date_time: ?[]const u8 = null;
var tags: ?[]const u8 = null;

// 解析后
return Post{
    .title = title orelse return error.MissingTitle,      // 必需
    .date_time = date_time orelse return error.MissingDateTime,  // 必需
    .tags = tags orelse "",  // 可选，使用空字符串默认值
};
```

---

## 5. 函数签名分析

### 5.1 main.zig 中的函数

```zig
// 入口点 - 返回错误联合
pub fn main() !void {
    // ...
}

// 辅助函数 - 无返回值
fn printHelp() void {
    // ...
}
```

| 函数 | 参数 | 返回 | 错误 |
|------|------|------|------|
| `main()` | 无 | `void` | `!` |
| `printHelp()` | 无 | `void` | 无 |

### 5.2 Blogger.zig 中的函数

#### Post 结构体方法

```zig
// 析构函数
pub fn deinit(self: *Post, allocator: Allocator) void

// 解析函数
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post

// 辅助函数（私有）
fn extractValue(allocator: Allocator, line: []const u8, prefix: []const u8) ![]const u8
fn content_offset_for_line(content: []const u8, line_num: usize) usize
```

| 函数 | 参数数 | 返回 | 错误 | 可见性 |
|------|--------|------|------|--------|
| `deinit()` | 2 | `void` | 无 | `pub` |
| `parse()` | 3 | `Post` | `!` | `pub` |
| `extractValue()` | 3 | `[]const u8` | `!` | `private` |
| `content_offset_for_line()` | 2 | `usize` | 无 | `private` |

#### Blogger 结构体方法

```zig
// 构造函数
pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger

// 主入口
pub fn generate(self: *Blogger) !void

// 私有方法
fn copyStaticFiles(self: *Blogger) !void
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void
fn generatePostPage(self: *Blogger, post: Post) !void
fn writeHtmlFileWithDate(self: *Blogger, markdown_filename: []const u8, date_time: []const u8, html: []const u8) !void
fn generateIndex(self: *Blogger, posts: []const Post) !void
fn generateAbout(self: *Blogger) !void
fn generateTagPages(self: *Blogger, posts: []const Post) !void
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool
```

| 函数 | 参数数 | 返回 | 错误 | 可见性 |
|------|--------|------|------|--------|
| `new()` | 3 | `Blogger` | 无 | `pub` |
| `generate()` | 1 | `void` | `!` | `pub` |
| `copyStaticFiles()` | 1 | `void` | `!` | `private` |
| `loadPosts()` | 2 | `void` | `!` | `private` |
| `generatePostPage()` | 2 | `void` | `!` | `private` |
| `writeHtmlFileWithDate()` | 4 | `void` | `!` | `private` |
| `generateIndex()` | 2 | `void` | `!` | `private` |
| `generateAbout()` | 1 | `void` | `!` | `private` |
| `generateTagPages()` | 2 | `void` | `!` | `private` |
| `sortPostsByDateDesc()` | 3 | `bool` | 无 | `private` |

---

## 6. 特殊参数模式

### 6.1 context 参数

标准库排序函数使用 `context` 参数模式：

```zig
// std.mem.sort 签名
fn sort(comptime T: type, items: []T, context: anytype, comptime lessThan: fn (@TypeOf(context), T, T) bool) void

// 项目使用
std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

// context 是 void (空值 {})
// sortPostsByDateDesc 第一个参数是 void 类型
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool
```

### 6.2 self 参数

Zig 结构体方法使用显式 `self` 参数：

```zig
// 方法定义
pub fn generate(self: *Blogger) !void {
    // 使用 self 访问字段
    std.debug.print("Posts: {s}\n", .{self.posts_dir});
}

// 两种调用方式等价
blogger.generate();           // 方法语法
Blogger.generate(&blogger);   // 函数语法
```

---

## 7. 返回值优化

### 7.1 命名返回值

Zig 不支持命名返回值，但可以使用局部变量：

```zig
// 不推荐 - 多余变量
fn create() Post {
    var post: Post = undefined;
    post.title = "Title";
    return post;
}

// 推荐 - 直接返回
fn create() Post {
    return .{
        .title = "Title",
        // ...
    };
}
```

### 7.2 早期返回

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    var lines = std.mem.splitScalar(u8, content, '\n');
    const first_line = lines.next() orelse return error.InvalidFormat;  // 早期返回
    
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;  // 早期返回
    }
    
    // ...
}
```

---

## 8. 完整实例分析

### 8.1 Post.parse() 参数与返回分析

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // [1] 参数分析
    // - allocator: 分配器，用于分配 Post 字段内存
    // - content: 输入内容（切片，不复制）
    // - filename: 文件名（切片，不复制）
    
    // [2] 返回分析
    // - 返回类型：!Post (可能失败的 Post)
    // - 成功：返回 Post 实例
    // - 失败：返回 error.InvalidFormat, error.MissingTitle, error.MissingDateTime
    
    var lines = std.mem.splitScalar(u8, content, '\n');
    const first_line = lines.next() orelse return error.InvalidFormat;
    
    // [3] 验证 frontmatter
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
    }

    // [4] 解析字段（可选类型）
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

    // [5] 计算内容偏移
    const content_start = content_offset_for_line(content, frontmatter_end);
    
    // [6] 构造并返回 Post
    return Post{
        .title = title orelse return error.MissingTitle,
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",  // tags 可选
        .content = try allocator.dupe(u8, content[content_start..]),
        .filename = try allocator.dupe(u8, filename),
    };
}
```

### 8.2 Blogger.new() 参数与返回分析

```zig
pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger {
    // [1] 参数分析
    // - allocator: 分配器，传递给 TemplateEngine
    // - posts_dir: 文章目录路径（切片引用）
    // - dest_dir: 输出目录路径（切片引用）
    
    // [2] 初始化依赖
    const engine = TemplateEngine.init(allocator, "templates");
    
    // [3] 返回 Blogger 实例
    return .{
        .allocator = allocator,
        .posts_dir = posts_dir,  // 只存储引用，不复制
        .dest_dir = dest_dir,    // 只存储引用，不复制
        .template_engine = engine,
    };
}
```

**内存语义**：

| 字段 | 所有权 | 说明 |
|------|--------|------|
| `allocator` | 引用 | 不拥有分配器，只使用 |
| `posts_dir` | 引用 | 切片引用，调用者保证生命周期 |
| `dest_dir` | 引用 | 切片引用，调用者保证生命周期 |
| `template_engine` | 拥有 | TemplateEngine 拥有内部资源 |

---

## 9. 注意事项

### 9.1 切片生命周期

```zig
// 错误 - 返回局部变量的切片
fn bad() []const u8 {
    var buffer: [100]u8 = undefined;
    return buffer[0..5];  // 错误：buffer 在函数返回后失效
}

// 正确 - 使用分配器
fn good(allocator: Allocator) ![]const u8 {
    return try allocator.dupe(u8, "hello");
}
```

### 9.2 指针与 const

```zig
// 可修改
fn modify(post: *Post) void {
    post.title = "New";
}

// 只读
fn read(post: *const Post) void {
    // post.title = "New";  // 错误：不能修改
}
```

### 9.3 错误返回一致性

```zig
// 推荐 - 统一的错误处理
fn process() !void {
    if (invalid) return error.InvalidInput;
    try doSomething();  // 使用 try 传播
}
```

---

## 相关文档

- [05-1 函数基础](./05-1-function-basics.md) - 函数定义与调用
- [05-3 泛型编程](./05-3-generics.md) - anytype 与 comptime 参数
- [08-2 内存所有权](../part-08-memory-management/08-2-memory-ownership.md) - 内存管理深入
