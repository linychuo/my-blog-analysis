# 2.1 环境要求

> 构建和运行 my-blog 所需的环境和工具

---

## 目录

- [核心要求](#核心要求)
- [安装 Zig 编译器](#安装-zig-编译器)
- [验证安装](#验证安装)
- [可选工具](#可选工具)
- [常见问题](#常见问题)

---

## 核心要求

### 必需工具

| 工具 | 最低版本 | 推荐版本 | 用途 |
|------|----------|----------|------|
| **Zig** | 0.15.2 | 0.15.2+ | 编译和运行 |
| **Git** | 2.0+ | 最新 | 代码管理 |

### 操作系统支持

| 系统 | 支持状态 | 说明 |
|------|----------|------|
| **Linux** | ✅ 完全支持 | x86_64, aarch64 |
| **macOS** | ✅ 完全支持 | Intel, Apple Silicon |
| **Windows** | ✅ 完全支持 | WS L2 或原生 |
| **FreeBSD** | ⚠️ 社区支持 | 可能需要额外配置 |

### 硬件要求

| 组件 | 最低要求 | 推荐配置 |
|------|----------|----------|
| **CPU** | 双核 | 四核或更多 |
| **内存** | 512 MB | 2 GB+ |
| **磁盘** | 100 MB | 500 MB+ |

---

## 安装 Zig 编译器

### 方法一：官方网站下载（推荐）

访问 [Zig 下载页面](https://ziglang.org/download/) 获取对应系统的二进制包。

#### Linux (x86_64)

```bash
# 1. 下载
wget https://ziglang.org/download/0.15.2/zig-linux-x86_64-0.15.2.tar.xz

# 2. 解压
tar -xf zig-linux-x86_64-0.15.2.tar.xz

# 3. 移动到系统目录
sudo mv zig-linux-x86_64-0.15.2 /opt/zig

# 4. 添加到 PATH
echo 'export PATH="/opt/zig:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

#### macOS (Intel/Apple Silicon)

```bash
# 使用 Homebrew（推荐）
brew install zig@0.15

# 或手动安装
wget https://ziglang.org/download/0.15.2/zig-macos-x86_64-0.15.2.tar.xz
# Apple Silicon:
# wget https://ziglang.org/download/0.15.2/zig-macos-aarch64-0.15.2.tar.xz

tar -xf zig-macos-*.tar.xz
sudo mv zig-macos-* /opt/zig
export PATH="/opt/zig:$PATH"
```

#### Windows

1. 下载 `zig-windows-x86_64-0.15.2.zip`
2. 解压到 `C:\zig`
3. 添加 `C:\zig` 到系统 PATH 环境变量
4. 重启终端验证

### 方法二：使用包管理器

#### Ubuntu/Debian

```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:zig-lang/zig
sudo apt update
sudo apt install zig-0.15
```

#### macOS (Homebrew)

```bash
brew install zig@0.15
```

---

## 验证安装

### 检查 Zig 版本

```bash
zig version
```

期望输出：`0.15.2`

### 测试 Zig 编译器

```bash
echo 'const std = @import("std");
pub fn main() void {
    std.debug.print("Hello, Zig!\n", .{});
}' > test.zig

zig run test.zig
rm test.zig
```

期望输出：`Hello, Zig!`

---

## 可选工具

### 代码编辑器/IDE

| 编辑器 | 插件 | 推荐度 |
|--------|------|--------|
| **VS Code** | Zig Language Server | ⭐⭐⭐⭐⭐ |
| **Neovim** | zig.vim + zls | ⭐⭐⭐⭐ |
| **IntelliJ** | Zig 插件 | ⭐⭐⭐ |

---

## 小结

| 步骤 | 操作 | 验证 |
|------|------|------|
| 1. 安装 Zig | 下载或使用包管理器 | `zig version` |
| 2. 配置 PATH | 添加到环境变量 | 终端可执行 `zig` |
| 3. 安装编辑器 | VS Code + Zig 插件 | 语法高亮正常 |

---

## 相关阅读

- [构建与运行](02-2-build-and-run.md) - 下一步：编译项目
- [命令行选项](02-3-cli-options.md) - 自定义构建参数
