# 07-2 Post 结构体

> 文章数据结构与 Frontmatter 解析

---

## 1. Post 结构体概览

### 1.1 完整代码

```zig
pub const Post = struct {
    title: []const u8,
    date_time: []const u8,
    tags: []const u8,
    content: []const u8,
    filename: []const u8,

    pub fn deinit(self: *Post, allocator: Allocator) void {
        allocator.free(self.title);
        allocator.free(self.date_time);
        allocator.free(self.tags);
        allocator.free(self.content);
        allocator.free(self.filename);
    }

    pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
        var lines = std.mem.splitScalar(u8, content, '\n');
        const first_line = lines.next() orelse return error.InvalidFormat;
        if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
            return error.InvalidFormat;
        }

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

        const content_start = content_offset_for_line(content, frontmatter_end);
        return Post{
            .title = title orelse return error.MissingTitle,
            .date_time = date_time orelse return error.MissingDateTime,
            .tags = tags orelse "",
            .content = try allocator.dupe(u8, content[content_start..]),
            .filename = try allocator.dupe(u8, filename),
        };
    }

    fn extractValue(allocator: Allocator, line: []const u8, prefix: []const u8) ![]const u8 {
        return allocator.dupe(u8, std.mem.trim(u8, line[prefix.len..], " \r\n\t\"'"));
    }

    fn content_offset_for_line(content: []const u8, line_num: usize) usize {
        var offset: usize = 0;
        var current_line: usize = 0;
        while (offset < content.len and current_line < line_num) {
            if (content[offset] == '\n') current_line += 1;
            offset += 1;
        }
        while (offset < content.len and (content[offset] == '\n' or content[offset] == '\r')) offset += 1;
        return offset;
    }
};
```

### 1.2 结构体字段

| 字段 | 类型 | 大小 | 说明 |
|------|------|------|------|
| `title` | `[]const u8` | 16 字节 | 文章标题 |
| `date_time` | `[]const u8` | 16 字节 | 发布日期时间 |
| `tags` | `[]const u8` | 16 字节 | 标签（空格分隔） |
| `content` | `[]const u8` | 16 字节 | Markdown 内容 |
| `filename` | `[]const u8` | 16 字节 | 源文件名 |

**总大小**：80 字节（64 位系统）

---

## 2. 内存管理：deinit()

### 2.1 函数签名

```zig
pub fn deinit(self: *Post, allocator: Allocator) void
```

**参数分析**：

| 参数 | 类型 | 用途 |
|------|------|------|
| `self` | `*Post` | 指向 Post 实例的指针 |
| `allocator` | `Allocator` | 用于释放内存的分配器 |

### 2.2 内存释放

```zig
pub fn deinit(self: *Post, allocator: Allocator) void {
    allocator.free(self.title);
    allocator.free(self.date_time);
    allocator.free(self.tags);
    allocator.free(self.content);
    allocator.free(self.filename);
}
```

**释放顺序**：

```
Post 实例
    │
    ├─ title (动态分配的字符串)
    ├─ date_time (动态分配的字符串)
    ├─ tags (动态分配的字符串)
    ├─ content (动态分配的字符串)
    └─ filename (动态分配的字符串)
```

### 2.3 使用模式

```zig
// 创建 Post
const post = try Post.parse(allocator, content, filename);

// 使用完毕后释放
defer post.deinit(allocator);

// 或者显式调用
post.deinit(allocator);
```

**在 Blogger 中的使用**：

```zig
var posts = std.ArrayList(Post){};
defer {
    // 批量释放所有 Post
    for (posts.items) |*post| post.deinit(self.allocator);
    posts.deinit(self.allocator);
}
```

---

## 3. Frontmatter 解析：parse()

### 3.1 函数签名

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post
```

**返回类型**：`!Post`（错误联合类型）

**可能的错误**：

| 错误 | 触发条件 |
|------|----------|
| `error.InvalidFormat` | 缺少 `---` 分隔符 |
| `error.MissingTitle` | 缺少 title 字段 |
| `error.MissingDateTime` | 缺少 date_time 字段 |
| `error.OutOfMemory` | 内存分配失败 |

### 3.2 Frontmatter 格式

**标准格式**：

```markdown
---
title: 我的文章
date_time: 2026-03-21
tags: zig blog
---

