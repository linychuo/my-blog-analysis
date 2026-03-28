# 2.3 命令行选项

> my-blog 命令行接口详解

---

## 目录

- [基本用法](#基本用法)
- [命令行选项](#命令行选项)
- [使用示例](#使用示例)
- [高级技巧](#高级技巧)

---

## 基本用法

### 命令格式

```bash
zig build run -- [OPTIONS]
```

**注意**: `--` 后面的参数传递给 my-blog 程序，而不是 Zig 构建系统。

---

## 命令行选项

| 选项 | 简写 | 默认值 | 说明 |
|------|------|--------|------|
| `--posts` | `-p` | `posts` | 文章源目录 |
| `--output` | `-o` | `build` | 输出目录 |
| `--help` | `-h` | - | 显示帮助 |

### --posts / -p

```bash
# 使用长选项
zig build run -- --posts ./blog-posts

# 使用简写
zig build run -- -p ./blog-posts
```

### --output / -o

```bash
zig build run -- --output ./public
```

### --help / -h

```bash
zig build run -- --help
```

**输出**:
```
my-blog - A static blog generator written in Zig

Usage: my-blog [OPTIONS]

Options:
  -p, --posts <dir>    Posts source directory (default: posts)
  -o, --output <dir>   Output directory (default: zig-out/blog)
  -h, --help           Show this help message
```

---

## 使用示例

```bash
# 默认运行
zig build run

# 指定文章目录
zig build run -- --posts ./my-posts

# 指定输出目录
zig build run -- --output ./public

# 组合使用
zig build run -- --posts ./content --output ./dist
```

---

## 高级技巧

### 使用别名简化命令

```bash
# 添加到 ~/.bashrc
alias blog-build='zig build run --'

# 使用
blog-build --posts ./posts --output ./dist
```

### CI/CD 集成

```yaml
# .github/workflows/deploy.yml
- name: Build
  run: zig build -Doptimize=ReleaseFast run -- --output ./public
```

---

## 常见错误

### 错误：缺少参数值

```bash
zig build run -- --posts
# Error: --posts requires a directory argument
```

### 错误：忘记 `--` 分隔符

```bash
# 错误
zig build run --posts ./posts

# 正确
zig build run -- --posts ./posts
```

---

## 小结

| 选项 | 用途 | 示例 |
|------|------|------|
| `--posts` | 指定文章目录 | `--posts ./my-posts` |
| `--output` | 指定输出目录 | `--output ./public` |
| `--help` | 显示帮助 | `--help` |

---

## 相关阅读

- [环境要求](02-1-requirements.md) - 前置条件
- [构建与运行](02-2-build-and-run.md) - 基本构建
- [输出结构](02-4-output-structure.md) - 生成的文件
