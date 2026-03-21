# 2.2 构建与运行

> 编译和运行 my-blog 项目的完整指南

---

## 目录

- [获取源代码](#获取源代码)
- [构建项目](#构建项目)
- [运行博客生成器](#运行博客生成器)
- [构建模式](#构建模式)
- [故障排除](#故障排除)

---

## 获取源代码

```bash
# 克隆仓库
git clone git@github.com:linychuo/my-blog.git
cd my-blog
```

---

## 构建项目

### 基本构建命令

```bash
# Debug 模式构建并运行
zig build run
```

**输出示例**:
```
Generating blog...
  Posts directory: posts
  Output directory: build
  Copied: style.css
  Loaded: my-first-post.markdown
  Generated: index.html
  Generated: about.html
  Generated: tags/zig.html
Blog generation complete!
```

### 仅构建不运行

```bash
# 仅编译
zig build

# 查看生成的文件
ls -la zig-out/bin/
```

---

## 运行博客生成器

### 默认运行

```bash
zig build run
```

### 自定义路径

```bash
zig build run -- --posts ./my-posts --output ./public
```

### 查看帮助

```bash
zig build run -- --help
```

---

## 构建模式

| 模式 | 命令 | 用途 |
|------|------|------|
| **Debug** | `zig build run` | 开发调试 |
| **ReleaseFast** | `zig build -Doptimize=ReleaseFast run` | 生产环境 |
| **ReleaseSafe** | `zig build -Doptimize=ReleaseSafe run` | 测试环境 |
| **ReleaseSmall** | `zig build -Doptimize=ReleaseSmall run` | 小体积 |

---

## 运行测试

```bash
# 运行所有测试
zig build test
```

---

## 故障排除

### 问题 1: 依赖库找不到

```bash
# 确认 libs/ 目录存在依赖
ls -la libs/
```

### 问题 2: Zig 版本不匹配

```bash
# 检查版本
zig version
```

### 问题 3: 模板文件找不到

```bash
# 确认在正确的目录运行
pwd
ls templates/
```

---

## 小结

| 命令 | 用途 |
|------|------|
| `zig build run` | Debug 模式构建并运行 |
| `zig build -Doptimize=ReleaseFast run` | Release 模式运行 |
| `zig build test` | 运行测试 |

---

## 相关阅读

- [环境要求](02-1-requirements.md) - 前置条件
- [命令行选项](02-3-cli-options.md) - 自定义参数
- [输出结构](02-4-output-structure.md) - 生成的文件
