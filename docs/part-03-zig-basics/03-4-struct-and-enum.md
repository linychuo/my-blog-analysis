# 03-4 结构体与枚举

> Zig 的复合类型系统

---

## 1. 结构体 (Struct)

### 1.1 结构体定义

结构体是 Zig 中最基本的复合类型，用于组合多个字段：

```zig
pub const Post = struct {
    title: []const u8,
    date_time: []const u8,
    tags: []const u8,
    content: []const u8,
    filename: []const u8,
};
```

### 1.2 项目中的核心结构体

#### Post 结构体

表示一篇博客文章：

```zig
pub const Post = struct {
    // 字段
    title: []const u8,
    date_time: []const u8,
    tags: []const u8,
    content: []const u8,
    filename: []const u8,

    // 方法：释放内存
    pub fn deinit(self: *Post, allocator: Allocator) void {
        allocator.free(self.title);
        allocator.free(self.date_time);
        allocator.free(self.tags);
        allocator.free(self.content);
        allocator.free(self.filename);
    }

    // 方法：解析 Markdown 文件
    pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
        var lines = std.mem.splitScalar(u8, content, '\n');
        const first_line = lines.next() orelse return error.InvalidFormat;
        
        // 检查 frontmatter 开始标记
        if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
            return error.InvalidFormat;
        }

        var title: ?[]const u8 = null;
        var date_time: ?[]const u8 = null;
        var tags: ?[]const u8 = null;
        var frontmatter_end: usize = 1;

        // 解析 frontmatter
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

    // 私有辅助函数
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

#### Blogger 结构体

博客生成器的核心逻辑：

```zig
pub const Blogger = struct {
    dest_dir: []const u8,
    posts_dir: []const u8,
    allocator: Allocator,
    template_engine: TemplateEngine,

    // 构造函数
    pub fn new(allocator: Allocator, posts_dir: []const u8, dest_dir: []const u8) Blogger {
        const engine = TemplateEngine.init(allocator, "templates");
        return .{
            .allocator = allocator,
            .posts_dir = posts_dir,
            .dest_dir = dest_dir,
            .template_engine = engine,
        };
    }

    // 生成整个网站
    pub fn generate(self: *Blogger) !void {
        std.debug.print("Generating blog...\n", .{});
        try std.fs.cwd().makePath(self.dest_dir);
        try self.copyStaticFiles();

        var posts = std.ArrayList(Post){};
        defer {
            for (posts.items) |*post| post.deinit(self.allocator);
            posts.deinit(self.allocator);
        }

        try self.loadPosts(&posts);
        std.mem.sort(Post, posts.items, {}, sortPostsByDateDesc);

        for (posts.items) |post| try self.generatePostPage(post);
        try self.generateIndex(posts.items);
        try self.generateAbout();
        try self.generateTagPages(posts.items);

        std.debug.print("Blog generation complete!\n", .{});
    }

    // 更多方法...
};
```

### 1.3 结构体实例化

```zig
// 完整实例化
const post = Post{
    .title = try allocator.dupe(u8, "My Blog Post"),
    .date_time = try allocator.dupe(u8, "2026-03-21"),
    .tags = try allocator.dupe(u8, "zig blog"),
    .content = try allocator.dupe(u8, "# Hello"),
    .filename = try allocator.dupe(u8, "post.markdown"),
};

// 使用 .{} 匿名结构体（用于返回值）
return .{
    .allocator = allocator,
    .posts_dir = posts_dir,
    .dest_dir = dest_dir,
};
```

### 1.4 结构体方法

Zig 结构体可以包含方法，方法的第一个参数通常是 `self`：

```zig
pub fn deinit(self: *Post, allocator: Allocator) void {
    // 使用 self 访问字段
    allocator.free(self.title);
}

// 调用方法
post.deinit(allocator);
```

### 1.5 结构体字段访问

```zig
const post = try Post.parse(allocator, content, "post.markdown");

// 访问字段
std.debug.print("Title: {s}\n", .{post.title});

