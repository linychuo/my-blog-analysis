# my-blog 技术文档总览

> 文档导航和阅读指南

---

## 📚 完整文档列表

### 第一部分：项目概览 ✅

- [1.1 项目背景与目标](part-01-project-overview/01-1-project-background.md)
- [1.2 技术栈选型](part-01-project-overview/01-2-tech-stack.md)
- [1.3 功能特性清单](part-01-project-overview/01-3-features.md)
- [1.4 项目结构说明](part-01-project-overview/01-4-project-structure.md)

### 第二部分：快速开始 ✅

- [2.1 环境要求](part-02-quick-start/02-1-requirements.md)
- [2.2 构建与运行](part-02-quick-start/02-2-build-and-run.md)
- [2.3 命令行选项](part-02-quick-start/02-3-cli-options.md)
- [2.4 输出结构](part-02-quick-start/02-4-output-structure.md)

### 第三部分：Zig 语言基础 ✅

- [3.1 变量与类型](part-03-zig-basics/03-1-variables-and-types.md)
- [3.2 重要类型详解](part-03-zig-basics/03-2-important-types.md)
- [3.3 可选类型与错误类型](part-03-zig-basics/03-3-optional-and-error.md)
- [3.4 结构体与枚举](part-03-zig-basics/03-4-struct-and-enum.md)

### 第四部分：控制流 ✅

- [4.1 条件语句 (if/else)](part-04-control-flow/04-1-if-else.md)
- [4.2 循环 (while/for)](part-04-control-flow/04-2-while-for.md)
- [4.3 Switch 表达式](part-04-control-flow/04-3-switch.md)
- [4.4 编译时执行 (comptime)](part-04-control-flow/04-4-comptime.md)

### 第五部分：函数 ✅

- [5.1 函数基础](part-05-functions/05-1-function-basics.md)
- [5.2 参数与返回值](part-05-functions/05-2-parameters-and-return.md)
- [5.3 泛型编程](part-05-functions/05-3-generics.md)

### 第六部分：构建系统 ✅

- [6.1 build.zig 结构](part-06-build-system/06-1-build-zig-structure.md)
- [6.2 std.Build API](part-06-build-system/06-2-std-build-api.md)
- [6.3 包管理](part-06-build-system/06-3-package-management.md)

### 第七部分：核心模块分析 ✅

- [7.1 入口点分析](part-07-core-modules/07-1-main-entry.md)
- [7.2 Post 结构体](part-07-core-modules/07-2-post-struct.md)
- [7.3 Blogger 结构体](part-07-core-modules/07-3-blogger-struct.md)
- [7.4 文件系统操作](part-07-core-modules/07-4-filesystem-ops.md)

### 第八部分：内存管理 ✅

- [8.1 内存分配器](part-08-memory-management/08-1-allocators.md)
- [8.2 内存所有权](part-08-memory-management/08-2-memory-ownership.md)
- [8.3 defer 模式](part-08-memory-management/08-3-defer-patterns.md)

### 第九部分：错误处理 ✅

- [9.1 错误类型](part-09-error-handling/09-1-error-types.md)
- [9.2 错误传播](part-09-error-handling/09-2-error-propagation.md)
- [9.3 错误处理模式](part-09-error-handling/09-3-error-handling-patterns.md)

### 第十部分：模板引擎 ✅

- [10.1 架构设计](part-10-template-engine/10-1-architecture.md)
- [10.2 模板语法](part-10-template-engine/10-2-template-syntax.md)
- [10.3 模板继承](part-10-template-engine/10-3-inheritance.md)
- [10.4 模板详解](part-10-template-engine/10-4-templates-detail.md)

### 第十一部分：Markdown 解析器 ✅

- [11.1 解析器架构](part-11-markdown-parser/11-1-parser-architecture.md)
- [11.2 块级元素](part-11-markdown-parser/11-2-block-elements.md)
- [11.3 内联元素](part-11-markdown-parser/11-3-inline-elements.md)
- [11.4 HTML 生成](part-11-markdown-parser/11-4-html-generation.md)

### 第十二部分：样式与前端 ⚪

