# 06-1 build.zig 结构

> Zig 构建系统配置与项目实例分析

---

## 1. build.zig 基础

### 1.1 什么是 build.zig

`build.zig` 是 Zig 项目的构建配置文件，使用 Zig 语言本身编写。它定义了：

- 编译目标（可执行文件、库等）
- 依赖管理
- 自定义构建步骤
- 测试配置

### 1.2 基本结构

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // 1. 配置目标和优化选项
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // 2. 创建模块和依赖
    // ...

    // 3. 创建可执行文件
    // ...

    // 4. 定义构建步骤
    // ...
}
```

---

## 2. 项目 build.zig 完整分析

### 2.1 完整代码

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // 导入依赖
    const zig_markdown_mod = b.dependency("zig_markdown", .{
        .target = target,
        .optimize = optimize,
    }).module("zig-markdown");

    const zig_handlebars_mod = b.dependency("zig_handlebars", .{
        .target = target,
        .optimize = optimize,
    }).module("zig-handlebars");

    // 创建 my_blog 模块
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

    // 创建可执行文件
    const exe = b.addExecutable(.{
        .name = "my-blog",
        .root_module = mod,
    });

    b.installArtifact(exe);

    // 运行步骤
    const run_step = b.step("run", "Run the app");
    const run_cmd = b.addRunArtifact(exe);
    run_step.dependOn(&run_cmd.step);
    run_cmd.step.dependOn(b.getInstallStep());
    run_cmd.setCwd(.{ .src_path = .{ .owner = b, .sub_path = "." } });

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    // 测试步骤
    const exe_tests = b.addTest(.{
        .root_module = exe.root_module,
    });

    const run_exe_tests = b.addRunArtifact(exe_tests);
    const test_step = b.step("test", "Run tests");
    test_step.dependOn(&run_exe_tests.step);
}
```

---

## 3. 构建配置详解

### 3.1 目标和优化选项

```zig
const target = b.standardTargetOptions(.{});
const optimize = b.standardOptimizeOption(.{});
```

**说明**：

| 变量 | 类型 | 用途 |
|------|------|------|
| `target` | `std.Build.ResolvedTarget` | 编译目标（CPU 架构、OS） |
| `optimize` | `std.builtin.OptimizeMode` | 优化模式（Debug/Release） |

**命令行覆盖**：

```bash
# 指定目标架构
zig build -Dtarget=x86_64-linux

# 指定优化模式
zig build -Doptimize=ReleaseFast
```

### 3.2 依赖导入

```zig
const zig_markdown_mod = b.dependency("zig_markdown", .{
    .target = target,
    .optimize = optimize,
}).module("zig-markdown");

const zig_handlebars_mod = b.dependency("zig_handlebars", .{
    .target = target,
    .optimize = optimize,
}).module("zig-handlebars");
```

**依赖链**：

```
my-blog (主程序)
├── zig-markdown (Markdown 解析器)
└── zig-handlebars (模板引擎)
```

### 3.3 模块定义

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

**模块配置字段**：

| 字段 | 值 | 说明 |
|------|-----|------|
| `root_source_file` | `b.path("src/main.zig")` | 入口源文件 |
| `target` | `target` | 编译目标 |
| `optimize` | `optimize` | 优化模式 |
| `imports` | `[2]Import` | 依赖模块列表 |

---

## 4. 可执行文件配置

### 4.1 创建可执行文件

```zig
const exe = b.addExecutable(.{
    .name = "my-blog",
    .root_module = mod,
});
```

**配置选项**：

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `"my-blog"` | 输出文件名 |
| `root_module` | `mod` | 根模块（包含所有依赖） |

### 4.2 安装工件

```zig
b.installArtifact(exe);
```

这会将编译好的可执行文件安装到 `zig-out/bin/` 目录。

---

## 5. 构建步骤

### 5.1 run 步骤

```zig
const run_step = b.step("run", "Run the app");
const run_cmd = b.addRunArtifact(exe);
run_step.dependOn(&run_cmd.step);
run_cmd.step.dependOn(b.getInstallStep());
run_cmd.setCwd(.{ .src_path = .{ .owner = b, .sub_path = "." } });

if (b.args) |args| {
    run_cmd.addArgs(args);
}
```

**步骤依赖链**：

```
run_step
└── run_cmd (运行程序)
    └── install_step (安装工件)
        └── compile (编译可执行文件)
```

**传递命令行参数**：

```bash
# 传递参数给程序
zig build run -- --posts ./content -o ./public
```

### 5.2 test 步骤

```zig
const exe_tests = b.addTest(.{
    .root_module = exe.root_module,
});

const run_exe_tests = b.addRunArtifact(exe_tests);
const test_step = b.step("test", "Run tests");
test_step.dependOn(&run_exe_tests.step);
```

**运行测试**：

```bash
zig build test
```

---

## 6. build.zig.zon 配置

### 6.1 完整配置

