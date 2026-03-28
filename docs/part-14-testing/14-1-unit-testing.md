# 14.1 Zig 单元测试

> **测试框架**: Zig 内置测试框架  
> **运行命令**: `zig build test`  
> **测试语法**: `test "description" { }`

## 14.1.1 Zig 测试基础

### 测试块语法

```zig
// 基本测试块
test "addition works" {
    const result = 2 + 2;
    try std.testing.expectEqual(@as(u32, 4), result);
}

// 多个测试块
test "string concatenation" {
    const allocator = std.testing.allocator;
    const str = try std.fmt.allocPrint(allocator, "{s} {s}", .{ "Hello", "World" });
    defer allocator.free(str);
    
    try std.testing.expectEqualStrings("Hello World", str);
}

test "empty test (will pass)" {
    // 空测试也会通过
}
```

### 测试断言函数

| 函数 | 用途 | 示例 |
|------|------|------|
| `expectEqual` | 值相等 | `try expectEqual(4, 2 + 2)` |
| `expectEqualStrings` | 字符串相等 | `try expectEqualStrings("abc", str)` |
| `expect` | 布尔条件 | `try expect(x > 0)` |
| `expectError` | 期望错误 | `try expectError(error.Overflow, fn())` |
| `expectApproxEqAbs` | 浮点近似 | `try expectApproxEqAbs(0.3, a + b, 0.001)` |

### 完整测试示例

```zig
const std = @import("std");
const testing = std.testing;

// 被测试的函数
fn add(a: i32, b: i32) i32 {
    return a + b;
}

fn divide(a: f64, b: f64) !f64 {
    if (b == 0) return error.DivisionByZero;
    return a / b;
}

// 测试
test "add positive numbers" {
    try testing.expectEqual(@as(i32, 5), add(2, 3));
}

test "add negative numbers" {
    try testing.expectEqual(@as(i32, -5), add(-2, -3));
}

test "divide normally" {
    const result = try divide(10.0, 2.0);
    try testing.expectApproxEqAbs(@as(f64, 5.0), result, 0.001);
}

test "divide by zero returns error" {
    try testing.expectError(error.DivisionByZero, divide(10.0, 0.0));
}
```

## 14.1.2 在 my-blog 中编写测试

### 测试 Post 结构

```zig
// 在 Blogger.zig 中添加测试
const Post = struct {
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
};

test "Post creation and cleanup" {
    const allocator = std.testing.allocator;
    
    var post = Post{
        .title = try allocator.dupe(u8, "Test Post"),
        .date_time = try allocator.dupe(u8, "2024-01-15 10:00"),
        .tags = try allocator.dupe(u8, "zig,test"),
        .content = try allocator.dupe(u8, "Test content"),
        .filename = try allocator.dupe(u8, "test-post.markdown"),
    };
    defer post.deinit(allocator);
    
    try std.testing.expectEqualStrings("Test Post", post.title);
    try std.testing.expectEqualStrings("2024-01-15 10:00", post.date_time);
}
```

### 测试 Frontmatter 解析

```zig
// Frontmatter 解析函数
fn parseFrontmatter(content: []const u8, allocator: Allocator) !Frontmatter {
    // 查找分隔符
    if (!std.mem.startsWith(u8, content, "---")) {
        return error.MissingFrontmatterDelimiter;
    }
    
    const end_index = std.mem.indexOf(u8, content[3..], "---") orelse 
        return error.UnclosedFrontmatter;
    
    const frontmatter_text = content[3..end_index + 3];
    
    // 解析字段
    var fm = Frontmatter.init(allocator);
    
    // 解析 title
    if (std.mem.indexOf(u8, frontmatter_text, "title:")) |title_pos| {
        const title_line = std.mem.trim(u8, frontmatter_text[title_pos + 6..], " \n\r");
        const newline = std.mem.indexOf(u8, title_line, "\n") orelse title_line.len;
        fm.title = try allocator.dupe(u8, title_line[0..newline]);
    }
    
    return fm;
}

// 测试
test "parse frontmatter with title" {
    const allocator = std.testing.allocator;
    const content = 
        \\---
        \\title: My Test Post
        \\date_time: 2024-01-15 10:00
        \\---
        \\Content starts here
    ;
    
    const fm = try parseFrontmatter(content, allocator);
    defer {
        allocator.free(fm.title);
        allocator.free(fm.date_time);
    }
    
    try std.testing.expectEqualStrings("My Test Post", fm.title);
    try std.testing.expectEqualStrings("2024-01-15 10:00", fm.date_time);
}

test "parse frontmatter missing delimiter" {
    const allocator = std.testing.allocator;
    const content = "No frontmatter here";
    
    try std.testing.expectError(
        error.MissingFrontmatterDelimiter,
        parseFrontmatter(content, allocator)
    );
}
```

