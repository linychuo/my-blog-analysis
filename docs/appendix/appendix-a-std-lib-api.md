# 附录 A: Zig 标准库 API 参考

> **Zig 版本**: 0.15.2  
> **适用范围**: my-blog 项目中使用的 std 库 API

## A.1 内存管理（std.mem）

### 分配器（Allocator）

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// 获取分配器
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
const allocator = gpa.allocator();

// 分配单个对象
const ptr = try allocator.create(Type);
allocator.destroy(ptr);

// 分配数组
const slice = try allocator.alloc(Type, count);
allocator.free(slice);

// 分配并初始化
const ptr = try allocator.create(Type);
ptr.* = Type{ .field = value };

// 重新分配
const new_slice = try allocator.realloc(old_slice, new_len);

// 复制字符串
const duped = try allocator.dupe(u8, original_string);
allocator.free(duped);
```

### 常用 API

| 函数 | 说明 | 示例 |
|------|------|------|
| `create(T)` | 分配单个对象 | `try allocator.create(i32)` |
| `alloc(T, n)` | 分配数组 | `try allocator.alloc(u8, 100)` |
| `free(ptr)` | 释放数组 | `allocator.free(slice)` |
| `destroy(ptr)` | 释放单个对象 | `allocator.destroy(ptr)` |
| `dupe(T, slice)` | 复制切片 | `try allocator.dupe(u8, str)` |
| `realloc(slice, n)` | 重新分配 | `try allocator.realloc(slice, 200)` |
| `alignedAlloc(T, align, n)` | 对齐分配 | `try allocator.alignedAlloc(u8, 4096, 100)` |

### 字符串操作

```zig
// 字符串比较
std.mem.eql(u8, str1, str2);  // 返回 bool

// 查找子串
std.mem.indexOf(u8, haystack, needle);  // 返回？usize
std.mem.lastIndexOf(u8, haystack, needle);

// 分割字符串
var iter = std.mem.split(u8, input, delimiter);
while (iter.next()) |part| {
    // 处理每个部分
}

// 修剪空白
std.mem.trim(u8, input, " \t\n\r");
std.mem.trimLeft(u8, input, " \t");
std.mem.trimRight(u8, input, "\n\r");

// 复制内存
@memcpy(dest, src);
std.mem.copyForwards(u8, dest, src);

// 填充内存
@memset(slice, value);
std.mem.set(u8, slice, value);
```

## A.2 格式化输出（std.fmt）

### 打印函数

```zig
// 调试输出（到 stderr）
std.debug.print("Value: {d}\n", .{value});

// 格式化到缓冲区
var buffer: [100]u8 = undefined;
const result = try std.fmt.bufPrint(&buffer, "Hello, {s}!", .{name});

// 格式化到分配器
const str = try std.fmt.allocPrint(allocator, "{d} + {d} = {d}", .{ a, b, a + b });
allocator.free(str);

// 写入 Writer
try writer.writeAll("Hello, World!\n");
try writer.print("Count: {d}\n", .{count});
```

### 格式化占位符

| 占位符 | 类型 | 示例 | 输出 |
|--------|------|------|------|
| `{}` | 任意 | `{}` | 自动格式化 |
| `{d}` | 十进制 | `{d}` | `42` |
| `{x}` | 十六进制 | `{x}` | `2a` |
| `{X}` | 十六进制大写 | `{X}` | `2A` |
| `{b}` | 二进制 | `{b}` | `101010` |
| `{o}` | 八进制 | `{o}` | `52` |
| `{s}` | 字符串 | `{s}` | `Hello` |
| `{c}` | 字符 | `{c}` | `A` |
| `{f}` | 浮点 | `{f}` | `3.14159` |
| `{f:.2}` | 浮点（2 位） | `{f:.2}` | `3.14` |
| `{e}` | 科学计数 | `{e}` | `3.14159e+00` |
| `{any}` | 任意（详细） | `{any}` | 详细输出 |
| `{?}` | 可选类型 | `{?}` | `null` 或值 |
| `{!}` | 错误 union | `{!}` | 错误或值 |

### 格式化选项

```zig
// 宽度填充
std.fmt.format(writer, "{d: >5}", .{42});  // "   42"
std.fmt.format(writer, "{d:0<5}", .{42});  // "42000"

