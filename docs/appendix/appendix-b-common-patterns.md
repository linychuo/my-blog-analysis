# 附录 B: 常用代码模式

> **适用范围**: Zig 0.15.2+  
> **来源**: my-blog 项目实践总结

## B.1 内存管理模式

### 1. 基本分配 - 释放模式

```zig
// 模式：分配后立即 defer 释放
pub fn process(allocator: Allocator) !void {
    const buffer = try allocator.alloc(u8, 1024);
    defer allocator.free(buffer);  // ✅ 标准模式
    
    // 使用 buffer...
}

// 模式：错误时立即释放
pub fn loadData(allocator: Allocator) !Data {
    const ptr = try allocator.create(Data);
    errdefer allocator.destroy(ptr);  // ✅ 错误时清理
    
    // 可能出错的操作...
    
    return ptr;  // 调用者负责释放
}
```

### 2. _owned_ 模式（所有权转移）

```zig
// 模式：函数返回分配的内存，调用者负责释放
pub fn loadFile(allocator: Allocator, path: []const u8) ![]u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();
    
    return try file.readToEndAlloc(allocator, 1024 * 1024);
    // ↑ 返回 owned slice，调用者必须 free
}

// 调用者责任
pub fn useFile() void {
    const allocator = std.heap.page_allocator;
    const content = loadFile(allocator, "data.txt") catch return;
    defer allocator.free(content);  // ✅ 调用者释放
    
    // 使用 content...
}
```

### 3. 双分配器模式

```zig
// 模式：一个分配器用于长期对象，一个用于临时对象
pub fn processPosts(allocator: Allocator, posts: []Post) !void {
    // 长期分配器：存储结果
    var results = std.ArrayList(Result).init(allocator);
    defer results.deinit();
    
    // 临时分配器：每轮迭代重用
    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit();
    
    for (posts) |post| {
        const temp_allocator = arena.allocator();
        
        // 使用临时分配器进行中间计算
        const temp_string = try temp_allocator.alloc(u8, 100);
        // ... 处理 ...
        
        // 将结果复制到长期分配器
        const final_result = try allocator.create(Result);
        try results.append(final_result.*);
        
        // 重置临时分配器（不释放内存，只是重置指针）
        arena.reset(.{ .retain_capacity = true });
    }
}
```

### 4. 结构体析构模式

```zig
// 模式：结构体带 deinit 方法
pub const BlogPost = struct {
    title: []const u8,
    content: []const u8,
    tags: []const u8,
    allocator: Allocator,
    
    pub fn init(allocator: Allocator) !BlogPost {
        return BlogPost{
            .title = try allocator.dupe(u8, "Untitled"),
            .content = try allocator.dupe(u8, ""),
            .tags = try allocator.dupe(u8, ""),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *BlogPost) void {
        self.allocator.free(self.title);
        self.allocator.free(self.content);
        self.allocator.free(self.tags);
    }
};

// 使用
pub fn createPost() void {
    var post = BlogPost.init(std.testing.allocator) catch return;
    defer post.deinit();  // ✅ 统一清理
    
    // 使用 post...
}
```

## B.2 错误处理模式

### 1. 错误转换模式

```zig
// 模式：将底层错误转换为业务错误
pub const MyError = error{
    FileNotFound,
    InvalidFormat,
    OutOfMemory,
};

pub fn loadPost(allocator: Allocator, path: []const u8) MyError!Post {
    const file = std.fs.cwd().openFile(path, .{}) catch |err| switch (err) {
        error.FileNotFound => return MyError.FileNotFound,
        else => return MyError.InvalidFormat,
    };
    defer file.close();
    
    const content = file.readToEndAlloc(allocator, 1024 * 1024) catch |err| switch (err) {
        error.OutOfMemory => return MyError.OutOfMemory,
        else => return MyError.InvalidFormat,
    };
    
    // 解析内容...
}
```

### 2. 错误合并模式

