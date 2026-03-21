# 06-2 std.Build API

> Zig 构建系统 API 详解与项目实例

---

## 1. std.Build 概述

### 1.1 核心类型

Zig 构建系统围绕以下核心类型构建：

| 类型 | 用途 |
|------|------|
| `std.Build` | 构建上下文，管理所有构建操作 |
| `std.Build.Module` | 模块（编译单元） |
| `std.Build.Step` | 构建步骤（可依赖的任务） |
| `std.Build.Artifact` | 构建产物（可执行文件、库等） |

### 1.2 项目中的使用

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // b 是构建上下文
    // 所有构建操作都通过 b 进行
}
```

---

## 2. 标准选项 API

### 2.1 standardTargetOptions

```zig
const target = b.standardTargetOptions(.{});
```

**功能**：提供标准的 `-Dtarget` 命令行选项。

**返回类型**：`std.Build.ResolvedTarget`

**可用选项**：

```bash
# 查看可用目标
zig targets

# 指定目标
zig build -Dtarget=x86_64-linux-gnu
zig build -Dtarget=aarch64-macos
```

**等价的手动配置**：

```zig
// 手动配置（不推荐）
const target = b.resolveTargetQuery(.{
    .cpu_arch = .x86_64,
    .os_tag = .linux,
    .abi = .gnu,
});

// 使用标准选项（推荐）
const target = b.standardTargetOptions(.{});
```

### 2.2 standardOptimizeOption

```zig
const optimize = b.standardOptimizeOption(.{});
```

**功能**：提供标准的 `-Doptimize` 命令行选项。

**返回类型**：`std.builtin.OptimizeMode`

**优化模式**：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `.Debug` | 无优化，完整调试信息 | 开发调试 |
| `.ReleaseSafe` | 适度优化，安全检查 | 生产环境（安全优先） |
| `.ReleaseFast` | 激进优化，较少检查 | 生产环境（性能优先） |
| `.ReleaseSmall` | 优化代码大小 | 嵌入式/小体积需求 |

**命令行使用**：

```bash
# Debug 构建（默认）
zig build

# Release 构建
zig build -Doptimize=ReleaseFast

# 最小体积构建
zig build -Doptimize=ReleaseSmall
```

---

## 3. 依赖管理 API

### 3.1 dependency()

```zig
const zig_markdown_mod = b.dependency("zig_markdown", .{
    .target = target,
    .optimize = optimize,
}).module("zig-markdown");
```

**签名**：

```zig
fn dependency(self: *Build, name: []const u8, options: anytype) *Module
```

**参数说明**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `[]const u8` | 依赖名称（来自 build.zig.zon） |
| `options` | `anytype` | 依赖配置选项 |

**常用选项**：

```zig
b.dependency("name", .{
    .target = target,           // 编译目标
    .optimize = optimize,       // 优化模式
    .some_option = value,       // 依赖特定选项
})
```

### 3.2 module()

```zig
.module("zig-markdown")
```

**功能**：从依赖中获取指定名称的模块。

**依赖的模块结构**：

```
zig-markdown 依赖
└── 导出模块
    ├── "zig-markdown" (主模块)
    └── ... (其他导出模块)
```

---

## 4. 模块 API

### 4.1 addModule

```zig
const mod = b.addModule("my_blog", .{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
    .imports = &.{
        .{
            .name = "zig-markdown",
            .module = zig_markdown_mod,
        },
        .{
            .name = "zig-handlebars",
            .module = zig_handlebars_mod,
        },
    },
});
```

**Module.Options 字段**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `root_source_file` | `?Build.LazyPath` | ✓ | 根源文件路径 |
| `target` | `ResolvedTarget` | ✗ | 编译目标（默认当前系统） |
| `optimize` | `OptimizeMode` | ✗ | 优化模式（默认 Debug） |
| `imports` | `[]const Import` | ✗ | 依赖模块列表 |
| `link_libc` | `bool` | ✗ | 是否链接 C 标准库 |

### 4.2 Import 结构

```zig
.{
    .name = "zig-markdown",     // 导入名称（用于 @import）
    .module = zig_markdown_mod, // 模块引用
}
```

**在源代码中使用**：

```zig
// src/main.zig
const markdown = @import("zig-markdown");
const handlebars = @import("zig-handlebars");
```

---

## 5. 可执行文件 API

### 5.1 addExecutable

```zig
const exe = b.addExecutable(.{
    .name = "my-blog",
    .root_module = mod,
});
```

**Executable.Options 字段**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | `[]const u8` | ✓ | 输出文件名 |
| `root_module` | `*Module` | ✓ | 根模块 |
| `target` | `ResolvedTarget` | ✗ | 目标（从模块继承） |
| `optimize` | `OptimizeMode` | ✗ | 优化（从模块继承） |

### 5.2 installArtifact

```zig
b.installArtifact(exe);
```

**功能**：将工件安装到 `zig-out/bin/`。

**输出位置**：

```
zig-out/
└── bin/
    └── my-blog    # Linux/macOS
    └── my-blog.exe # Windows
