# 09-2 错误传播 (Error Propagation)

> Zig try/catch 机制与项目实例分析

---

## 1. 错误传播基础

### 1.1 什么是错误传播

错误传播是指将函数内部的错误传递给调用者处理的机制。

**Zig 的错误传播方式**：

| 方式 | 关键字 | 用途 |
|------|--------|------|
| try | `try` | 传播错误给调用者 |
| catch | `catch` | 本地处理错误 |
| orelse | `orelse` | 处理可选类型 |

### 1.2 try 关键字

**基本语法**：

```zig
// try 用于传播错误
fn caller() !void {
    try mayFail();  // 如果失败，立即返回错误
}

fn mayFail() !void {
    return error.SomeError;
}
```

**执行流程**：

```
try mayFail()
    │
    ├─ 成功 → 继续执行
    │
    └─ 失败 → 立即返回错误
            │
            ▼
        caller() 返回相同错误
```

---

## 2. try 在项目中的使用

### 2.1 main.zig 中的 try

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // [1] 获取命令行参数
    var args = try std.process.argsWithAllocator(allocator);
    //         ^^^ 如果内存分配失败，main() 立即返回
    defer args.deinit();
    _ = args.skip();

    // ... 参数解析

    // [2] 创建并运行 Blogger
    var blogger = Blogger.new(allocator, posts_dir, dest_dir);
    try blogger.generate();
    //  ^^^ 如果生成失败，main() 立即返回
}
```

**错误传播链**：

```
blogger.generate() 失败
    │
    ▼
try 传播错误
    │
    ▼
main() 返回错误
    │
    ▼
Zig 运行时转换为进程退出码
```

### 2.2 Blogger.generate() 中的 try

```zig
pub fn generate(self: *Blogger) !void {
    std.debug.print("Generating blog...\n", .{});
    
    // [1] 创建输出目录
    try std.fs.cwd().makePath(self.dest_dir);
    //  ^^^ 如果创建失败（权限不足等），立即返回

    // [2] 复制静态文件
    try self.copyStaticFiles();
    //  ^^^ 传播 copyStaticFiles 的错误

    // [3] 加载文章
    var posts = std.ArrayList(Post){};
    try self.loadPosts(&posts);
    //  ^^^ 传播加载错误

    // [4] 排序文章
    std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

    // [5] 生成页面
    for (posts.items) |post| try self.generatePostPage(post);
    //                       ^^^ 每个文章生成失败都会传播
    
    try self.generateIndex(posts.items);
    try self.generateAbout();
    try self.generateTagPages(posts.items);
    //  ^^^ 任何一步失败都会停止后续操作
}
```

**try 的短路行为**：

```
generate() 执行流程:
    │
    ├─ try makePath() ✓
    │
    ├─ try copyStaticFiles() ✓
    │
    ├─ try loadPosts() ✓
    │
    ├─ try generatePostPage(post1) ✓
    │
    ├─ try generatePostPage(post2) ✗ 失败！
    │   │
    │   ▼
    │   立即返回，不再执行后续
    │
    └─ try generateIndex() ← 不会执行
```

### 2.3 Post.parse() 中的 try

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // ... 验证逻辑

    while (lines.next()) |line| {
        if (std.mem.startsWith(u8, trimmed, "title:")) {
            title = try extractValue(allocator, trimmed, "title:");
            //    ^^^ 如果内存分配失败，立即返回
        } else if (std.mem.startsWith(u8, trimmed, "date_time:")) {
            date_time = try extractValue(allocator, trimmed, "date_time:");
        } else if (std.mem.startsWith(u8, trimmed, "tags:")) {
            tags = try extractValue(allocator, trimmed, "tags:");
        }
    }

    return Post{
        .title = title orelse return error.MissingTitle,
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",
        .content = try allocator.dupe(u8, content[content_start..]),
        //         ^^^ 内存分配失败时返回 error.OutOfMemory
        .filename = try allocator.dupe(u8, filename),
        //         ^^^ 内存分配失败时返回 error.OutOfMemory
    };
}
```

---

## 3. catch 关键字

### 3.1 catch 基本用法

```zig
// 提供默认值
const value = mayFail() catch defaultValue;

// 执行代码块
const value = mayFail() catch {
    std.debug.print("Failed!\n", .{});
    defaultValue;
};

// 捕获错误
mayFail() catch |err| {
    std.debug.print("Error: {}\n", .{err});
};
```

### 3.2 项目中的 catch 使用

#### 优雅降级

