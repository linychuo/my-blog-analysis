# 03-1 变量与类型

> Zig 语言的变量声明与基础类型系统

---

## 1. 变量声明

### 1.1 不可变变量 (const)

在 Zig 中，默认情况下变量是不可变的，使用 `const` 关键字声明：

```zig
const name: []const u8 = "my-blog";
const version: u8 = 1;
```

**项目实例** - `build.zig` 中的常量声明：

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
}
```

### 1.2 可变变量 (var)

当需要修改变量值时，使用 `var` 关键字：

```zig
var counter: u32 = 0;
counter += 1; // ✓ 允许修改
```

**项目实例** - `main.zig` 中的可变变量：

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var args = try std.process.argsWithAllocator(allocator);
    defer args.deinit();
    _ = args.skip();

    var posts_dir: []const u8 = "posts";  // 可变，默认值
    var dest_dir: []const u8 = "build";

    // 根据命令行参数修改
    while (args.next()) |arg| {
        if (std.mem.eql(u8, arg, "--posts")) {
            posts_dir = args.next() orelse return error.MissingArg;
        }
    }
}
```

---

## 2. 类型系统

### 2.1 基础类型

| 类型 | 描述 | 项目中的使用 |
|------|------|-------------|
| `void` | 空类型，无值 | 函数无返回值 |
| `bool` | 布尔类型 | 条件判断 |
| `i8`, `i16`, `i32`, `i64` | 有符号整数 | 数值计算 |
| `u8`, `u16`, `u32`, `u64` | 无符号整数 | 索引、计数 |
| `f32`, `f64` | 浮点数 | - |
| `usize` | 指针大小整数 | 数组长度、索引 |
| `isize` | 有符号指针大小 | - |

### 2.2 重要类型

#### `[]const u8` - 字符串切片

Zig 中没有内置字符串类型，使用字节切片表示字符串：

```zig
const title: []const u8 = "Hello Zig";
const content: []const u8 = post.content;  // 博客内容
```

**项目中的广泛应用**：

```zig
// Blogger.zig
pub const Post = struct {
    title: []const u8,      // 标题
    date_time: []const u8,  // 日期时间
    tags: []const u8,       // 标签
    content: []const u8,    // Markdown 内容
    filename: []const u8,   // 文件名
};
```

#### `[]T` - 切片类型

切片是 Zig 中最常用的类型之一，表示连续内存的视图：

```zig
const slice: []u8 = buffer[0..10];  // 从 buffer 创建的切片
```

---

## 3. 类型推断

Zig 支持类型推断，使用 `@TypeOf()` 或在声明时省略类型：

```zig
const allocator = gpa.allocator();  // 推断为 Allocator
var posts = std.ArrayList(Post){};  // 推断为 ArrayList(Post)
```

**项目实例**：

```zig
// 类型推断为 GeneralPurposeAllocator(.{})
var gpa = std.heap.GeneralPurposeAllocator(.{}){};

// 类型推断为 ArrayList(Post)
var posts = std.ArrayList(Post){};
```

---

## 4. 字面量

### 4.1 字符串字面量

```zig
const greeting = "Hello";  // 类型为 *const [5:0]u8
```

### 4.2 匿名结构体字面量

在 Zig 0.15.2 中，常用匿名结构体传递配置：

```zig
// build.zig
const zig_markdown_mod = b.dependency("zig_markdown", .{
    .target = target,
    .optimize = optimize,
});
```

这里的 `.{}` 是匿名结构体字面量。

---

## 5. 类型转换

### 5.1 隐式转换

Zig 不允许大多数隐式类型转换，保证类型安全。

### 5.2 显式转换

使用内置函数进行类型转换：

```zig
const num: u32 = 42;
const str = std.fmt.bufPrint(&buf, "{d}", .{num});
```

**项目实例** - 数字转字符串：

```zig
// Blogger.zig - 生成标签页面时
var post_count_buf: [16]u8 = undefined;
const post_count_str = std.fmt.bufPrint(&post_count_buf, "{d}", .{tag_posts.items.len}) catch "0";
```

---

## 6. 内存中的类型布局

### 6.1 结构体内存布局

```zig
pub const Post = struct {
    title: []const u8,      // 16 字节 (指针 + 长度)
    date_time: []const u8,  // 16 字节
    tags: []const u8,       // 16 字节
    content: []const u8,    // 16 字节
    filename: []const u8,   // 16 字节
};  // 总共 80 字节
```

每个切片 (`[]const u8`) 包含：
- 指针 (8 字节)
- 长度 (8 字节)

---

## 7. 实战：Post 结构体的类型分析

```zig
pub const Post = struct {
    // 所有字段都是切片类型，指向堆分配的内存
    title: []const u8,
    date_time: []const u8,
    tags: []const u8,
    content: []const u8,
    filename: []const u8,

    // 释放所有字段占用的内存
    pub fn deinit(self: *Post, allocator: Allocator) void {
        allocator.free(self.title);
        allocator.free(self.date_time);
        allocator.free(self.tags);
        allocator.free(self.content);
        allocator.free(self.filename);
    }
};
```

---

## 小结

| 概念 | 关键点 |
|------|--------|
| `const` vs `var` | 默认不可变，需要修改时用 `var` |
| 字符串 | 使用 `[]const u8` 切片表示 |
| 切片 | `[]T` 是最常用的引用类型 |
| 类型推断 | Zig 可以自动推断大多数类型 |
| 匿名结构体 | `.{}` 用于传递配置选项 |

---

## 相关阅读

- [03-2 重要类型详解](./03-2-important-types.md) - 深入讲解切片、可选类型等
- [08-1 内存分配器](../part-08-memory-management/08-1-allocators.md) - 理解 Zig 内存管理
- [Zig 官方文档 - 类型](https://ziglang.org/documentation/master/#Types)
