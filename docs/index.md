# my-blog 技术文档

> 完整剖析 Zig 静态博客生成器的技术实现

[![Zig Version](https://img.shields.io/badge/zig-0.15.2-orange)](https://ziglang.org)
[![Documentation](https://img.shields.io/badge/docs-53%20articles-blue)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

---

## 🎯 关于本项目

**my-blog** 是一个使用 Zig 0.15.2 编写的静态博客生成器，从 Markdown 文件生成完整的静态 HTML 网站。

=== "核心特性"
    - ⚡ **高性能** - 编译型语言，快速生成
    - 🎨 **现代 UI** - 响应式设计，暗色/亮色主题
    - 📝 **Markdown** - CommonMark + 数学公式 + 代码高亮
    - 🏷️ **标签系统** - 自动标签页生成
    - 🔧 **可扩展** - 模块化架构

=== "技术栈"
    | 组件 | 技术 |
    |------|------|
    | 语言 | Zig 0.15.2 |
    | Markdown | 自定义 zig-markdown |
    | 模板 | 自定义 zig-handlebars |
    | 数学公式 | KaTeX 0.12.0 |
    | 代码高亮 | highlight.js 11.9.0 |

=== "项目链接"
    - 📦 [源代码仓库](https://github.com/linychuo/my-blog)
    - 📖 [Zig 官方文档](https://ziglang.org/documentation)
    - 📚 [标准库参考](https://ziglang.org/documentation/master/std)

---

## 📚 文档结构

本文档共 **14 个部分**，包含 **53 篇文章**，全面覆盖项目各个方面。

### 🟢 基础篇

| 部分 | 主题 | 文章数 | 核心内容 |
|------|------|--------|----------|
| [第一部分](part-01-project-overview/index.md) | 项目概览 | 4 篇 | 背景、技术栈、功能、结构 |
| [第二部分](part-02-quick-start/index.md) | 快速开始 | 4 篇 | 环境、构建、CLI、输出 |
| [第三部分](part-03-zig-basics/index.md) | Zig 基础 | 4 篇 | 变量、类型、结构体、枚举 |
| [第四部分](part-04-control-flow/index.md) | 控制流 | 4 篇 | if/else、循环、switch、comptime |
| [第五部分](part-05-functions/index.md) | 函数 | 3 篇 | 基础、参数、泛型 |

### 🟡 进阶篇

| 部分 | 主题 | 文章数 | 核心内容 |
|------|------|--------|----------|
| [第六部分](part-06-build-system/index.md) | 构建系统 | 3 篇 | build.zig、std.Build、包管理 |
| [第七部分](part-07-core-modules/index.md) | 核心模块 | 4 篇 | main.zig、Post、Blogger、文件系统 |
| [第八部分](part-08-memory-management/index.md) | 内存管理 | 3 篇 | 分配器、所有权、defer 模式 |
| [第九部分](part-09-error-handling/index.md) | 错误处理 | 3 篇 | 错误类型、传播、处理模式 |

### 🔴 高级篇

| 部分 | 主题 | 文章数 | 核心内容 |
|------|------|--------|----------|
| [第十部分](part-10-template-engine/index.md) | 模板引擎 | 4 篇 | 架构、语法、继承、详解 |
| [第十一部分](part-11-markdown-parser/index.md) | Markdown 解析器 | 4 篇 | 架构、块级、内联、HTML 生成 |
| [第十二部分](part-12-styling/index.md) | 样式与前端 | 3 篇 | CSS 架构、主题系统、前端功能 |
| [第十三部分](part-13-extension-guides/index.md) | 扩展指南 | 4 篇 | 写作、RSS、搜索、优化 |
| [第十四部分](part-14-testing/index.md) | 测试与调试 | 3 篇 | 单元测试、调试、内存检测 |

### 📖 附录

| 附录 | 主题 | 文章数 |
|------|------|--------|
| [附录 A](appendix/appendix-a-std-lib-api.md) | 标准库 API | 1 篇 |
| [附录 B](appendix/appendix-b-common-patterns.md) | 常用模式 | 1 篇 |
| [附录 C](appendix/appendix-c-faq.md) | 常见问题 | 1 篇 |

---

## 🗺️ 阅读路径

<div class="grid cards" markdown>

### 🟢 初学者路径

**刚接触 Zig 或本项目？**

1. [项目背景](part-01-project-overview/01-1-project-background.md)
2. [快速开始](part-02-quick-start/02-2-build-and-run.md)
3. [Zig 基础](part-03-zig-basics/index.md)
4. [核心模块](part-07-core-modules/index.md)

[:octicons-arrow-right-24: 开始入门](part-01-project-overview/index.md)

---

### 🟡 进阶路径

**已有 Zig 基础，想深入了解？**

1. [构建系统](part-06-build-system/index.md)
2. [内存管理](part-08-memory-management/index.md)
3. [错误处理](part-09-error-handling/index.md)
4. [模板引擎](part-10-template-engine/index.md)

[:octicons-arrow-right-24: 深入学习](part-06-build-system/index.md)

---

### 🔴 架构师路径

**关注架构设计与实现？**

1. [核心模块分析](part-07-core-modules/index.md)
2. [模板引擎架构](part-10-template-engine/10-1-architecture.md)
3. [解析器架构](part-11-markdown-parser/11-1-parser-architecture.md)
4. [性能优化](part-13-extension-guides/13-4-performance-optimization.md)

[:octicons-arrow-right-24: 架构设计](part-07-core-modules/index.md)

</div>

---

## 📊 完成状态

!!! success "文档已完成"

    **所有 53 篇文章已完成** · 14 个部分 · 100% 覆盖

| 状态 | 数量 |
|------|------|
| ✅ 已完成 | 53 篇 |
| 🔄 进行中 | 0 篇 |
| ⏳ 待开始 | 0 篇 |

---

## 💡 使用建议

### 本文档适合谁

- **Zig 初学者** - 学习 Zig 语言基础和实际项目应用
- **博客开发者** - 参考静态博客生成器的实现方式
- **编译器爱好者** - 了解 Markdown 解析器和模板引擎的设计
- **架构师** - 参考模块化设计和可扩展架构

### 如何有效使用

1. **按需阅读** - 使用左侧导航栏快速定位感兴趣的主题
2. **实践结合** - 边阅读边运行 `zig build run` 查看效果
3. **代码实验** - 修改源码观察输出变化，加深理解
4. **参考附录** - 遇到问题时查阅 [常用模式](appendix/appendix-b-common-patterns.md) 和 [FAQ](appendix/appendix-c-faq.md)

---

## 📝 更新记录

!!! note "最后更新"

    **2026-03-22** - 全部 53 篇文章已完成

| 日期 | 更新内容 |
|------|----------|
| 2026-03-22 | ✅ 完成全部文档（53/53） |
| 2026-03-22 | ✅ 完成扩展指南、测试调试、附录 |
| 2026-03-22 | ✅ 完成样式与前端、Markdown 解析器 |
| 2026-03-21 | ✅ 完成模板引擎、核心模块、内存管理 |
| 2026-03-21 | ✅ 完成错误处理、构建系统、函数 |
| 2026-03-21 | ✅ 完成控制流、Zig 基础、快速开始 |
| 2026-03-21 | ✨ 创建文档仓库，完成项目概览 |

---

## 🔧 本地预览

```bash
# 安装依赖
pip install -r requirements.txt

# 启动本地预览服务器
mkdocs serve

# 访问 http://127.0.0.1:8000
```

---

**© 2026 my-blog-analysis** · [MIT License](https://github.com/linychuo/my-blog-analysis/blob/main/LICENSE)