- [12.1 CSS 架构](part-12-styling/12-1-css-architecture.md)
- [12.2 主题系统](part-12-styling/12-2-theme-system.md)
- [12.3 前端功能](part-12-styling/12-3-frontend-features.md)

### 第十三部分：扩展指南 ⚪

- [13.1 编写博客](part-13-extension-guides/13-1-writing-posts.md)
- [13.2 添加 RSS](part-13-extension-guides/13-2-adding-rss.md)
- [13.3 添加搜索](part-13-extension-guides/13-3-adding-search.md)
- [13.4 性能优化](part-13-extension-guides/13-4-performance-optimization.md)

### 第十四部分：测试与调试 ⚪

- [14.1 单元测试](part-14-testing/14-1-unit-testing.md)
- [14.2 调试技巧](part-14-testing/14-2-debugging.md)
- [14.3 内存泄漏检测](part-14-testing/14-3-memory-leak-detection.md)

### 附录 ⚪

- [附录 A：标准库 API](appendix/appendix-a-std-api.md)
- [附录 B：常用模式](appendix/appendix-b-common-patterns.md)
- [附录 C：常见问题](appendix/appendix-c-faq.md)

---

## 📊 文档进度

| 部分 | 完成状态 | 文档数 | 进度 |
|------|----------|--------|------|
| 项目概览 | ✅ 已完成 | 4/4 | 100% |
| 快速开始 | ✅ 已完成 | 4/4 | 100% |
| Zig 基础 | ✅ 已完成 | 4/4 | 100% |
| 控制流 | ✅ 已完成 | 4/4 | 100% |
| 函数 | ✅ 已完成 | 3/3 | 100% |
| 构建系统 | ✅ 已完成 | 3/3 | 100% |
| 核心模块 | ✅ 已完成 | 4/4 | 100% |
| 内存管理 | ✅ 已完成 | 3/3 | 100% |
| 错误处理 | ✅ 已完成 | 3/3 | 100% |
| 模板引擎 | ✅ 已完成 | 4/4 | 100% |
| Markdown 解析器 | ✅ 已完成 | 4/4 | 100% |
| 样式与前端 | ⚪ 待开始 | 0/3 | 0% |
| 扩展指南 | ⚪ 待开始 | 0/4 | 0% |
| 测试调试 | ⚪ 待开始 | 0/3 | 0% |
| 附录 | ⚪ 待开始 | 0/3 | 0% |
| **总计** | | **40/53** | **75%** |

---

## 🗺️ 阅读路径推荐

### 🟢 初学者路径

```
项目概览 → 快速开始 → Zig 基础 → 核心模块
```

1. [项目背景](part-01-project-overview/01-1-project-background.md) - 了解项目
2. [技术栈](part-01-project-overview/01-2-tech-stack.md) - 技术选型
3. [功能特性](part-01-project-overview/01-3-features.md) - 功能列表
4. [项目结构](part-01-project-overview/01-4-project-structure.md) - 目录结构
5. [环境要求](part-02-quick-start/02-1-requirements.md) - 安装 Zig
6. [构建与运行](part-02-quick-start/02-2-build-and-run.md) - 编译项目

### 🟡 进阶路径

```
构建系统 → 内存管理 → 错误处理 → 核心模块分析
```

### 🔴 高级路径

```
模板引擎架构 → Markdown 解析器 → 性能优化 → 扩展开发
```

---

## 📝 文档更新记录

| 日期 | 更新内容 |
|------|----------|
| 2026-03-21 | 完成第十一部分（Markdown 解析器）4 篇文档 |
| 2026-03-21 | 完成第十部分（模板引擎）4 篇文档 |
| 2026-03-21 | 完成第三部分（Zig 语言基础）4 篇文档 |
| 2026-03-21 | 完成第二部分（快速开始）4 篇文档 |
| 2026-03-21 | 创建文档仓库，完成第一部分（项目概览） |

---

## 🔗 相关链接

- [源代码仓库](https://github.com/linychuo/my-blog)
- [Zig 官方文档](https://ziglang.org/documentation)
- [Zig 标准库参考](https://ziglang.org/documentation/master/std)

---

**最后更新**: 2026-03-21
