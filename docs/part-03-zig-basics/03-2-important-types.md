# 03-2 重要类型详解

> 深入理解 Zig 的核心类型系统

---

## 1. 切片类型 (Slice)

### 1.1 切片是什么

切片是 Zig 中最常用的类型之一，表示连续内存的**视图**：

```zig
const slice: []T = buffer[start..end];
```

切片包含两个字段：
- **指针** - 指向内存起始位置
- **长度** - 切片包含的元素数量

### 1.2 切片在项目中的应用

**字符串处理** - 所有字符串都是 `[]const u8`：

```zig
// Blogger.zig
pub const Blogger = struct {
    dest_dir: []const u8,    // 输出目录路径
    posts_dir: []const u8,   // 文章目录路径
    allocator: Allocator,
    template_engine: TemplateEngine,
};
```

**解析 Frontmatter**：

```zig
fn extractValue(allocator: Allocator, line: []const u8, prefix: []const u8) ![]const u8 {
    // 截取冒号后的值
    return allocator.dupe(u8, std.mem.trim(u8, line[prefix.len..], " \r\n\t\"'"));
}
```

### 1.3 切片操作

```zig
const text = "Hello, Zig!";

// 切片截取
const hello = text[0..5];      // "Hello"
const zig = text[7..10];       // "Zig"

// 使用切片方法
const trimmed = std.mem.trim(u8, text, " !");  // "Hello, Zig"
```

---

## 2. 可选类型 (Optional)

### 2.1 可选类型语法

```zig
const value: ?T = null;  // 可能为 null 的值
```

### 2.2 项目中的可选类型

**Frontmatter 解析** - 字段可能不存在：

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    var title: ?[]const u8 = null;      // 标题可能存在
    var date_time: ?[]const u8 = null;  // 日期可能存在
    var tags: ?[]const u8 = null;       // 标签可能存在

    // 解析过程中赋值
    if (std.mem.startsWith(u8, trimmed, "title:")) {
        title = try extractValue(allocator, trimmed, "title:");
    }

    // 使用 orelse 提供默认值或返回错误
    return Post{
        .title = title orelse return error.MissingTitle,
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",  // 标签可选，默认为空字符串
        // ...
    };
}
```

### 2.3 可选类型的使用模式

| 模式 | 语法 | 说明 |
|------|------|------|
| 捕获 | `if (value) |v| { ... }` | 安全解包 |
| 默认值 | `value orelse default` | 提供备选值 |
| 错误传播 | `value orelse return err` | 返回错误 |
| 断言 | `value.?` | 假设不为 null |

---

## 3. 错误类型 (Error Type)

### 3.1 错误类型基础

```zig
const MyError = error{
    MissingTitle,
    MissingDateTime,
    InvalidFormat,
};
```

### 3.2 项目中的错误类型

**Post 解析错误**：

```zig
pub fn parse(...) !Post {
    const first_line = lines.next() orelse return error.InvalidFormat;
    
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
    }

    return Post{
        .title = title orelse return error.MissingTitle,
        .date_time = date_time orelse return error.MissingDateTime,
        // ...
    };
}
```

### 3.3 错误类型推断

Zig 可以自动推断错误类型：

```zig
pub fn main() !void {  // !void 表示可能返回错误的 void
    // ...
}
```

---

## 4. 结构体 (Struct)

### 4.1 结构体定义

```zig
pub const Post = struct {
    // 字段
    title: []const u8,
    date_time: []const u8,
    tags: []const u8,
    content: []const u8,
    filename: []const u8,

    // 方法
    pub fn deinit(self: *Post, allocator: Allocator) void {
        allocator.free(self.title);
        // ...
    }

    pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
        // ...
    }
};
```

### 4.2 结构体实例化

```zig
const post = Post{
    .title = try allocator.dupe(u8, "My Blog Post"),
    .date_time = try allocator.dupe(u8, "2026-03-21"),
    .tags = try allocator.dupe(u8, "zig blog"),
    .content = try allocator.dupe(u8, "# Hello"),
    .filename = try allocator.dupe(u8, "post.markdown"),
};
```

---

## 5. 枚举 (Enum)

### 5.1 枚举定义

虽然项目中没有大量使用枚举，但枚举是 Zig 的重要类型：

```zig
const FileKind = enum {
    file,
    directory,
    symlink,
};
```

### 5.2 项目中的枚举使用

**文件系统遍历**：

```zig
while (try walker.next()) |entry| {
    if (entry.kind == .file) {  // .file 是 FileKind 枚举值
        // 处理文件
    }
}
```

---

## 6. 数组与列表

### 6.1 固定大小数组

```zig
var buffer: [1024]u8 = undefined;  // 未初始化的缓冲区
var count_buf: [16]u8 = undefined;
```

**项目实例**：

```zig
// 用于数字转字符串的缓冲区
var post_count_buf: [16]u8 = undefined;
const post_count_str = std.fmt.bufPrint(&post_count_buf, "{d}", .{count}) catch "0";
```

### 6.2 动态数组 (ArrayList)

```zig
var posts = std.ArrayList(Post){};
defer {
    for (posts.items) |*post| post.deinit(self.allocator);
    posts.deinit(self.allocator);
}

// 添加元素
try posts.append(allocator, post);
```

---

## 7. 函数类型

### 7.1 函数作为类型

```zig
// 函数指针类型
const Comparator = fn (a: Post, b: Post) bool;
```

### 7.2 项目中的函数类型使用

**排序比较函数**：

```zig
fn sortPostsByDateDesc(a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}

// 使用
std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);
```

---

## 8. 类型总结表

| 类型 | 语法 | 大小 | 项目中的用途 |
|------|------|------|-------------|
| 切片 | `[]T` | 16 字节 | 字符串、路径 |
| 可选 | `?T` | T + 1 字节 | 可选字段 |
| 错误 | `!T` | T + 错误 | 函数返回 |
| 结构体 | `struct {}` | 字段总和 | Post, Blogger |
| 枚举 | `enum {}` | 最小存储 | 文件类型 |
| 数组 | `[N]T` | N * sizeof(T) | 缓冲区 |
| 列表 | `ArrayList(T)` | 动态 | 文章集合 |

---

## 相关阅读

- [03-1 变量与类型](./03-1-variables-and-types.md) - 基础类型介绍
- [03-3 可选类型与错误处理](./03-3-optional-and-error.md) - 深入讲解
- [08-1 内存分配器](../part-08-memory-management/08-1-allocators.md) - 内存管理
