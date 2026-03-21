# 04-1 条件语句 (if/else)

> Zig 语言的条件控制流与项目实例分析

---

## 1. if 语句基础

### 1.1 基本语法

Zig 的 `if` 语句用于根据条件执行不同的代码分支：

```zig
if (condition) {
    // 条件为真时执行
} else {
    // 条件为假时执行
}
```

**项目实例** - `Blogger.zig` 中验证 frontmatter 格式：

```zig
const first_line = lines.next() orelse return error.InvalidFormat;
if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
    return error.InvalidFormat;
}
```

### 1.2 if 表达式

在 Zig 中，`if` 是一个**表达式**，可以返回值：

```zig
const value = if (x > 0) 1 else -1;
```

**项目实例** - `writeHtmlFileWithDate` 中处理月份填充：

```zig
const month = if (month_end - pos == 1) blk: {
    month_buf[0] = '0';
    month_buf[1] = date_time[pos];
    break :blk month_buf[0..2];
} else date_time[pos..month_end];
```

---

## 2. if/else if 链

### 2.1 多条件分支

当需要处理多个条件时，使用 `if/else if` 链：

```zig
if (condition1) {
    // 条件 1 为真
} else if (condition2) {
    // 条件 2 为真
} else if (condition3) {
    // 条件 3 为真
} else {
    // 所有条件都为假
}
```

**项目实例** - `main.zig` 中解析命令行参数：

```zig
while (args.next()) |arg| {
    if (std.mem.eql(u8, arg, "--posts") or std.mem.eql(u8, arg, "-p")) {
        posts_dir = args.next() orelse {
            std.debug.print("Error: --posts requires a directory argument\n", .{});
            std.process.exit(1);
        };
    } else if (std.mem.eql(u8, arg, "--output") or std.mem.eql(u8, arg, "-o")) {
        dest_dir = args.next() orelse {
            std.debug.print("Error: --output requires a directory argument\n", .{});
            std.process.exit(1);
        };
    } else if (std.mem.eql(u8, arg, "--help") or std.mem.eql(u8, arg, "-h")) {
        printHelp();
        return;
    }
}
```

### 2.2 项目中的 if/else if 模式

**项目实例** - `Post.parse()` 解析 frontmatter 字段：

```zig
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

---

## 3. if 与 orelse 组合

### 3.1 orelse 运算符

`orelse` 用于处理可选类型 (`?T`)，当值为 `null` 时提供备选方案：

```zig
const value: ?u32 = null;
const result = value orelse 0;  // 如果 value 为 null，则 result = 0
```

### 3.2 orelse 返回错误

**项目实例** - `Post.parse()` 中验证必需字段：

```zig
return Post{
    .title = title orelse return error.MissingTitle,
    .date_time = date_time orelse return error.MissingDateTime,
    .tags = tags orelse "",  // tags 可选，提供空字符串默认值
};
```

### 3.3 orelse 代码块

`orelse` 后面可以跟一个代码块，用于复杂逻辑：

**项目实例** - `main.zig` 中处理参数缺失：

```zig
posts_dir = args.next() orelse {
    std.debug.print("Error: --posts requires a directory argument\n", .{});
    std.process.exit(1);
};
```

---

## 4. 文件类型检查

### 4.1 目录遍历中的条件判断

**项目实例** - `copyStaticFiles()` 中检查文件类型：

```zig
while (try walker.next()) |entry| {
    if (entry.kind == .file) {
        const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
        defer self.allocator.free(src_path);
        // ... 复制文件逻辑
    }
}
```

### 4.2 多条件组合

**项目实例** - `loadPosts()` 中筛选 Markdown 文件：

```zig
while (try walker.next()) |entry| {
    if (entry.kind == .file and std.mem.endsWith(u8, entry.path, ".markdown")) {
        // 排除 about.markdown，它单独处理
        if (std.mem.eql(u8, std.fs.path.basename(entry.path), "about.markdown")) {
            continue;
        }
        // ... 加载文章逻辑
    }
}
```

---

## 5. 错误恢复中的 if/else

### 5.1 优雅降级

**项目实例** - `copyStaticFiles()` 处理缺失的静态目录：

```zig
fn copyStaticFiles(self: *Blogger) !void {
    const static_dir = "static";
    var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
        std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
        return;  // 优雅降级，不生成静态文件但继续运行
    };
    defer dir.close();
    // ...
}
```

### 5.2 错误捕获与继续处理

**项目实例** - `loadPosts()` 跳过解析失败的文章：

```zig
const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| {
    std.debug.print(" Skipped ({s}): {s}\n", .{ @errorName(err), entry.path });
    continue;  // 跳过坏文章，继续处理其他文章
};

