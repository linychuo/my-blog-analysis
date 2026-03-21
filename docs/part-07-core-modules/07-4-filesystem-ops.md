# 07-4 文件系统操作

> Zig 标准库文件系统 API 在项目中的应用

---

## 1. 文件系统 API 概览

### 1.1 核心类型

Zig 标准库的文件系统 API 主要围绕以下类型构建：

| 类型 | 用途 |
|------|------|
| `std.fs.Dir` | 目录句柄，表示打开的目录 |
| `std.fs.File` | 文件句柄，表示打开的文件 |
| `std.fs.cwd()` | 获取当前工作目录 |
| `std.fs.path` | 路径操作工具 |

### 1.2 项目中的使用场景

| 场景 | 使用的 API | 位置 |
|------|-----------|------|
| 打开目录 | `openDir()` | `loadPosts()`, `copyStaticFiles()` |
| 遍历目录 | `walk()` | `loadPosts()`, `copyStaticFiles()` |
| 创建目录 | `makePath()` | `generate()`, `writeHtmlFileWithDate()` |
| 打开文件 | `openFile()` | `loadPosts()`, `generateAbout()` |
| 创建文件 | `createFile()` | `writeHtmlFileWithDate()`, `generateIndex()` |
| 读取文件 | `readToEndAlloc()` | `loadPosts()`, `copyStaticFiles()` |
| 写入文件 | `writeAll()` | `writeHtmlFileWithDate()` |
| 路径操作 | `path.join()`, `path.dirname()` | 多处使用 |

---

## 2. 目录操作

### 2.1 获取当前工作目录

```zig
// 获取当前工作目录
const cwd = std.fs.cwd();

// 创建输出目录
try std.fs.cwd().makePath(self.dest_dir);
```

**cwd() 返回**：`std.fs.Dir` 类型，表示当前工作目录

### 2.2 打开目录

```zig
fn copyStaticFiles(self: *Blogger) !void {
    const static_dir = "static";
    var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
        std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
        return;
    };
    defer dir.close();
    // ...
}
```

**openDir() 参数**：

```zig
fn openDir(path: []const u8, options: OpenDirOptions) !Dir
```

**OpenDirOptions 字段**：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `iterate` | `bool` | `false` | 是否需要遍历目录 |
| `no_follow` | `bool` | `false` | 不跟随符号链接 |

**项目中的使用**：

```zig
// 需要遍历
var dir = try std.fs.cwd().openDir(self.posts_dir, .{ .iterate = true });

// 不需要遍历
var dir = try std.fs.cwd().openDir("templates", .{});
```

### 2.3 创建目录

```zig
// 创建单个目录
try std.fs.cwd().makePath("output");

// 创建嵌套目录
try std.fs.cwd().makePath("2026/03/21");
```

**makePath() 特点**：

| 特点 | 说明 |
|------|------|
| 递归创建 | 自动创建所有父目录 |
| 幂等操作 | 目录已存在时不报错 |
| 跨平台 | Windows/Linux/macOS 行为一致 |

**项目中的使用**：

```zig
// 在 generate() 中
try std.fs.cwd().makePath(self.dest_dir);

// 在 writeHtmlFileWithDate() 中
const dir_path = std.fs.path.dirname(output_path.items);
if (dir_path) |dp| {
    try std.fs.cwd().makePath(dp);
}
```

---

## 3. 目录遍历

### 3.1 walk() API

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    var dir = try std.fs.cwd().openDir(self.posts_dir, .{ .iterate = true });
    defer dir.close();

    var walker = try dir.walk(self.allocator);
    defer walker.deinit();

    while (try walker.next()) |entry| {
        // 处理每个条目
    }
}
```

**Walker 类型**：`std.fs.Dir.Walker`

**Walker 方法**：

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `next()` | `!?Entry` | 获取下一个条目 |
| `deinit()` | `void` | 清理 walker 资源 |

### 3.2 Entry 结构

```zig
while (try walker.next()) |entry| {
    if (entry.kind == .file) {
        // 处理文件
    }
}
```

**Entry 字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `kind` | `Entry.Kind` | 条目类型（文件/目录/符号链接） |
| `path` | `[]const u8` | 相对路径 |

**Entry.Kind 枚举**：

```zig
pub const Kind = enum {
    directory,
    file,
    symlink,
    // ...
};
```

### 3.3 遍历结果示例

```
posts/
├── about.markdown       → entry.path = "about.markdown", kind = .file
├── first-post.markdown  → entry.path = "first-post.markdown", kind = .file
└── drafts/
    └── wip.markdown     → entry.path = "drafts/wip.markdown", kind = .file
