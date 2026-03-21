# 09-1 错误类型 (Error Types)

> Zig 错误处理机制与项目实例分析

---

## 1. 错误类型基础

### 1.1 什么是错误类型

在 Zig 中，错误是一种**第一类公民**（first-class citizen），用于表示操作失败。

**基本语法**：

```zig
// 错误集定义
const MyError = error{
    FileNotFound,
    PermissionDenied,
    OutOfMemory,
};

// 单个错误
const err = error.FileNotFound;
```

### 1.2 错误集

错误集是一组相关错误的集合：

```zig
// 显式定义错误集
const ParseError = error{
    InvalidFormat,
    MissingField,
    OutOfMemory,
};

// 推断错误集
fn parse() !void {
    // 编译器自动推断可能的错误
}
```

---

## 2. 项目中的错误类型

### 2.1 Post.parse() 的错误

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    var lines = std.mem.splitScalar(u8, content, '\n');
    const first_line = lines.next() orelse return error.InvalidFormat;
    //                                        ^^^^^^^^^^^^^^^^^^^^^^^^
    //                                        返回错误
    
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
        //     ^^^^^^^^^^^^^^^^^^^
        //     frontmatter 格式错误
    }

    // ... 解析逻辑

    return Post{
        .title = title orelse return error.MissingTitle,
        //                       ^^^^^^^^^^^^^^^^^^^^^^
        //                       缺少 title 字段
        .date_time = date_time orelse return error.MissingDateTime,
        //                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //                         缺少 date_time 字段
        .tags = tags orelse "",
        .content = try allocator.dupe(u8, content[content_start..]),
        //         ^^^ 可能返回 error.OutOfMemory
        .filename = try allocator.dupe(u8, filename),
        //         ^^^ 可能返回 error.OutOfMemory
    };
}
```

**Post.parse() 可能返回的错误**：

| 错误 | 触发条件 | 严重性 |
|------|----------|--------|
| `error.InvalidFormat` | frontmatter 格式错误 | 高 |
| `error.MissingTitle` | 缺少 title 字段 | 高 |
| `error.MissingDateTime` | 缺少 date_time 字段 | 高 |
| `error.OutOfMemory` | 内存分配失败 | 致命 |

### 2.2 错误传播链

```
main() !void
    │
    └─> Blogger.generate() !void
            │
            ├─> Post.parse() !Post
            │       ├─ error.InvalidFormat
            │       ├─ error.MissingTitle
            │       ├─ error.MissingDateTime
            │       └─ error.OutOfMemory
            │
            ├─> std.fs.cwd().makePath() !void
            │       └─ error.PathAlreadyExists, etc.
            │
            └─> markdown.toHtml() ![]const u8
                    └─ error.OutOfMemory
```

---

## 3. 错误联合类型

### 3.1 基本语法

```zig
// 错误联合类型
fn operation() !void {
    // 可能返回错误或 void
}

// 带具体返回类型
fn getValue() !i32 {
    // 可能返回错误或 i32
}
```

### 3.2 项目中的错误联合

```zig
// main.zig
pub fn main() !void {
    // 可能返回任何错误
}

// Blogger.zig
pub fn generate(self: *Blogger) !void {
    // 可能返回文件操作、内存分配等错误
}

pub fn parse(allocator: Allocator, ...) !Post {
    // 可能返回解析错误或内存错误
}
```

### 3.3 错误类型推断

```zig
// 编译器自动推断错误集
fn parse() !Post {
    return error.InvalidFormat;  // 推断包含此错误
}

// 等价于显式声明
fn parse() error{InvalidFormat, OutOfMemory}!Post {
    return error.InvalidFormat;
}
```

---

## 4. 标准库错误集

### 4.1 常见错误

Zig 标准库定义了通用错误集：

```zig
// std.fs 文件操作错误
error{
    FileNotFound,      // 文件不存在
    PathAlreadyExists, // 路径已存在
    NotDir,            // 不是目录
    IsDir,             // 是目录
    PermissionDenied,  // 权限不足
    OutOfMemory,       // 内存不足
}
```

### 4.2 项目中的标准库错误

```zig
// 文件操作可能返回的错误
const file = try std.fs.cwd().openFile(about_path, .{});
//                                  ^^^^^^^^^^^^^^^^^^^
// 可能返回：
// - error.FileNotFound
// - error.PermissionDenied
// - error.IsDir

