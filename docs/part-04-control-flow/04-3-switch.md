# 04-3 switch 表达式

> Zig 语言的多分支选择与项目实例分析

---

## 1. switch 基础

### 1.1 基本语法

Zig 的 `switch` 是一个**表达式**，用于根据值选择多个分支之一：

```zig
const value: u32 = 2;
const result = switch (value) {
    1 => "one",
    2 => "two",
    3 => "three",
    else => "other",
};
```

### 1.2 switch 是表达式

与 C 语言不同，Zig 的 `switch` 是表达式而不是语句，它**返回值**：

```zig
// switch 必须覆盖所有可能的值
const x = switch (day) {
    1 => "Monday",
    2 => "Tuesday",
    else => "Other day",
};
```

### 1.3 else 分支的强制性

除非 `switch` 覆盖了所有可能的值，否则必须包含 `else` 分支：

```zig
// 错误 - 未覆盖所有情况
const result = switch (x) {
    1 => "one",
    2 => "two",
    // 缺少 else 或其他分支
};

// 正确
const result = switch (x) {
    1 => "one",
    2 => "two",
    else => "other",
};
```

---

## 2. 项目中的 switch 使用分析

### 2.1 项目现状

在 my-blog 项目中，`switch` 表达式的使用相对较少。项目主要使用 `if/else if` 链来处理多分支逻辑。

**原因分析**：

1. **命令行参数解析** - 使用字符串比较，适合 `if/else if` 链
2. **文件类型检查** - 使用简单的 `if (entry.kind == .file)`
3. **frontmatter 解析** - 使用 `if/else if` 检查字段名

### 2.2 潜在的 switch 使用场景

虽然项目当前未使用 `switch`，但以下场景适合使用：

#### 场景 1：文章状态处理

```zig
pub const PostStatus = enum {
    draft,
    published,
    archived,
};

const status: PostStatus = .published;

switch (status) {
    .draft => std.debug.print("Draft post\n", .{}),
    .published => std.debug.print("Published post\n", .{}),
    .archived => std.debug.print("Archived post\n", .{}),
}
```

#### 场景 2：错误类型处理

```zig
someOperation() catch |err| switch (err) {
    error.FileNotFound => std.debug.print("File not found\n", .{}),
    error.InvalidFormat => std.debug.print("Invalid format\n", .{}),
    else => |e| std.debug.print("Other error: {}\n", .{e}),
};
```

---

## 3. switch 与 enum 配合

### 3.1 枚举类型定义

```zig
pub const FileKind = enum {
    file,
    directory,
    symlink,
};
```

### 3.2 switch 处理枚举

```zig
fn handleFile(kind: FileKind) void {
    switch (kind) {
        .file => std.debug.print("Regular file\n", .{}),
        .directory => std.debug.print("Directory\n", .{}),
        .symlink => std.debug.print("Symbolic link\n", .{}),
    }
}
```

### 3.3 项目中的 enum 使用

**项目实例** - 虽然项目未显式定义 enum，但标准库的 `std.fs.Dir.Entry.Kind` 是枚举类型：

```zig
// std.fs.Dir.Entry.Kind 是一个枚举
if (entry.kind == .file) {
    // 处理文件
} else if (entry.kind == .directory) {
    // 处理目录
}

// 使用 switch 的等价写法
switch (entry.kind) {
    .file => {
        // 处理文件
    },
    .directory => {
        // 处理目录
    },
    else => {
        // 其他类型（symlink 等）
    },
}
```

---

## 4. switch 的高级用法

### 4.1 多值匹配

一个分支可以匹配多个值：

```zig
const result = switch (x) {
    1, 2, 3 => "small",
    4, 5, 6 => "medium",
    else => "large",
};
```

### 4.2 值范围匹配

```zig
const grade = switch (score) {
    90..100 => "A",
    80..89 => "B",
    70..79 => "C",
    60..69 => "D",
    else => "F",
};
```

### 4.3 带代码块的分支

分支可以包含代码块，返回最后一个表达式的值：

```zig
const result = switch (option) {
    1 => blk: {
        const temp = computeValue();
        break :blk temp * 2;
    },
    else => 0,
};
```

---

## 5. switch 与错误集

### 5.1 错误捕获 switch

Zig 允许在 `catch` 表达式中使用 `switch` 处理不同类型的错误：

```zig
operation() catch |err| switch (err) {
    error.FileNotFound => {
        std.debug.print("File not found\n", .{});
        return error.MissingFile;
    },
    error.InvalidFormat => {
        std.debug.print("Invalid format\n", .{});
        return error.BadData;
    },
    else => |e| {
        std.debug.print("Unknown error: {}\n", .{e});
        return e;
    },
};
```

### 5.2 项目中的错误处理对比

**项目实例** - `loadPosts()` 使用简单的 `catch |err|`：

```zig
const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| {
    std.debug.print(" Skipped ({s}): {s}\n", .{ @errorName(err), entry.path });
    continue;  // 统一处理所有错误：跳过并继续
};
```

**使用 switch 的改进版本**：

```zig
const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| switch (err) {
    error.InvalidFormat => {
        std.debug.print(" Skipped (invalid format): {s}\n", .{entry.path});
        continue;
    },
    error.MissingTitle => {
        std.debug.print(" Skipped (missing title): {s}\n", .{entry.path});
        continue;
    },
    error.MissingDateTime => {
        std.debug.print(" Skipped (missing date): {s}\n", .{entry.path});
        continue;
    },
    else => |e| {
        std.debug.print(" Skipped (unknown error {s}): {s}\n", .{ @errorName(e), entry.path });
        continue;
    },
};
```

