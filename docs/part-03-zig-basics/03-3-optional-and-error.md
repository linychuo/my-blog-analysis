# 03-3 可选类型与错误处理

> Zig 的空值安全与错误处理机制

---

## 1. 可选类型 (Optional Types)

### 1.1 什么是可选类型

可选类型表示一个值可能存在也可能不存在：

```zig
const value: ?i32 = null;      // 值为空
const another: ?i32 = 42;      // 值为 42
```

### 1.2 项目中的可选类型

**Frontmatter 解析** - 字段是可选的：

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    var title: ?[]const u8 = null;
    var date_time: ?[]const u8 = null;
    var tags: ?[]const u8 = null;  // tags 是可选的

    // 解析...
    
    return Post{
        .title = title orelse return error.MissingTitle,
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",  // tags 可选，默认为空
        // ...
    };
}
```

### 1.3 可选类型的使用模式

#### 模式 1: if 捕获

```zig
const value: ?i32 = getValue();

if (value) |v| {
    // v 的类型是 i32，这里保证不为 null
    std.debug.print("Value: {d}\n", .{v});
} else {
    // value 为 null
    std.debug.print("No value\n", .{});
}
```

#### 模式 2: orelse 提供默认值

```zig
const tags = post.tags orelse "";  // 如果为 null，使用空字符串
```

#### 模式 3: orelse 返回错误

```zig
const title = title_opt orelse return error.MissingTitle;
```

#### 模式 4: while 循环解包

```zig
var node: ?Node = getFirstNode();
while (node) |n| {
    process(n);
    node = n.next;
}
```

---

## 2. 错误类型 (Error Types)

### 2.1 错误集合定义

```zig
const ParseError = error{
    InvalidFormat,
    MissingTitle,
    MissingDateTime,
};
```

### 2.2 项目中的错误类型

**Post 解析可能返回的错误**：

```zig
pub fn parse(...) !Post {
    // 没有下一行 → 格式错误
    const first_line = lines.next() orelse return error.InvalidFormat;
    
    // 第一行不是 --- → 格式错误
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
    }
    
    // 缺少 title 字段
    const title = title orelse return error.MissingTitle;
    
    // 缺少 date_time 字段
    const date_time = date_time orelse return error.MissingDateTime;
}
```

### 2.3 错误推断

Zig 自动推断函数可能返回的错误：

```zig
pub fn main() !void {
    // !void = 可能返回错误的 void
}

pub fn generate(self: *Blogger) !void {
    // 可能返回各种错误
}
```

### 2.4 错误并集

```zig
const MyError = error{A, B} || error{C, D};  // error{A, B, C, D}
```

---

## 3. 错误处理模式

### 3.1 try - 传播错误

```zig
pub fn generate(self: *Blogger) !void {
    try std.fs.cwd().makePath(self.dest_dir);  // 错误向上传播
    try self.copyStaticFiles();
    try self.loadPosts(&posts);
}
```

**项目中的广泛应用**：

```zig
// Blogger.zig - generatePostPage 函数
pub fn generatePostPage(self: *Blogger, post: Post) !void {
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);

    var ctx = Context.init(self.allocator);
    defer ctx.deinit();

    try ctx.set("title", post.title);
    try ctx.set("date_time", post.date_time);
    try ctx.set("content", html_content);
    
    // ...
}
```

### 3.2 catch - 捕获错误

```zig
// 捕获错误并提供默认值
const value = riskyOperation() catch defaultValue;

// 捕获错误并返回新错误
const result = parse() catch return error.ParseFailed;

// 捕获错误并执行代码
const file = openFile() catch |err| {
    std.debug.print("Failed to open: {}\n", .{err});
    return error.FileNotFound;
};
```

**项目实例**：

```zig
// 打开静态文件目录，如果不存在则跳过
var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
    std.debug.print("  Warning: static directory not found, skipping\n", .{});
    return;
};
defer dir.close();

// Post 解析失败时跳过文件
const post = Post.parse(self.allocator, content, path) catch |err| {
    std.debug.print("  Skipped ({s}): {s}\n", .{ @errorName(err), entry.path });
    continue;
};
```

### 3.3 try 与 catch 的区别

| 关键字 | 行为 | 使用场景 |
|--------|------|---------|
| `try` | 遇到错误立即返回 | 函数内部传播错误 |
| `catch` | 捕获并处理错误 | 需要恢复或转换错误 |

---

## 4. 错误类型与可选类型的组合

### 4.1 可选的错误

```zig
const maybe_error: ?ParseError = null;
```

### 4.2 返回可选或错误的函数

```zig
pub fn findPost(name: []const u8) !?Post {
    // 可能返回错误，也可能返回 null（没找到）
}
```

---

## 5. @errorName - 获取错误名称

```zig
const post = Post.parse(...) catch |err| {
    std.debug.print("Skipped ({s}): {s}\n", .{ 
        @errorName(err),  // 将错误转换为字符串名称
        entry.path 
    });
    continue;
};
```

输出示例：
```
Skipped (MissingTitle): posts/draft.markdown
```

---

## 6. 实战：完整的错误处理流程

### 6.1 main.zig 的错误处理

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();  // 检测内存泄漏
    
    const allocator = gpa.allocator();

    // 可能失败的内存分配
    var args = try std.process.argsWithAllocator(allocator);
    defer args.deinit();

    // 创建 Blogger 并生成网站
    var blogger = Blogger.new(allocator, posts_dir, dest_dir);
    try blogger.generate();  // 任何错误都会向上传播
}
```

### 6.2 Blogger.generate() 的错误处理

```zig
pub fn generate(self: *Blogger) !void {
    try std.fs.cwd().makePath(self.dest_dir);  // 可能失败
    try self.copyStaticFiles();                // 可能失败

    var posts = std.ArrayList(Post){};
    defer {
        for (posts.items) |*post| post.deinit(self.allocator);
        posts.deinit(self.allocator);
    }

    try self.loadPosts(&posts);                // 可能失败
    std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

    for (posts.items) |post| try self.generatePostPage(post);
    try self.generateIndex(posts.items);
    try self.generateAbout();
    try self.generateTagPages(posts.items);
}
```

---

## 7. 错误处理最佳实践

### 7.1 使用 try 传播

在函数内部，使用 `try` 让错误自然传播：

```zig
pub fn process() !void {
    try step1();
    try step2();
    try step3();
}
```

### 7.2 在边界处处理错误

在程序边界（如 main 函数）处理最终错误：

```zig
pub fn main() !void {
    // 错误会在这里终止程序并打印错误信息
    try myFunction();
}
```

### 7.3 使用 catch 提供降级方案

```zig
// 文件不存在时使用默认内容
const content = readFile() catch defaultContent;
```

---

## 8. 小结

| 概念 | 语法 | 用途 |
|------|------|------|
| 可选类型 | `?T` | 表示值可能为 null |
| 错误类型 | `!T` | 表示函数可能失败 |
| try | `try expr` | 传播错误 |
| catch | `expr catch handler` | 捕获处理错误 |
| orelse | `opt orelse default` | 可选值的备选方案 |
| @errorName | `@errorName(err)` | 获取错误名称字符串 |

---

## 相关阅读

- [03-1 变量与类型](./03-1-variables-and-types.md)
- [03-2 重要类型详解](./03-2-important-types.md)
- [09-1 错误类型](../part-09-error-handling/09-1-error-types.md) - 深入错误处理