```zig
// 模式：合并多个错误集
pub const ParseError = error{
    InvalidSyntax,
    MissingField,
} || std.fmt.ParseError || std.mem.Allocator.Error;

pub fn parseFrontmatter(text: []const u8) ParseError!Frontmatter {
    // 可能返回多种错误
}
```

### 3. 错误忽略模式

```zig
// 模式：忽略特定错误
pub fn cleanupFile(path: []const u8) void {
    std.fs.cwd().deleteFile(path) catch |err| switch (err) {
        error.FileNotFound => {},  // ✅ 文件不存在是可以接受的
        else => {
            std.debug.print("Failed to delete: {}\n", .{err});
        },
    };
}

// 模式：提供默认值
pub fn getPort(config: ?Config) u16 {
    return config orelse Config{ .port = 8080 }.port;
}

// 模式：catch 默认值
pub fn parseCount(text: []const u8) u32 {
    return std.fmt.parseInt(u32, text, 10) catch 0;  // 解析失败返回 0
}
```

### 4. try-catch 组合模式

```zig
// 模式：try 用于成功路径，catch 用于错误恢复
pub fn robustLoad(allocator: Allocator) !Data {
    // 尝试主路径
    const data = loadFromPrimary(allocator) catch 
        loadFromBackup(allocator) catch 
        return error.FailedToLoad;
    
    return data;
}

// 模式：错误时清理并返回
pub fn processWithCleanup(allocator: Allocator) !Result {
    var resource = try acquireResource();
    errdefer resource.release();  // ✅ 错误时自动清理
    
    const data = try resource.read();
    
    return Result{
        .data = data,
        .resource = resource,  // 成功时转移所有权
    };
}
```

## B.3 字符串处理模式

### 1. 字符串分割

```zig
// 模式：分割逗号分隔的字符串
pub fn parseTags(allocator: Allocator, tags_str: []const u8) ![][]const u8 {
    var tags = std.ArrayList([]const u8).init(allocator);
    errdefer {
        for (tags.items) |tag| allocator.free(tag);
        tags.deinit();
    }
    
    var iter = std.mem.splitScalar(u8, tags_str, ',');
    while (iter.next()) |tag| {
        const trimmed = std.mem.trim(u8, tag, " \t\n\r");
        if (trimmed.len > 0) {
            try tags.append(try allocator.dupe(u8, trimmed));
        }
    }
    
    return tags.toOwnedSlice();
}

// 使用
pub fn useTags() void {
    const allocator = std.testing.allocator;
    const tags = parseTags(allocator, "zig, rust, go") catch return;
    defer {
        for (tags) |tag| allocator.free(tag);
        allocator.free(tags);
    }
    
    for (tags) |tag| {
        std.debug.print("Tag: {s}\n", .{tag});
    }
}
```

### 2. 字符串构建

```zig
// 模式：使用 ArrayList 构建字符串
pub fn buildPath(allocator: Allocator, parts: [][]const u8) ![]const u8 {
    var result = std.ArrayList(u8).init(allocator);
    errdefer result.deinit();
    
    for (parts, 0..) |part, i| {
        if (i > 0) try result.append('/');
        try result.appendSlice(part);
    }
    
    return result.toOwnedSlice();
}

// 模式：使用 bufPrint 到固定缓冲
pub fn formatTitle(title: []const u8, date: []const u8) [256]u8 {
    var buffer: [256]u8 = undefined;
    const result = std.fmt.bufPrint(&buffer, "[{s}] {s}", .{date, title}) catch 
        return undefined;  // 或处理错误
    
    // 返回整个数组（调用者需要注意只使用有效部分）
    return buffer;
}
```

### 3. 字符串插值

