# my-blog 技术分析文档

> 对 Zig 静态博客生成器的完整技术剖析

[![Documentation Status](https://img.shields.io/badge/docs-in--progress-blue)]()
[![Zig Version](https://img.shields.io/badge/zig-0.15.2-orange)](https://ziglang.org)
[![License](https://img.shields.io/badge/license-MIT-green)]()

---

## 📖 在线文档

**完整技术文档请访问**: [linychuo.github.io/my-blog-analysis](https://linychuo.github.io/my-blog-analysis)

---

## 📚 文档结构

本文档集共 **14 个部分**，**53 篇文章**，完整覆盖：

### 基础篇
- **项目概览** - 背景、技术栈、功能、结构
- **快速开始** - 环境、构建、CLI、输出
- **Zig 语言基础** - 变量、类型、结构体、枚举
- **控制流** - if/else、循环、switch、comptime
- **函数** - 基础、参数、泛型

### 进阶篇
- **构建系统** - build.zig、std.Build API、包管理
- **核心模块分析** - 入口点、Post、Blogger、文件系统
- **内存管理** - 分配器、所有权、defer 模式
- **错误处理** - 类型、传播、处理模式

### 高级篇
- **模板引擎** - 架构、语法、继承
- **Markdown 解析器** - 解析器、块级、内联、HTML 生成
- **样式与前端** - CSS 架构、主题系统、前端功能
- **扩展指南** - 编写博客、RSS、搜索、性能优化
- **测试与调试** - 单元测试、调试技巧、内存检测

### 附录
- 标准库 API、常用模式、常见问题

---

## 🎯 关于项目

**my-blog** 是一个使用 Zig 0.15.2 编写的静态博客生成器，具有以下特点：

- ⚡ **高性能** - 编译型语言，生成速度快
- 🎨 **现代设计** - 响应式布局，暗色/亮色主题
- 📝 **Markdown 支持** - CommonMark + 数学公式 + 代码高亮
- 🏷️ **标签系统** - 自动生成标签页面
- 🔧 **可扩展** - 模块化设计，易于添加新功能

### 技术栈

| 组件 | 技术 |
|------|------|
| 语言 | Zig 0.15.2 |
| Markdown 解析 | 自定义实现 (zig-markdown) |
| 模板引擎 | 自定义 Handlebars 风格 (zig-handlebars) |
| 数学公式 | KaTeX 0.12.0 |
| 代码高亮 | highlight.js 11.9.0 |

---

## 🔗 相关链接

- **源代码仓库**: [github.com/linychuo/my-blog](https://github.com/linychuo/my-blog)
- **Zig 官网**: [ziglang.org](https://ziglang.org)
- **Zig 文档**: [ziglang.org/documentation](https://ziglang.org/documentation)

---

## 📄 许可证

MIT License