```

**walker 遍历顺序**：

```
[1] about.markdown
[2] first-post.markdown
[3] drafts/wip.markdown  (递归进入子目录)
```

---

## 4. 文件操作

### 4.1 打开文件

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    // ...
    while (try walker.next()) |entry| {
        if (entry.kind == .file) {
            const file = try dir.openFile(entry.path, .{});
            defer file.close();
            
            const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);
            
            // 处理文件内容
        }
    }
}
```

**openFile() 签名**：

```zig
fn openFile(self: *Dir, sub_path: []const u8, options: OpenFileOptions) !File
```

**常用选项**：

```zig
// 默认选项
dir.openFile("file.txt", .{});

// 只读模式
dir.openFile("file.txt", .{ .mode = .read_only });

// 写入模式
dir.openFile("file.txt", .{ .mode = .write_only });
```

### 4.2 创建文件

```zig
fn writeHtmlFileWithDate(self: *Blogger, markdown_filename: []const u8, date_time: []const u8, html: []const u8) !void {
    // ... 构建 output_path
    
    const file = try std.fs.cwd().createFile(output_path.items, .{});
    defer file.close();
    try file.writeAll(html);
}
```

**createFile() 签名**：

```zig
fn createFile(self: *Dir, sub_path: []const u8, options: CreateFileOptions) !File
```

**CreateFileOptions 字段**：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `read` | `bool` | `false` | 是否允许读取 |
| `truncate` | `bool` | `true` | 是否截断已存在的文件 |

### 4.3 读取文件

```zig
// 读取整个文件到分配的内存
const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
defer self.allocator.free(content);
```

**readToEndAlloc() 参数**：

```zig
fn readToEndAlloc(self: *File, allocator: Allocator, max_size: usize) ![]u8
```

| 参数 | 说明 |
|------|------|
| `allocator` | 用于分配内存的分配器 |
| `max_size` | 最大读取字节数（防止 DoS） |

**项目中的 max_size**：

```zig
// 限制 1MB
const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
```

### 4.4 写入文件

```zig
const file = try std.fs.cwd().createFile(output_path.items, .{});
defer file.close();
try file.writeAll(html);
```

**writeAll() 特点**：

| 特点 | 说明 |
|------|------|
| 原子写入 | 要么全部写入，要么返回错误 |
| 阻塞操作 | 直到所有数据写入完成 |
| 错误处理 | 磁盘满、权限不足等返回错误 |

---

## 5. 路径操作

### 5.1 path.join()

```zig
const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
defer self.allocator.free(src_path);
```

**签名**：

```zig
fn join(allocator: Allocator, paths: []const []const u8) ![]const u8
```

**示例**：

```zig
// Linux/macOS
path.join(allocator, &.{ "static", "css", "style.css" })
    → "static/css/style.css"

// Windows
path.join(allocator, &.{ "static", "css", "style.css" })
    → "static\\css\\style.css"
```

### 5.2 path.dirname()

```zig
const dest_dir_path = std.fs.path.dirname(dest_path);
if (dest_dir_path) |dir_path| {
    try std.fs.cwd().makePath(dir_path);
}
```

**返回值**：`?[]const u8`（可选类型）

**示例**：

```zig
dirname("foo/bar/baz.txt") → "foo/bar"
dirname("file.txt")        → ""
dirname("/")               → null
```

### 5.3 path.basename()

```zig
if (std.mem.eql(u8, std.fs.path.basename(entry.path), "about.markdown")) {
    continue;
}
```

**示例**：

```zig
basename("foo/bar/baz.txt") → "baz.txt"
basename("file.txt")        → "file.txt"
basename("/dir/")           → "dir"
```

---

## 6. 完整实例分析

### 6.1 copyStaticFiles() 完整流程

```zig
fn copyStaticFiles(self: *Blogger) !void {
    // [1] 打开 static 目录
    const static_dir = "static";
    var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
        std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
        return;
    };
    defer dir.close();

    // [2] 创建输出目录
    try std.fs.cwd().makePath(self.dest_dir);

    // [3] 初始化 walker
    var walker = try dir.walk(self.allocator);
    defer walker.deinit();

    // [4] 遍历目录
    while (try walker.next()) |entry| {
        if (entry.kind == .file) {
            // [5] 构建源路径和目标路径
            const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
            defer self.allocator.free(src_path);
            const dest_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, entry.path });
            defer self.allocator.free(dest_path);

            // [6] 创建父目录
            const dest_dir_path = std.fs.path.dirname(dest_path);
            if (dest_dir_path) |dir_path| {
                try std.fs.cwd().makePath(dir_path);
            }

            // [7] 打开源文件
            var src_file = try dir.openFile(entry.path, .{});
            defer src_file.close();
            
            // [8] 创建目标文件
            const dest_file = try std.fs.cwd().createFile(dest_path, .{});
            defer dest_file.close();
            
            // [9] 读取并写入内容
            const content = try src_file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);
            try dest_file.writeAll(content);

            std.debug.print(" Copied: {s}\n", .{entry.path});
        }
    }
}
```