```zig
fn copyStaticFiles(self: *Blogger) !void {
    const static_dir = "static";
    var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
        // 静态目录不存在时，优雅降级
        std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
        return;  // 不中断程序
    };
    defer dir.close();
    
    // ... 复制逻辑
}
```

**catch vs try**：

```zig
// try - 传播错误
try std.fs.cwd().openDir(path, .{});
// 失败 → 函数返回错误

// catch - 本地处理
std.fs.cwd().openDir(path, .{}) catch {
    // 本地处理，函数可能继续执行
};
```

#### 错误恢复

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    // ... 目录遍历

    while (try walker.next()) |entry| {
        if (entry.kind == .file) {
            // ... 读取文件

            // 错误恢复：解析失败不中断整体
            const post = Post.parse(self.allocator, content, filename) catch |err| {
                std.debug.print(" Skipped ({s}): {s}\n", .{ 
                    @errorName(err), 
                    entry.path 
                });
                continue;  // 跳过坏文章
            };
            
            try posts.append(self.allocator, post);
        }
    }
}
```

**错误恢复流程**：

```
Post.parse() 失败
    │
    ▼
catch |err| 捕获错误
    │
    ├─ @errorName(err) 获取错误名称
    │
    ├─ std.debug.print 打印诊断
    │
    └─ continue 继续下一次循环
            │
            ▼
        处理下一篇文章
```

#### 生成关于页面

```zig
fn generateAbout(self: *Blogger) !void {
    const about_path = try std.fs.path.join(self.allocator, &.{ 
        self.posts_dir, 
        "about.markdown" 
    });
    defer self.allocator.free(about_path);

    // 关于页面可选：不存在时不报错
    const file = std.fs.cwd().openFile(about_path, .{}) catch {
        std.debug.print(" Skipped: about.markdown not found\n", .{});
        return;  // 优雅返回
    };
    defer file.close();

    // ... 生成关于页面
}
```

---

## 4. catch 与错误集

### 4.1 捕获特定错误

```zig
operation() catch |err| switch (err) {
    error.FileNotFound => {
        std.debug.print("File not found\n", .{});
        return error.MissingFile;
    },
    error.PermissionDenied => {
        std.debug.print("Permission denied\n", .{});
        return error.CannotAccess;
    },
    else => |e| {
        std.debug.print("Unknown error: {}\n", .{e});
        return e;
    },
};
```

### 4.2 项目中的错误分类

```zig
// 改进的 loadPosts 错误处理
const post = Post.parse(...) catch |err| switch (err) {
    error.InvalidFormat => {
        std.debug.print(" Skipped (invalid format): {s}\n", .{entry.path});
        continue;
    },
    error.MissingTitle => {
        std.debug.print(" Skipped (no title): {s}\n", .{entry.path});
        continue;
    },
    error.MissingDateTime => {
        std.debug.print(" Skipped (no date): {s}\n", .{entry.path});
        continue;
    },
    error.OutOfMemory => {
        // 内存错误需要立即处理
        return error.OutOfMemory;
    },
    else => |e| {
        std.debug.print(" Skipped (unknown: {s}): {s}\n", .{ 
            @errorName(e), entry.path 
        });
        continue;
    },
};
```

---

## 5. orelse 与错误传播

### 5.1 orelse 基础

```zig
// orelse 用于可选类型
const value: ?i32 = null;
const result = value orelse 0;  // null 时使用默认值

// orelse 返回错误
const required = value orelse return error.MissingValue;
```

### 5.2 项目中的 orelse

```zig
pub fn parse(...) !Post {
    // [1] 获取第一行
    const first_line = lines.next() orelse return error.InvalidFormat;
    //                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                  如果没有第一行，返回错误

    // [2] 验证必需字段
    return Post{
        .title = title orelse return error.MissingTitle,
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     title 为 null 时返回错误
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",
        //   ^^^^^^^^^^^^^^^^
        //   tags 为 null 时使用空字符串
    };
}
```

### 5.3 orelse 代码块

```zig
// orelse 后面可以跟代码块
posts_dir = args.next() orelse {
    std.debug.print("Error: --posts requires a directory argument\n", .{});
    std.process.exit(1);
};
```

**执行流程**：

```
args.next()
    │
    ├─ 返回 Some(value) → 赋值给 posts_dir
    │
    └─ 返回 null → 执行 orelse 块
        │
        ├─ 打印错误信息
        │
        └─ 退出程序 (exit code 1)
```

---

## 6. 错误传播模式

### 6.1 直接传播

```zig
// 最简单的模式：使用 try 直接传播
fn caller() !void {
    try mayFail();
}