### 测试 Markdown 转换

```zig
// Markdown 转 HTML
fn markdownToHtml(markdown: []const u8, allocator: Allocator) ![]u8 {
    var html = std.ArrayList(u8).init(allocator);
    
    var iter = std.mem.split(u8, markdown, "\n");
    while (iter.next()) |line| {
        if (std.mem.startsWith(u8, line, "# ")) {
            // H1 标题
            try html.writer().writeAll("<h1>");
            try html.writer().writeAll(line[2..]);
            try html.writer().writeAll("</h1>\n");
        } else if (std.mem.startsWith(u8, line, "## ")) {
            // H2 标题
            try html.writer().writeAll("<h2>");
            try html.writer().writeAll(line[3..]);
            try html.writer().writeAll("</h2>\n");
        } else if (line.len > 0) {
            // 普通段落
            try html.writer().writeAll("<p>");
            try html.writer().writeAll(line);
            try html.writer().writeAll("</p>\n");
        }
    }
    
    return html.toOwnedSlice();
}

test "markdown heading conversion" {
    const allocator = std.testing.allocator;
    const markdown = "# Hello World";
    
    const html = try markdownToHtml(markdown, allocator);
    defer allocator.free(html);
    
    try std.testing.expectEqualStrings("<h1>Hello World</h1>\n", html);
}

test "markdown paragraph conversion" {
    const allocator = std.testing.allocator;
    const markdown = "This is a paragraph.";
    
    const html = try markdownToHtml(markdown, allocator);
    defer allocator.free(html);
    
    try std.testing.expectEqualStrings("<p>This is a paragraph.</p>\n", html);
}
```

## 14.1.3 测试组织

### 文件内测试组织

```zig
// Blogger.zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// ===== 类型定义 =====
pub const Post = struct {
    // ...
};

pub const Blogger = struct {
    // ...
};

// ===== 公共函数 =====
pub fn generateBlog() void {
    // ...
}

// ===== 内部函数 =====
fn parseFrontmatter() void {
    // ...
}

// ===== 测试（放在文件末尾） =====
test "Blogger.postCount" {
    // ...
}

test "Blogger.generateIndex" {
    // ...
}
```

### 独立测试文件

```
src/
├── Blogger.zig          # 源代码
├── Blogger_test.zig     # 独立测试文件
└── utils.zig
└── utils_test.zig
```

```zig
// Blogger_test.zig
const std = @import("std");
const Blogger = @import("Blogger.zig").Blogger;

test "Blogger initialization" {
    const allocator = std.testing.allocator;
    const blogger = try Blogger.init(allocator, "./posts", "./output");
    defer blogger.deinit();
    
    try std.testing.expectEqualStrings("./posts", blogger.posts_dir);
    try std.testing.expectEqualStrings("./output", blogger.dest_dir);
}
```

## 14.1.4 测试运行

### 运行所有测试

```bash
# 运行项目中的所有测试
zig build test

# 带详细输出
zig build test -- --verbose
```

### 运行特定测试

```bash
# 运行包含特定字符串的测试
zig test src/Blogger.zig --test-filter "post"

# 运行单个测试文件
zig test src/Blogger_test.zig
```

### 测试输出

```
$ zig build test
1/5 test "Post creation and cleanup"... OK
2/5 test "parse frontmatter with title"... OK
3/5 test "parse frontmatter missing delimiter"... OK
4/5 test "markdown heading conversion"... OK
5/5 test "markdown paragraph conversion"... OK

All 5 tests passed.
```

