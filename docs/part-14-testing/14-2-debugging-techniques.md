# 14.2 调试技巧

> **调试工具**: GDB, LLDB, Zig 内置调试器  
> **调试模式**: Debug Build, ReleaseSafe  
> **日志输出**: `std.debug.print`

## 14.2.1 构建模式对比

### Zig 构建模式

| 模式 | 命令 | 特点 | 用途 |
|------|------|------|------|
| Debug | `zig build` | 完整调试信息，慢，安全 | 开发和调试 |
| ReleaseSafe | `zig build -Doptimize=ReleaseSafe` | 优化 + 安全检查 | 测试环境 |
| ReleaseFast | `zig build -Doptimize=ReleaseFast` | 最大优化，不安全 | 生产环境 |
| ReleaseSmall | `zig build -Doptimize=ReleaseSmall` | 优化代码大小 | 嵌入式 |

### Debug 模式特性

```bash
# Debug 构建（默认）
zig build

# Debug 模式特点：
# - 包含完整调试符号
# - 启用边界检查
# - 启用未初始化内存检测
# - 启用内存泄漏检测
# - 编译速度慢，运行速度慢
```

### 不同模式下的行为差异

```zig
const std = @import("std");

pub fn main() void {
    var buffer: [10]u8 = undefined;
    
    // Debug 模式：会捕获未初始化内存使用
    // Release 模式：可能读取到垃圾值
    std.debug.print("{s}\n", .{buffer});  // ⚠️ 未初始化
    
    // Debug 模式：会捕获数组越界
    // Release 模式：未定义行为
    buffer[15] = 42;  // ⚠️ 越界
}
```

## 14.2.2 日志输出调试

### std.debug.print

```zig
const std = @import("std");

pub fn processPosts(posts: []Post) void {
    std.debug.print("Processing {d} posts\n", .{posts.len});
    
    for (posts, 0..) |post, i| {
        std.debug.print("[{d}] Title: {s}, Date: {s}\n", .{
            i,
            post.title,
            post.date_time,
        });
        
        // 处理逻辑
    }
}
```

### 格式化选项

| 占位符 | 类型 | 示例 |
|--------|------|------|
| `{}` | 任意类型 | `std.debug.print("Value: {}\n", .{42})` |
| `{d}` | 十进制整数 | `std.debug.print("Count: {d}\n", .{100})` |
| `{x}` | 十六进制 | `std.debug.print("Hex: {x}\n", .{255})` |
| `{b}` | 二进制 | `std.debug.print("Bin: {b}\n", .{0b1010})` |
| `{s}` | 字符串 | `std.debug.print("Name: {s}\n", .{"Alice"})` |
| `{c}` | 字符 | `std.debug.print("Char: {c}\n", .{'A'})` |
| `{f}` | 浮点数 | `std.debug.print("Pi: {f}\n", .{3.14159})` |
| `{f:.2}` | 浮点数（2 位小数） | `std.debug.print("Pi: {f:.2}\n", .{3.14159})` |
| `{any}` | 任意类型（详细） | `std.debug.print("Data: {any}\n", .{data})` |

### 条件日志

```zig
const DEBUG = true;

fn debugLog(comptime format: []const u8, args: anytype) void {
    if (DEBUG) {
        std.debug.print("[DEBUG] " ++ format, args);
    }
}

pub fn parseFrontmatter(content: []const u8) !Frontmatter {
    debugLog("Parsing frontmatter: {s}\n", .{content[0..@min(content.len, 50)]});
    
    const fm = try parse(content);
    
    debugLog("Parsed title: {s}\n", .{fm.title});
    
    return fm;
}
```

### 日志级别系统

```zig
const LogLevel = enum {
    debug,
    info,
    warn,
    error,
};

const current_log_level = LogLevel.debug;

fn log(comptime level: LogLevel, comptime format: []const u8, args: anytype) void {
    if (@intFromEnum(level) >= @intFromEnum(current_log_level)) {
        const prefix = switch (level) {
            LogLevel.debug => "[DEBUG] ",
            LogLevel.info => "[INFO]  ",
            LogLevel.warn => "[WARN]  ",
            LogLevel.error => "[ERROR] ",
        };
        std.debug.print(prefix ++ format, args);
    }
}

pub fn generateBlog() void {
    log(.info, "Starting blog generation\n", .{});
    
    const posts = loadPosts() catch |err| {
        log(.error, "Failed to load posts: {}\n", .{err});
        return;
    };
    
    log(.debug, "Loaded {d} posts\n", .{posts.len});
}
```