```

---

## 6. 步骤 (Step) API

### 6.1 step()

```zig
const run_step = b.step("run", "Run the app");
const test_step = b.step("test", "Run tests");
```

**签名**：

```zig
fn step(self: *Build, name: []const u8, description: []const u8) *Step
```

**参数**：

| 参数 | 说明 |
|------|------|
| `name` | 步骤名称（命令行使用） |
| `description` | 步骤描述（`zig build --help` 显示） |

### 6.2 dependOn

```zig
run_step.dependOn(&run_cmd.step);
run_cmd.step.dependOn(b.getInstallStep());
```

**功能**：建立步骤依赖关系。

**依赖链示例**：

```
run_step
  └─> run_cmd.step
        └─> install_step
              └─> compile_step
```

### 6.3 可用步骤

| 步骤 | 获取方式 | 说明 |
|------|----------|------|
| `install` | `b.getInstallStep()` | 安装所有工件 |
| `run` | 自定义 | 运行程序 |
| `test` | 自定义 | 运行测试 |
| `clean` | 内置 | 清理构建产物 |

---

## 7. 运行工件 API

### 7.1 addRunArtifact

```zig
const run_cmd = b.addRunArtifact(exe);
```

**功能**：创建一个运行可执行文件的步骤。

**返回类型**：`*std.Build.RunStep`

### 7.2 配置运行环境

```zig
// 设置工作目录
run_cmd.setCwd(.{ .src_path = .{ .owner = b, .sub_path = "." } });

// 添加命令行参数
if (b.args) |args| {
    run_cmd.addArgs(args);
}
```

**setCwd 选项**：

```zig
// 相对于构建目录
run_cmd.setCwd(.{ .src_path = .{ .owner = b, .sub_path = "data" } });

// 相对于源目录
run_cmd.setCwd(.{ .cwd_dir = .{ .owner = b, .sub_path = "." } });
```

### 7.3 添加参数

```zig
// 添加单个参数
run_cmd.addArg("--posts");
run_cmd.addArg("./content");

// 添加多个参数
run_cmd.addArgs(&.{ "--output", "./public" });

// 从构建命令行传递
if (b.args) |args| {
    run_cmd.addArgs(args);
}
```

**使用方式**：

```bash
# zig build run -- 后面的参数传递给程序
zig build run -- --posts ./content -o ./public
```

---

## 8. 测试 API

### 8.1 addTest

```zig
const exe_tests = b.addTest(.{
    .root_module = exe.root_module,
});
```

**Test.Options 字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `root_module` | `*Module` | 测试模块（通常复用可执行文件模块） |
| `target` | `ResolvedTarget` | 目标（可选） |
| `optimize` | `OptimizeMode` | 优化（可选） |

### 8.2 运行测试

```zig
const run_exe_tests = b.addRunArtifact(exe_tests);
const test_step = b.step("test", "Run tests");
test_step.dependOn(&run_exe_tests.step);
```

**测试命令**：

```bash
# 运行所有测试
zig build test

# 运行特定测试
zig build test -- --test-filter "Post.parse"
```

### 8.3 测试覆盖率

```zig
// 添加覆盖率选项
const coverage = b.option(bool, "coverage", "Enable coverage") orelse false;
if (coverage) {
    // 配置覆盖率收集
}
```

---

## 9. 路径 API

### 9.1 path()

```zig
.root_source_file = b.path("src/main.zig"),
```

**功能**：创建相对于项目根目录的路径。

**返回类型**：`Build.LazyPath`

### 9.2 路径类型

```zig
// 相对于项目根目录
b.path("src/main.zig")

