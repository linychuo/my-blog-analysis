# 04-2 循环 (while/for)

> Zig 语言的循环控制流与项目实例分析

---

## 1. while 循环基础

### 1.1 基本语法

Zig 的 `while` 循环在条件为真时重复执行代码：

```zig
while (condition) {
    // 循环体
}
```

**项目实例** - `main.zig` 中遍历命令行参数：

```zig
var args = try std.process.argsWithAllocator(allocator);
defer args.deinit();
_ = args.skip(); // 跳过程序名

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

### 1.2 while 与可选类型捕获

当 `while` 条件返回可选类型时，可以在循环内捕获值：

```zig
while (iterator.next()) |item| {
    // item 的类型是 T，而不是 ?T
}
```

**项目实例** - `Post.parse()` 遍历 frontmatter 行：

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

## 2. while 循环的 continue 表达式

### 2.1 continue 语法

Zig 的 `while` 循环支持 `continue` 表达式，在每次迭代后执行：

```zig
while (condition) : (continue_expression) {
    // 循环体
}
```

**项目实例** - `content_offset_for_line()` 中计算行偏移：

```zig
fn content_offset_for_line(content: []const u8, line_num: usize) usize {
    var offset: usize = 0;
    var current_line: usize = 0;
    while (offset < content.len and current_line < line_num) : (offset += 1) {
        if (content[offset] == '\n') current_line += 1;
    }
    // 跳过尾随换行符
    while (offset < content.len and (content[offset] == '\n' or content[offset] == '\r')) : (offset += 1) {}
    return offset;
}
```

### 2.2 多变量更新

**项目实例** - `writeHtmlFileWithDate()` 中解析日期组件：

```zig
var month_end = pos;
while (month_end < date_time.len and date_time[month_end] != '-') : (month_end += 1) {}

