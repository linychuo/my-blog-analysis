# 07-1 入口点分析 (main.zig)

> my-blog 程序入口点与命令行解析

---

## 1. main.zig 概览

### 1.1 文件结构

```
main.zig (57 lines)
├── Imports (3 lines)
├── main() function (34 lines)
└── printHelp() function (13 lines)
```

### 1.2 完整代码

```zig
const std = @import("std");
const markdown = @import("zig-markdown");
const Blogger = @import("Blogger.zig").Blogger;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Parse command line arguments
    var args = try std.process.argsWithAllocator(allocator);
    defer args.deinit();
    _ = args.skip(); // Skip program name

    var posts_dir: []const u8 = "posts";
    var dest_dir: []const u8 = "build";

    // Parse optional arguments: --posts --output
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

    // Create blogger and generate site
    var blogger = Blogger.new(allocator, posts_dir, dest_dir);
    try blogger.generate();
}

fn printHelp() void {
    std.debug.print(
        \\hello-zig - A static blog generator written in Zig
        \\
        \\Usage: hello-zig [OPTIONS]
        \\
        \\Options:
        \\  -p, --posts    Posts source directory (default: posts)
        \\  -o, --output   Output directory (default: zig-out/blog)
        \\  -h, --help     Show this help message
        \\
        \\Example:
        \\  hello-zig --posts ./blog-posts --output ./public
        \\
    , .{});
}
```

---

## 2. 导入部分分析

### 2.1 标准库导入

```zig
const std = @import("std");
```

**说明**：
- `@import("std")` 是 Zig 标准库的入口
- 在编译时解析，无运行时开销
- 提供内存分配、文件系统、进程管理等 API

### 2.2 依赖导入

```zig
const markdown = @import("zig-markdown");
const Blogger = @import("Blogger.zig").Blogger;
```

**依赖链**：

```
main.zig
├── zig-markdown (外部依赖)
│   └── Markdown → HTML 转换
└── Blogger.zig (本地模块)
    └── 博客生成核心逻辑
```

**导入语法解析**：

```zig
// 导入整个模块
const markdown = @import("zig-markdown");

// 导入模块中的特定类型
const Blogger = @import("Blogger.zig").Blogger;
// 等价于
const blogger_mod = @import("Blogger.zig");
const Blogger = blogger_mod.Blogger;
```

---

## 3. main() 函数详解

### 3.1 函数签名

```zig
pub fn main() !void
```

**解析**：

| 部分 | 含义 |
|------|------|
| `pub` | 公共函数，程序入口点 |
| `fn main()` | 主函数名称 |
| `!void` | 返回错误联合类型，可能失败但无返回值 |

**错误传播**：
- 任何未处理的错误会终止程序并返回错误码
- Zig 运行时自动将错误转换为进程退出码

### 3.2 内存分配器设置

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
defer _ = gpa.deinit();
const allocator = gpa.allocator();
```

**内存管理流程**：

```
1. 创建 GeneralPurposeAllocator
   │
   ▼
2. 注册 defer 清理
   │
   ▼
3. 获取 allocator 引用
   │
   ▼
4. 程序执行中使用 allocator
   │
   ▼
5. defer 自动调用 deinit()
```

**GeneralPurposeAllocator 特点**：

| 特点 | 说明 |
|------|------|
| 调试友好 | 检测内存泄漏 |
| 通用 | 适用于各种分配模式 |
| 性能适中 | 比 Arena 慢，比 Page 快 |

**defer 解析**：

```zig
defer _ = gpa.deinit();
// 在 main() 返回时执行
// _ 表示忽略返回值
```

### 3.3 命令行参数解析

```zig
var args = try std.process.argsWithAllocator(allocator);
defer args.deinit();
_ = args.skip(); // Skip program name
```

**参数迭代器**：

```
命令行：./my-blog --posts ./content -o ./public

args 迭代器:
  [0] "./my-blog"     ← skip() 跳过
  [1] "--posts"
  [2] "./content"
  [3] "-o"
  [4] "./public"
```

### 3.4 默认值设置

```zig
var posts_dir: []const u8 = "posts";
var dest_dir: []const u8 = "build";
```

**变量类型**：

| 变量 | 类型 | 默认值 | 可变性 |
|------|------|--------|--------|
| `posts_dir` | `[]const u8` | `"posts"` | `var` (可变) |
| `dest_dir` | `[]const u8` | `"build"` | `var` (可变) |

**为什么使用 `var`**：
- 命令行参数可能修改这些值
- 字符串字面量是 `[]const u8` 类型

### 3.5 参数解析循环

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

**控制流分析**：

```
while (args.next()) |arg|
    │
    ├─ if "--posts" or "-p"
    │   └─ 获取下一个参数作为目录
    │
    ├─ else if "--output" or "-o"
    │   └─ 获取下一个参数作为输出目录
    │
    └─ else if "--help" or "-h"
        └─ 显示帮助并退出
```

**orelse 模式**：

```zig
args.next() orelse {
    // 如果 next() 返回 null（没有更多参数）
    // 执行错误处理
    std.debug.print("Error: ...\n", .{});
    std.process.exit(1);
}
```

### 3.6 博客生成调用

```zig
var blogger = Blogger.new(allocator, posts_dir, dest_dir);
try blogger.generate();
```

**执行流程**：

```
Blogger.new()
    │
    ▼