// 修改字段（需要 var）
var mutable_post = post;
mutable_post.title = "New Title";
```

---

## 2. 枚举 (Enum)

### 2.1 枚举定义

枚举是一组命名的常量：

```zig
const FileKind = enum {
    file,
    directory,
    symlink,
};
```

### 2.2 项目中的枚举使用

**文件系统遍历**：

```zig
while (try walker.next()) |entry| {
    // entry.kind 是 FileKind 枚举类型
    if (entry.kind == .file) {
        // 处理文件
        const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
        defer self.allocator.free(src_path);
        
        // 复制文件...
    }
}
```

### 2.3 带值的枚举

```zig
const Status = enum(u8) {
    pending = 0,
    active = 1,
    done = 2,
};
```

### 2.4 枚举方法

```zig
const Color = enum {
    red,
    green,
    blue,

    pub fn toHex(self: Color) []const u8 {
        return switch (self) {
            .red => "#FF0000",
            .green => "#00FF00",
            .blue => "#0000FF",
        };
    }
};
```

---

## 3. 结构体与枚举的组合使用

### 3.1 结构体中包含枚举

```zig
pub const FileEntry = struct {
    path: []const u8,
    kind: std.fs.File.Kind,  // 枚举类型
    size: u64,
};
```

### 3.2 使用 switch 处理枚举

```zig
const kind_name = switch (entry.kind) {
    .file => "file",
    .directory => "directory",
    .sym_link => "symlink",
    else => "unknown",
};
```

---

## 4. 实战：Post 结构体的完整实现

### 4.1 内存管理

```zig
pub fn deinit(self: *Post, allocator: Allocator) void {
    // 释放所有堆分配的内存
    allocator.free(self.title);
    allocator.free(self.date_time);
    allocator.free(self.tags);
    allocator.free(self.content);
    allocator.free(self.filename);
}

// 使用模式
var posts = std.ArrayList(Post){};
defer {
    // 释放每个 Post 的内存
    for (posts.items) |*post| post.deinit(self.allocator);
    // 释放 ArrayList 本身
    posts.deinit(self.allocator);
}
```

### 4.2 解析 Frontmatter

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    var lines = std.mem.splitScalar(u8, content, '\n');
    
    // 检查开始标记
    const first_line = lines.next() orelse return error.InvalidFormat;
    if (!std.mem.eql(u8, std.mem.trim(u8, first_line, " \r\n\t"), "---")) {
        return error.InvalidFormat;
    }

    // 解析字段
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

    // 计算内容起始位置
    const content_start = content_offset_for_line(content, frontmatter_end);
    
    // 构造并返回 Post
    return Post{
        .title = title orelse return error.MissingTitle,
        .date_time = date_time orelse return error.MissingDateTime,
        .tags = tags orelse "",
        .content = try allocator.dupe(u8, content[content_start..]),
        .filename = try allocator.dupe(u8, filename),
    };
}
```

---

## 5. 结构体内存布局

### 5.1 字段顺序影响内存使用

```zig
// Post 结构体的内存布局
// 每个 []const u8 切片 = 16 字节（8 字节指针 + 8 字节长度）
pub const Post = struct {
    title: []const u8,      // 偏移 0，大小 16
    date_time: []const u8,  // 偏移 16，大小 16
    tags: []const u8,       // 偏移 32，大小 16
    content: []const u8,    // 偏移 48，大小 16
    filename: []const u8,   // 偏移 64，大小 16
};  // 总计 80 字节
```

### 5.2 内存对齐

Zig 会自动对齐结构体字段以获得最佳性能：

```zig
const Small = struct {
    a: u8,    // 1 字节
    // 7 字节填充
    b: u64,   // 8 字节
};  // 总计 16 字节（不是 9 字节）
```

---

## 6. 小结

| 概念 | 语法 | 特点 |
|------|------|------|
| 结构体 | `struct {}` | 组合多个字段，可包含方法 |
| 枚举 | `enum {}` | 命名常量集合 |
| 方法 | `pub fn foo(self: *T) void` | 第一个参数是 self |
| 实例化 | `Type{ .field = value }` | 使用命名字段 |
| 匿名结构体 | `.{ .field = value }` | 类型推断 |
| 内存管理 | `defer post.deinit(allocator)` | 手动释放 |

---

## 相关阅读

- [03-1 变量与类型](./03-1-variables-and-types.md)
- [07-2 Blogger 结构体](../part-07-core-modules/07-2-blogger-struct.md) - 核心模块分析
- [07-3 Post 结构体](../part-07-core-modules/07-3-post-struct.md) - 详细解析
