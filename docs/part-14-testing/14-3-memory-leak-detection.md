# 14.3 内存泄漏检测

> **检测工具**: Zig 内置检测器，Valgrind  
> **分配器**: GeneralPurposeAllocator, TestingAllocator  
> **检测模式**: Debug, ReleaseSafe

## 14.3.1 Zig 内存管理基础

### 内存分配模式

```
┌─────────────────────────────────────────────────────────┐
│              Zig 内存分配器层次结构                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              GeneralPurposeAllocator             │   │
│  │  - 完整功能的通用分配器                          │   │
│  │  - 检测内存泄漏和双重释放                        │   │
│  │  - 性能较慢，用于 Debug 模式                      │   │
│  └──────────────────┬──────────────────────────────┘   │
│                     │                                   │
│         ┌───────────┼───────────┐                       │
│         │           │           │                       │
│         ▼           ▼           ▼                       │
│  ┌────────────┐ ┌────────┐ ┌──────────────┐            │
│  │ ArenaAlloc │ │ Fixed  │ │ TestingAlloc │            │
│  │            │ │ Buffer │ │              │            │
│  │ 批量分配   │ │ 固定大小│ │ 泄漏检测    │            │
│  │ 不单独释放 │ │ 高性能  │ │ 测试专用    │            │
│  └────────────┘ └────────┘ └──────────────┘            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 分配器对比

| 分配器 | 用途 | 泄漏检测 | 性能 |
|--------|------|---------|------|
| GeneralPurposeAllocator | 通用场景 | ✅ | 慢 |
| ArenaAllocator | 批量分配 | ❌ | 快 |
| FixedBufferAllocator | 固定大小 | ❌ | 最快 |
| TestingAllocator | 测试 | ✅ | 中等 |

## 14.3.2 内置内存泄漏检测

### GeneralPurposeAllocator 配置

```zig
const std = @import("std");

pub fn main() !void {
    // 配置分配器
    var gpa = std.heap.GeneralPurposeAllocator(.{
        .stack_trace_frames = 10,      // 堆栈跟踪帧数
        .enable_memory_limit = false,  // 不限制内存
        .safety = true,                // 启用安全检查
    }){};
    defer _ = gpa.deinit();  // deinit 时会检查泄漏
    
    const allocator = gpa.allocator();
    
    // 使用 allocator 分配内存
    const ptr = try allocator.create(i32);
    const slice = try allocator.alloc(u8, 100);
    
    // 忘记释放会导致泄漏警告
    // allocator.destroy(ptr);  // ⚠️ 如果忘记调用
    // allocator.free(slice);   // ⚠️ 如果忘记调用
}
```

### 泄漏检测输出

```
$ zig build run

// 如果有内存泄漏，会输出：
memory leak detected
====================

Allocated here:
    /path/to/src/main.zig:25:42 in main
        const ptr = try allocator.create(i32);

Allocated 4 bytes

Freed here:
    (never freed)  // ⚠️ 从未释放

Total leaks: 1
```

### 修复内存泄漏

```zig
// 错误示例：泄漏
pub fn processData(allocator: Allocator) !void {
    const buffer = try allocator.alloc(u8, 1024);
    // 处理数据...
    // ⚠️ 忘记 free，函数返回时泄漏
}

// 正确示例：使用 defer
pub fn processData(allocator: Allocator) !void {
    const buffer = try allocator.alloc(u8, 1024);
    defer allocator.free(buffer);  // ✅ 函数返回时自动释放
    
    // 处理数据...
}

// 正确示例：错误时也释放
pub fn loadData(allocator: Allocator) ![]u8 {
    const data = try allocator.alloc(u8, 1024);
    errdefer allocator.free(data);  // ✅ 错误时释放
    
    const file = try std.fs.cwd().openFile("data.txt", .{});
    errdefer file.close();
    
    try file.readAll(data);
    
    return data;  // 调用者负责释放
}
```

## 14.3.3 测试中的内存检测

### TestingAllocator 自动检测

```zig
const std = @import("std");
const testing = std.testing;

test "no memory leak" {
    const allocator = testing.allocator;
    
    const ptr = try allocator.create(i32);
    const slice = try allocator.alloc(u8, 100);
    
    // 测试结束前释放
    allocator.destroy(ptr);
    allocator.free(slice);
    
    // 如果忘记释放，测试会失败并报告泄漏
}

test "memory leak detected" {
    const allocator = testing.allocator;
    
    const ptr = try allocator.create(i32);
    // ⚠️ 忘记 allocator.destroy(ptr)
    
    // 测试失败：
    // Test [fail]: memory leak detected
    // 1 bytes leaked
}
```

### 检测分配计数

```zig
test "verify allocation count" {
    var fixed_buffer = std.heap.FixedBufferAllocator.init(&[_]u8{0} ** 1024);
    const allocator = fixed_buffer.allocator();
    
    const ptr1 = try allocator.create(i32);
    const ptr2 = try allocator.create(i32);
    
    // 验证只分配了预期的内存
    try std.testing.expectEqual(@as(usize, 8), fixed_buffer.end_index);
    
    allocator.destroy(ptr1);
    allocator.destroy(ptr2);
}
```

## 14.3.4 常见内存泄漏场景

### 场景 1：忘记释放

```zig
// ❌ 泄漏示例
pub fn loadFile(allocator: Allocator, path: []const u8) ![]u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    const content = try file.readToEndAlloc(allocator, 1024 * 1024);
    return content;  // 调用者需要 free
}

