# 05-3 泛型编程

> Zig 语言的泛型与编译时多态

---

## 1. Zig 的泛型理念

### 1.1 编译时多态

Zig 不使用传统的泛型语法（如 `<T>`），而是通过 **编译时参数** 和 **类型推断** 实现泛型：

```zig
// 传统泛型（C++/Java 风格）
// fn max<T>(a: T, b: T) T

// Zig 风格 - 使用 comptime 类型参数
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

// 调用
const a = max(i32, 5, 10);
const b = max(f64, 3.14, 2.71);
```

### 1.2 项目中的泛型使用

在 my-blog 项目中，泛型使用相对简单，主要通过标准库间接使用：

```zig
// 项目中使用标准库的泛型函数
var posts = std.ArrayList(Post){};  // ArrayList 是泛型结构体
var tag_map = std.StringHashMap(std.ArrayListUnmanaged(Post)).init(self.allocator);

// 泛型排序
std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);
```

---

## 2. comptime 类型参数

### 2.1 类型作为参数

```zig
// 函数接收类型作为 comptime 参数
fn process(comptime T: type, value: T) void {
    std.debug.print("Type: {s}\n", .{@typeName(T)});
}

// 调用
process(i32, 42);
process([]const u8, "hello");
process(Post, my_post);
```

### 2.2 类型推断

Zig 可以推断类型参数，无需显式指定：

```zig
// 标准库中的泛型函数
pub fn ArrayList(comptime T: type) type {
    return struct {
        items: []T,
        // ...
    };
}

// 调用时指定类型
const IntList = ArrayList(i32);
const PostList = ArrayList(Post);

// 或使用变量
var list = ArrayList(Post).init(allocator);
```

---

## 3. anytype 参数

### 3.1 anytype 基础

`anytype` 允许函数接受任何类型的参数：

```zig
// 接收任何类型的参数
fn printValue(value: anytype) void {
    const T = @TypeOf(value);
    std.debug.print("Value: {}\n", .{value});
}

// 调用
printValue(42);
printValue("hello");
printValue(my_struct);
```

### 3.2 项目中的潜在应用

虽然项目当前未使用 `anytype`，但以下场景适合：

```zig
// 通用的 HTML 渲染器
fn renderHtml(item: anytype) ![]const u8 {
    const T = @TypeOf(item);
    return switch (T) {
        Post => renderPost(item),
        Page => renderPage(item),
        else => @compileError("Unsupported type: " ++ @typeName(T)),
    };
}

// 通用的日志函数
fn log(comptime level: []const u8, message: anytype) void {
    std.debug.print("[{s}] {}\n", .{ level, message });
}
```

---

## 4. 泛型结构体

### 4.1 标准库中的泛型结构体

**项目实例** - `std.ArrayList` 和 `std.StringHashMap`：

```zig
// ArrayList 是泛型结构体
pub fn ArrayList(comptime T: type) type {
    return struct {
        items: []T,
        capacity: usize,
        allocator: Allocator,
        
        pub fn init(allocator: Allocator) ArrayList(T) {
            return .{
                .items = &.{},
                .capacity = 0,
                .allocator = allocator,
            };
        }
        
        pub fn append(self: *ArrayList(T), item: T) !void {
            // ...
        }
    };
}

// 项目中使用
var posts = std.ArrayList(Post){};  // Post 类型的 ArrayList
var tag_posts = std.ArrayListUnmanaged(Post){};  // 无托管版本
```

### 4.2 HashMap 泛型

**项目实例** - `generateTagPages()` 中的标签映射：

```zig
fn generateTagPages(self: *Blogger, posts: []const Post) !void {
    // StringHashMap 是泛型，值是 ArrayListUnmanaged(Post)
    var tag_map = std.StringHashMap(std.ArrayListUnmanaged(Post)).init(self.allocator);
    defer {
        var it = tag_map.iterator();
        while (it.next()) |entry| {
            entry.value_ptr.deinit(self.allocator);
        }
        tag_map.deinit();
    }

    // 构建标签 -> 文章列表的映射
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
    // ...
}
```

**类型分析**：

```
std.StringHashMap(std.ArrayListUnmanaged(Post))
├── Key:   []const u8 (字符串)
└── Value: std.ArrayListUnmanaged(Post)
    └── 元素：Post
```

