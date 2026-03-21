# 09-3 错误处理模式

> Zig 高级错误处理技巧与项目实例分析

---

## 1. 错误处理模式概览

### 1.1 常见模式分类

| 模式 | 用途 | 关键字 |
|------|------|--------|
| 传播模式 | 将错误传递给调用者 | `try` |
| 恢复模式 | 从错误中恢复继续执行 | `catch` + `continue` |
| 降级模式 | 使用默认值替代 | `catch` default |
| 转换模式 | 转换错误类型 | `catch switch` |
| 包装模式 | 添加错误上下文 | `catch |err|` |

### 1.2 模式选择指南

```
操作失败
    │
    ├─ 致命错误？ → 传播 (try)
    │
    ├─ 可恢复？ → 恢复 (catch + continue)
    │
    ├─ 有默认值？ → 降级 (catch default)
    │
    └─ 需要转换？ → 转换 (catch switch)
```

---

## 2. 传播模式

### 2.1 直接传播

```zig
// 最简单的模式：让调用者处理
fn parse(allocator: Allocator, content: []const u8) !Post {
    // 任何错误都传播给调用者
    const title = try extractTitle(content);
    const date = try extractDate(content);
    
    return Post{
        .title = title,
        .date_time = date,
    };
}
```

**项目实例**：

```zig
pub fn generate(self: *Blogger) !void {
    // 所有错误都传播给 main()
    try std.fs.cwd().makePath(self.dest_dir);
    try self.copyStaticFiles();
    try self.loadPosts(&posts);
    // ...
}
```

### 2.2 传播的优缺点

| 优点 | 缺点 |
|------|------|
| 简单直接 | 调用者需要处理所有错误 |
| 不丢失信息 | 可能导致调用者负担过重 |
| 适合库函数 | 不适合所有场景 |

---

## 3. 恢复模式

### 3.1 循环中的恢复

```zig
// 项目实例：loadPosts()
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    while (try walker.next()) |entry| {
        if (entry.kind == .file) {
            // 解析失败不中断整体流程
            const post = Post.parse(...) catch |err| {
                std.debug.print(" Skipped ({s}): {s}\n", .{ 
                    @errorName(err), entry.path 
                });
                continue;  // 恢复：继续处理下一篇
            };
            try posts.append(self.allocator, post);
        }
    }
}
```

**恢复流程**：

```
遍历文章文件
    │
    ├─ 文件 1: 解析成功 → 添加到列表
    │
    ├─ 文件 2: 解析失败 → 打印错误，continue
    │
    ├─ 文件 3: 解析成功 → 添加到列表
    │
    └─ 文件 4: 解析失败 → 打印错误，continue
            │
            ▼
    循环继续，不中断
```

### 3.2 批量操作中的恢复

```zig
// 通用模式：批量处理中的错误恢复
fn processAll(items: []const Item) void {
    for (items) |item| {
        processOne(item) catch |err| {
            std.debug.print("Failed to process: {}\n", .{err});
            continue;  // 继续处理其他项目
        };
    }
}
```

---

## 4. 降级模式

### 4.1 使用默认值

```zig
// 静态文件可选：不存在时不报错
fn copyStaticFiles(self: *Blogger) !void {
    var dir = std.fs.cwd().openDir("static", .{ .iterate = true }) catch {
        std.debug.print(" Warning: static directory not found\n", .{});
        return;  // 降级：没有静态文件也能运行
    };
    defer dir.close();
    
    // ... 复制逻辑
}
```

**降级策略**：

```
尝试打开 static/ 目录
    │
    ├─ 成功 → 复制静态文件
    │
    └─ 失败 → 打印警告，返回
            │
            ▼
        程序继续运行（无静态文件）
```

### 4.2 可选功能降级

```zig
// 关于页面可选
fn generateAbout(self: *Blogger) !void {
    const file = std.fs.cwd().openFile(about_path, .{}) catch {
        std.debug.print(" Skipped: about.markdown not found\n", .{});
        return;  // 降级：没有关于页面也能运行
    };
    defer file.close();
    
    // ... 生成关于页面
}
```

---

## 5. 转换模式

### 5.1 错误类型转换

```zig
// 将底层错误转换为业务错误
fn loadPost(path: []const u8) !Post {
    const content = std.fs.cwd().readFile(path) catch |err| switch (err) {
        error.FileNotFound => return error.PostNotFound,
        error.PermissionDenied => return error.CannotReadPost,
        error.IsDir => return error.InvalidPath,
        else => |e| return e,  // 其他错误原样传播
    };
    return Post.parse(..., content, ...);
}
```

**错误映射**：

```
文件系统错误              业务逻辑错误
    │                          │
    ├─ FileNotFound      →    PostNotFound
    ├─ PermissionDenied  →    CannotReadPost
    ├─ IsDir             →    InvalidPath
    └─ 其他              →    原样传播
```