这里是文章内容...
```

**字段说明**：

| 字段 | 必需 | 说明 |
|------|------|------|
| `title` | ✓ | 文章标题 |
| `date_time` | ✓ | 发布日期（YYYY-MM-DD 格式） |
| `tags` | ✗ | 标签列表（空格分隔） |

### 3.3 解析流程

```
输入：完整的 Markdown 文件内容
    │
    ▼
[1] 验证起始分隔符 "---"
    │
    ▼
[2] 逐行解析 frontmatter
    │
    ├─ 提取 title
    ├─ 提取 date_time
    └─ 提取 tags（可选）
    │
    ▼
[3] 找到结束分隔符 "---"
    │
    ▼
[4] 计算内容起始位置
    │
    ▼
[5] 构造并返回 Post 实例
```

### 3.4 分隔符验证

```zig
var lines = std.mem.splitScalar(u8, content, '\n');
const first_line = lines.next() orelse return error.InvalidFormat;

if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
    return error.InvalidFormat;
}
```

**验证逻辑**：

```
content = "---\ntitle: Hello\n---\nContent..."
    │
    ▼
splitScalar('\n') → 迭代器
    │
    ▼
lines.next() → "---"
    │
    ▼
trim(" \r\n\t") → "---"
    │
    ▼
eql("---") → true ✓
```

### 3.5 字段提取循环

```zig
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
```

**可选类型使用**：

```zig
var title: ?[]const u8 = null;  // 初始为 null
// ...
title = try extractValue(...);  // 解析后为 Some(value)
// ...
.title = title orelse return error.MissingTitle  // 验证
```

### 3.6 必需字段验证

```zig
return Post{
    .title = title orelse return error.MissingTitle,
    .date_time = date_time orelse return error.MissingDateTime,
    .tags = tags orelse "",  // tags 可选，默认空字符串
    .content = try allocator.dupe(u8, content[content_start..]),
    .filename = try allocator.dupe(u8, filename),
};
```

**orelse 模式对比**：

```zig
// 必需字段 - 缺失时返回错误
title orelse return error.MissingTitle

// 可选字段 - 缺失时使用默认值
tags orelse ""
```

---

## 4. 辅助函数

### 4.1 extractValue()

```zig
fn extractValue(allocator: Allocator, line: []const u8, prefix: []const u8) ![]const u8 {
    return allocator.dupe(u8, std.mem.trim(u8, line[prefix.len..], " \r\n\t\"'"));
}
```

**功能**：从 `key: value` 格式中提取 value

**解析示例**：

```
输入：line = "title: 我的文章", prefix = "title:"
    │
    ▼
line[prefix.len..] → " 我的文章"
    │
    ▼
trim(" \r\n\t\"'") → "我的文章"
    │
    ▼
allocator.dupe() → 分配并复制字符串
    │
    ▼
返回：[]const u8 = "我的文章"
```

**支持的格式**：

```zig
// 标准格式
"title: 标题"       → "标题"

// 带引号
"title: \"标题\""   → "标题"

// 带空格
"title:   标题  "   → "标题"

// 单引号
"title: '标题'"    → "标题"
```

### 4.2 content_offset_for_line()

```zig
fn content_offset_for_line(content: []const u8, line_num: usize) usize {
    var offset: usize = 0;
    var current_line: usize = 0;
    while (offset < content.len and current_line < line_num) {
        if (content[offset] == '\n') current_line += 1;
        offset += 1;
    }
    while (offset < content.len and (content[offset] == '\n' or content[offset] == '\r')) {
        offset += 1;
    }
    return offset;
}
```

**功能**：计算第 line_num 行之后的字节偏移量

**示例**：

```
content = "---\ntitle: Hi\n---\n\n文章内容..."
          0123  5678901   15 678901234
                    1         2

line_num = 3 (frontmatter_end)
    │
    ▼
遍历字符直到第 3 行
    │
    ▼
offset = 17 (跳过所有换行符)
    │
    ▼
返回 17 → content[17..] = "文章内容..."
```

**两个 while 循环**：

```zig
// [1] 找到目标行的起始位置
while (offset < content.len and current_line < line_num) {
    if (content[offset] == '\n') current_line += 1;
    offset += 1;
}