---

## 5. 泛型函数模式

### 5.1 比较函数模式

**项目实例** - `sortPostsByDateDesc` 配合 `std.mem.sort`：

```zig
// std.mem.sort 是泛型函数
pub fn sort(comptime T: type, items: []T, context: anytype, 
            comptime lessThan: fn (@TypeOf(context), T, T) bool) void {
    // ...
}

// 项目使用
std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

// 比较函数
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}
```

**类型匹配**：

| std.mem.sort 参数 | 项目实际值 |
|-------------------|------------|
| `T` | `Post` |
| `items` | `posts.items` (`[]Post`) |
| `context` | `{}` (void) |
| `lessThan` | `sortPostsByDateDesc` |

### 5.2 可选的 context 参数

```zig
// 简单比较 - 不需要 context
fn sortPostsByDateDesc(_: void, a: Post, b: Post) bool {
    return std.mem.order(u8, a.date_time, b.date_time).compare(.gt);
}

// 复杂比较 - 需要 context
fn sortByTag(context: struct { tag: []const u8 }, a: Post, b: Post) bool {
    const a_has_tag = std.mem.indexOfScalar(u8, a.tags, context.tag) != null;
    const b_has_tag = std.mem.indexOfScalar(u8, b.tags, context.tag) != null;
    return a_has_tag and !b_has_tag;
}

// 使用
const ctx = .{ .tag = "zig" };
std.mem.sort(Post, posts.items, ctx, sortByTag);
```

---

## 6. 类型反射与泛型

### 6.1 @TypeOf 和 @typeName

```zig
// 获取变量类型
const x = 42;
const T = @TypeOf(x);  // T = comptime_int

// 获取类型名称
const name = @typeName(T);  // "comptime_int"
```

**项目实例** - 错误名称：

```zig
const post = Post.parse(allocator, content, filename) catch |err| {
    std.debug.print(" Skipped ({s}): {s}\n", .{ 
        @errorName(err),  // 将错误类型转换为字符串
        entry.path 
    });
    continue;
};
```

### 6.2 @typeInfo 反射

```zig
// 获取结构体字段信息
comptime {
    const info = @typeInfo(Post);
    std.debug.print("Post has {d} fields\n", .{info.Struct.fields.len});
    
    for (info.Struct.fields) |field| {
        std.debug.print("  Field: {s} ({s})\n", .{ 
            field.name, 
            @typeName(field.type) 
        });
    }
}
```

**Post 结构体的字段信息**：

```
Post has 5 fields
  Field: title ([]const u8)
  Field: date_time ([]const u8)
  Field: tags ([]const u8)
  Field: content ([]const u8)
  Field: filename ([]const u8)
```

---

## 7. 完整实例：通用 HTML 生成器

### 7.1 使用 anytype 的通用渲染

```zig
// 通用 HTML 标签生成
pub fn tag(comptime tag_name: []const u8, content: []const u8, 
           allocator: Allocator) ![]const u8 {
    var buffer = std.ArrayList(u8).init(allocator);
    errdefer buffer.deinit();
    
    try buffer.appendSlice("<");
    try buffer.appendSlice(tag_name);
    try buffer.appendSlice(">");
    try buffer.appendSlice(content);
    try buffer.appendSlice("</");
    try buffer.appendSlice(tag_name);
    try buffer.appendSlice(">");
    
    return buffer.toOwnedSlice();
}

// 专用函数
pub fn div(content: []const u8, allocator: Allocator) ![]const u8 {
    return tag("div", content, allocator);
}

pub fn article(content: []const u8, allocator: Allocator) ![]const u8 {
    return tag("article", content, allocator);
}
```

### 7.2 使用类型特征的泛型

```zig
// 检查类型是否有特定字段
fn hasTitleField(comptime T: type) bool {
    const info = @typeInfo(T);
    if (info != .Struct) return false;
    
    for (info.Struct.fields) |field| {
        if (std.mem.eql(u8, field.name, "title")) {
            return true;
        }
    }
    return false;
}

// 条件编译
fn render(comptime T: type, item: T) !void {
    if (comptime hasTitleField(T)) {
        std.debug.print("Rendering with title: {s}\n", .{item.title});
    } else {
        std.debug.print("Rendering without title\n", .{});
    }
}
```

---

## 8. 项目中的泛型扩展建议