### 6.2 writeHtmlFileWithDate() 日期目录

```zig
fn writeHtmlFileWithDate(self: *Blogger, markdown_filename: []const u8, date_time: []const u8, html: []const u8) !void {
    // [1] 解析日期
    const year = date_time[0..4];  // "2026"
    // ... 解析 month 和 day
    
    // [2] 构建输出路径
    var output_path = std.ArrayList(u8){};
    defer output_path.deinit(self.allocator);
    try output_path.appendSlice(self.allocator, self.dest_dir);
    try output_path.appendSlice(self.allocator, "/");
    try output_path.appendSlice(self.allocator, year);
    try output_path.appendSlice(self.allocator, "/");
    try output_path.appendSlice(self.allocator, month);
    try output_path.appendSlice(self.allocator, "/");
    try output_path.appendSlice(self.allocator, day);
    try output_path.appendSlice(self.allocator, "/");
    try output_path.appendSlice(self.allocator, output_filename.items);

    // [3] 创建目录结构
    const dir_path = std.fs.path.dirname(output_path.items);
    if (dir_path) |dp| {
        try std.fs.cwd().makePath(dp);
    }

    // [4] 创建并写入文件
    const file = try std.fs.cwd().createFile(output_path.items, .{});
    defer file.close();
    try file.writeAll(html);
}
```

**输出路径示例**：

```
输入：
  markdown_filename = "my-first-post.markdown"
  date_time = "2026-03-21"
  dest_dir = "zig-out/blog"

输出：
  zig-out/blog/2026/03/21/my-first-post.html
```

---

## 7. 错误处理模式

### 7.1 优雅降级

```zig
var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
    std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
    return;  // 不中断程序，继续执行
};
```

### 7.2 文件不存在处理

```zig
const file = std.fs.cwd().openFile(about_path, .{}) catch {
    std.debug.print(" Skipped: about.markdown not found\n", .{});
    return;  // 关于页面可选
};
```

### 7.3 错误传播

```zig
// 关键操作失败时传播错误
try std.fs.cwd().makePath(self.dest_dir);
// 如果失败，generate() 立即返回错误
```

---

## 8. 资源管理

### 8.1 defer 清理模式

```zig
// 目录句柄
var dir = try std.fs.cwd().openDir(path, .{});
defer dir.close();

// 文件句柄
var file = try dir.openFile(entry.path, .{});
defer file.close();

// Walker
var walker = try dir.walk(self.allocator);
defer walker.deinit();

// 动态路径
const src_path = try std.fs.path.join(self.allocator, &.{ ... });
defer self.allocator.free(src_path);
```

### 8.2 内存管理

```zig
// 读取文件内容
const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
defer self.allocator.free(content);

// 构建路径
const dest_path = try std.fs.path.join(self.allocator, &.{ ... });
defer self.allocator.free(dest_path);
```

---

## 9. 性能考虑

### 9.1 批量操作

```zig
// 不推荐：逐个文件打开/关闭
for (files) |file| {
    var f = try openFile(file);
    // ...
    f.close();
}

// 推荐：保持文件打开直到完成
// 项目中使用 defer 自动管理
```

### 9.2 内存限制

```zig
// 设置最大读取大小，防止大文件耗尽内存
const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);  // 1MB 限制
```

---

## 10. 跨平台考虑

### 10.1 路径分隔符

```zig
// path.join() 自动处理
// Linux/macOS: "foo/bar"
// Windows:     "foo\\bar"
```

### 10.2 大小写敏感性

```zig
// Linux: 文件名大小写敏感
// Windows/macOS: 文件名大小写不敏感

// 项目假设：文件名完全匹配
if (std.mem.eql(u8, std.fs.path.basename(entry.path), "about.markdown")) {
    continue;
}
```

---

## 相关文档

- [07-3 Blogger 结构体](./07-3-blogger-struct.md) - 使用文件系统 API
- [06-2 std.Build API](../part-06-build-system/06-2-std-build-api.md) - 构建系统中的路径处理
- [08-1 内存分配器](../part-08-memory-management/08-1-allocators.md) - 内存分配与文件读取