// 对齐
std.fmt.format(writer, "{s: <10}", .{"Hi"});   // "Hi        "
std.fmt.format(writer, "{s: ^10}", .{"Hi"});   // "    Hi    "
std.fmt.format(writer, "{s: >10}", .{"Hi"});   // "        Hi"

// 精度
std.fmt.format(writer, "{f:.2}", .{3.14159});  // "3.14"

// 分组
std.fmt.format(writer, "{d:3}", .{1000});  // "1 000"
```

## A.3 文件系统（std.fs）

### 文件操作

```zig
const fs = std.fs;

// 打开文件
const file = try fs.cwd().openFile("path/to/file.txt", .{});
defer file.close();

// 创建文件
const new_file = try fs.cwd().createFile("new.txt", .{});
defer new_file.close();

// 读取文件
const content = try file.readToEndAlloc(allocator, max_size);
defer allocator.free(content);

// 写入文件
try file.writeAll("Hello, World!");
try file.print("Count: {d}\n", .{count});

// 读取到缓冲区
var buffer: [1024]u8 = undefined;
const bytes_read = try file.read(&buffer);

// 删除文件
try fs.cwd().deleteFile("old.txt");

// 重命名文件
try fs.cwd().rename("old.txt", "new.txt");

// 复制文件
try fs.cwd().copyFile("src.txt", "dest.txt", .{});
```

### 目录操作

```zig
// 打开目录
const dir = try fs.cwd().openDir("path/to/dir", .{});
defer dir.close();

// 创建目录
try fs.cwd().makeDir("new_dir");
try dir.makeDir("subdir");

// 遍历目录
var walker = try dir.walk(allocator);
defer walker.deinit();

while (try walker.next()) |entry| {
    std.debug.print("{s}\n", .{entry.path});
}

// 检查文件是否存在
dir.access("file.txt", .{}) catch |err| switch (err) {
    error.FileNotFound => {
        // 文件不存在
    },
    else => return err,
};

// 获取文件信息
const stat = try dir.statFile("file.txt");
std.debug.print("Size: {d}\n", .{stat.size});
```

### 路径操作

```zig
// 连接路径
const path = try std.fs.path.join(allocator, &.{ "home", "user", "file.txt" });
defer allocator.free(path);

// 获取目录名
const dir_name = std.fs.path.dirname("/home/user/file.txt");  // "/home/user"

// 获取文件名
const base_name = std.fs.path.basename("/home/user/file.txt");  // "file.txt"

// 获取扩展名
const ext = std.fs.path.extension("/home/user/file.txt");  // ".txt"

// 判断是否为绝对路径
const is_abs = std.fs.path.isAbsolute("/home/user");  // true
```

## A.4 数据结构（std）

### ArrayList

```zig
// 创建
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();

// 或使用已分配缓冲
var fixed_list = std.ArrayList(u8).initFixed(buffer);

// 添加元素
try list.append(42);
try list.appendSlice(&.{ 1, 2, 3 });

// 访问
const first = list.items[0];
const slice = list.items;  // []T

// 长度
const len = list.items.len;
const capacity = list.capacity;

// 删除
_ = list.pop();  // 删除并返回最后一个
list.removeAt(0);  // 删除指定位置
list.orderedRemove(0);  // 有序删除

// 查找
const index = list.indexOf(42);
const contains = list.contains(42);

// 调整大小
try list.resize(100);
try list.ensureTotalCapacity(1000);
```

### HashMap

```zig
// 定义类型
const StringMap = std.AutoHashMap([]const u8, i32);

// 创建
var map = StringMap.init(allocator);
defer map.deinit();

// 或带初始容量
var map = StringMap.initCapacity(allocator, 100);

// 插入
try map.put("key", 42);
const result = try map.putIfAbsent("key", 100);

// 获取
const value = map.get("key");  // 返回？i32
const entry = map.getEntry("key");  // 返回？*Entry

// 删除
_ = map.remove("key");  // 返回 bool