---

## 6. if/else if vs switch

### 6.1 何时使用 if/else if

**项目实例** - 字符串比较场景：

```zig
// 适合 if/else if - 字符串比较
if (std.mem.eql(u8, arg, "--posts") or std.mem.eql(u8, arg, "-p")) {
    posts_dir = args.next() orelse return;
} else if (std.mem.eql(u8, arg, "--output") or std.mem.eql(u8, arg, "-o")) {
    dest_dir = args.next() orelse return;
} else if (std.mem.eql(u8, arg, "--help") or std.mem.eql(u8, arg, "-h")) {
    printHelp();
    return;
}
```

**原因**：
- `switch` 不支持字符串直接匹配
- `switch` 要求编译时已知的值
- 字符串比较需要运行时计算

### 6.2 何时使用 switch

**适合 switch 的场景**：

```zig
// 1. 枚举值匹配
pub const BuildMode = enum { debug, release, fast };
const mode: BuildMode = .release;
switch (mode) {
    .debug => { },
    .release => { },
    .fast => { },
}

// 2. 整数状态码
switch (status_code) {
    200 => "OK",
    404 => "Not Found",
    500 => "Server Error",
    else => "Unknown",
}

// 3. 错误类型分类
catch |err| switch (err) {
    error.FileNotFound => { },
    error.PermissionDenied => { },
    else => |e| { },
}
```

---

## 7. 完整实例：配置文件解析器

虽然 my-blog 项目未使用 `switch`，但以下是一个完整的使用实例：

```zig
pub const ConfigKey = enum {
    title,
    author,
    posts_dir,
    output_dir,
    theme,
};

pub const ConfigValue = union(enum) {
    string: []const u8,
    number: u32,
    boolean: bool,
};

fn parseConfig(key: ConfigKey, value: ConfigValue) void {
    switch (key) {
        .title, .author => {
            if (value == .string) {
                std.debug.print("{s}: {s}\n", .{ @tagName(key), value.string });
            }
        },
        .posts_dir, .output_dir => {
            if (value == .string) {
                std.debug.print("Directory: {s}\n", .{value.string});
            }
        },
        .theme => {
            switch (value) {
                .string => |theme| {
                    if (std.mem.eql(u8, theme, "dark") or std.mem.eql(u8, theme, "light")) {
                        std.debug.print("Theme: {s}\n", .{theme});
                    } else {
                        std.debug.print("Invalid theme: {s}\n", .{theme});
                    }
                },
                else => std.debug.print("Theme must be a string\n", .{}),
            }
        },
    }
}
```

---

## 8. 模式匹配与 anytype

### 8.1 使用 switch 处理联合 (Union)

```zig
const Value = union(enum) {
    int: i32,
    float: f64,
    text: []const u8,
};

fn printValue(v: Value) void {
    switch (v) {
        .int => |n| std.debug.print("Integer: {d}\n", .{n}),
        .float => |f| std.debug.print("Float: {d}\n", .{f}),
        .text => |s| std.debug.print("Text: {s}\n", .{s}),
    }
}
```

### 8.2 项目中的潜在应用

如果项目扩展支持多种内容类型：

```zig
pub const Content = union(enum) {
    markdown: []const u8,
    html: []const u8,
    json: []const u8,
};

fn renderContent(content: Content) ![]const u8 {
    return switch (content) {
        .markdown => |md| try markdown.toHtml(allocator, md),
        .html => |html| html,
        .json => |json| try jsonToHtml(allocator, json),
    };
}
```

---

## 9. 注意事项

### 9.1 switch 必须穷尽

```zig
// 错误 - 未覆盖所有情况
const x: u8 = 5;
switch (x) {
    1 => "one",
    2 => "two",
    // 编译错误：未覆盖 0, 3-255
}

// 正确
switch (x) {
    1 => "one",
    2 => "two",
    else => "other",
}
```

### 9.2 switch 不支持字符串

```zig
// 错误
const s = "hello";
switch (s) {
    "hello" => "greeting",  // 不支持
    else => "unknown",
}

// 正确 - 使用 if/else if
if (std.mem.eql(u8, s, "hello")) {
    // ...
} else if (std.mem.eql(u8, s, "world")) {
    // ...
}
```

### 9.3 switch 分支不能有 fallthrough

与 C 不同，Zig 的 `switch` 每个分支自动 `break`：

```zig
// 错误 - Zig 不支持 fallthrough
switch (x) {
    1 => {
        std.debug.print("one\n", .{});
        // 不能 fallthrough 到下一个分支
    },
    2 => { },
}
```

---

## 10. 总结对比

| 特性 | if/else if | switch |
|------|-----------|--------|
| 字符串比较 | ✓ 支持 | ✗ 不支持 |
| 范围匹配 | ✓ 支持 | ✓ 支持 (`..`) |
| 枚举匹配 | ✓ 支持 | ✓ 支持（更简洁） |
| 错误处理 | ✓ `catch \|err\|` | ✓ `catch\|err\| switch` |
| 返回值 | ✓ 表达式 | ✓ 表达式 |
| 多值匹配 | ✗ 需要 `or` | ✓ 支持 (`1, 2, 3 =>`) |
| 编译时检查 | ✗ 运行时 | ✓ 穷尽性检查 |

---

## 相关文档

- [04-1 条件语句 (if/else)](./04-1-if-else.md) - 条件控制流基础
- [04-4 comptime](./04-4-comptime.md) - 编译时执行
- [03-2 重要类型](../part-03-zig-basics/03-2-important-types.md) - 枚举类型详解
