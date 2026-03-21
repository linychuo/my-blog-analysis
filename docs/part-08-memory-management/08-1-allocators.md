# 08-1 内存分配器 (Allocators)

> Zig 内存分配机制与项目实例分析

---

## 1. 内存分配器概览

### 1.1 什么是 Allocator

在 Zig 中，内存分配器 (`Allocator`) 是一个核心抽象，用于：

- 动态分配内存
- 释放已分配的内存
- 管理内存池

**类型定义**：

```zig
pub const Allocator = @import("std/mem/Allocator.zig").Allocator;
```

### 1.2 项目中的分配器使用

在 my-blog 项目中，分配器贯穿整个程序：

```
main() 
  │
  ├─ 创建 GeneralPurposeAllocator
  │
  ▼
传递给 Blogger.new()
  │
  ├─ 传递给 TemplateEngine
  │
  ▼
传递给所有需要内存的方法
  │
  ├─ Post.parse() - 解析文章
  ├─ generatePostPage() - 生成 HTML
  └─ loadPosts() - 读取文件
```

---

## 2. 标准库分配器类型

### 2.1 GeneralPurposeAllocator

**项目使用**：

```zig
// main.zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // 使用 allocator 分配内存
    var blogger = Blogger.new(allocator, posts_dir, dest_dir);
    try blogger.generate();
}
```

**特点**：

| 特点 | 说明 |
|------|------|
| 调试友好 | 检测内存泄漏、重复释放 |
| 通用 | 适用于各种分配模式 |
| 性能 | 中等（比 Arena 慢，比 Page 快） |
| 线程安全 | 支持多线程访问 |

**配置选项**：

```zig
// 默认配置
var gpa = std.heap.GeneralPurposeAllocator(.{}){};

// 自定义配置
var gpa = std.heap.GeneralPurposeAllocator(.{
    .stack_trace_frames = 6,      // 堆栈跟踪帧数
    .enable_memory_limit = false, // 启用内存限制
    .retry_on_oom = false,        // OOM 时重试
}){};
```

### 2.2 ArenaAllocator

**使用场景**：批量分配，一次性释放

```zig
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();

const allocator = arena.allocator();

// 多次分配
const str1 = try allocator.dupe(u8, "hello");
const str2 = try allocator.dupe(u8, "world");
const arr = try allocator.alloc(i32, 100);

// 一次性释放所有内存
arena.deinit();  // 不需要单独释放 str1, str2, arr
```

**项目潜在应用**：

```zig
// 在 generate() 中使用 Arena
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();
const allocator = arena.allocator();

// 所有临时分配使用 arena allocator
// generate() 返回时一次性释放
```

### 2.3 FixedBufferAllocator

**使用场景**：预分配固定大小缓冲区

```zig
var buffer: [1024]u8 = undefined;
var fba = std.heap.FixedBufferAllocator.init(&buffer);
const allocator = fba.allocator();

// 分配内存（从 buffer 中切割）
const data = try allocator.alloc(u8, 256);

// 注意：不能单独释放，只能重置整个分配器
fba.reset();  // 释放所有分配
```

### 2.4 分配器对比

| 分配器 | 分配速度 | 释放方式 | 适用场景 |
|--------|----------|----------|----------|
| `GeneralPurposeAllocator` | 中 | 逐个释放 | 通用场景 |
| `ArenaAllocator` | 快 | 一次性释放 | 临时数据、解析 |
| `FixedBufferAllocator` | 最快 | 重置/无法单独释放 | 嵌入式、缓冲区 |
| `page_allocator` | 慢 | 逐个释放 | 底层分配 |

---

## 3. 分配器 API

### 3.1 分配内存

```zig
// 分配单个对象
const ptr = try allocator.create(Type);

// 分配数组
const slice = try allocator.alloc(Type, count);

// 分配并复制
const copy = try allocator.dupe(Type, existing_slice);
```

### 3.2 释放内存

```zig
// 释放单个对象
allocator.destroy(ptr);

// 释放数组
allocator.free(slice);
```

### 3.3 重新分配

