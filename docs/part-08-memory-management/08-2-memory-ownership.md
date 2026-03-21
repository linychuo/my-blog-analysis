# 08-2 内存所有权 (Memory Ownership)

> Zig 内存所有权模式与项目实例分析

---

## 1. 内存所有权概念

### 1.1 什么是内存所有权

在 Zig 中，**内存所有权**指的是：

- 谁负责分配内存
- 谁负责释放内存
- 谁有权访问和修改内存

**核心原则**：

```
分配者 → 拥有 → 释放
```

### 1.2 项目中的所有权链

```
main() 分配 GeneralPurposeAllocator
    │
    ▼
Blogger 借用 allocator（不拥有）
    │
    ├─ 使用 allocator 分配 Post 字段
    │   │
    │   └─ Post 拥有字段内存
    │       │
    │       └─ Post.deinit() 释放
    │
    └─ 使用 allocator 分配临时 HTML
        │
        └─ 函数返回前释放
```

---

## 2. 所有权模式

### 2.1 创建者拥有模式

```zig
// 分配内存的函数负责释放
fn createData(allocator: Allocator) !Data {
    return Data{
        .name = try allocator.dupe(u8, "example"),
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     分配内存，返回给调用者
    };
}

// 调用者获得所有权
const data = try createData(allocator);
defer data.deinit(allocator);  // 调用者负责释放
```

**项目实例 - Post.parse()**：

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // 分配内存并返回 Post
    return Post{
        .title = try allocator.dupe(u8, title_value),
        .content = try allocator.dupe(u8, content_value),
        .filename = try allocator.dupe(u8, filename),
    };
}

// 调用者获得 Post 的所有权
const post = try Post.parse(allocator, content, filename);
defer post.deinit(allocator);  // 调用者负责释放
```

### 2.2 借用模式

```zig
// 函数借用内存，不拥有
fn processData(data: []const u8) void {
    // 使用 data，但不分配也不释放
    std.debug.print("Data: {s}\n", .{data});
}

// 调用者保持所有权
const data = try allocator.dupe(u8, "hello");
defer allocator.free(data);
processData(data);  // 借用
```

**项目实例 - Blogger.new()**：

```zig
pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger {
    return .{
        .allocator = allocator,      // 借用 allocator
        .posts_dir = posts_dir,      // 借用路径切片
        .dest_dir = dest_dir,        // 借用路径切片
        .template_engine = engine,
    };
}

// 调用者保持所有权
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
const allocator = gpa.allocator();

var blogger = Blogger.new(allocator, "posts", "build");
// blogger 借用 allocator，不拥有
```

### 2.3 转移所有权模式

```zig
// 所有权转移
fn transfer(allocator: Allocator) !Container {
    const data = try allocator.dupe(u8, "transferred");
    // data 的所有权转移给 Container
    return Container{ .data = data };
}

const container = try transfer(allocator);
// container 现在拥有 data
defer allocator.free(container.data);
```

---

## 3. Post 结构体的所有权分析

### 3.1 字段所有权

```zig
pub const Post = struct {
    title: []const u8,      // Post 拥有
    date_time: []const u8,  // Post 拥有
    tags: []const u8,       // Post 拥有
    content: []const u8,    // Post 拥有
    filename: []const u8,   // Post 拥有
    
    pub fn deinit(self: *Post, allocator: Allocator) void {
        // Post 释放所有拥有的内存
        allocator.free(self.title);
        allocator.free(self.date_time);
        allocator.free(self.tags);
        allocator.free(self.content);
        allocator.free(self.filename);
    }
};
```

**所有权流程**：

```
Post.parse() 分配内存
    │
    ├─ title (分配)
    ├─ date_time (分配)
    ├─ tags (分配)
    ├─ content (分配)
    └─ filename (分配)
    │
    ▼
返回 Post 实例
    │
    ▼
Post 拥有所有字段内存
    │
    ▼
调用者负责调用 deinit()
    │
    ▼
deinit() 释放所有字段
```

### 3.2 使用示例

```zig
// [1] 解析文章（分配内存）
const post = try Post.parse(allocator, content, filename);

// [2] Post 现在拥有所有字段
std.debug.print("Title: {s}\n", .{post.title});

// [3] 使用完毕后释放
defer post.deinit(allocator);