创建 Blogger 实例
    │
    ▼
blogger.generate()
    │
    ▼
生成完整博客站点
    │
    ▼
try 传播可能的错误
```

---

## 4. printHelp() 函数分析

### 4.1 函数签名

```zig
fn printHelp() void
```

| 部分 | 含义 |
|------|------|
| `fn` (无 `pub`) | 私有函数，只在 main.zig 内可见 |
| `void` | 无返回值，永不失败 |

### 4.2 多行字符串字面量

```zig
std.debug.print(
    \\hello-zig - A static blog generator written in Zig
    \\
    \\Usage: hello-zig [OPTIONS]
    \\
    \\Options:
    \\  -p, --posts    Posts source directory (default: posts)
    \\  -o, --output   Output directory (default: zig-out/blog)
    \\  -h, --help     Show this help message
    \\
    \\Example:
    \\  hello-zig --posts ./blog-posts --output ./public
    \\
, .{});
```

**多行字符串语法**：

```zig
// 每行以 \\ 开头
// 编译器自动移除行首的 \\ 和空格
// 保留原始格式

const help = 
    \\Line 1
    \\Line 2
    \\
;
```

**格式化输出**：

```zig
std.debug.print("Format: {s}, Number: {d}\n", .{ "hello", 42 });
//                                        ^^^^ 参数数组
```

---

## 5. 错误处理分析

### 5.1 try 的使用

```zig
var args = try std.process.argsWithAllocator(allocator);
//         ^^^ 如果失败，main() 立即返回错误

try blogger.generate();
//  ^^^ 如果 generate() 失败，main() 立即返回错误
```

**错误传播链**：

```
main() !void
  │
  ├─ argsWithAllocator() → 可能失败
  │   └─ 内存不足
  │
  └─ blogger.generate() → 可能失败
      ├─ 文件读写错误
      ├─ 模板解析错误
      └─ Markdown 解析错误
```

### 5.2 orelse 错误处理

```zig
posts_dir = args.next() orelse {
    std.debug.print("Error: --posts requires a directory argument\n", .{});
    std.process.exit(1);
};
```

**处理逻辑**：

```
args.next()
    │
    ├─ 返回 Some(value) → 赋值给 posts_dir
    │
    └─ 返回 null → 执行 orelse 块
        │
        ├─ 打印错误信息
        │
        └─ 以错误码 1 退出程序
```

---

## 6. 程序执行流程

### 6.1 完整流程图

```
程序启动
    │
    ▼
创建内存分配器 (gpa)
    │
    ▼
解析命令行参数
    │
    ├─ 无参数 → 使用默认值 (posts/, build/)
    ├─ --posts DIR → 设置 posts_dir
    ├─ --output DIR → 设置 dest_dir
    └─ --help → 显示帮助，退出
    │
    ▼
创建 Blogger 实例
    │
    ▼
调用 blogger.generate()
    │
    ├─ 复制静态文件
    ├─ 加载文章
    ├─ 生成文章页面
    ├─ 生成首页
    ├─ 生成关于页面
    └─ 生成标签页面
    │
    ▼
程序结束
    │
    ▼
defer 清理内存分配器
```

### 6.2 命令行使用示例

```bash
# 使用默认值
zig build run

# 指定文章目录和输出目录
zig build run -- --posts ./content --output ./public

# 使用短选项
zig build run -- -p ./blog -o ./site

# 显示帮助
zig build run -- --help
```

---

## 7. 代码设计模式

### 7.1 资源管理模式

```zig
// 1. 创建资源
var gpa = std.heap.GeneralPurposeAllocator(.{}){};

// 2. 注册清理
defer _ = gpa.deinit();

// 3. 使用资源
const allocator = gpa.allocator();
// ... 使用 allocator ...

// 4. 自动清理（defer 在函数返回时执行）
```

### 7.2 参数验证模式

```zig
value = args.next() orelse {
    // 参数缺失时的错误处理
    std.debug.print("Error: ...\n", .{});
    std.process.exit(1);
};
```

### 7.3 默认值模式

```zig
// 先设置默认值
var posts_dir: []const u8 = "posts";

// 命令行参数可覆盖
while (args.next()) |arg| {
    if (...) {
        posts_dir = args.next() orelse ...;
    }
}
```

---

## 8. 注意事项

### 8.1 内存泄漏检测

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
defer _ = gpa.deinit();  // 检测泄漏的关键

// 如果有未释放的内存，deinit() 会触发警告
```

### 8.2 字符串生命周期

```zig
var posts_dir: []const u8 = "posts";  // 指向静态字符串
// 程序整个生命周期都有效

// 如果是动态分配的字符串，需要确保在释放前使用
```

### 8.3 错误码转换

```zig
// Zig 自动将错误转换为进程退出码
// error.OutOfMemory → 退出码 1
// 成功返回 → 退出码 0

// 手动退出
std.process.exit(1);  // 错误退出
std.process.exit(0);  // 成功退出
```

---

## 相关文档

- [07-2 Blogger 结构体](./07-2-blogger-struct.md) - 核心生成逻辑
- [06-1 build.zig 结构](../part-06-build-system/06-1-build-zig-structure.md) - 构建配置
- [05-1 函数基础](../part-05-functions/05-1-function-basics.md) - main() 函数签名