```zig
// 模式：使用 allocPrint 进行字符串插值
pub fn createSlug(allocator: Allocator, title: []const u8, date: []const u8) ![]const u8 {
    return try std.fmt.allocPrint(allocator, "{s}-{s}", .{date, title});
}

// 模式：条件格式化
pub fn formatStatus(status: ?Status) []const u8 {
    return switch (status) {
        null => "unknown",
        .active => "active",
        .inactive => "inactive",
    };
}
```

## B.4 集合操作模式

### 1. 过滤和转换

```zig
// 模式：过滤数组
pub fn filterPosts(posts: []Post, allocator: Allocator) ![]Post {
    var filtered = std.ArrayList(Post).init(allocator);
    errdefer filtered.deinit();
    
    for (posts) |post| {
        if (post.published) {
            try filtered.append(post);
        }
    }
    
    return filtered.toOwnedSlice();
}

// 模式：转换数组
pub fn extractTitles(posts: []Post, allocator: Allocator) ![][]const u8 {
    var titles = std.ArrayList([]const u8).init(allocator);
    errdefer {
        for (titles.items) |t| allocator.free(t);
        titles.deinit();
    }
    
    for (posts) |post| {
        try titles.append(try allocator.dupe(u8, post.title));
    }
    
    return titles.toOwnedSlice();
}
```

### 2. 去重模式

```zig
// 模式：使用 HashMap 去重
pub fn uniqueTags(allocator: Allocator, all_tags: [][]const u8) ![][]const u8 {
    var set = std.StringHashMap(void).init(allocator);
    defer set.deinit();
    
    for (all_tags) |tags| {
        for (tags) |tag| {
            try set.put(try allocator.dupe(u8, tag), {});
        }
    }
    
    var result = std.ArrayList([]const u8).init(allocator);
    errdefer {
        for (result.items) |t| allocator.free(t);
        result.deinit();
    }
    
    var it = set.keyIterator();
    while (it.next()) |key| {
        try result.append(key.*);
    }
    
    return result.toOwnedSlice();
}
```

### 3. 分组模式

```zig
// 模式：按标签分组
pub const TagGroup = struct {
    tag: []const u8,
    posts: std.ArrayList(Post),
};

pub fn groupByTag(allocator: Allocator, posts: []Post) !std.AutoHashMap([]const u8, std.ArrayList(Post)) {
    var groups = std.AutoHashMap([]const u8, std.ArrayList(Post)).init(allocator);
    
    for (posts) |post| {
        var iter = std.mem.splitScalar(u8, post.tags, ',');
        while (iter.next()) |tag| {
            const trimmed = std.mem.trim(u8, tag, " \t");
            
            const group = groups.getOrPut(trimmed) catch |err| {
                // 清理已分配的资源
                var it = groups.valueIterator();
                while (it.next()) |list| list.deinit();
                groups.deinit();
                return err;
            };
            
            if (!group.found_existing) {
                group.value_ptr.* = std.ArrayList(Post).init(allocator);
            }
            
            try group.value_ptr.append(post);
        }
    }
    
    return groups;
}
```

## B.5 迭代器模式

### 1. 自定义迭代器

```zig
// 模式：实现迭代器
pub const LineIterator = struct {
    lines: std.mem.Split(u8, []const u8, "\n"),
    
    pub fn next(self: *LineIterator) ?[]const u8 {
        return self.lines.next();
    }
};

pub fn iterateLines(content: []const u8) LineIterator {
    return LineIterator{
        .lines = std.mem.split(u8, content, "\n"),
    };
}

// 使用
pub fn processLines(content: []const u8) void {
    var it = iterateLines(content);
    while (it.next()) |line| {
        std.debug.print("Line: {s}\n", .{line});
    }
}
```

### 2. 生成器模式

```zig
// 模式：惰性生成序列
pub const RangeIterator = struct {
    current: i32,
    end: i32,
    
    pub fn next(self: *RangeIterator) ?i32 {
        if (self.current >= self.end) return null;
        const value = self.current;
        self.current += 1;
        return value;
    }
};

pub fn range(start: i32, end: i32) RangeIterator {
    return RangeIterator{ .current = start, .end = end };
}

// 使用
pub fn sumRange() i32 {
    var sum: i32 = 0;
    var it = range(0, 10);
    while (it.next()) |n| {
        sum += n;
    }
    return sum;
}
```

