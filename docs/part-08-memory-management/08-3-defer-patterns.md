# 08-3 defer 模式

> Zig defer 语句使用技巧与项目实例分析

---

## 1. defer 基础

### 1.1 什么是 defer

`defer` 语句用于注册一个代码块，在**当前作用域结束时**执行。

**基本语法**：

```zig
defer {
    // 清理代码
}

// 或者单行
defer expression;
```

### 1.2 defer 执行时机

```zig
fn example() void {
    defer std.debug.print("Cleanup 1\n", .{});
    defer std.debug.print("Cleanup 2\n", .{});
    
    std.debug.print("Main code\n", .{});
}  // ← defer 在此按**后进先出**顺序执行

// 输出:
// Main code
// Cleanup 2
// Cleanup 1
```

**执行顺序**：后进先出 (LIFO)

---

## 2. 项目中的 defer 模式

### 2.1 main.zig 中的 defer

```zig
pub fn main() !void {
    // [1] 分配器清理
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    //     ^^^^^^^^^^^^^^^^
    //     程序结束时释放所有内存
    const allocator = gpa.allocator();

    // [2] 命令行参数清理
    var args = try std.process.argsWithAllocator(allocator);
    defer args.deinit();
    //     ^^^^^^^^^^^^
    //     释放参数迭代器

    // ... 主逻辑
}  // ← defer 按顺序执行:
   //   1. args.deinit()
   //   2. gpa.deinit()
```

### 2.2 Blogger.zig 中的 defer

#### 批量清理模式

```zig
pub fn generate(self: *Blogger) !void {
    // 创建文章列表
    var posts = std.ArrayList(Post){};
    defer {
        // 批量清理所有 Post
        for (posts.items) |*post| post.deinit(self.allocator);
        //      ^^^^^^^^^^^ 遍历每个 Post
        //                    ^^^^^^^^^^^ 调用 deinit()
        posts.deinit(self.allocator);
        // ^^^^^^^^^^^ 释放 ArrayList 本身
    }
    
    try self.loadPosts(&posts);
    // ... 生成页面
}  // ← 函数返回时执行 defer 块
```

#### 临时资源清理

```zig
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] Markdown → HTML
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     函数返回前释放
    
    // [2] 模板上下文
    var ctx = Context.init(self.allocator);
    defer ctx.deinit();
    //     ^^^^^^^^^^
    //     释放上下文资源
    
    // [3] 标签 HTML
    var tags_html = std.ArrayList(u8){};
    defer tags_html.deinit(self.allocator);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     释放 ArrayList
    
    // [4] 页面 HTML
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     释放渲染结果
    
    // [5] 完整标题
    var full_title = std.ArrayList(u8){};
    defer full_title.deinit(self.allocator);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     释放标题缓冲区
    
    // [6] 最终 HTML
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^
    //     释放最终 HTML
    
    // ... 写入文件
}  // ← 所有 defer 在此按 LIFO 顺序执行
```

### 2.3 Post.zig 中的 defer

```zig
pub fn parse(allocator: Allocator, content: []const u8, filename: []const u8) !Post {
    var lines = std.mem.splitScalar(u8, content, '\n');
    // 注意：lines 是迭代器，不需要 defer
    
    const first_line = lines.next() orelse return error.InvalidFormat;
    // 早期返回，不需要清理
    
    // ... 解析逻辑
    
    return Post{
        .title = title orelse return error.MissingTitle,
        // 如果这里返回错误，之前分配的 title 会泄漏！
        // 需要小心处理
    };
}
```

**注意**：`parse()` 中如果有多个早期返回点，需要确保已分配的内存被释放。

---

## 3. defer 变体

### 3.1 errdefer

`errdefer` 只在**函数返回错误时**执行。

**语法**：

```zig
errdefer expression;
// 或
errdefer {
    // 清理代码
}
```

**使用场景**：

```zig
fn parse(allocator: Allocator, content: []const u8) !Post {
    var title: ?[]const u8 = null;
    
    // 如果后续操作失败，释放已分配的 title
    errdefer {
        if (title) |t| allocator.free(t);
    }
    
    // 分配 title
    title = try allocator.dupe(u8, "Title");
    
    // 可能失败的操作
    const date_time = try allocator.dupe(u8, "2026-03-21");
    // 如果这里失败，errdefer 会释放 title
    
    return Post{
        .title = title.?,
        .date_time = date_time,
    };
}
```

### 3.2 defer vs errdefer

| 特性 | defer | errdefer |
|------|-------|----------|
| 执行时机 | 总是执行 | 仅返回错误时执行 |
| 用途 | 常规清理 | 错误恢复 |
| 项目使用 | 广泛 | 较少 |