fn mayFail() !void {
    return error.SomeError;
}
```

### 6.2 错误转换

```zig
// 将底层错误转换为业务错误
fn loadPost(path: []const u8) !Post {
    const content = std.fs.cwd().readFile(path) catch |err| switch (err) {
        error.FileNotFound => return error.PostNotFound,
        error.PermissionDenied => return error.CannotReadPost,
        else => |e| return e,
    };
    return Post.parse(..., content, ...);
}
```

### 6.3 错误包装

```zig
// 包装错误添加上下文
fn processPosts() !void {
    loadPosts() catch |err| {
        std.debug.print("Failed to load posts: {}\n", .{err});
        return error.LoadFailed;
    };
}
```

### 6.4 部分错误处理

```zig
// 只处理特定错误，传播其他
fn process() !void {
    operation() catch |err| {
        if (err == error.FileNotFound) {
            std.debug.print("Using default\n", .{});
            return;
        }
        // 其他错误继续传播
        return err;
    };
}
```

---

## 7. 完整实例分析

### 7.1 generatePostPage() 错误处理

```zig
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] Markdown → HTML
    const html_content = try markdown.toHtml(self.allocator, post.content);
    //                   ^^^ 如果转换失败，立即返回
    defer self.allocator.free(html_content);

    // [2] 初始化上下文
    var ctx = Context.init(self.allocator);
    defer ctx.deinit();
    
    try ctx.set("title", post.title);
    //  ^^^ 如果设置失败，立即返回
    try ctx.set("date_time", post.date_time);
    try ctx.set("content", html_content);

    // [3] 构建标签 HTML
    var tags_html = std.ArrayList(u8){};
    defer tags_html.deinit(self.allocator);
    
    var tag_iter = std.mem.tokenizeScalar(u8, post.tags, ' ');
    while (tag_iter.next()) |tag| {
        try tags_html.appendSlice(self.allocator, "<a href=\"/tags/" ++ tag ++ "\">" ++ tag ++ "</a>");
        //  ^^^ 如果追加失败，立即返回
    }
    try ctx.set("tags", tags_html.items);

    // [4] 渲染模板
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    //                  ^^^ 如果渲染失败，立即返回
    defer self.allocator.free(page_html);

    // [5] 设置页面标题
    var full_title = std.ArrayList(u8){};
    defer full_title.deinit(self.allocator);
    try full_title.appendSlice(self.allocator, post.title);
    try full_title.appendSlice(self.allocator, " - ");
    try ctx.set("page_title", full_title.items);

    // [6] 渲染最终 HTML
    self.template_engine.setPageContent(page_html);
    const html = try self.template_engine.render("layout.hbs", &ctx);
    //             ^^^ 如果渲染失败，立即返回
    defer self.allocator.free(html);

    // [7] 写入文件
    try self.writeHtmlFileWithDate(post.filename, post.date_time, html);
    //  ^^^ 如果写入失败，立即返回
}
```

### 7.2 错误传播链

```
generatePostPage() 可能的错误点:
    │
    ├─ markdown.toHtml() → error.OutOfMemory
    │
    ├─ ctx.set() → error.OutOfMemory
    │
    ├─ tags_html.appendSlice() → error.OutOfMemory
    │
    ├─ template_engine.render() → error.OutOfMemory, error.TemplateNotFound
    │
    ├─ full_title.appendSlice() → error.OutOfMemory
    │
    └─ writeHtmlFileWithDate() → error.PathAlreadyExists, error.PermissionDenied
            │
            ▼
    所有错误都通过 try 传播给 generate()
```

---

## 8. 注意事项

### 8.1 try 的位置

```zig
// 推荐：try 靠近函数调用
const value = try mayFail();

// 不推荐：try 在表达式末尾
const value = mayFail() try + 1;  // 语法错误！
```

### 8.2 catch 的滥用

```zig
// 不推荐：忽略所有错误
operation() catch {};  // 静默失败

// 推荐：至少记录错误
operation() catch |err| {
    std.debug.print("Warning: {}\n", .{err});
};
```

### 8.3 错误传播与资源管理

```zig
// 注意：try 之前的资源需要正确管理
fn process() !void {
    var resource = try acquireResource();
    defer releaseResource(resource);
    
    try doSomething();  // 如果失败，defer 仍然执行
}
```

---

## 相关文档

- [09-1 错误类型](./09-1-error-types.md) - 错误类型基础
- [09-3 错误处理模式](./09-3-error-handling-patterns.md) - 高级错误处理
- [04-1 条件语句](../part-04-control-flow/04-1-if-else.md) - orelse 运算符