## 14.2.3 使用 GDB 调试

### 编译带调试符号

```bash
# 编译为可执行文件（包含调试符号）
zig build-exe src/main.zig -g

# 或使用 build 命令
zig build -g

# -g 标志生成调试符号，可用 GDB/LLDB 调试
```

### GDB 基本命令

```bash
# 启动 GDB
gdb zig-out/bin/my-blog

# GDB 会话
(gdb) break main              # 在 main 函数设置断点
(gdb) break Blogger.zig:100   # 在指定行设置断点
(gdb) run                     # 运行程序
(gdb) next                    # 单步执行（不进入函数）
(gdb) step                    # 单步执行（进入函数）
(gdb) continue                # 继续执行到下一个断点
(gdb) print variable          # 打印变量值
(gdb) backtrace               # 显示调用栈
(gdb) quit                    # 退出 GDB
```

### 调试 Zig 程序示例

```bash
# 1. 编译带调试符号
zig build -g

# 2. 启动 GDB
gdb zig-out/bin/my-blog

# 3. GDB 会话示例
(gdb) break src/main.zig:25        # 在 main.zig 第 25 行设置断点
Breakpoint 1 at 0x12345678

(gdb) run --posts ./test-posts     # 带参数运行
Starting program: /path/to/my-blog --posts ./test-posts

Breakpoint 1, main () at src/main.zig:25
25          const posts = try blogger.loadPosts();

(gdb) print posts                  # 打印 posts 变量
$1 = {len = 5, ptr = 0x7ffff0001000}

(gdb) next                         # 执行下一行
26          try blogger.generateIndex(posts);

(gdb) step                         # 进入函数
generateIndex (self=0x7fffffffe000, posts=...) at src/Blogger.zig:150

(gdb) backtrace                    # 查看调用栈
#0  generateIndex (self=0x7fffffffe000, posts=...) at src/Blogger.zig:150
#1  0x0000000000123456 in main () at src/main.zig:26

(gdb) print self.dest_dir          # 打印结构体字段
$2 = {len = 12, ptr = 0x7ffff0002000 "zig-out/blog"}

(gdb) continue                     # 继续执行
```

## 14.2.4 使用 LLDB 调试

### LLDB 基本命令

```bash
# 启动 LLDB
lldb zig-out/bin/my-blog

# LLDB 会话
(lldb) breakpoint set --file main.zig --line 25   # 设置断点
(lldb) run                                        # 运行
(lldb) thread step-over                           # 单步（不进入）
(lldb) thread step-in                             # 单步（进入）
(lldb) thread continue                            # 继续
(lldb) frame variable                             # 打印变量
(lldb) thread backtrace                           # 调用栈
(lldb) quit
```

### LLDB Python 脚本

```python
# lldb_script.py
import lldb

def print_post_summary(frame, bp_loc, dict):
    """打印 Post 结构摘要"""
    post = frame.FindVariable("post")
    title = post.GetChildMemberWithName("title")
    print(f"Post: {title.GetValue()}")

# 在 LLDB 中加载
(lldb) command script import lldb_script
(lldb) breakpoint command add -F lldb_script.print_post_summary
```

## 14.2.5 核心转储调试

### 启用核心转储

```bash
# Linux 启用核心转储
ulimit -c unlimited

# 查看核心转储位置
cat /proc/sys/kernel/core_pattern

# 设置核心转储位置
echo "/tmp/core.%e.%p" > /proc/sys/kernel/core_pattern
```

### 分析核心转储

```bash
# 程序崩溃后生成 core 文件
./zig-out/bin/my-blog  # 崩溃
# core 文件生成在 /tmp/core.my-blog.12345

# 使用 GDB 分析
gdb zig-out/bin/my-blog /tmp/core.my-blog.12345

# 查看崩溃位置
(gdb) backtrace
(gdb) frame 0
(gdb) info locals
```

## 14.2.6 常见调试场景

### 场景 1：段错误（Segmentation Fault）

```zig
// 错误示例
pub fn processPosts(posts: []Post) void {
    for (0..posts.len + 1) |i| {  // ⚠️ 越界
        std.debug.print("{s}\n", .{posts[i].title});
    }
}

// 调试步骤
// 1. 使用 Debug 模式编译
zig build -g

// 2. 运行程序（会崩溃）
./zig-out/bin/my-blog

// 3. 查看错误信息
// Segmentation fault at address 0x...

// 4. 修复
for (0..posts.len) |i| {  // ✅ 正确
    std.debug.print("{s}\n", .{posts[i].title});
}
```

### 场景 2：内存泄漏