// 或者显式释放
post.deinit(allocator);
```

---

## 4. Blogger 中的所有权

### 4.1 Blogger 字段所有权

```zig
pub const Blogger = struct {
    dest_dir: []const u8,       // 借用
    posts_dir: []const u8,      // 借用
    allocator: Allocator,       // 借用
    template_engine: TemplateEngine,  // 拥有
};
```

**所有权分析**：

| 字段 | 所有权 | 说明 |
|------|--------|------|
| `dest_dir` | 借用 | 引用调用者提供的路径 |
| `posts_dir` | 借用 | 引用调用者提供的路径 |
| `allocator` | 借用 | 引用调用者的分配器 |
| `template_engine` | 拥有 | Blogger 内部管理 |

### 4.2 generate() 中的临时所有权

```zig
pub fn generate(self: *Blogger) !void {
    // [1] 创建 ArrayList（拥有内部内存）
    var posts = std.ArrayList(Post){};
    defer {
        // 释放所有 Post 拥有的内存
        for (posts.items) |*post| post.deinit(self.allocator);
        // 释放 ArrayList 本身
        posts.deinit(self.allocator);
    }
    
    // [2] 加载文章（posts 拥有）
    try self.loadPosts(&posts);
    
    // [3] 生成页面（临时分配，函数内释放）
    for (posts.items) |post| try self.generatePostPage(post);
    
    // [4] defer 块在函数返回时执行
}  // ← defer 块在此执行
```

---

## 5. 临时分配的所有权

### 5.1 函数内分配和释放

```zig
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] 分配 HTML 内容
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     函数返回前释放
    
    // [2] 分配标签 HTML
    var tags_html = std.ArrayList(u8){};
    defer tags_html.deinit(self.allocator);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     函数返回前释放
    
    // [3] 渲染模板
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     函数返回前释放
    
    // [4] 最终 HTML
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^
    //     函数返回前释放
    
    // [5] 写入文件后，html 不再需要
}  // ← 所有 defer 在此执行
```

### 5.2 所有权不转移

```zig
// post 的内容所有权不转移
fn generatePostPage(self: *Blogger, post: Post) !void {
    // 读取 post.content（借用）
    const html_content = try markdown.toHtml(self.allocator, post.content);
    //                                         ^^^^^^^^^^^^
    //                                         借用，不拥有
    
    // post.content 在函数结束后仍然有效（由 Post 拥有）
}
```

---

## 6. ArrayList 的所有权

### 6.1 ArrayList 拥有内部内存

```zig
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();  // ArrayList 释放内部内存

// 添加元素
try list.append('a');
try list.appendSlice("hello");

// list 拥有 items 内存
// list.deinit() 时释放
```

### 6.2 所有权转移

```zig
// 转移所有权给调用者
fn getData(allocator: Allocator) ![]u8 {
    var list = std.ArrayList(u8).init(allocator);
    try list.appendSlice("data");
    
    // 转移所有权（不释放）
    return list.toOwnedSlice();
    //     ^^^^^^^^^^^^^^^^^^^
    //     调用者负责释放
}

// 调用者获得所有权
const data = try getData(allocator);
defer allocator.free(data);  // 调用者释放
```

### 6.3 项目中的 ArrayList

```zig
// 构建输出路径
var output_path = std.ArrayList(u8){};
defer output_path.deinit(self.allocator);
//     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//     函数返回前释放

try output_path.appendSlice(self.allocator, self.dest_dir);
try output_path.appendSlice(self.allocator, "/");
try output_path.appendSlice(self.allocator, year);
// ...
```

---

## 7. 字符串所有权

### 7.1 字符串字面量

```zig
// 字符串字面量存储在静态内存
const static_str = "hello";
// 不需要释放，程序整个生命周期有效

// 类型：[]const u8
```

### 7.2 动态分配字符串

```zig
// 动态分配
const dynamic_str = try allocator.dupe(u8, "hello");
defer allocator.free(dynamic_str);
//     ^^^^^^^^^^^^^^^^^^^^^^^^^
//     必须释放

// 类型：[]const u8（与字面量相同）
```

### 7.3 项目中的字符串

```zig
// 默认值使用字面量（不分配）
var posts_dir: []const u8 = "posts";
//                           ^^^^^^
//                           静态内存，不需要释放

// 命令行参数覆盖（仍然借用）
posts_dir = args.next() orelse ...;
//          ^^^^^^^^^^^
//          借用命令行参数的内存

