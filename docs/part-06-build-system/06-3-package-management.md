# 06-3 包管理

> Zig 依赖管理与项目实例分析

---

## 1. Zig 包管理概述

### 1.1 包管理系统

Zig 的包管理系统基于以下组件：

| 组件 | 文件 | 用途 |
|------|------|------|
| **包清单** | `build.zig.zon` | 定义包元数据和依赖 |
| **构建配置** | `build.zig` | 导入和使用依赖 |
| **包缓存** | `~/.cache/zig/` | 下载的依赖缓存 |

### 1.2 项目依赖结构

```
my-blog/
├── build.zig.zon          # 包清单
├── build.zig              # 构建配置
└── libs/                  # 本地依赖
    ├── zig-markdown/      # Markdown 解析器
    └── zig-handlebars/    # 模板引擎
```

---

## 2. build.zig.zon 详解

### 2.1 完整配置

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

### 2.2 元数据字段

#### name

```zig
.name = .my_blog,
```

- **类型**：标识符（以 `.` 开头）
- **用途**：包的唯一标识
- **命名规则**：小写字母、数字、下划线

#### version

```zig
.version = "0.1.0",
```

- **类型**：语义化版本字符串
- **格式**：`MAJOR.MINOR.PATCH`
- **示例**：`"1.0.0"`, `"0.15.2"`, `"2.1.0-beta"`

#### fingerprint

```zig
.fingerprint = 0x32781750f0cc4d,
```

- **类型**：64 位整数
- **用途**：唯一构建标识，用于缓存验证
- **生成**：通常由工具自动生成

#### minimum_zig_version

```zig
.minimum_zig_version = "0.15.2",
```

- **类型**：版本字符串
- **用途**：指定最低 Zig 编译器版本
- **检查**：构建时验证 Zig 版本

---

## 3. 依赖配置

### 3.1 本地路径依赖

**项目实例**：

```zig
.dependencies = .{
    .zig_markdown = .{
        .path = "libs/zig-markdown",
    },
    .zig_handlebars = .{
        .path = "libs/zig-handlebars",
    },
}
```

**特点**：

| 特点 | 说明 |
|------|------|
| 路径 | 相对于 `build.zig.zon` 的位置 |
| 版本 | 由本地仓库的版本决定 |
| 离线 | 不需要网络连接 |
| 控制 | 完全控制依赖代码 |

### 3.2 Git 子模块模式

项目使用 Git 子模块管理本地依赖：

```bash
# 初始化子模块
git submodule init

# 更新子模块
git submodule update
```

**目录结构**：

```
my-blog/
├── .gitmodules              # 子模块配置
├── libs/
│   ├── zig-markdown/       # Git 子模块
│   │   ├── build.zig.zon
│   │   └── src/
│   └── zig-handlebars/     # Git 子模块
│       ├── build.zig.zon
│       └── src/
```

### 3.3 远程依赖（URL）

虽然项目使用本地依赖，但 Zig 也支持远程依赖：

```zig
.dependencies = .{
    .some_lib = .{
        .url = "https://github.com/user/some-lib/archive/refs/tags/v1.0.0.tar.gz",
        .hash = "1220abcdef...",  // SHA256 哈希
    },
}
```

**远程依赖类型**：

| 类型 | 示例 | 说明 |
|------|------|------|
| Git URL | `https://github.com/...` | Git 仓库 |
| Tarball | `*.tar.gz`, `*.zip` | 压缩归档 |
| Local | `.path = "..."` | 本地路径 |

---

## 4. 依赖导入与使用

### 4.1 在 build.zig 中导入

```zig
pub fn build(b: *std.Build) void {
    // 导入依赖
    const zig_markdown_mod = b.dependency("zig_markdown", .{
        .target = target,
        .optimize = optimize,
    }).module("zig-markdown");

    const zig_handlebars_mod = b.dependency("zig_handlebars", .{
        .target = target,
        .optimize = optimize,
    }).module("zig-handlebars");
}
```

**导入流程**：

```
build.zig.zon 定义依赖
       │
       ▼
b.dependency() 加载依赖
       │
       ▼
.module() 获取导出模块
       │
       ▼
添加到 imports 列表
       │
       ▼
源代码中 @import() 使用
```