```zig
// 错误示例
pub fn loadPost(allocator: Allocator, path: []const u8) !Post {
    const title = try allocator.alloc(u8, 100);
    const content = try allocator.alloc(u8, 1000);
    
    // ... 加载数据
    
    return Post{
        .title = title,
        .content = content,
    };
    // ⚠️ 忘记释放内存
}

// 调试：启用内存泄漏检测
zig build test  // 会自动检测

// 修复
pub fn loadPost(allocator: Allocator, path: []const u8) !Post {
    const title = try allocator.alloc(u8, 100);
    errdefer allocator.free(title);  // ✅ 错误时释放
    
    const content = try allocator.alloc(u8, 1000);
    errdefer allocator.free(content);  // ✅ 错误时释放
    
    // ... 加载数据
    
    return Post{
        .title = title,
        .content = content,
    };
    // 调用者负责释放
}
```

### 场景 3：未初始化内存

```zig
// 错误示例
pub fn computeHash(data: []const u8) u64 {
    var hash: u64 = undefined;  // ⚠️ 未初始化
    // 忘记赋值就直接使用
    return hash;
}

// 调试：Debug 模式会捕获
// 使用 std.mem.zeroes 初始化

// 修复
pub fn computeHash(data: []const u8) u64 {
    var hash: u64 = 0;  // ✅ 显式初始化
    // 或者
    var hash2: u64 = @zeroes();  // ✅ 零初始化
    
    for (data) |byte| {
        hash = hash * 31 + byte;
    }
    
    return hash;
}
```

### 场景 4：错误处理不当

```zig
// 错误示例
pub fn processFile(path: []const u8) !void {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();
    
    const content = try file.readToEndAlloc(std.testing.allocator, 1024 * 1024);
    // ⚠️ 如果 readToEndAlloc 失败，file 不会被 close
    
    // 处理内容
}

// 修复：使用 errdefer
pub fn processFile(path: []const u8) !void {
    const file = try std.fs.cwd().openFile(path, .{});
    errdefer file.close();  // ✅ 错误时关闭
    defer file.close();
    
    const content = try file.readToEndAlloc(std.testing.allocator, 1024 * 1024);
    defer std.testing.allocator.free(content);
    
    // 处理内容
}
```

## 14.2.7 调试技巧总结

### 1. 使用 std.debug.dump

```zig
// 打印堆栈跟踪
pub fn debugStackTrace() void {
    std.debug.dumpStackTrace(std.debug.getStackTrace());
}

// 在关键位置调用
pub fn processPost(post: Post) void {
    if (post.title.len == 0) {
        std.debug.print("Empty title detected!\n", .{});
        std.debug.dumpStackTrace(std.debug.getStackTrace());
        return;
    }
}
```

### 2. 使用 std.testing.refAllDecls

```zig
// 测试中检查所有声明
test "verify all functions exist" {
    try std.testing.refAllDecls(@This());
}
```

### 3. 编译时调试

```zig
// 使用 @compileLog 在编译时打印信息
pub fn process(comptime T: type) void {
    @compileLog("Processing type: ", T);
    // 编译时会输出类型信息
}

// 编译输出
// Processing type: i32
```

### 4. 断言调试

```zig
// 使用 std.debug.assert
pub fn divide(a: f64, b: f64) f64 {
    std.debug.assert(b != 0);  // ⚠️ 违反会崩溃
    return a / b;
}

// 只在 Debug 模式生效
// Release 模式会被优化掉
```

## 14.2.8 调试检查清单

调试前准备：

- [ ] 使用 Debug 模式编译（`zig build -g`）
- [ ] 启用内存泄漏检测
- [ ] 准备最小复现用例
- [ ] 收集错误信息和日志

调试过程中：

- [ ] 添加日志输出定位问题
- [ ] 使用断点单步执行
- [ ] 检查变量值和调用栈
- [ ] 尝试 GDB/LLDB 深入分析

调试后验证：

- [ ] 修复后运行所有测试
- [ ] 检查是否引入新问题
- [ ] 移除调试代码和日志
- [ ] 更新文档和注释

## 相关文档

- [14.1 单元测试](./14-1-unit-testing.md) - 测试基础
- [14.3 内存泄漏检测](./14-3-memory-leak-detection.md) - 内存调试
- [9.2 错误传播](../part-09-error-handling/09-2-error-propagation.md) - 错误处理

---

*本文档介绍了 Zig 程序的调试技巧，包括日志输出、GDB/LLDB 调试器使用、核心转储分析和常见调试场景的解决方法。*