```zig
.{
    .name = .my_blog,
    .version = "0.1.0",
    .fingerprint = 0x32781750f0cc4d,
    .minimum_zig_version = "0.15.2",
    .dependencies = .{
        .zig_markdown = .{
            .path = "libs/zig-markdown",
        },
        .zig_handlebars = .{
            .path = "libs/zig-handlebars",
        },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
        "templates",
        "static",
        "libs/zig-markdown",
        "libs/zig-handlebars",
    },
}
```

### 6.2 字段说明

| 字段 | 值 | 说明 |
|------|-----|------|
| `.name` | `.my_blog` | 包名（标识符） |
| `.version` | `"0.1.0"` | 语义化版本号 |
| `.fingerprint` | `0x32781750f0cc4d` | 唯一构建指纹 |
| `.minimum_zig_version` | `"0.15.2"` | 最低 Zig 版本要求 |

### 6.3 依赖配置

```zig
.dependencies = .{
    .zig_markdown = .{
        .path = "libs/zig-markdown",  // 本地路径依赖
    },
    .zig_handlebars = .{
        .path = "libs/zig-handlebars",  // 本地路径依赖
    },
}
```

**依赖类型**：

| 类型 | 示例 | 说明 |
|------|------|------|
| 本地路径 | `.path = "libs/..."` | 本地子模块或目录 |
| URL | `.url = "https://..."` | 远程 Git 仓库 |

### 6.4 包含路径

```zig
.paths = .{
    "build.zig",
    "build.zig.zon",
    "src",
    "templates",
    "static",
    "libs/zig-markdown",
    "libs/zig-handlebars",
}
```

这些路径用于确定包的内容范围。

---

## 7. 构建流程

### 7.1 完整构建流程

```
zig build
    │
    ▼
1. 解析 build.zig.zon
    │
    ▼
2. 加载依赖 (zig-markdown, zig-handlebars)
    │
    ▼
3. 执行 build() 函数
    │
    ├── 创建模块 (my_blog)
    │   ├── 配置 imports
    │   └── 设置 root_source_file
    │
    ├── 创建可执行文件 (my-blog)
    │   └── 编译所有源文件
    │
    └── 安装工件
        └── 输出到 zig-out/bin/my-blog
```

### 7.2 输出目录结构

```
zig-out/
├── bin/
│   └── my-blog          # 可执行文件
├── blog/                # 生成的博客（运行后）
│   ├── index.html
│   ├── about.html
│   ├── style.css
│   └── tags/
└── lib/                 # 库文件（如果有）
```

---

## 8. 常用构建命令

| 命令 | 说明 |
|------|------|
| `zig build` | 默认构建（编译 + 安装） |
| `zig build run` | 编译并运行 |
| `zig build run -- --help` | 运行并传递参数 |
| `zig build test` | 运行测试 |
| `zig build -Doptimize=ReleaseFast` | Release 模式构建 |
| `zig build -Dtarget=x86_64-linux` | 指定目标平台 |
| `zig build clean` | 清理构建产物 |

---

## 9. 扩展示例

### 9.1 添加自定义步骤

```zig
// 添加文档生成步骤
const docs_step = b.step("docs", "Generate documentation");
const docs_cmd = b.addSystemCommand(&.{ "zig", "doc" });
docs_step.dependOn(&docs_cmd.step);
```

### 9.2 条件编译

```zig
// 根据优化模式选择功能
if (optimize == .Debug) {
    // Debug 模式：启用额外检查
    mod.addCMacro("DEBUG", "1");
} else {
    // Release 模式：禁用调试代码
    mod.addCMacro("DEBUG", "0");
}
```

### 9.3 多目标构建

```zig
// 同时构建多个目标
const targets = .{
    .{ .arch = .x86_64, .os = .linux },
    .{ .arch = .aarch64, .os = .macos },
    .{ .arch = .x86_64, .os = .windows },
};

for (targets) |t| {
    const target = b.resolveTargetQuery(.{
        .cpu_arch = t.arch,
        .os_tag = t.os,
    });
    // 为每个目标创建可执行文件
}
```

---

## 10. 注意事项

### 10.1 路径处理

```zig
// 推荐：使用 b.path()
.root_source_file = b.path("src/main.zig"),

// 不推荐：硬编码字符串
.root_source_file = "src/main.zig",
```

### 10.2 依赖顺序

```zig
// 正确：先导入依赖，再创建模块
const dep_mod = b.dependency("dep", .{}).module("dep");
const mod = b.addModule("main", .{
    .imports = &.{.{ .name = "dep", .module = dep_mod }},
});

// 错误：循环依赖会导致编译失败
```

### 10.3 内存管理

```zig
// build.zig 中的内存由构建系统管理
// 不需要手动释放

pub fn build(b: *std.Build) void {
    // 所有分配都使用 b.allocator
    // 构建完成后自动释放
}
```

---

## 相关文档

- [06-2 std.Build API](./06-2-std-build-api.md) - 构建 API 详解
- [06-3 包管理](./06-3-package-management.md) - 依赖管理深入
- [04-4 comptime](../part-04-control-flow/04-4-comptime.md) - @import 与编译时执行
