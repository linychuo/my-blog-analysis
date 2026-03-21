# my-blog 技术分析文档

> 对 Zig 静态博客生成器的完整技术剖析

[![Documentation Status](https://img.shields.io/badge/docs-in--progress-blue)]()
[![Zig Version](https://img.shields.io/badge/zig-0.15.2-orange)](https://ziglang.org)
[![License](https://img.shields.io/badge/license-MIT-green)]()

---

## 📚 文档导航

### 第一部分：项目概览
- [项目背景与目标](part-01-project-overview/01-1-project-background.md)
- [技术栈选型](part-01-project-overview/01-2-tech-stack.md)
- [功能特性清单](part-01-project-overview/01-3-features.md)
- [项目结构说明](part-01-project-overview/01-4-project-structure.md)

### 第二部分：快速开始
- [环境要求](part-02-quick-start/02-1-requirements.md)
- [构建与运行](part-02-quick-start/02-2-build-and-run.md)
- [命令行选项](part-02-quick-start/02-3-cli-options.md)
- [输出结构](part-02-quick-start/02-4-output-structure.md)

### 第三部分：Zig 语言基础
- [变量与类型](part-03-zig-basics/03-1-variables-and-types.md)
- [重要类型详解](part-03-zig-basics/03-2-important-types.md)
- [可选类型与错误类型](part-03-zig-basics/03-3-optional-and-error.md)
- [结构体与枚举](part-03-zig-basics/03-4-struct-and-enum.md)

### 第四部分：控制流 ✅
- [条件语句 (if/else)](part-04-control-flow/04-1-if-else.md)
- [循环 (while/for)](part-04-control-flow/04-2-while-for.md)
- [Switch 表达式](part-04-control-flow/04-3-switch.md)
- [编译时执行 (comptime)](part-04-control-flow/04-4-comptime.md)

### 第五部分：函数 ✅
- [函数基础](part-05-functions/05-1-function-basics.md)
- [参数与返回值](part-05-functions/05-2-parameters-and-return.md)
- [泛型编程](part-05-functions/05-3-generics.md)

### 第六部分：构建系统 ✅
- [build.zig 结构](part-06-build-system/06-1-build-zig-structure.md)
- [std.Build API](part-06-build-system/06-2-std-build-api.md)
- [包管理](part-06-build-system/06-3-package-management.md)

### 第七部分：核心模块分析 ✅
- [入口点分析](part-07-core-modules/07-1-main-entry.md)
- [Blogger 结构体](part-07-core-modules/07-2-blogger-struct.md)
- [Post 结构体](part-07-core-modules/07-3-post-struct.md)
- [文件系统操作](part-07-core-modules/07-4-filesystem-ops.md)

### 第八部分：内存管理 ✅
- [内存分配器](part-08-memory-management/08-1-allocators.md)
- [内存所有权](part-08-memory-management/08-2-memory-ownership.md)
- [defer 模式](part-08-memory-management/08-3-defer-patterns.md)

### 第九部分：错误处理 ✅
- [错误类型](part-09-error-handling/09-1-error-types.md)
- [错误传播](part-09-error-handling/09-2-error-propagation.md)
- [错误处理模式](part-09-error-handling/09-3-error-handling-patterns.md)

### 第十部分：模板引擎 ✅
- [架构设计](part-10-template-engine/10-1-architecture.md)
- [模板语法](part-10-template-engine/10-2-template-syntax.md)
- [模板继承](part-10-template-engine/10-3-inheritance.md)
- [模板详解](part-10-template-engine/10-4-templates-detail.md)

### 第十一部分：Markdown 解析器 ✅
- [解析器架构](part-11-markdown-parser/11-1-parser-architecture.md)
- [块级元素](part-11-markdown-parser/11-2-block-elements.md)
- [内联元素](part-11-markdown-parser/11-3-inline-elements.md)
- [HTML 生成](part-11-markdown-parser/11-4-html-generation.md)

### 第十二部分：样式与前端
- [CSS 架构](part-12-styling/12-1-css-architecture.md)
- [主题系统](part-12-styling/12-2-theme-system.md)
- [前端功能](part-12-styling/12-3-frontend-features.md)

### 第十三部分：扩展指南
- [编写博客](part-13-extension-guides/13-1-writing-posts.md)
- [添加 RSS](part-13-extension-guides/13-2-adding-rss.md)
- [添加搜索](part-13-extension-guides/13-3-adding-search.md)
- [性能优化](part-13-extension-guides/13-4-performance-optimization.md)

### 第十四部分：测试与调试
- [单元测试](part-14-testing/14-1-unit-testing.md)
- [调试技巧](part-14-testing/14-2-debugging.md)
- [内存泄漏检测](part-14-testing/14-3-memory-leak-detection.md)

### 附录
- [标准库 API](appendix/appendix-a-std-api.md)
- [常用模式](appendix/appendix-b-common-patterns.md)
- [常见问题](appendix/appendix-c-faq.md)

---

## 📖 阅读指南

### 🟢 初学者路径
如果你刚接触 Zig 或这个项目，建议按以下顺序阅读：

1. **项目概览** - 了解项目背景和功能
2. **快速开始** - 上手构建和运行
3. **Zig 语言基础** - 掌握必要的 Zig 语法
4. **核心模块分析** - 理解项目主体结构

### 🟡 进阶开发者路径
如果你已有 Zig 基础，想深入了解项目实现：

1. **构建系统** - 理解项目构建配置
2. **内存管理** - 掌握 Zig 内存管理模式
3. **错误处理** - 学习 Zig 错误处理机制
4. **模板引擎与 Markdown 解析器** - 深入核心算法

### 🔴 架构师路径
如果你想参考项目架构设计：

1. **核心模块分析** - 模块划分
2. **模板引擎架构** - 设计模式
3. **Markdown 解析器架构** - 状态机设计
4. **扩展指南** - 可扩展性设计

---

## 📊 文档进度

| 部分 | 状态 | 文档数 | 已完成 |
|------|------|--------|--------|
| 项目概览 | 🟢 已完成 | 4 | 4 |
| 快速开始 | 🟢 已完成 | 4 | 4 |
| Zig 语言基础 | 🟢 已完成 | 4 | 4 |
| 控制流 | 🟢 已完成 | 4 | 4 |
| 函数 | 🟢 已完成 | 3 | 3 |
| 构建系统 | 🟢 已完成 | 3 | 3 |
| 核心模块 | 🟢 已完成 | 4 | 4 |
| 内存管理 | 🟢 已完成 | 3 | 3 |
| 错误处理 | 🟢 已完成 | 3 | 3 |
| 模板引擎 | 🟢 已完成 | 4 | 4 |
| Markdown 解析器 | 🟢 已完成 | 4 | 4 |
| 样式与前端 | ⚪ 待开始 | 3 | 0 |
| 扩展指南 | ⚪ 待开始 | 4 | 0 |
| 测试调试 | ⚪ 待开始 | 3 | 0 |
| 附录 | ⚪ 待开始 | 3 | 0 |
| **总计** | | **53** | **40** |

**图例**：🟢 已完成 | 🟡 进行中 | ⚪ 待开始

---

## 🔧 关于项目

**my-blog** 是一个用 Zig 编写的静态博客生成器，具有以下特点：

- ⚡ **高性能** - 编译型语言，生成速度快
- 🎨 **现代设计** - 响应式布局，暗色/亮色主题
- 📝 **Markdown 支持** - 完整 CommonMark + 扩展（数学公式、代码高亮）
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

### 相关链接

- **源代码仓库**: [github.com/linychuo/my-blog](https://github.com/linychuo/my-blog)
- **Zig 官网**: [ziglang.org](https://ziglang.org)
- **Zig 文档**: [ziglang.org/documentation](https://ziglang.org/documentation)

---

## 📝 更新日志

### 2026-03-21 (Part 11)
- ✅ 完成第十一部分：Markdown 解析器（4 篇）
  - 解析器架构
  - 块级元素
  - 内联元素
  - HTML 生成

### 2026-03-21 (Part 10)
- ✅ 完成第十部分：模板引擎（4 篇）
  - 架构设计
  - 模板语法
  - 模板继承
  - 模板详解

### 2026-03-21 (Part 9)
- ✅ 完成第九部分：错误处理（3 篇）
  - 错误类型
  - 错误传播
  - 错误处理模式

### 2026-03-21 (Part 8)
- ✅ 完成第八部分：内存管理（3 篇）
  - 内存分配器
  - 内存所有权
  - defer 模式

### 2026-03-21 (Part 7)
- ✅ 完成第七部分：核心模块分析（4 篇）
  - 入口点分析 (main.zig)
  - Blogger 结构体
  - Post 结构体
  - 文件系统操作

### 2026-03-21 (Part 6)
- ✅ 完成第六部分：构建系统（3 篇）
  - build.zig 结构
  - std.Build API
  - 包管理

### 2026-03-21 (Part 5)
- ✅ 完成第五部分：函数（3 篇）
  - 函数基础
  - 参数与返回值
  - 泛型编程

### 2026-03-21 (Part 4)
- ✅ 完成第四部分：控制流（4 篇）
  - 条件语句 (if/else)
  - 循环 (while/for)
  - Switch 表达式
  - 编译时执行 (comptime)

### 2026-03-21 (Part 3)
- ✅ 完成第三部分：Zig 语言基础（4 篇）
  - 变量与类型
  - 重要类型详解
  - 可选类型与错误处理
  - 结构体与枚举

### 2026-03-21 (Part 2)
- ✅ 完成第二部分：快速开始（4 篇）
  - 环境要求
  - 构建与运行
  - 命令行选项
  - 输出结构

### 2026-03-21 (Part 1)
- ✨ 创建文档仓库
- 📋 完成文档大纲规划
- ✅ 完成第一部分：项目概览（4 篇）
  - 项目背景与目标
  - 技术栈选型
  - 功能特性清单
  - 项目结构说明

---

## 📄 许可证

MIT License