try posts.append(post);
```

---

## 6. 字符串比较与条件

### 6.1 使用 std.mem.eql 进行比较

在 Zig 中，字符串比较不能直接用 `==`，需要使用 `std.mem.eql`：

```zig
// 错误写法
if (arg == "--help") { }

// 正确写法
if (std.mem.eql(u8, arg, "--help")) { }
```

**项目实例** - `main.zig` 中多处使用：

```zig
if (std.mem.eql(u8, arg, "--posts") or std.mem.eql(u8, arg, "-p")) {
    // ...
} else if (std.mem.eql(u8, arg, "--help") or std.mem.eql(u8, arg, "-h")) {
    printHelp();
    return;
}
```

### 6.2 检查字符串前缀和后缀

**项目实例** - 使用 `startsWith` 和 `endsWith`：

```zig
// 检查前缀
if (std.mem.startsWith(u8, trimmed, "title:")) {
    title = try extractValue(allocator, trimmed, "title:");
}

// 检查后缀
if (entry.kind == .file and std.mem.endsWith(u8, entry.path, ".markdown")) {
    // 处理 Markdown 文件
}
```

---

## 7. 完整实例分析

### 7.1 writeHtmlFileWithDate 中的条件逻辑

这个函数展示了复杂的条件判断，用于解析日期字符串并创建目录结构：

```zig
fn writeHtmlFileWithDate(self: *Blogger, markdown_filename: []const u8, date_time: []const u8, html: []const u8) !void {
    // [1] 验证日期长度
    if (date_time.len < 8) return;

    // [2] 提取年份
    const year = date_time[0..4];
    
    // [3] 提取月份（处理前导零）
    var pos: usize = 5;
    var month_end = pos;
    while (month_end < date_time.len and date_time[month_end] != '-') : (month_end += 1) {}
    var month_buf: [2]u8 = undefined;
    const month = if (month_end - pos == 1) blk: {
        month_buf[0] = '0';
        month_buf[1] = date_time[pos];
        break :blk month_buf[0..2];
    } else date_time[pos..month_end];

    // [4] 提取日期（处理前导零）
    pos = month_end + 1;
    var day_end = pos;
    while (day_end < date_time.len and date_time[day_end] != ' ' and date_time[day_end] != '-') : (day_end += 1) {}
    var day_buf: [2]u8 = undefined;
    const day = if (day_end - pos == 1) blk: {
        day_buf[0] = '0';
        day_buf[1] = date_time[pos];
        break :blk day_buf[0..2];
    } else date_time[pos..day_end];

    // [5] 构建输出路径
    // ...
}
```

---

## 8. 常见模式总结

| 模式 | 代码示例 | 用途 |
|------|----------|------|
| 验证输入 | `if (input.len == 0) return error.EmptyInput;` | 参数验证 |
| 可选字段 | `field orelse return error.MissingField` | 必需字段检查 |
| 默认值 | `field orelse ""` | 可选字段默认值 |
| 多条件 | `if (cond1) {...} else if (cond2) {...} else {...}` | 多分支选择 |
| 错误恢复 | `operation() catch |err| { log(err); continue; }` | 容错处理 |
| 类型检查 | `if (entry.kind == .file)` | 文件系统操作 |
| 字符串比较 | `if (std.mem.eql(u8, str, "value"))` | 字符串匹配 |

---

## 9. 注意事项

### 9.1 if 不是语句是表达式

Zig 的 `if` 可以返回值，因此以下代码是合法的：

```zig
const x: u32 = if (true) 1 else 2;
```

### 9.2 条件必须是布尔值

与 C 语言不同，Zig 不允许将整数作为条件：

```zig
// 错误
if (x) { }  // x 必须是 bool 类型

// 正确
if (x != 0) { }
```

### 9.3 else 是必需的（对于可选类型）

当处理可选类型时，必须处理 `null` 情况：

```zig
const opt: ?u32 = null;
const val = opt orelse 0;  // 必须提供 orelse
```

---

## 相关文档

- [03-3 可选类型与错误处理](../part-03-zig-basics/03-3-optional-and-error.md) - orelse 运算符详解
- [04-2 循环 (while/for)](./04-2-while-for.md) - 循环控制流
- [04-3 switch 表达式](./04-3-switch.md) - 多分支选择