### 4.2 在源代码中使用

**src/main.zig**：

```zig
const std = @import("std");
const markdown = @import("zig-markdown");
const TemplateEngine = @import("zig-handlebars").TemplateEngine;

pub fn main() !void {
    // 使用依赖
    const html = try markdown.toHtml(allocator, markdown_content);
    
    var engine = TemplateEngine.init(allocator, "templates");
    // ...
}
```

**@import 名称匹配**：

```zig
// build.zig 中的名称
.{ .name = "zig-markdown", .module = zig_markdown_mod }

// src/main.zig 中的使用
const markdown = @import("zig-markdown");  // 名称必须匹配
```

---

## 5. paths 字段详解

### 5.1 包含路径

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

**用途**：

| 用途 | 说明 |
|------|------|
| 包范围 | 定义包包含的文件和目录 |
| 发布过滤 | 发布包时只包含这些路径 |
| 依赖解析 | 帮助构建系统定位资源 |

### 5.2 路径类型

```zig
.paths = .{
    // 单个文件
    "build.zig",
    "build.zig.zon",
    
    // 目录（递归包含）
    "src",              // 所有源文件
    "templates",        // 所有模板文件
    "static",           // 所有静态资源
    
    // 依赖目录
    "libs/zig-markdown",
    "libs/zig-handlebars",
}
```

---

## 6. 依赖的依赖

### 6.1 传递依赖

如果依赖本身也有依赖：

```
my-blog
├── zig-markdown
│   └── (无依赖)
└── zig-handlebars
    └── (无依赖)
```

**处理机制**：

- Zig 构建系统自动解析传递依赖
- 无需在 `build.zig.zon` 中显式声明

### 6.2 依赖冲突

如果两个依赖需要不同版本的同一库：

```
my-blog
├── lib-a (需要 utils v1.0)
└── lib-b (需要 utils v2.0)
```

**Zig 的解决方案**：

- 每个依赖独立解析其依赖
- 使用哈希区分不同版本
- 避免版本冲突

---

## 7. 包管理命令

### 7.1 获取依赖

```bash
# 自动下载并缓存依赖
zig build

# 依赖会被缓存到 ~/.cache/zig/
```

### 7.2 查看依赖

```bash
# 查看构建信息
zig build --summary

# 查看缓存的依赖
ls ~/.cache/zig/
```

### 7.3 清理缓存

```bash
# 清理构建缓存
zig build clean

# 手动清理依赖缓存
rm -rf ~/.cache/zig/
```

---

## 8. 依赖开发工作流

### 8.1 本地依赖开发

```bash
# 1. 克隆依赖仓库
cd libs/zig-markdown
git pull origin main

# 2. 返回项目目录
cd ../..

# 3. 重新构建
zig build
```

### 8.2 更新子模块

```bash
# 更新所有子模块
git submodule update --remote

# 更新特定子模块
git submodule update --remote libs/zig-markdown

# 提交更新
git add libs/zig-markdown
git commit -m "Update zig-markdown submodule"
```

### 8.3 添加新依赖

```bash
# 1. 克隆依赖到 libs/
cd libs
git clone https://github.com/user/new-lib.git

# 2. 添加为子模块（可选）
git submodule add https://github.com/user/new-lib.git libs/new-lib

# 3. 更新 build.zig.zon
# 添加：
# .new_lib = .{ .path = "libs/new-lib" }

# 4. 更新 build.zig
# 添加导入和使用代码
```

---

## 9. 完整实例：添加新依赖

### 9.1 场景：添加 RSS 生成库

假设要添加一个 RSS 生成库：

**步骤 1：获取依赖**

```bash
cd libs
git clone https://github.com/user/zig-rss.git
```

**步骤 2：更新 build.zig.zon**

```zig
.{
    .name = .my_blog,
    .version = "0.1.0",
    .dependencies = .{
        .zig_markdown = .{ .path = "libs/zig-markdown" },
        .zig_handlebars = .{ .path = "libs/zig-handlebars" },
        // 新增
        .zig_rss = .{ .path = "libs/zig-rss" },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
        "templates",
        "static",
        "libs/zig-markdown",
        "libs/zig-handlebars",
        "libs/zig-rss",  // 新增
    },
}
```