```zig
// 调整大小
const new_slice = try allocator.realloc(old_slice, new_size);

// 收缩（不移动内存）
allocator.shrink(slice, new_size);
```

---

## 4. 项目中的分配器使用模式

### 4.1 分配器传递模式

```zig
// main.zig - 创建分配器
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
defer _ = gpa.deinit();
const allocator = gpa.allocator();

// 传递给 Blogger
var blogger = Blogger.new(allocator, posts_dir, dest_dir);

// Blogger.zig - 存储分配器
pub const Blogger = struct {
    allocator: Allocator,
    
    pub fn new(allocator: Allocator, ...) Blogger {
        return .{
            .allocator = allocator,
            // ...
        };
    }
};
```

### 4.2 Post.parse() 中的分配

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // [1] 提取 title
    title = try extractValue(allocator, trimmed, "title:");
    //      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //      allocator.dupe() 分配新内存
    
    // [2] 提取 date_time
    date_time = try extractValue(allocator, trimmed, "date_time:");
    
    // [3] 提取 tags
    tags = try extractValue(allocator, trimmed, "tags:");
    
    // [4] 复制内容
    .content = try allocator.dupe(u8, content[content_start..]);
    
    // [5] 复制文件名
    .filename = try allocator.dupe(u8, filename);
    
    return Post{ ... };
}
```

**extractValue() 实现**：

```zig
fn extractValue(allocator: Allocator, line: []const u8, prefix: []const u8) ![]const u8 {
    // 分配并复制字符串
    return allocator.dupe(u8, std.mem.trim(u8, line[prefix.len..], " \r\n\t\"'"));
}
```

### 4.3 文件读取中的分配

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    // ...
    while (try walker.next()) |entry| {
        // 读取整个文件到分配的内存
        const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
        defer self.allocator.free(content);
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     使用完毕后立即释放
        
        // 解析 Post（会分配新内存存储字段）
        const post = try Post.parse(self.allocator, content, filename);
        // content 在此处释放，但 post 的字段使用新分配的内存
    }
}
```

### 4.4 HTML 生成中的分配

```zig
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] Markdown → HTML（分配新内存）
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);
    
    // [2] 构建标签 HTML
    var tags_html = std.ArrayList(u8){};
    defer tags_html.deinit(self.allocator);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     ArrayList 内部管理内存
    
    // [3] 渲染模板（分配新内存）
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);
    
    // [4] 构建完整标题
    var full_title = std.ArrayList(u8){};
    defer full_title.deinit(self.allocator);
    
    // [5] 最终 HTML
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);
}
```

---

## 5. ArrayList 与分配器

### 5.1 ArrayList 初始化

```zig
// 显式初始化
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();

// 使用 initCapacity 预分配
var list = try std.ArrayList(u8).initCapacity(allocator, 1024);
defer list.deinit();
```

### 5.2 ArrayList 操作

```zig
// 添加元素
try list.append('a');
try list.appendSlice("hello");

// 获取切片
const slice = list.items;

// 释放内存
list.deinit();  // 释放所有内部内存
```

### 5.3 项目中的 ArrayList

```zig
// 构建标签 HTML
var tags_html = std.ArrayList(u8){};
defer tags_html.deinit(self.allocator);

var tag_iter = std.mem.tokenizeScalar(u8, post.tags, ' ');
while (tag_iter.next()) |tag| {
    try tags_html.appendSlice(self.allocator, "<a href=\"/tags/");
    try tags_html.appendSlice(self.allocator, tag);
    try tags_html.appendSlice(self.allocator, "\">");
    try tags_html.appendSlice(self.allocator, tag);
    try tags_html.appendSlice(self.allocator, "</a> ");
}
```

---

## 6. 内存泄漏检测

### 6.1 GeneralPurposeAllocator 的泄漏检测

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
defer _ = gpa.deinit();  // 如果有泄漏，这里会触发警告
const allocator = gpa.allocator();

// 分配但未释放
const leak = try allocator.alloc(u8, 100);
// 忘记释放：allocator.free(leak);