// [2] 跳过后续的空白行（换行符）
while (offset < content.len and (content[offset] == '\n' or content[offset] == '\r')) {
    offset += 1;
}
```

---

## 5. 内存分配分析

### 5.1 分配点

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    // ...
    
    // [1] 提取 title
    title = try extractValue(allocator, trimmed, "title:");
    //      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //      allocator.dupe() 分配内存
    
    // [2] 提取 date_time
    date_time = try extractValue(allocator, trimmed, "date_time:");
    //          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //          allocator.dupe() 分配内存
    
    // [3] 提取 tags（如果有）
    tags = try extractValue(allocator, trimmed, "tags:");
    //       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //       allocator.dupe() 分配内存
    
    // [4] 复制内容
    .content = try allocator.dupe(u8, content[content_start..]);
    //         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //         分配新内存
    
    // [5] 复制文件名
    .filename = try allocator.dupe(u8, filename);
    //          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //          分配新内存
}
```

### 5.2 内存所有权

```
调用者传递 content (切片引用)
    │
    ▼
parse() 分配新内存存储 Post 字段
    │
    ├─ title (新分配)
    ├─ date_time (新分配)
    ├─ tags (新分配)
    ├─ content (新分配)
    └─ filename (新分配)
    │
    ▼
返回 Post (拥有所有字段的所有权)
    │
    ▼
调用者负责通过 deinit() 释放
```

### 5.3 内存使用估算

对于一篇典型文章：

| 字段 | 典型大小 | 分配开销 |
|------|----------|----------|
| `title` | 50 字节 | ~16 字节 |
| `date_time` | 10 字节 | ~16 字节 |
| `tags` | 30 字节 | ~16 字节 |
| `content` | 5000 字节 | ~16 字节 |
| `filename` | 20 字节 | ~16 字节 |
| **总计** | **~5110 字节** | **~80 字节** |

---

## 6. 错误处理

### 6.2 错误传播

```zig
// extractValue 中的错误传播
title = try extractValue(allocator, trimmed, "title:");
//      ^^^^ 如果分配失败，立即返回 error.OutOfMemory

// 内容复制的错误传播
.content = try allocator.dupe(u8, content[content_start..]);
//         ^^^^ 如果分配失败，立即返回 error.OutOfMemory
```

### 6.3 错误恢复

```zig
// 在 loadPosts() 中的错误处理
const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| {
    std.debug.print(" Skipped ({s}): {s}\n", .{ @errorName(err), entry.path });
    continue;  // 跳过坏文章，继续处理其他文章
};
```

**错误恢复策略**：

```
解析失败的文章
    │
    ├─ 打印错误信息（包括错误类型和文件路径）
    │
    └─ continue → 跳过该文章
        │
        ▼
    继续处理下一篇文章
```

---

## 7. 使用示例

### 7.1 基本使用

```zig
const std = @import("std");
const Post = @import("Blogger.zig").Post;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const markdown_content = 
        \\---
        \\title: 我的第一篇文章
        \\date_time: 2026-03-21
        \\tags: zig blog
        \\---
        \\
        \\这是文章内容...
    ;

    // 解析文章
    const post = try Post.parse(allocator, markdown_content, "my-post.markdown");
    defer post.deinit(allocator);

    // 使用文章数据
    std.debug.print("Title: {s}\n", .{post.title});
    std.debug.print("Date: {s}\n", .{post.date_time});
    std.debug.print("Tags: {s}\n", .{post.tags});
}
```

### 7.2 批量加载

```zig
var posts = std.ArrayList(Post).init(allocator);
defer {
    for (posts.items) |*post| {
        post.deinit(allocator);
    }
    posts.deinit();
}

// 加载多篇文章
try posts.append(try Post.parse(allocator, content1, "post1.markdown"));
try posts.append(try Post.parse(allocator, content2, "post2.markdown"));
```

---

## 8. 设计模式

### 8.1 解析器模式

```
原始输入 (String)
    │
    ▼
验证格式 (validate)
    │
    ▼
提取字段 (extract)
    │
    ▼
构造对象 (construct)
    │
    ▼
返回结果 (Result/Error)
```

### 8.2 资源管理模式

```zig
// 创建时分配内存
const post = try Post.parse(allocator, ...);

// 使用 defer 确保释放
defer post.deinit(allocator);

// 安全使用
// ...
```

---

## 相关文档

- [07-3 Blogger 结构体](./07-3-blogger-struct.md) - 使用 Post 生成页面
- [05-2 参数与返回值](../part-05-functions/05-2-parameters-and-return.md) - 内存所有权
- [08-2 内存所有权](../part-08-memory-management/08-2-memory-ownership.md) - 内存管理深入