pub fn process() void {
    const allocator = std.heap.page_allocator;
    const content = loadFile(allocator, "data.txt") catch return;
    // ⚠️ 忘记 allocator.free(content)
}

// ✅ 修复
pub fn process() void {
    const allocator = std.heap.page_allocator;
    const content = loadFile(allocator, "data.txt") catch return;
    defer allocator.free(content);  // ✅ 自动释放
    
    // 处理内容...
}
```

### 场景 2：错误路径泄漏

```zig
// ❌ 泄漏示例
pub fn processMultipleFiles(allocator: Allocator, paths: [][]const u8) !void {
    var contents = std.ArrayList([]u8).init(allocator);
    
    for (paths) |path| {
        const content = try loadFile(allocator, path);
        try contents.append(content);
        
        // 如果这里出错，之前分配的内容都会泄漏
        if (someCondition()) return error.SomethingWentWrong;
    }
}

// ✅ 修复
pub fn processMultipleFiles(allocator: Allocator, paths: [][]const u8) !void {
    var contents = std.ArrayList([]u8).init(allocator);
    errdefer {
        // 错误时释放所有已分配的内容
        for (contents.items) |content| {
            allocator.free(content);
        }
        contents.deinit();
    }
    
    for (paths) |path| {
        const content = try loadFile(allocator, path);
        errdefer allocator.free(content);  // 当前内容错误时释放
        
        try contents.append(content);
    }
    
    // 成功后，调用者负责清理
}
```

### 场景 3：循环引用

```zig
// ❌ 泄漏示例（使用 ArenaAllocator 时）
const Node = struct {
    value: i32,
    next: ?*Node = null,
    prev: ?*Node = null,  // 双向引用
};

pub fn createLinkedList(allocator: Allocator) !void {
    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit();  // ⚠️ Arena 会泄漏所有未释放的内存
    
    const node1 = try arena.allocator().create(Node);
    const node2 = try arena.allocator().create(Node);
    
    node1.next = node2;
    node2.prev = node1;  // 循环引用
    
    // ArenaAllocator 无法检测单个对象的释放
}

// ✅ 修复：使用 GeneralPurposeAllocator
pub fn createLinkedList(allocator: Allocator) !void {
    const node1 = try allocator.create(Node);
    errdefer allocator.destroy(node1);
    
    const node2 = try allocator.create(Node);
    errdefer allocator.destroy(node2);
    
    node1.next = node2;
    node2.prev = node1;
    
    // 手动管理生命周期
    allocator.destroy(node2);
    allocator.destroy(node1);
}
```

### 场景 4：字符串泄漏

```zig
// ❌ 泄漏示例
pub fn buildString(allocator: Allocator) ![]u8 {
    var result = std.ArrayList(u8).init(allocator);
    
    try result.appendSlice("Hello, ");
    try result.appendSlice("World!");
    
    return result.toOwnedSlice();  // 调用者需要 free
}

pub fn useString() void {
    const allocator = std.heap.page_allocator;
    const str = buildString(allocator) catch return;
    // ⚠️ 忘记 allocator.free(str)
}

// ✅ 修复
pub fn useString() void {
    const allocator = std.heap.page_allocator;
    const str = buildString(allocator) catch return;
    defer allocator.free(str);  // ✅ 自动释放
    
    std.debug.print("{s}\n", .{str});
}
```

## 14.3.5 使用 Valgrind 检测

### Valgrind 安装

```bash
# Ubuntu/Debian
sudo apt-get install valgrind

# macOS
brew install valgrind

# Fedora
sudo dnf install valgrind
```

### 编译用于 Valgrind

```bash
# 编译不带优化的 Debug 版本
zig build-exe src/main.zig -g -ODebug

# 或使用 build 命令
zig build -g
```

### 运行 Valgrind

```bash
# 基本内存检查
valgrind --leak-check=full ./zig-out/bin/my-blog

# 详细输出
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes ./zig-out/bin/my-blog

# 输出到文件
valgrind --leak-check=full --log-file=valgrind.log ./zig-out/bin/my-blog
```

### Valgrind 输出解读

```
==12345== Memcheck, a memory error detector
==12345== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==12345== Using Valgrind-3.15.0 and LibVEX

