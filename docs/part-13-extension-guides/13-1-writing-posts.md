# 13.1 编写博客文章指南

> **核心文件**: `posts/*.markdown`  
> **文件格式**: Markdown + YAML Frontmatter  
> **输出路径**: `zig-out/blog/YYYY/MM/DD/slug.html`

## 13.1.1 文章文件结构

### 完整文件示例

```markdown
---
title: 我的第一篇博客文章
date_time: 2024-01-15 10:30
tags:
  - zig
  - 入门教程
---

# 欢迎来到我的博客

这是我的第一篇博客文章，使用 **Zig** 编写的静态博客生成器生成。

## 为什么要用 Zig？

Zig 是一门系统编程语言，具有以下特点：

1. 简单清晰 - 没有隐藏控制流
2. 高性能 - 与 C 语言相当
3. 安全性 - 编译时检查边界

## 代码示例

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, {s}!\n", .{"World"});
}
```

## 数学公式

质能方程：$E = mc^2$

块级公式：

$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

这就是为什么我喜欢 Zig！
```

## 13.1.2 Frontmatter 详解

### 必需字段

| 字段 | 类型 | 格式 | 说明 |
|------|------|------|------|
| `title` | 字符串 | `"文章标题"` | 文章标题，显示在首页和文章页 |
| `date_time` | 日期时间 | `YYYY-MM-DD HH:MM` | 发布时间，决定 URL 路径 |

### 可选字段

| 字段 | 类型 | 格式 | 说明 |
|------|------|------|------|
| `tags` | 数组 | `["tag1", "tag2"]` 或多行 | 文章标签，用于分类和标签页 |

### Frontmatter 格式

```yaml
# 标准格式（推荐）
---
title: 文章标题
date_time: 2024-01-15 10:30
tags:
  - zig
  - 教程
  - 入门
---

# 正文内容
```

```yaml
# 紧凑格式（单行标签）
---
title: 文章标题
date_time: 2024-01-15 10:30
tags: ["zig", "教程", "入门"]
---

# 正文内容
```

### 字段验证规则

```zig
// Blogger.zig 中的验证逻辑（伪代码）
if (!frontmatter.has("title")) {
    return error.MissingTitle;
}
if (!frontmatter.has("date_time")) {
    return error.MissingDateTime;
}
// tags 是可选的，默认为空数组
```

## 13.1.3 文件命名规范

### 文件名格式

```
posts/
├── about.markdown              # 关于页面（特殊）
├── 2024-01-15-my-first-post.markdown
├── 2024-01-20-zig-basics.markdown
└── 2024-02-01-advanced-patterns.markdown
```

### 命名规则

| 规则 | 说明 | 示例 |
|------|------|------|
| 推荐格式 | `YYYY-MM-DD-slug.markdown` | `2024-01-15-hello-world.markdown` |
| 字符限制 | 小写字母、数字、连字符 | `my-post-2024.markdown` |
| 避免空格 | 使用连字符代替 | `my-post` ✅ `my post` ❌ |
| 避免特殊字符 | 不使用中文、符号 | `zig-intro` ✅ `zig 入门` ❌ |

### 输出 URL 结构

```
输入文件：posts/2024-01-15-hello-world.markdown
输出路径：zig-out/blog/2024/01/15/hello-world.html
访问 URL：  /2024/01/15/hello-world.html
```

## 13.1.4 Markdown 语法支持

### 基础语法

```markdown
# 标题（H1-H6）
## 二级标题
### 三级标题

**粗体文字** 和 *斜体文字*

- 无序列表项 1
- 无序列表项 2

1. 有序列表项 1
2. 有序列表项 2

[链接文字](https://example.com)

![图片描述](image.jpg)

`行内代码`
```

### 代码块

````markdown
<!-- 带语言标识的代码块（推荐） -->
```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello!\n", .{});
}
```

<!-- 自动检测语言的代码块 -->
```
def hello():
    print("Hello!")
```
````

### 表格

```markdown
| 列 1 | 列 2 | 列 3 |
|------|------|------|
| 值 1 | 值 2 | 值 3 |
| 值 4 | 值 5 | 值 6 |
```

### 引用块

```markdown
> 这是一段引用文字
> 
> 可以有多行
> 
> > 嵌套引用
```

### 水平线

```markdown
---

***

___
```

## 13.1.5 扩展语法

### 数学公式

```markdown
<!-- 行内公式 -->
欧拉恒等式是 $e^{i\pi} + 1 = 0$。

<!-- 块级公式 -->
$$
\begin{aligned}
f(x) &= x^2 + 2x + 1 \\
     &= (x + 1)^2
\end{aligned}
$$

<!-- 多行对齐公式 -->
$$
\begin{aligned}
\nabla \cdot \mathbf{E} &= \frac{\rho}{\varepsilon_0} \\
\nabla \cdot \mathbf{B} &= 0
\end{aligned}
$$
```

### 转义字符

```markdown
<!-- 显示特殊字符 -->
\$ 显示美元符号：\$100

<!-- 显示反斜杠 -->
\\ 显示路径：C:\\Users\\name