// Post 字段（动态分配）
pub fn parse(...) !Post {
    return Post{
        .title = try allocator.dupe(u8, title_value),
        //       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //       动态分配，Post 拥有
    };
}
```

---

## 8. 切片所有权

### 8.1 切片不拥有内存

```zig
// 切片只是引用
const slice = data[0..10];
// 不拥有 data 的内存

// 切片可以复制（浅拷贝）
const slice2 = slice;
// 两个切片指向同一内存
```

### 8.2 项目中的切片

```zig
// Blogger 借用路径切片
pub const Blogger = struct {
    posts_dir: []const u8,  // 借用，不拥有
    dest_dir: []const u8,   // 借用，不拥有
};

// Post 拥有字段切片
pub const Post = struct {
    title: []const u8,  // 拥有，需要释放
    content: []const u8, // 拥有，需要释放
};
```

---

## 9. 常见所有权错误

### 9.1 过早释放

```zig
// 错误：释放后使用
const data = try allocator.dupe(u8, "hello");
allocator.free(data);
use(data);  // 未定义行为！

// 正确：使用 defer
const data = try allocator.dupe(u8, "hello");
defer allocator.free(data);
use(data);  // 安全使用
```

### 9.2 双重释放

```zig
// 错误：释放两次
allocator.free(ptr);
allocator.free(ptr);  // 崩溃！

// 正确：只释放一次
defer allocator.free(ptr);
```

### 9.3 忘记释放

```zig
// 错误：内存泄漏
fn leak() !void {
    const data = try allocator.alloc(u8, 100);
    // 忘记释放
}

// 正确
fn noLeak() !void {
    const data = try allocator.alloc(u8, 100);
    defer allocator.free(data);
}
```

### 9.4 混淆借用和拥有

```zig
// 错误：尝试释放借用的内存
fn wrong(allocator: Allocator) !void {
    const static_str = "hello";  // 静态内存
    allocator.free(static_str);  // 错误！
}

// 错误：不释放拥有的内存
fn leak() !Post {
    return Post{
        .title = try allocator.dupe(u8, "title"),
        // 忘记：调用者需要调用 deinit()
    };
}
```

---

## 10. 所有权最佳实践

### 10.1 明确所有权

```zig
// 好的命名暗示所有权
fn createData(allocator: Allocator) !Data  // 创建，返回所有权
fn processData(data: []const u8) void      // 处理，借用
fn destroyData(data: *Data) void           // 销毁，转移所有权
```

### 10.2 使用 defer 确保释放

```zig
// 总是使用 defer
const data = try allocator.alloc(u8, size);
defer allocator.free(data);

// 即使有错误返回也能正确释放
```

### 10.3 文档化所有权

```zig
/// 创建新的 Post 实例
/// 调用者拥有返回的 Post，负责调用 deinit()
pub fn parse(allocator: Allocator, ...) !Post { }

/// 处理数据（借用）
/// 不获取所有权，调用者负责释放
pub fn process(data: []const u8) void { }
```

---

## 11. 完整实例

### 11.1 所有权链示例

```zig
const std = @import("std");

pub fn main() !void {
    // [1] main 拥有分配器
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // [2] 创建文章（main 调用，Post 拥有内容）
    const content = "---\ntitle: Hello\ndate_time: 2026-03-21\n---\nContent";
    const post = try Post.parse(allocator, content, "test.markdown");
    defer post.deinit(allocator);  // main 负责释放
    
    // [3] 使用文章
    std.debug.print("Title: {s}\n", .{post.title});
}

const Post = struct {
    title: []const u8,
    date_time: []const u8,
    content: []const u8,
    filename: []const u8,
    
    /// 解析文章
    /// 返回的 Post 拥有所有字段内存
    pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
        // 分配并拥有字段内存
        return Post{
            .title = try allocator.dupe(u8, "Hello"),
            .date_time = try allocator.dupe(u8, "2026-03-21"),
            .content = try allocator.dupe(u8, "Content"),
            .filename = try allocator.dupe(u8, filename),
        };
    }
    
    /// 释放 Post 拥有的所有内存
    pub fn deinit(self: *Post, allocator: Allocator) void {
        allocator.free(self.title);
        allocator.free(self.date_time);
        allocator.free(self.content);
        allocator.free(self.filename);
    }
};
```

---

## 相关文档

- [08-1 内存分配器](./08-1-allocators.md) - 分配器基础
- [08-3 defer 模式](./08-3-defer-patterns.md) - defer 使用技巧
- [07-2 Post 结构体](../part-07-core-modules/07-2-post-struct.md) - Post 详解