// 相对于依赖
dependency.path("src/dependency.zig")

// 绝对路径（不推荐）
b.path("/absolute/path/file.zig")
```

### 9.3 LazyPath 使用

```zig
// 用于源文件
.addExecutable(.{
    .root_module = mod,
});

// 用于资源文件
exe.addCopyFileToOutput(source_file, "dest/path.txt");
```

---

## 10. 完整实例分析

### 10.1 项目 build.zig 逐行解析

```zig
// [1] 导入标准库
const std = @import("std");

// [2] build 函数入口
pub fn build(b: *std.Build) void {
    
    // [3] 获取标准选项
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // [4] 导入依赖模块
    const zig_markdown_mod = b.dependency("zig_markdown", .{
        .target = target,
        .optimize = optimize,
    }).module("zig-markdown");

    const zig_handlebars_mod = b.dependency("zig_handlebars", .{
        .target = target,
        .optimize = optimize,
    }).module("zig-handlebars");

    // [5] 创建主模块
    const mod = b.addModule("my_blog", .{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
        .imports = &.{
            .{ .name = "zig-markdown", .module = zig_markdown_mod },
            .{ .name = "zig-handlebars", .module = zig_handlebars_mod },
        },
    });

    // [6] 创建可执行文件
    const exe = b.addExecutable(.{
        .name = "my-blog",
        .root_module = mod,
    });

    // [7] 安装工件
    b.installArtifact(exe);

    // [8] 创建 run 步骤
    const run_step = b.step("run", "Run the app");
    const run_cmd = b.addRunArtifact(exe);
    run_step.dependOn(&run_cmd.step);
    run_cmd.step.dependOn(b.getInstallStep());
    run_cmd.setCwd(.{ .src_path = .{ .owner = b, .sub_path = "." } });

    // [9] 传递命令行参数
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    // [10] 创建 test 步骤
    const exe_tests = b.addTest(.{
        .root_module = exe.root_module,
    });

    const run_exe_tests = b.addRunArtifact(exe_tests);
    const test_step = b.step("test", "Run tests");
    test_step.dependOn(&run_exe_tests.step);
}
```

---

## 11. 高级用法

### 11.1 条件编译

```zig
// 根据目标平台选择配置
if (target.result.os.tag == .windows) {
    mod.addCMacro("PLATFORM_WINDOWS", "1");
} else {
    mod.addCMacro("PLATFORM_UNIX", "1");
}
```

### 11.2 自定义选项

```zig
// 添加自定义构建选项
const enable_feature = b.option(
    bool,
    "enable-feature",
    "Enable experimental feature"
) orelse false;

if (enable_feature) {
    mod.addCMacro("ENABLE_FEATURE", "1");
}
```

### 11.3 多模块构建

```zig
// 创建多个模块
const lib_mod = b.addModule("mylib", .{
    .root_source_file = b.path("src/lib.zig"),
});

const exe_mod = b.addModule("myapp", .{
    .root_source_file = b.path("src/main.zig"),
    .imports = &.{
        .{ .name = "mylib", .module = lib_mod },
    },
});
```

---

## 12. 注意事项

### 12.1 模块导入顺序

```zig
// 正确：先创建依赖模块
const dep = b.dependency("dep", .{}).module("dep");
const mod = b.addModule("main", .{
    .imports = &.{.{ .name = "dep", .module = dep }},
});

// 错误：模块未定义就使用
const mod = b.addModule("main", .{
    .imports = &.{.{ .name = "dep", .module = dep }}, // dep 未定义！
});
const dep = b.dependency("dep", .{}).module("dep");
```

### 12.2 步骤依赖循环

```zig
// 错误：循环依赖
step_a.dependOn(&step_b.step);
step_b.dependOn(&step_a.step);  // 会导致构建失败
```

### 12.3 内存管理

```zig
// build.zig 中的内存由构建系统自动管理
// 不需要手动释放分配的资源

pub fn build(b: *std.Build) void {
    // 所有分配都使用 b.allocator
    // 构建完成后自动清理
}
```

---

## 相关文档

- [06-1 build.zig 结构](./06-1-build-zig-structure.md) - 构建配置基础
- [06-3 包管理](./06-3-package-management.md) - 依赖管理深入
- [04-4 comptime](../part-04-control-flow/04-4-comptime.md) - @import 机制