## 14.1.5 测试最佳实践

### 1. 测试命名规范

```zig
// ✅ 好的命名：描述行为
test "parse frontmatter with valid title" { }
test "parse frontmatter missing required field" { }
test "generate html with proper escaping" { }

// ❌ 差的命名：太模糊
test "test1" { }
test "parser test" { }
test "should work" { }
```

### 2. 测试隔离

```zig
// ✅ 每个测试使用独立的分配器
test "test with allocator" {
    const allocator = std.testing.allocator;
    // 使用 allocator 分配内存
    // 测试结束后自动释放
}

// ❌ 避免共享状态
var shared_state: i32 = 0;  // 不要在测试间共享状态

test "test1" {
    shared_state = 1;  // 会影响其他测试
}
```

### 3. 测试边界条件

```zig
test "empty input" {
    const result = try processContent("");
    try std.testing.expectEqualStrings("", result);
}

test "single character" {
    const result = try processContent("a");
    try std.testing.expectEqualStrings("A", result);
}

test "very long input" {
    var long_input: [10000]u8 = undefined;
    @memset(&long_input, 'a');
    const result = try processContent(&long_input);
    try std.testing.expect(result.len > 0);
}

test "special characters" {
    const result = try processContent("<>&\"'");
    try std.testing.expectEqualStrings("&lt;&gt;&amp;&quot;&apos;", result);
}
```

### 4. 测试错误处理

```zig
test "error on invalid input" {
    try std.testing.expectError(
        error.InvalidInput,
        processInvalidContent()
    );
}

test "error propagation" {
    const allocator = std.testing.allocator;
    try std.testing.expectError(
        error.OutOfMemory,
        allocateTooMuch(allocator)
    );
}
```

## 14.1.6 表格驱动测试

```zig
// 使用表格驱动测试多个案例
test "markdown bold conversion" {
    const TestCase = struct {
        input: []const u8,
        expected: []const u8,
    };
    
    const test_cases = [_]TestCase{
        .{ .input = "**bold**", .expected = "<strong>bold</strong>" },
        .{ .input = "**multi word**", .expected = "<strong>multi word</strong>" },
        .{ .input = "**nested *italic***", .expected = "<strong>nested *italic*</strong>" },
    };
    
    inline for (test_cases) |tc| {
        const result = try convertBold(tc.input, std.testing.allocator);
        defer std.testing.allocator.free(result);
        try std.testing.expectEqualStrings(tc.expected, result);
    }
}
```

## 14.1.7 测试覆盖率

### 启用覆盖率检查

```bash
# Zig 目前没有内置覆盖率工具
# 可以使用第三方工具如 kcov

# 编译测试二进制
zig test src/Blogger.zig --test-no-exec -femit-bin=test-binary

# 使用 kcov 运行
kcov coverage/ ./test-binary
```

### 手动覆盖率检查

```zig
// 添加覆盖率标记
var test_coverage = struct {
    parse_frontmatter: bool = false,
    generate_html: bool = false,
    copy_assets: bool = false,
}{};

test "parse frontmatter" {
    test_coverage.parse_frontmatter = true;
    // ... 测试逻辑
}

test "generate html" {
    test_coverage.generate_html = true;
    // ... 测试逻辑
}

test "check coverage" {
    try std.testing.expect(test_coverage.parse_frontmatter);
    try std.testing.expect(test_coverage.generate_html);
    try std.testing.expect(test_coverage.copy_assets);
}
```

## 相关文档

- [14.2 调试技巧](./14-2-debugging-techniques.md) - 调试方法和工具
- [14.3 内存泄漏检测](./14-3-memory-leak-detection.md) - 内存管理测试
- [9.3 错误处理模式](../part-09-error-handling/09-3-error-handling-patterns.md) - 错误处理测试

---

*本文档介绍了 Zig 单元测试的基础知识，包括测试块语法、断言函数、测试组织和最佳实践，以及如何在 my-blog 项目中编写有效的测试。*