// 目录创建可能返回的错误
try std.fs.cwd().makePath(self.dest_dir);
//  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
// 可能返回：
// - error.PathAlreadyExists
// - error.PermissionDenied
// - error.DiskQuota
```

---

## 5. 错误定义与使用

### 5.1 定义自定义错误

```zig
// 在模块级别定义错误集
pub const ParseError = error{
    InvalidFormat,
    MissingTitle,
    MissingDateTime,
    EmptyContent,
};

// 使用错误集
pub fn parse() ParseError!Post {
    // ...
}
```

### 5.2 合并错误集

```zig
// 合并多个错误集
const AllErrors = ParseError || std.fs.File.OpenError || std.mem.Allocator.Error;

// 使用合并的错误集
pub fn parseFile(allocator: Allocator, path: []const u8) AllErrors!Post {
    // ...
}
```

### 5.3 anyerror 类型

```zig
// anyerror 表示任何错误类型
fn mayFail(flag: bool) anyerror!void {
    if (flag) return error.SomeError;
}

// 项目中较少使用，通常让编译器推断
```

---

## 6. 错误比较与转换

### 6.1 错误比较

```zig
// 错误可以比较
if (err == error.FileNotFound) {
    std.debug.print("File not found\n", .{});
}

// 错误集成员测试
const is_fs_error = @as(anyerror, err) == error.FileNotFound;
```

### 6.2 错误名称

```zig
// 获取错误名称（字符串）
const err = error.FileNotFound;
const name = @errorName(err);  // "FileNotFound"

// 项目中的使用
const post = Post.parse(...) catch |err| {
    std.debug.print(" Skipped ({s}): {s}\n", .{ 
        @errorName(err),  // 错误名称
        entry.path 
    });
    continue;
};
```

---

## 7. 错误处理模式

### 7.1 立即返回错误

```zig
// 最简单的错误处理
fn parse() !Post {
    if (invalid) return error.InvalidFormat;
    //         ^^^^^^ 立即返回
}
```

### 7.2 错误转换

```zig
// 将一种错误转换为另一种
fn parse() !Post {
    someOperation() catch |err| switch (err) {
        error.FileNotFound => return error.MissingFile,
        error.PermissionDenied => return error.CannotRead,
        else => |e| return e,
    };
}
```

### 7.3 错误包装

```zig
// 包装错误添加上下文
fn loadPost(path: []const u8) !Post {
    const content = std.fs.cwd().readFile(path) catch |err| {
        std.debug.print("Failed to load {s}: {}\n", .{ path, err });
        return error.LoadFailed;
    };
    return Post.parse(..., content, ...);
}
```

---

## 8. 完整实例分析

### 8.1 Post.parse() 错误处理

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // [1] 验证输入非空
    if (content.len == 0) return error.EmptyContent;
    
    // [2] 分割行
    var lines = std.mem.splitScalar(u8, content, '\n');
    const first_line = lines.next() orelse return error.InvalidFormat;
    //                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                  如果没有行，返回错误
    
    // [3] 验证 frontmatter 起始符
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
        //     ^^^^^^^^^^^^^^^^^^^
        //     格式错误
    }

    // [4] 解析字段
    var title: ?[]const u8 = null;
    var date_time: ?[]const u8 = null;
    var tags: ?[]const u8 = null;

    while (lines.next()) |line| {
        const trimmed = std.mem.trim(u8, line, " \r\n\t");
        if (std.mem.eql(u8, trimmed, "---")) break;
        
        if (std.mem.startsWith(u8, trimmed, "title:")) {
            title = try extractValue(allocator, trimmed, "title:");
            //    ^^^ 如果分配失败，返回 error.OutOfMemory
        } else if (std.mem.startsWith(u8, trimmed, "date_time:")) {
            date_time = try extractValue(allocator, trimmed, "date_time:");
        } else if (std.mem.startsWith(u8, trimmed, "tags:")) {
            tags = try extractValue(allocator, trimmed, "tags:");
        }
    }

    // [5] 验证必需字段
    return Post{
        .title = title orelse return error.MissingTitle,
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     如果 title 为 null，返回错误
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",
        //   ^^^^^^^^^^^^^^^^
        //   tags 可选，使用默认值
        .content = try allocator.dupe(u8, content[content_start..]),
        .filename = try allocator.dupe(u8, filename),
    };
}
```