### 5.2 添加错误上下文

```zig
// 包装错误添加详细信息
fn processPosts() !void {
    loadPosts() catch |err| {
        std.debug.print("Failed to load posts from '{s}': {}\n", .{
            self.posts_dir,
            err,
        });
        return error.LoadFailed;
    };
}
```

---

## 6. 组合模式

### 6.1 try + catch 组合

```zig
// 大部分操作使用 try，特定操作使用 catch
fn generate(self: *Blogger) !void {
    // 关键操作：失败则停止
    try std.fs.cwd().makePath(self.dest_dir);
    try self.loadPosts(&posts);
    
    // 可选操作：失败则跳过
    self.copyStaticFiles() catch {
        std.debug.print(" Warning: skipping static files\n", .{});
    };
    
    // 继续生成页面
    for (posts.items) |post| {
        try self.generatePostPage(post);
    }
}
```

### 6.2 分层错误处理

```zig
// 不同层次使用不同策略
fn main() !void {
    // 顶层：传播所有错误
    try run();
}

fn run() !void {
    // 中间层：部分恢复
    loadPosts() catch |err| {
        std.debug.print("Warning: {s}\n", .{@errorName(err)});
        // 但不返回错误，继续执行
    };
    
    // 关键操作：传播错误
    try generatePages();
}

fn loadPosts() !void {
    // 底层：详细错误处理
    while (...) |entry| {
        Post.parse(...) catch |err| {
            // 记录并跳过
            continue;
        };
    }
}
```

---

## 7. 项目中的错误处理分析

### 7.1 main.zig 错误处理

```zig
pub fn main() !void {
    // [1] 分配器设置（不处理错误）
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // [2] 参数解析（部分恢复）
    var args = try std.process.argsWithAllocator(allocator);
    defer args.deinit();
    _ = args.skip();

    var posts_dir: []const u8 = "posts";
    var dest_dir: []const u8 = "build";

    while (args.next()) |arg| {
        if (std.mem.eql(u8, arg, "--posts") or std.mem.eql(u8, arg, "-p")) {
            posts_dir = args.next() orelse {
                // 参数缺失：错误退出
                std.debug.print("Error: --posts requires a directory argument\n", .{});
                std.process.exit(1);
            };
        } else if (std.mem.eql(u8, arg, "--help") or std.mem.eql(u8, arg, "-h")) {
            printHelp();
            return;  // 正常退出
        }
    }

    // [3] 博客生成（传播错误）
    var blogger = Blogger.new(allocator, posts_dir, dest_dir);
    try blogger.generate();
    //  ^^^ 任何错误都会导致程序终止
}
```

**错误处理策略**：

| 位置 | 策略 | 理由 |
|------|------|------|
| 分配器 | 无处理 | 分配失败应终止 |
| 参数解析 | orelse 退出 | 参数错误无法继续 |
| 博客生成 | try 传播 | 生成失败应报告 |

### 7.2 Blogger.generate() 错误处理

```zig
pub fn generate(self: *Blogger) !void {
    std.debug.print("Generating blog...\n", .{});
    
    // [1] 创建目录（传播）
    try std.fs.cwd().makePath(self.dest_dir);

    // [2] 复制静态文件（传播，但内部有降级）
    try self.copyStaticFiles();
    // copyStaticFiles 内部 catch 处理目录不存在

    // [3] 初始化文章列表
    var posts = std.ArrayList(Post){};
    defer {
        for (posts.items) |*post| post.deinit(self.allocator);
        posts.deinit(self.allocator);
    }

    // [4] 加载文章（传播，但内部有恢复）
    try self.loadPosts(&posts);
    // loadPosts 内部 catch 跳过错文

    // [5] 排序（不失败）
    std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

    // [6] 生成页面（传播）
    for (posts.items) |post| try self.generatePostPage(post);
    try self.generateIndex(posts.items);
    try self.generateAbout();
    try self.generateTagPages(posts.items);
}
```

**分层错误处理**：

```
generate()
    │
    ├─ copyStaticFiles()
    │   └─ catch: 目录不存在 → 警告并返回
    │
    ├─ loadPosts()
    │   └─ catch: 解析失败 → 跳过文章
    │
    └─ generatePostPage()
        └─ try: 失败 → 传播并停止
```