var day_end = pos;
while (day_end < date_time.len and date_time[day_end] != ' ' and date_time[day_end] != '-') : (day_end += 1) {}
```

---

## 3. for 循环基础

### 3.1 遍历数组/切片

`for` 循环用于遍历数组或切片：

```zig
const items = [_]u32{ 1, 2, 3, 4, 5 };
for (items) |item| {
    std.debug.print("{d}\n", .{item});
}
```

### 3.2 带索引的 for 循环

```zig
for (items, 0..) |item, index| {
    std.debug.print("{d}: {d}\n", .{ index, item });
}
```

### 3.3 项目中的 for 循环

**项目实例** - `Blogger.generate()` 遍历文章列表：

```zig
pub fn generate(self: *Blogger) !void {
    // ...
    
    // 清理已加载的文章
    defer {
        for (posts.items) |*post| post.deinit(self.allocator);
        posts.deinit(self.allocator);
    }

    try self.loadPosts(&posts);
    std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

    // 为每篇文章生成页面
    for (posts.items) |post| try self.generatePostPage(post);
    
    try self.generateIndex(posts.items);
    try self.generateAbout();
    try self.generateTagPages(posts.items);
}
```

---

## 4. 目录遍历中的循环

### 4.1 使用 walk 遍历目录树

**项目实例** - `copyStaticFiles()` 复制静态文件：

```zig
fn copyStaticFiles(self: *Blogger) !void {
    const static_dir = "static";
    var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
        std.debug.print(" Warning: static directory not found, skipping static files\n", .{});
        return;
    };
    defer dir.close();

    try std.fs.cwd().makePath(self.dest_dir);
    var walker = try dir.walk(self.allocator);
    defer walker.deinit();

    while (try walker.next()) |entry| {
        if (entry.kind == .file) {
            const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
            defer self.allocator.free(src_path);
            const dest_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, entry.path });
            defer self.allocator.free(dest_path);

            const dest_dir_path = std.fs.path.dirname(dest_path);
            if (dest_dir_path) |dir_path| {
                try std.fs.cwd().makePath(dir_path);
            }

            var src_file = try dir.openFile(entry.path, .{});
            defer src_file.close();
            const dest_file = try std.fs.cwd().createFile(dest_path, .{});
            defer dest_file.close();

            const content = try src_file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);
            try dest_file.writeAll(content);

            std.debug.print(" Copied: {s}\n", .{entry.path});
        }
    }
}
```

### 4.2 加载文章时的循环

**项目实例** - `loadPosts()` 加载所有 Markdown 文件：

```zig
fn loadPosts(self: *Blogger, posts: *std.ArrayList(Post)) !void {
    var dir = try std.fs.cwd().openDir(self.posts_dir, .{ .iterate = true });
    defer dir.close();

    var walker = try dir.walk(self.allocator);
    defer walker.deinit();

    while (try walker.next()) |entry| {
        if (entry.kind == .file and std.mem.endsWith(u8, entry.path, ".markdown")) {
            // 排除 about.markdown
            if (std.mem.eql(u8, std.fs.path.basename(entry.path), "about.markdown")) {
                continue;
            }

            const file = try dir.openFile(entry.path, .{});
            defer file.close();
            const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);

            const post = Post.parse(self.allocator, content, std.fs.path.basename(entry.path)) catch |err| {
                std.debug.print(" Skipped ({s}): {s}\n", .{ @errorName(err), entry.path });
                continue;
            };

            try posts.append(post);
            std.debug.print(" Loaded: {s}\n", .{entry.path});
        }
    }
}
```

---

## 5. 标签 (Tag) 处理中的循环

### 5.1 构建标签映射

**项目实例** - `generateTagPages()` 构建 tag -> posts 映射：

```zig
fn generateTagPages(self: *Blogger, posts: []const Post) !void {
    var tag_map = std.StringHashMap(std.ArrayListUnmanaged(Post)).init(self.allocator);
    defer {
        var it = tag_map.iterator();
        while (it.next()) |entry| {
            entry.value_ptr.deinit(self.allocator);
        }
        tag_map.deinit();
    }

    // 遍历所有文章，构建标签映射
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

    // 遍历标签映射，生成每个标签页面
    var it = tag_map.iterator();
    while (it.next()) |entry| {
        const tag_name = entry.key_ptr.*;
        const tag_posts = entry.value_ptr;
        // ... 生成标签页面
    }
}
```

### 5.2 生成标签 HTML

**项目实例** - `generatePostPage()` 生成标签链接：

```zig
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
try ctx.set("tags", tags_html.items);
```

---

## 6. 日期解析中的循环

### 6.1 解析日期字符串

**项目实例** - `writeHtmlFileWithDate()` 解析 `YYYY-MM-DD` 格式：

```zig
fn writeHtmlFileWithDate(self: *Blogger, markdown_filename: []const u8, date_time: []const u8, html: []const u8) !void {
    if (date_time.len < 8) return;

    const year = date_time[0..4];
    
    // 解析月份
    var pos: usize = 5;
    var month_end = pos;
    while (month_end < date_time.len and date_time[month_end] != '-') : (month_end += 1) {}
    var month_buf: [2]u8 = undefined;
    const month = if (month_end - pos == 1) blk: {
        month_buf[0] = '0';
        month_buf[1] = date_time[pos];
        break :blk month_buf[0..2];
    } else date_time[pos..month_end];

    // 解析日期
    pos = month_end + 1;
    var day_end = pos;
    while (day_end < date_time.len and date_time[day_end] != ' ' and date_time[day_end] != '-') : (day_end += 1) {}
    var day_buf: [2]u8 = undefined;
    const day = if (day_end - pos == 1) blk: {
        day_buf[0] = '0';
        day_buf[1] = date_time[pos];
        break :blk day_buf[0..2];
    } else date_time[pos..day_end];

    // ... 构建输出路径
}
```

---

## 7. 生成索引页面的循环

### 7.1 构建文章列表 HTML

**项目实例** - `generateIndex()` 生成首页文章列表：

```zig
fn generateIndex(self: *Blogger, posts: []const Post) !void {
    var posts_html = std.ArrayList(u8){};
    defer posts_html.deinit(self.allocator);

    for (posts) |post| {
        // [1] 构建文章链接
        var filename = std.ArrayList(u8){};
        defer filename.deinit(self.allocator);

        if (post.date_time.len >= 8) {
            try filename.appendSlice(self.allocator, post.date_time[0..4]);
            try filename.appendSlice(self.allocator, "/");
            
            // 解析月份
            var pos: usize = 5;
            var month_end = pos;
            while (month_end < post.date_time.len and post.date_time[month_end] != '-') : (month_end += 1) {}
            const month = post.date_time[pos..month_end];
            if (month.len == 1) try filename.appendSlice(self.allocator, "0");
            try filename.appendSlice(self.allocator, month);
            try filename.appendSlice(self.allocator, "/");
            
            // 解析日期
            pos = month_end + 1;
            var day_end = pos;
            while (day_end < post.date_time.len and post.date_time[day_end] != ' ' and post.date_time[day_end] != '-') : (day_end += 1) {}
            const day = post.date_time[pos..day_end];
            if (day.len == 1) try filename.appendSlice(self.allocator, "0");
            try filename.appendSlice(self.allocator, day);
            try filename.appendSlice(self.allocator, "/");
        }

        // [2] 处理文件名
        const base_len = post.filename.len;
        if (base_len >= 9 and std.mem.endsWith(u8, post.filename, ".markdown")) {
            try filename.appendSlice(self.allocator, post.filename[0 .. base_len - 9]);
        } else {
            try filename.appendSlice(self.allocator, post.filename);
        }
        try filename.appendSlice(self.allocator, ".html");

        // [3] 构建标签 HTML
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

        // [4] 构建文章条目
        try posts_html.appendSlice(self.allocator, "<article>\n<h2><a href=\"");
        try posts_html.appendSlice(self.allocator, filename.items);
        try posts_html.appendSlice(self.allocator, "\">");
        try posts_html.appendSlice(self.allocator, post.title);
        try posts_html.appendSlice(self.allocator, "</a></h2>\n<time>");
        try posts_html.appendSlice(self.allocator, post.date_time);
        try posts_html.appendSlice(self.allocator, "</time>\n<div class=\"tags\">");
        try posts_html.appendSlice(self.allocator, tags_html.items);
        try posts_html.appendSlice(self.allocator, "</div>\n</article>\n");
    }

    // ... 渲染模板
}
```

---

## 8. 循环模式总结

| 模式 | 代码示例 | 用途 |
|------|----------|------|
| 迭代器循环 | `while (iter.next()) \|item\|` | 标准库迭代器 |
| 目录遍历 | `while (try walker.next()) \|entry\|` | 文件系统遍历 |
| 数组遍历 | `for (items) \|item\|` | 遍历集合 |
| 带 continue | `while (cond) : (i += 1)` | 计数器更新 |
| 嵌套循环 | `for (posts) \|post\| { while (tags...) }` | 标签处理 |
| 提前退出 | `while (...) \|line\| { if (done) break; }` | 解析 frontmatter |
| 跳过迭代 | `if (bad) continue;` | 错误恢复 |

---

## 9. defer 与循环的配合

### 9.1 循环内的 defer

在循环中使用 `defer` 需要注意作用域：

```zig
while (try walker.next()) |entry| {
    if (entry.kind == .file) {
        const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
        defer self.allocator.free(src_path);  // 每次迭代都会释放
        // ...
    }
}
```

### 9.2 循环后的 defer

```zig
var posts = std.ArrayList(Post){};
defer {
    // 循环结束后统一清理
    for (posts.items) |*post| post.deinit(self.allocator);
    posts.deinit(self.allocator);
}
```

---

## 10. 注意事项

### 10.1 while 条件中的错误传播

```zig
// 正确 - 使用 try 传播错误
while (try walker.next()) |entry| {
    // ...
}

// 错误 - 未处理错误
while (walker.next()) |entry| {  // 编译错误
    // ...
}
```

### 10.2 for 循环不能直接修改元素

```zig
// 错误 - 不能修改循环变量
for (items) |item| {
    item += 1;  // 编译错误
}

// 正确 - 使用指针
for (items) |*item| {
    item.* += 1;
}
```

### 10.3 循环中的内存管理

在循环中分配内存时，确保正确释放：

```zig
for (posts) |post| {
    var buffer = std.ArrayList(u8){};
    defer buffer.deinit(self.allocator);  // 每次迭代都会释放
    // ...
}
```

---

## 相关文档

- [04-1 条件语句 (if/else)](./04-1-if-else.md) - 条件控制流
- [04-3 switch 表达式](./04-3-switch.md) - 多分支选择
- [03-3 可选类型与错误处理](../part-03-zig-basics/03-3-optional-and-error.md) - 错误传播