### 8.2 loadPosts() 错误恢复

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    var dir = try std.fs.cwd().openDir(self.posts_dir, .{ .iterate = true });
    defer dir.close();

    var walker = try dir.walk(self.allocator);
    defer walker.deinit();

    while (try walker.next()) |entry| {
        if (entry.kind == .file and std.mem.endsWith(u8, entry.path, ".markdown")) {
            // 跳过 about.markdown
            if (std.mem.eql(u8, std.fs.path.basename(entry.path), "about.markdown")) {
                continue;
            }

            const file = try dir.openFile(entry.path, .{});
            defer file.close();
            const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);

            // 错误恢复：解析失败不中断整体流程
            const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| {
                std.debug.print(" Skipped ({s}): {s}\n", .{ 
                    @errorName(err),  // 打印错误名称
                    entry.path 
                });
                continue;  // 跳过坏文章
            };
            
            try posts.append(self.allocator, post);
            std.debug.print(" Loaded: {s}\n", .{entry.path});
        }
    }
}
```

**错误恢复策略**：

```
Post.parse() 失败
    │
    ├─ catch |err| 捕获错误
    │
    ├─ @errorName(err) 获取错误名称
    │
    ├─ 打印诊断信息
    │
    └─ continue 继续处理下一篇文章
```

---

## 9. 错误类型最佳实践

### 9.1 使用有意义的错误

```zig
// 推荐：具体错误
return error.MissingTitle;
return error.InvalidFormat;

// 不推荐：模糊错误
return error.SomeError;
return error.Failure;
```

### 9.2 错误命名约定

```zig
// 使用 PascalCase
error.FileNotFound
error.InvalidFormat
error.OutOfMemory

// 错误名称应描述问题
error.MissingTitle      // ✓ 好
error.TitleError        // ✗ 模糊
error.Bad               // ✗ 太模糊
```

### 9.3 错误粒度

```zig
// 推荐：适度粒度
error{
    FileNotFound,
    PermissionDenied,
    InvalidFormat,
}

// 不推荐：过度细分
error{
    FileNotFoundLevel1,
    FileNotFoundLevel2,
    // ...
}
```

---

## 10. 注意事项

### 10.1 错误泄漏

```zig
// 错误：忘记处理错误
fn leak() void {
    someOperation();  // 错误被忽略
}

// 正确：使用 try 或 catch
fn noLeak() !void {
    try someOperation();  // 传播错误
}
```

### 10.2 错误吞没

```zig
// 不推荐：吞没错误无提示
someOperation() catch {};  // 静默失败

// 推荐：至少记录错误
someOperation() catch |err| {
    std.debug.print("Warning: {}\n", .{err});
};
```

### 10.3 错误与可选类型

```zig
// 错误 vs 可选类型
fn find(id: u32) !Item    // 可能失败
fn find(id: u32) ?Item    // 可能不存在

// 选择指南：
// - 使用 !T 当操作可能失败（文件、网络、解析）
// - 使用 ?T 当值可能不存在（查找、搜索）
```

---

## 相关文档

- [09-2 错误传播](./09-2-error-propagation.md) - try/catch 详解
- [09-3 错误处理模式](./09-3-error-handling-patterns.md) - 高级错误处理
- [05-1 函数基础](../part-05-functions/05-1-function-basics.md) - 错误返回类型