## B.6 编译时编程模式

### 1. comptime 循环

```zig
// 模式：编译时生成代码
pub fn registerHandlers() void {
    const handlers = [_]struct {
        name: []const u8,
        handler: *const fn () void,
    }{
        .{ .name = "init", .handler = initHandler },
        .{ .name = "load", .handler = loadHandler },
        .{ .name = "save", .handler = saveHandler },
    };
    
    comptime for (handlers, 0..) |h, i| {
        @export(h.handler, .{ .name = "handler_" ++ std.fmt.comptimePrint("{d}", .{i}) });
    };
}
```

### 2. 泛型模式

```zig
// 模式：泛型函数
pub fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

// 使用
const a = max(i32, 10, 20);
const b = max(f64, 3.14, 2.71);

// 模式：泛型结构
pub fn Box(comptime T: type) type {
    return struct {
        value: T,
        
        pub fn init(v: T) @This() {
            return .{ .value = v };
        }
    };
}

// 使用
const IntBox = Box(i32);
const box = IntBox.init(42);
```

### 3. 编译时反射

```zig
// 模式：遍历结构体字段
pub fn printFields(comptime T: type, value: T) void {
    inline for (std.meta.fields(T)) |field| {
        const field_value = @field(value, field.name);
        std.debug.print("{s}: {}\n", .{ field.name, field_value });
    }
}

// 使用
const Config = struct {
    name: []const u8,
    count: i32,
    enabled: bool,
};

const config = Config{
    .name = "test",
    .count = 42,
    .enabled = true,
};

printFields(Config, config);
```

## B.7 测试模式

### 1. 表格驱动测试

```zig
test "parse integers" {
    const TestCase = struct {
        input: []const u8,
        expected: i32,
        should_fail: bool = false,
    };
    
    const cases = [_]TestCase{
        .{ .input = "42", .expected = 42 },
        .{ .input = "-10", .expected = -10 },
        .{ .input = "abc", .expected = 0, .should_fail = true },
        .{ .input = "", .expected = 0, .should_fail = true },
    };
    
    inline for (cases) |c| {
        if (c.should_fail) {
            try std.testing.expectError(
                error.InvalidFormat,
                std.fmt.parseInt(i32, c.input, 10)
            );
        } else {
            const result = try std.fmt.parseInt(i32, c.input, 10);
            try std.testing.expectEqual(c.expected, result);
        }
    }
}
```

### 2. 测试夹具模式

```zig
// 模式：测试夹具
fn createTestPost(allocator: Allocator) !Post {
    return Post{
        .title = try allocator.dupe(u8, "Test Post"),
        .content = try allocator.dupe(u8, "Test content"),
        .date_time = try allocator.dupe(u8, "2024-01-15 10:00"),
        .tags = try allocator.dupe(u8, "test"),
        .filename = try allocator.dupe(u8, "test.markdown"),
    };
}

test "Post processing" {
    const allocator = std.testing.allocator;
    var post = try createTestPost(allocator);
    defer {
        allocator.free(post.title);
        allocator.free(post.content);
        allocator.free(post.date_time);
        allocator.free(post.tags);
        allocator.free(post.filename);
    }
    
    // 测试逻辑...
}
```

## 相关文档

- [附录 A: std 库 API](./appendix-a-std-lib-api.md) - 标准库参考
- [附录 C: FAQ](./appendix-c-faq.md) - 常见问题
- [9.3 错误处理模式](../part-09-error-handling/009-3-error-handling-patterns.md) - 错误处理

---

*本文档总结了 my-blog 项目中的常用 Zig 代码模式，包括内存管理、错误处理、字符串操作、集合操作和测试模式。*