**步骤 3：更新 build.zig**

```zig
pub fn build(b: *std.Build) void {
    // 现有依赖
    const zig_markdown_mod = b.dependency("zig_markdown", .{}).module("zig-markdown");
    const zig_handlebars_mod = b.dependency("zig_handlebars", .{}).module("zig-handlebars");
    
    // 新增依赖
    const zig_rss_mod = b.dependency("zig_rss", .{
        .target = target,
        .optimize = optimize,
    }).module("zig-rss");

    const mod = b.addModule("my_blog", .{
        .root_source_file = b.path("src/main.zig"),
        .imports = &.{
            .{ .name = "zig-markdown", .module = zig_markdown_mod },
            .{ .name = "zig-handlebars", .module = zig_handlebars_mod },
            .{ .name = "zig-rss", .module = zig_rss_mod },  // 新增
        },
    });
    
    // ...
}
```

**步骤 4：在源代码中使用**

```zig
// src/Blogger.zig
const rss = @import("zig-rss");

pub fn generateRss(self: *Blogger, posts: []const Post) !void {
    var feed = rss.Feed.init(self.allocator);
    defer feed.deinit();
    
    for (posts) |post| {
        try feed.addItem(.{
            .title = post.title,
            .link = "/posts/" ++ post.filename,
            .description = post.content,
        });
    }
    
    const rss_xml = try feed.render();
    // 写入文件
}
```

---

## 10. 依赖最佳实践

### 10.1 版本锁定

使用 Git 子模块或提交哈希锁定版本：

```bash
# 锁定到特定提交
cd libs/zig-markdown
git checkout abc123
cd ../..
git add libs/zig-markdown
git commit -m "Lock zig-markdown to abc123"
```

### 10.2 依赖最小化

```zig
// 只导入需要的模块
const markdown = @import("zig-markdown");
// 而不是导入整个依赖

// 在 build.zig 中
.module("zig-markdown")  // 只导入主模块
```

### 10.3 依赖文档化

在 README 中记录依赖：

```markdown
## Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| zig-markdown | local | Markdown parsing |
| zig-handlebars | local | Template rendering |
```

---

## 11. 常见问题

### 11.1 依赖未找到

**错误**：

```
error: dependency 'zig_markdown' not found
```

**解决**：

```bash
# 检查 libs/ 目录
ls libs/

# 初始化子模块
git submodule init
git submodule update
```

### 11.2 模块名称不匹配

**错误**：

```
error: module 'zig-markdown' not found in dependency
```

**解决**：

```zig
// 检查 build.zig 中的模块名称
b.dependency("zig_markdown", .{}).module("zig-markdown");
//                                  ^^^^^^^^^^^^^^^^
//                                  必须与依赖导出的名称匹配

// 检查依赖的 build.zig.zon
// 查看其导出的模块名称
```

### 11.3 路径错误

**错误**：

```
error: path 'libs/zig-markdown' does not exist
```

**解决**：

```bash
# 检查路径是否正确
ls -la libs/zig-markdown/

# 如果是相对路径，确保相对于 build.zig.zon
```

---

## 12. 总结

### 12.1 包管理流程

```
1. 在 build.zig.zon 中声明依赖
       │
       ▼
2. 在 build.zig 中导入依赖
       │
       ▼
3. 配置模块 imports
       │
       ▼
4. 在源代码中 @import 使用
```

### 12.2 依赖类型对比

| 类型 | 优点 | 缺点 |
|------|------|------|
| 本地路径 | 离线可用、完全控制 | 需要手动更新 |
| Git 子模块 | 版本锁定、易于协作 | 需要 Git 知识 |
| 远程 URL | 自动下载、最新版本 | 需要网络、版本不稳定 |

---

## 相关文档

- [06-1 build.zig 结构](./06-1-build-zig-structure.md) - 构建配置基础
- [06-2 std.Build API](./06-2-std-build-api.md) - 构建 API 详解
- [01-4 项目结构](../part-01-project-overview/01-4-project-structure.md) - libs/ 目录说明