### 7.3 Post.parse() 错误处理

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // [1] 验证输入
    var lines = std.mem.splitScalar(u8, content, '\n');
    const first_line = lines.next() orelse return error.InvalidFormat;
    //                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                  空文件 → 错误

    // [2] 验证 frontmatter
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
        //     ^^^^^^^^^^^^^^^^^^^
        //         格式错误 → 错误
    }

    // [3] 解析字段
    var title: ?[]const u8 = null;
    var date_time: ?[]const u8 = null;
    var tags: ?[]const u8 = null;

    while (lines.next()) |line| {
        const trimmed = std.mem.trim(u8, line, " \r\n\t");
        if (std.mem.eql(u8, trimmed, "---")) break;
        
        if (std.mem.startsWith(u8, trimmed, "title:")) {
            title = try extractValue(allocator, trimmed, "title:");
            //    ^^^ 内存分配失败 → 传播
        } else if (std.mem.startsWith(u8, trimmed, "date_time:")) {
            date_time = try extractValue(allocator, trimmed, "date_time:");
        } else if (std.mem.startsWith(u8, trimmed, "tags:")) {
            tags = try extractValue(allocator, trimmed, "tags:");
        }
    }

    // [4] 验证必需字段
    return Post{
        .title = title orelse return error.MissingTitle,
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     缺少 title → 错误
        .date_time = date_time orelse return error.MissingDateTime,
        //         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //         缺少 date_time → 错误
        .tags = tags orelse "",
        //   ^^^^^^^^^^^^^^^^
        //   缺少 tags → 使用默认值
        .content = try allocator.dupe(u8, content[content_start..]),
        .filename = try allocator.dupe(u8, filename),
    };
}
```

**错误分类**：

| 错误类型 | 处理方式 | 理由 |
|----------|----------|------|
| 空文件 | orelse 返回 | 无法解析 |
| 格式错误 | return error | 无效输入 |
| 内存失败 | try 传播 | 系统问题 |
| 缺少 title | orelse 返回 | 必需字段 |
| 缺少 tags | orelse 默认 | 可选字段 |

---

## 8. 错误处理最佳实践

### 8.1 选择合适的模式

```
决策树:
    │
    ├─ 错误是否致命？
    │   ├─ 是 → try 传播
    │   └─ 否 → 继续判断
    │
    ├─ 是否有默认值？
    │   ├─ 是 → catch default
    │   └─ 否 → 继续判断
    │
    ├─ 是否可以跳过？
    │   ├─ 是 → catch + continue
    │   └─ 否 → 继续判断
    │
    └─ 是否需要转换？
        ├─ 是 → catch switch
        └─ 否 → try 传播
```

### 8.2 错误信息清晰

```zig
// 推荐：清晰的错误信息
std.debug.print(" Skipped ({s}): {s}\n", .{ 
    @errorName(err),  // 错误类型
    entry.path        // 相关文件
});

// 不推荐：模糊的错误信息
std.debug.print("Error!\n", .{});
```

### 8.3 资源管理配合

```zig
// 确保错误处理时资源正确释放
fn process() !void {
    var resource = try acquire();
    defer release(resource);  // 即使错误也会释放
    
    try doSomething();  // 失败时 defer 仍执行
}
```

---

## 9. 常见错误处理反模式

### 9.1 静默失败

```zig
// 错误：忽略错误
operation() catch {};

// 正确：至少记录
operation() catch |err| {
    std.debug.print("Warning: {}\n", .{err});
};
```

### 9.2 过度包装

```zig
// 不推荐：层层包装
fn a() !void {
    b() catch |err| return error.Wrap1;
}
fn b() !void {
    c() catch |err| return error.Wrap2;
}
fn c() !void {
    return error.Original;
}

// 推荐：直接传播
fn a() !void {
    try b();
}
fn b() !void {
    try c();
}
fn c() !void {
    return error.Original;
}
```

### 9.3 错误吞没

```zig
// 错误：吞没关键错误
try mayFail() catch {};  // 语法允许，但逻辑错误

// 正确：根据情况选择
try mayFail();           // 传播
mayFail() catch handle(); // 处理
```

---

## 10. 总结

### 10.1 模式对比

| 模式 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| 传播 | 致命错误 | 简单 | 调用者负担 |
| 恢复 | 可跳过错误 | 健壮 | 可能丢失数据 |
| 降级 | 可选功能 | 灵活 | 功能不完整 |
| 转换 | 抽象边界 | 清晰接口 | 额外代码 |

### 10.2 项目中的模式应用

| 位置 | 模式 | 实现 |
|------|------|------|
| `main()` | 传播 | `try blogger.generate()` |
| `copyStaticFiles()` | 降级 | `catch { return; }` |
| `loadPosts()` | 恢复 | `catch |err| continue` |
| `generateAbout()` | 降级 | `catch { return; }` |
| `Post.parse()` | 传播 | `try`, `orelse return` |

---

## 相关文档

- [09-1 错误类型](./09-1-error-types.md) - 错误类型基础
- [09-2 错误传播](./09-2-error-propagation.md) - try/catch 详解
- [08-3 defer 模式](../part-08-memory-management/08-3-defer-patterns.md) - 错误与资源管理