---

## 4. defer 与循环

### 4.1 循环内的 defer

```zig
while (try walker.next()) |entry| {
    if (entry.kind == .file) {
        const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
        defer self.allocator.free(src_path);
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     每次迭代都会释放
        
        const content = try src_file.readToEndAlloc(self.allocator, 1024 * 1024);
        defer self.allocator.free(content);
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     每次迭代都会释放
        
        // ... 处理文件
    }
}  // ← 每次迭代结束时执行 defer
```

**执行流程**：

```
迭代 1:
  ├─ 分配 src_path
  ├─ 分配 content
  ├─ 处理
  └─ defer 释放 content, src_path

迭代 2:
  ├─ 分配 src_path
  ├─ 分配 content
  ├─ 处理
  └─ defer 释放 content, src_path
```

### 4.2 for 循环中的 defer

```zig
for (posts.items) |post| {
    var filename = std.ArrayList(u8){};
    defer filename.deinit(self.allocator);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     每次迭代都会释放
    
    // ... 构建文件名
}
```

---

## 5. defer 与错误处理

### 5.1 defer 在错误传播中的作用

```zig
fn process(allocator: Allocator) !void {
    var data = try allocator.alloc(u8, 100);
    defer allocator.free(data);
    //     ^^^^^^^^^^^^^^^^^^^^
    //     即使 try 失败也会执行
    
    try doSomething(data);  // 如果失败，defer 仍然执行
    try doAnotherThing();   // 如果失败，defer 仍然执行
}
```

### 5.2 多个 defer 的执行顺序

```zig
fn complex(allocator: Allocator) !void {
    var resource1 = try acquireResource1();
    defer releaseResource1(resource1);
    
    var resource2 = try acquireResource2();
    defer releaseResource2(resource2);
    
    var data = try allocator.alloc(u8, 100);
    defer allocator.free(data);
    
    // ... 使用资源
}  // ← 执行顺序:
   //   1. allocator.free(data)
   //   2. releaseResource2(resource2)
   //   3. releaseResource1(resource1)
```

---

## 6. defer 最佳实践

### 6.1 立即注册清理

```zig
// 推荐：分配后立即 defer
const data = try allocator.alloc(u8, size);
defer allocator.free(data);

// 不推荐：延迟 defer
const data = try allocator.alloc(u8, size);
// ... 一些代码 ...
defer allocator.free(data);  // 容易忘记
```

### 6.2 成对使用

```zig
// 分配和释放成对
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();

var file = try std.fs.cwd().openFile("test.txt", .{});
defer file.close();

var dir = try std.fs.cwd().openDir("test", .{});
defer dir.close();
```

### 6.3 嵌套作用域的 defer

```zig
fn nested() void {
    defer std.debug.print("Outer\n", .{});
    
    {
        defer std.debug.print("Inner\n", .{});
        std.debug.print("Block\n", .{});
    }  // ← Inner defer 在此执行
    
    std.debug.print("After block\n", .{});
}  // ← Outer defer 在此执行

// 输出:
// Block
// Inner
// After block
// Outer
```

---

## 7. 完整实例分析

### 7.1 copyStaticFiles() 中的 defer

```zig
fn copyStaticFiles(self: *Blogger) !void {
    // [1] 打开目录
    var dir = std.fs.cwd().openDir(static_dir, .{ .iterate = true }) catch {
        std.debug.print(" Warning: static directory not found\n", .{});
        return;
    };
    defer dir.close();
    //     ^^^^^^^^^^
    //     关闭目录句柄

    // [2] 创建 walker
    var walker = try dir.walk(self.allocator);
    defer walker.deinit();
    //     ^^^^^^^^^^^^^
    //     清理 walker

    // [3] 遍历文件
    while (try walker.next()) |entry| {
        if (entry.kind == .file) {
            // 每次迭代分配的路径
            const src_path = try std.fs.path.join(self.allocator, &.{ static_dir, entry.path });
            defer self.allocator.free(src_path);
            //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            //     每次迭代释放
            
            const dest_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, entry.path });
            defer self.allocator.free(dest_path);
            //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            //     每次迭代释放

            // 打开源文件
            var src_file = try dir.openFile(entry.path, .{});
            defer src_file.close();
            //     ^^^^^^^^^^^^^^
            //     关闭文件句柄

            // 创建目标文件
            const dest_file = try std.fs.cwd().createFile(dest_path, .{});
            defer dest_file.close();
            //     ^^^^^^^^^^^^^^^
            //     关闭文件句柄

            // 读取内容
            const content = try src_file.readToEndAlloc(self.allocator, 1024 * 1024);
            defer self.allocator.free(content);
            //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
            //     释放内容内存

            // 写入文件
            try dest_file.writeAll(content);
        }
    }
}  // ← 函数返回时执行外层的 defer
```