<!-- 显示星号 -->
\* 不是斜体：\*emphasis\*
```

## 13.1.6 文章组织建议

### 分类策略

| 分类方式 | 适用场景 | 示例标签 |
|---------|---------|---------|
| 按技术 | 技术栈分类 | `zig`, `rust`, `python` |
| 按难度 | 读者水平 | `beginner`, `intermediate`, `advanced` |
| 按类型 | 内容形式 | `tutorial`, `guide`, `reference`, `opinion` |
| 按系列 | 连载文章 | `zig-series-1`, `zig-series-2` |

### 标签最佳实践

```markdown
<!-- ✅ 推荐：3-5 个标签 -->
tags:
  - zig
  - 系统编程
  - 入门教程

<!-- ❌ 避免：太多标签 -->
tags:
  - zig
  - 编程
  - 教程
  - 入门
  - 系统
  - 语言
  - 技术
  - 博客

<!-- ❌ 避免：太少标签 -->
tags: ["zig"]  <!-- 太单一 -->
```

### 系列文章组织

```markdown
<!-- 第一篇 -->
---
title: Zig 入门系列（一）：环境搭建
date_time: 2024-01-01 10:00
tags:
  - zig
  - zig-series-1
  - 环境搭建
---

<!-- 第二篇 -->
---
title: Zig 入门系列（二）：基础语法
date_time: 2024-01-08 10:00
tags:
  - zig
  - zig-series-2
  - 基础语法
---
```

## 13.1.7 内容模板

### 技术教程模板

```markdown
---
title: [技术名称] 教程：[具体主题]
date_time: YYYY-MM-DD HH:MM
tags:
  - [技术名]
  - 教程
  - [难度级别]
---

# [技术名称]：[具体主题]

## 简介

简要介绍本教程要解决的问题。

## 前置条件

- [ prerequisite 1 ]
- [ prerequisite 2 ]

## 步骤一：[步骤名称]

详细说明和代码示例。

```[language]
// 代码示例
```

## 步骤二：[步骤名称]

继续详细说明。

## 总结

回顾要点和下一步建议。

## 参考资料

- [Zig 官方文档](https://ziglang.org/documentation)
- [Zig 标准库参考](https://ziglang.org/documentation/master/std)
```

### 项目展示模板

```markdown
---
title: 项目展示：[项目名称]
date_time: YYYY-MM-DD HH:MM
tags:
  - 项目展示
  - [技术栈]
  - 开源
---

# [项目名称]

## 项目简介

一句话描述项目。

## 功能特点

- 特点 1
- 特点 2
- 特点 3

## 技术栈

| 技术 | 用途 |
|------|------|
| Zig | 后端 |
| React | 前端 |

## 安装使用

```bash
# 安装命令
```

## 项目链接

- [GitHub 仓库](https://github.com/linychuo/my-blog)
- [在线演示](https://linychuo.github.io/my-blog)
```

## 13.1.8 常见问题

### Q: 如何修改已发布文章？

```bash
# 1. 编辑源文件
vim posts/2024-01-15-hello-world.markdown

# 2. 重新生成
zig build run

# 3. 验证输出
cat zig-out/blog/2024/01/15/hello-world.html
```

### Q: 如何删除文章？

```bash
# 删除源文件
rm posts/2024-01-15-hello-world.markdown

# 重新生成（会自动删除输出文件）
zig build run
```

### Q: 如何设置文章为草稿？

my-blog 目前不支持草稿功能，变通方法：

```bash
# 方法 1：移出 posts 目录
mv posts/draft-post.markdown drafts/

# 方法 2：重命名文件扩展名
mv posts/draft-post.markdown posts/draft-post.markdown.bak
```

### Q: 日期时间格式错误怎么办？

```yaml
# ✅ 正确格式
date_time: 2024-01-15 10:30

# ❌ 错误格式
date_time: 2024/01/15        # 分隔符错误
date_time: 2024-01-15T10:30  # ISO 格式（可能不支持）
date_time: January 15, 2024  # 文本格式（不支持）
```

## 13.1.9 发布检查清单

在发布文章前，请确认：

- [ ] Frontmatter 格式正确（title, date_time 必填）
- [ ] 文件名包含日期（`YYYY-MM-DD-slug.markdown`）
- [ ] 标签合理（3-5 个）
- [ ] Markdown 语法正确（无未闭合的代码块）
- [ ] 图片路径正确（使用相对路径或绝对 URL）
- [ ] 链接有效（无 404）
- [ ] 本地预览正常（`zig build run` 后检查）
- [ ] 拼写检查通过

## 相关文档

- [13.2 添加 RSS 订阅](./13-2-rss-feed.md) - 生成 RSS 订阅源
- [13.3 集成搜索功能](./13-3-search-integration.md) - 添加站内搜索
- [11.1 Markdown 解析器架构](../part-11-markdown-parser/11-1-parser-architecture.md) - Markdown 解析原理

---

*本文档介绍了如何在 my-blog 中编写和发布博客文章，包括 frontmatter 格式、Markdown 语法、文件命名规范和内容组织建议。*