// 遍历
var it = map.iterator();
while (it.next()) |entry| {
    std.debug.print("{s} = {d}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
}

// 数量
const count = map.count();
```

### StringHashMap

```zig
// 专门用于字符串键的 HashMap
const StringMap = std.StringHashMap(i32);

var map = StringMap.init(allocator);
defer map.deinit();

// 使用（与 AutoHashMap 类似，但针对字符串优化）
try map.put("key", 42);
```

## A.5 时间相关（std.time）

### 时间戳

```zig
// 获取当前时间戳（毫秒）
const ms = std.time.milliTimestamp();

// 微秒
const us = std.time.microTimestamp();

// 纳秒
const ns = std.time.nanoTimestamp();

// 计算耗时
const start = std.time.milliTimestamp();
// ... 执行操作 ...
const end = std.time.milliTimestamp();
const duration = end - start;
```

### 睡眠

```zig
// 毫秒睡眠
std.time.sleep(std.time.ns_per_ms * 100);  // 睡眠 100ms

// 微秒睡眠
std.time.sleep(std.time.us_per_ms * 500);  // 睡眠 500μs
```

### 时间常量

```zig
const ns_per_us = std.time.ns_per_us;       // 1000
const ns_per_ms = std.time.ns_per_ms;       // 1_000_000
const ns_per_s = std.time.ns_per_s;         // 1_000_000_000
const us_per_ms = std.time.us_per_ms;       // 1000
const us_per_s = std.time.us_per_s;         // 1_000_000
const ms_per_s = std.time.ms_per_s;         // 1000
```

## A.6 日志（std.log）

### 日志级别

```zig
// 设置日志级别
pub const log_level = std.log.Level.info;

// 日志输出
std.log.info("Info message: {s}", .{"data"});
std.log.warn("Warning: {d}", .{count});
std.log.err("Error: {}", .{err});
std.log.debug("Debug info", .{});  // 只在 debug 级别输出
```

### 自定义日志

```zig
// 自定义日志前缀
pub const log = struct {
    pub fn log(
        comptime level: std.log.Level,
        comptime scope: @Type(.EnumLiteral),
        comptime format: []const u8,
        args: anytype,
    ) void {
        std.debug.print("[MyApp] " ++ format, args);
    }
};
```

## A.7 加密哈希（std.crypto.hash）

### 常用哈希

```zig
// MD5（不推荐用于安全场景）
var md5_hash: [16]u8 = undefined;
std.crypto.hash.Md5.hash(data, &md5_hash, .{});

// SHA256
var sha256_hash: [32]u8 = undefined;
std.crypto.hash.Sha256.hash(data, &sha256_hash, .{});

// 输出为十六进制
var hex: [64]u8 = undefined;
_ = std.fmt.binToHex(&sha256_hash, &hex, .lower);
```

## A.8 随机数（std.random）

### 生成随机数

```zig
// 获取随机生成器
var rng = std.random.DefaultPrng.init(blk: {
    var seed: u64 = undefined;
    try std.os.getrandom(std.mem.asBytes(&seed));
    break :blk seed;
});
const random = rng.random();

// 生成整数
const num = random.int(u32);
const bounded = random.intRangeAtMost(u32, 1, 100);

// 生成浮点数
const float = random.float(f64);  // 0.0 <= x < 1.0

// 随机选择
const items = [_]i32{ 1, 2, 3, 4, 5 };
const chosen = random.choice(&items);

// 打乱数组
var arr = [_]i32{ 1, 2, 3, 4, 5 };
random.shuffle(&arr);

// 填充随机字节
var buffer: [16]u8 = undefined;
try std.os.getrandom(&buffer);
```

## A.9 位操作（std.bit）

### 位集

```zig
// 定义位集
const BitSet = std.bit_set.IntegerBitSet(64);
var bits = BitSet.initEmpty();

// 设置位
bits.set(5);
bits.set(10);

// 清除位
bits.unset(5);

// 检查位
const is_set = bits.isSet(10);

// 计数
const count = bits.count();
```

## A.10 错误处理（std.debug）

### 断言和检查

```zig
// 断言（只在 Debug 模式生效）
std.debug.assert(x > 0);

// 打印堆栈跟踪
std.debug.dumpStackTrace(std.debug.getStackTrace());

// 获取调用栈
const trace = std.debug.getStackTrace();
```

## 相关文档

- [附录 B: 常用代码模式](./appendix-b-common-patterns.md) - 常见编程模式
- [附录 C: 常见问题解答](./appendix-c-faq.md) - FAQ
- [8.1 分配器基础](../part-08-memory-management/08-1-allocators.md) - 内存管理基础

---

*本文档汇总了 my-blog 项目中常用的 Zig 标准库 API，包括内存管理、格式化、文件系统、数据结构等核心模块。*