### 7.2 generateIndex() 中的 defer

```zig
fn generateIndex(self: *Blogger, posts: []const Post) !void {
    // [1] 主 HTML 缓冲区
    var posts_html = std.ArrayList(u8){};
    defer posts_html.deinit(self.allocator);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     函数返回前释放

    // [2] 遍历文章
    for (posts) |post| {
        // 每次迭代的文件名缓冲区
        var filename = std.ArrayList(u8){};
        defer filename.deinit(self.allocator);
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     每次迭代释放

        // 每次迭代的标签 HTML
        var tags_html = std.ArrayList(u8){};
        defer tags_html.deinit(self.allocator);
        //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //     每次迭代释放

        // ... 构建 HTML
    }

    // [3] 模板上下文
    var ctx = Context.init(self.allocator);
    defer ctx.deinit();
    //     ^^^^^^^^^^
    //     释放上下文

    // [4] 页面 HTML
    const page_html = try self.template_engine.render("index.hbs", &ctx);
    defer self.allocator.free(page_html);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     释放渲染结果

    // [5] 最终 HTML
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^
    //     释放最终 HTML

    // [6] 输出路径
    const output_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, "index.html" });
    defer self.allocator.free(output_path);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     释放路径字符串

    // ... 写入文件
}
```

---

## 8. defer 陷阱

### 8.1 defer 在循环中的误用

```zig
// 错误：defer 在循环外
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();

for (items) |item| {
    try list.append(item);
    // 如果这里出错，list 已经 defer
    // 但 item 可能泄漏
}

// 正确：每次迭代单独管理
for (items) |item| {
    var temp = try allocator.dupe(u8, item);
    defer allocator.free(temp);  // 每次迭代释放
    // ...
}
```

### 8.2 defer 与早期返回

```zig
// 错误：早期返回跳过 defer
fn process() !void {
    var data = try allocator.alloc(u8, 100);
    defer allocator.free(data);
    
    if (someCondition) {
        return;  // defer 仍然会执行，正确
    }
    
    // 但如果 data 在条件前已经部分使用...
}

// 正确：使用 errdefer 处理部分初始化
fn parse() !Post {
    var title: ?[]const u8 = null;
    errdefer if (title) |t| allocator.free(t);
    
    title = try allocator.dupe(u8, "Title");
    
    // 如果后续失败，errdefer 释放 title
    return Post{ .title = title.? };
}
```

### 8.3 defer 顺序依赖

```zig
// 注意 defer 执行顺序
var a = acquireA();
defer releaseA(a);

var b = acquireB(a);  // b 依赖 a
defer releaseB(b);

// 执行顺序: releaseB, releaseA (正确)
// 如果反过来会出错
```

---

## 9. defer 性能

### 9.1 defer 开销

`defer` 的开销极小，相当于：

```zig
// defer 编译后类似
fn example() {
    // 注册清理回调（几乎零开销）
    // ... 主代码 ...
    // 在作用域末尾内联展开清理代码
}
```

### 9.2 优化建议

```zig
// 推荐：defer 内联展开
defer allocator.free(data);  // 零开销

// 避免：复杂的 defer 块
defer {
    // 复杂逻辑可能影响性能
    if (condition) {
        cleanup1();
    } else {
        cleanup2();
    }
}
```

---

## 10. 总结

### 10.1 defer 使用检查清单

- [ ] 分配内存后立即 `defer` 释放
- [ ] 打开文件/目录后 `defer` 关闭
- [ ] 初始化资源后 `defer` 清理
- [ ] 循环内的临时资源使用 `defer`
- [ ] 错误恢复使用 `errdefer`
- [ ] 注意多个 `defer` 的执行顺序

### 10.2 项目中的 defer 模式总结

| 位置 | 模式 | 示例 |
|------|------|------|
| `main()` | 根分配器清理 | `defer _ = gpa.deinit()` |
| `generate()` | 批量清理 | `defer { for (...) post.deinit() }` |
| `generatePostPage()` | 临时资源 | 多个 `defer allocator.free()` |
| `copyStaticFiles()` | 循环内清理 | 每次迭代 `defer` 释放 |
| `loadPosts()` | 文件句柄 | `defer file.close()` |

---

## 相关文档

- [08-1 内存分配器](./08-1-allocators.md) - 分配器基础
- [08-2 内存所有权](./08-2-memory-ownership.md) - 所有权模式
- [05-1 函数基础](../part-05-functions/05-1-function-basics.md) - defer 基础语法