// deinit() 时会检测到泄漏
// 输出：memory leak detected
```

### 6.2 项目中的正确模式

```zig
// main.zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
defer _ = gpa.deinit();  // 确保所有内存都已释放
const allocator = gpa.allocator();

// Blogger 使用 allocator，但不拥有它
// 所有分配的内存都在各自的作用域内释放
```

---

## 7. 分配器选择指南

### 7.1 场景与推荐

| 场景 | 推荐分配器 | 理由 |
|------|------------|------|
| 主程序入口 | `GeneralPurposeAllocator` | 检测泄漏，通用 |
| 临时解析 | `ArenaAllocator` | 批量释放，快速 |
| 固定缓冲区 | `FixedBufferAllocator` | 零开销，确定 |
| 性能关键 | 自定义池 | 针对特定模式优化 |
| 多线程 | `GeneralPurposeAllocator` | 线程安全 |

### 7.2 项目中的分配器层次

```
程序层级
├── main() - GeneralPurposeAllocator (根分配器)
│
├── Blogger - 使用根分配器
│   │
│   ├─ Post.parse() - 使用根分配器
│   ├─ generatePostPage() - 使用根分配器
│   └─ loadPosts() - 使用根分配器
│
└── TemplateEngine - 使用根分配器
```

---

## 8. 性能考虑

### 8.1 减少分配次数

```zig
// 不推荐：多次小分配
for (items) |item| {
    const str = try allocator.dupe(u8, item);
    defer allocator.free(str);
}

// 推荐：使用 ArrayList 一次分配
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();
for (items) |item| {
    try list.appendSlice(item);
}
```

### 8.2 避免不必要的复制

```zig
// 不推荐：无意义的复制
const copy = try allocator.dupe(u8, original);
defer allocator.free(copy);
use(copy);

// 推荐：直接使用引用
use(original);
```

### 8.3 预分配容量

```zig
// 知道大致大小时预分配
var list = try std.ArrayList(u8).initCapacity(allocator, expected_size);
defer list.deinit();
```

---

## 9. 常见错误

### 9.1 忘记释放

```zig
// 错误：忘记释放
const data = try allocator.alloc(u8, 100);
use(data);
// 忘记：allocator.free(data);

// 正确：使用 defer
const data = try allocator.alloc(u8, 100);
defer allocator.free(data);
use(data);
```

### 9.2 重复释放

```zig
// 错误：重复释放
allocator.free(ptr);
allocator.free(ptr);  // 崩溃！

// 正确：设置 null
allocator.free(ptr);
ptr = null;
```

### 9.3 使用已释放内存

```zig
// 错误：释放后使用
allocator.free(slice);
use(slice);  // 未定义行为！

// 正确：释放前使用完毕
use(slice);
allocator.free(slice);
```

---

## 10. 完整实例

### 10.1 内存管理最佳实践

```zig
const std = @import("std");

pub fn main() !void {
    // [1] 创建根分配器
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();  // 确保清理
    const allocator = gpa.allocator();
    
    // [2] 使用分配器
    const data = try processData(allocator);
    defer data.deinit(allocator);  // 确保清理
    
    // [3] 输出结果
    std.debug.print("Result: {s}\n", .{data.result});
}

fn processData(allocator: std.mem.Allocator) !Result {
    // [4] 分配临时内存
    var temp = std.ArrayList(u8).init(allocator);
    defer temp.deinit();  // 函数返回时释放
    
    // [5] 处理数据
    try temp.appendSlice("processing...");
    
    // [6] 分配结果内存（调用者负责释放）
    return Result{
        .result = try allocator.dupe(u8, temp.items),
    };
}

const Result = struct {
    result: []const u8,
    
    fn deinit(self: *Result, allocator: std.mem.Allocator) void {
        allocator.free(self.result);
    }
};
```

---

## 相关文档

- [08-2 内存所有权](./08-2-memory-ownership.md) - 内存所有权模式
- [08-3 defer 模式](./08-3-defer-patterns.md) - defer 使用技巧
- [05-2 参数与返回值](../part-05-functions/05-2-parameters-and-return.md) - 分配器参数