### 8.1 通用的页面生成器

```zig
// 当前：每个页面类型有独立函数
fn generateIndex(self: *Blogger, posts: []const Post) !void { }
fn generateAbout(self: *Blogger) !void { }
fn generateTagPages(self: *Blogger, posts: []const Post) !void { }

// 扩展：通用页面生成器
pub fn PageType = enum {
    index,
    about,
    tag,
    post,
};

fn generatePage(self: *Blogger, comptime page_type: PageType, 
                context: anytype) !void {
    switch (page_type) {
        .index => try renderIndex(self, context),
        .about => try renderAbout(self),
        .tag => try renderTag(self, context),
        .post => try renderPost(self, context),
    }
}
```

### 8.2 通用的缓存机制

```zig
// 通用缓存结构体
fn Cache(comptime V: type) type {
    return struct {
        map: std.StringHashMap(V),
        
        pub fn init(allocator: Allocator) Cache(V) {
            return .{
                .map = std.StringHashMap(V).init(allocator),
            };
        }
        
        pub fn getOrPut(self: *Cache(V), key: []const u8) !*V {
            const gop = try self.map.getOrPut(key);
            if (!gop.found_existing) {
                gop.value_ptr.* = try createDefault(V);
            }
            return gop.value_ptr;
        }
    };
}

// 使用
var html_cache = Cache([]const u8).init(allocator);
```

---

## 9. 编译时计算与泛型

### 9.1 comptime 循环生成代码

```zig
// 编译时生成多个标签函数
const html_tags = [_][]const u8{ "div", "span", "article", "section" };

comptime {
    for (html_tags) |tag_name| {
        // 在编译时为每个标签生成函数
        // 这需要更复杂的元编程技巧
    }
}
```

### 9.2 条件编译

```zig
// 根据类型特征选择实现
fn process(item: anytype) !void {
    const T = @TypeOf(item);
    
    if (comptime std.mem.startsWith(u8, @typeName(T), "[]")) {
        // 处理切片类型
        std.debug.print("Processing slice\n", .{});
    } else if (comptime @typeInfo(T) == .Struct) {
        // 处理结构体类型
        std.debug.print("Processing struct\n", .{});
    }
}
```

---

## 10. 注意事项

### 10.1 anytype 的使用限制

```zig
// 错误 - anytype 参数不能用于运行时多态
fn process(value: anytype) void {
    // value 的类型在编译时确定
}

var f = process;  // 错误：不能将泛型函数作为值
f(42);
f("hello");

// 正确 - 每次调用都是不同的函数实例
process(42);
process("hello");
```

### 10.2 comptime 参数的要求

```zig
// 错误 - comptime 参数必须是编译时已知
fn process(comptime x: usize) void { }

pub fn main() void {
    var y: usize = 5;
    process(y);  // 错误：y 不是 comptime 已知
}

// 正确
process(5);  // 字面量是 comptime 已知
```

### 10.3 代码膨胀

```zig
// 泛型函数会为每个类型生成独立代码
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

// 以下调用会生成多个函数实例
_ = max(i32, 1, 2);      // 生成 max_i32
_ = max(f64, 1.0, 2.0);  // 生成 max_f64
_ = max(Post, a, b);     // 生成 max_Post

// 对于大量类型，考虑使用接口模式
```

---

## 11. 总结

### 11.1 Zig 泛型特点

| 特点 | 说明 |
|------|------|
| 编译时多态 | 类型在编译时确定 |
| comptime 参数 | 类型作为参数传递 |
| anytype | 接受任何类型 |
| 类型推断 | 自动推断类型参数 |
| 零开销 | 无运行时泛型开销 |

### 11.2 项目中的泛型使用

| 场景 | 使用方式 |
|------|----------|
| `ArrayList(Post)` | 标准库泛型结构体 |
| `StringHashMap(T)` | 标准库泛型 HashMap |
| `std.mem.sort` | 泛型排序函数 |
| `@errorName` | 编译时内置函数 |

---

## 相关文档

- [04-4 comptime](../part-04-control-flow/04-4-comptime.md) - 编译时执行
- [05-1 函数基础](./05-1-function-basics.md) - 函数定义
- [06-2 std.Build API](../part-06-build-system/06-2-std-build-api.md) - 构建系统中的泛型使用