==12345== HEAP SUMMARY:
==12345==     in use at exit: 1,024 bytes in 10 blocks
==12345==   total heap usage: 100 allocs, 90 frees, 10,240 bytes allocated
==12345== 
==12345== 1,024 bytes in 10 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x4C3089F: malloc (in valgrind)
==12345==    by 0x123456: std.heap.GeneralPurposeAllocator.alloc (main.zig:25)
==12345==    by 0x234567: loadData (Blogger.zig:100)
==12345==    by 0x345678: main (main.zig:10)
==12345== 
==12345== LEAK SUMMARY:
==12345==    definitely lost: 1,024 bytes in 10 blocks
==12345==    indirectly lost: 0 bytes in 0 blocks
==12345==      possibly lost: 0 bytes in 0 blocks
==12345==    still reachable: 0 bytes in 0 blocks
==12345==         suppressed: 0 bytes in 0 blocks
```

### Valgrind 选项说明

| 选项 | 说明 |
|------|------|
| `--leak-check=full` | 完整泄漏检查 |
| `--show-leak-kinds=all` | 显示所有泄漏类型 |
| `--track-origins=yes` | 跟踪未初始化值来源 |
| `--error-exitcode=1` | 有错误时返回非零退出码 |

## 14.3.6 内存分配器选择

### 场景推荐

| 场景 | 推荐分配器 | 理由 |
|------|-----------|------|
| 开发调试 | GeneralPurposeAllocator | 完整泄漏检测 |
| 单元测试 | TestingAllocator | 自动检测泄漏 |
| 批量临时分配 | ArenaAllocator | 高性能，批量释放 |
| 固定大小缓冲 | FixedBufferAllocator | 最快，无系统调用 |
| 生产环境 | GeneralPurposeAllocator | 安全性和性能平衡 |

### ArenaAllocator 使用

```zig
pub fn processDocument(allocator: Allocator) !void {
    // 使用 Arena 管理临时内存
    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit();  // 一次性释放所有内存
    
    const temp_allocator = arena.allocator();
    
    // 大量临时分配
    var strings = std.ArrayList([]u8).init(temp_allocator);
    for (0..1000) |i| {
        const str = try temp_allocator.alloc(u8, 100);
        try strings.append(str);
    }
    
    // 处理...
    
    // arena.deinit() 会释放所有临时内存
    // 无需手动管理每个分配
}
```

### FixedBufferAllocator 使用

```zig
pub fn parseHeader(buffer: []const u8) !Header {
    // 使用固定缓冲分配器
    var fixed_buffer = std.heap.FixedBufferAllocator.init(&[_]u8{0} ** 1024);
    const allocator = fixed_buffer.allocator();
    
    // 在固定缓冲内分配
    const title = try allocator.alloc(u8, 100);
    const content = try allocator.alloc(u8, 500);
    
    // 如果超出缓冲大小，会返回 OutOfMemory 错误
    // 无需担心泄漏，缓冲是栈上的
}
```

## 14.3.7 内存调试检查清单

### 开发阶段

- [ ] 使用 GeneralPurposeAllocator
- [ ] 启用 Debug 模式（`zig build -g`）
- [ ] 运行测试检查泄漏（`zig build test`）
- [ ] 检查所有 `errdefer` 是否正确放置

### 测试阶段

- [ ] 使用 TestingAllocator 运行所有测试
- [ ] 运行 Valgrind 检查（`valgrind --leak-check=full`）
- [ ] 长时间运行测试（检测累积泄漏）
- [ ] 边界条件测试（空输入、大输入）

### 发布前

- [ ] 切换到 ReleaseSafe 模式测试
- [ ] 性能测试（确保无过度分配）
- [ ] 内存使用监控（RSS 稳定）
- [ ] 压力测试（高负载下无泄漏）

## 14.3.8 内存性能优化

### 减少分配次数

```zig
// ❌ 多次分配
pub fn buildPath(allocator: Allocator, parts: [][]const u8) ![]u8 {
    var result = try allocator.alloc(u8, 0);
    
    for (parts) |part| {
        result = try allocator.realloc(result, result.len + part.len + 1);
        // ⚠️ 每次都重新分配
    }
    
    return result;
}

// ✅ 预计算大小，一次分配
pub fn buildPath(allocator: Allocator, parts: [][]const u8) ![]u8 {
    // 计算总大小
    var total_size: usize = 0;
    for (parts) |part| {
        total_size += part.len + 1;
    }
    
    // 一次分配
    var result = try allocator.alloc(u8, total_size);
    var offset: usize = 0;
    
    for (parts) |part| {
        @memcpy(result[offset..offset + part.len], part);
        offset += part.len;
        result[offset] = '/';
        offset += 1;
    }
    
    return result;
}
```

### 使用缓冲区

```zig
// 使用栈上缓冲区避免分配
pub fn formatPostTitle(post: Post) [256]u8 {
    var buffer: [256]u8 = undefined;
    const result = std.fmt.bufPrint(&buffer, "[{s}] {s}", .{post.date_time, post.title}) catch 
        return undefined;
    return buffer;  // 返回数组（不是切片），无泄漏风险
}
```

## 相关文档

- [14.1 单元测试](./14-1-unit-testing.md) - 测试基础
- [14.2 调试技巧](./14-2-debugging-techniques.md) - 调试方法
- [8.2 所有权模式](../part-08-memory-management/08-2-memory-ownership.md) - 内存所有权

---

*本文档介绍了 Zig 程序中的内存泄漏检测方法，包括内置检测器使用、Valgrind 工具、常见泄漏场景和预防技巧。*
