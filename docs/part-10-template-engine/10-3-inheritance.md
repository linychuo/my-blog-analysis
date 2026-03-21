# 10-3 模板继承

> Handlebars 模板继承机制与项目实例分析

---

## 1. 模板继承概念

### 1.1 什么是模板继承

模板继承是一种设计模式，允许子模板继承父模板的结构，并注入特定内容。

**优势**：

| 优势 | 说明 |
|------|------|
| DRY 原则 | 避免重复的 HTML 结构代码 |
| 一致性 | 所有页面共享相同的布局 |
| 易维护 | 修改布局只需改父模板 |
| 关注点分离 | 父模板管结构，子模板管内容 |

### 1.2 项目中的继承层次

```
layout.hbs (父模板/主布局)
    │
    ├─────────────┬─────────────┬─────────────┐
    │             │             │             │
index.hbs     post.hbs     about.hbs     tag.hbs
(首页)        (文章页)       (关于页)       (标签页)
```

---

## 2. 父模板：layout.hbs

### 2.1 完整代码

```handlebars
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{title}}</title>
    <link rel="stylesheet" href="/style.css">
</head>
<body>
    <header>
        <h1><a href="/">{{siteTitle}}</a></h1>
        <nav>
            <a href="/">Home</a>
            <a href="/about.html">About</a>
        </nav>
    </header>

    <main>
        {{~> page}}
    </main>

    <aside>
        {{~> sidebar}}
    </aside>

    <footer>
        <p>Generated with <a href="https://ziglang.org">Zig</a> · © {{year}}</p>
    </footer>
</body>
</html>
```

### 2.2 结构分析

| 区域 | 内容 | 来源 |
|------|------|------|
| `<head>` | 元数据、标题、样式 | 父模板 |
| `<header>` | 站点标题、导航 | 父模板 |
| `<main>` | 主内容 | `{{~> page}}` 注入 |
| `<aside>` | 侧边栏 | `{{~> sidebar}}` 注入 |
| `<footer>` | 版权信息 | 父模板 |

### 2.3 占位块

```handlebars
<main>
    {{~> page}}
    <!-- ^^^^^^
         占位符：page 块
         子模板定义此块的内容
    -->
</main>

<aside>
    {{~> sidebar}}
    <!-- ^^^^^^^^^
         占位符：sidebar 块
         子模板定义此块的内容（可选）
    -->
</aside>
```

---

## 3. 子模板继承模式

### 3.1 标准子模板结构

所有子模板都遵循相同模式：

```handlebars
{{#*inline "page"}}
<!-- 主内容区域 -->
{{/inline}}

{{#*inline "sidebar"}}
<!-- 侧边栏内容（可选） -->
{{/inline}}

{{~> (parent)~}}
```

### 3.2 继承语法详解

**`{{#*inline "name"}}`** - 定义内联块

```handlebars
{{#*inline "page"}}
{{{posts}}}
{{/inline}}
```

| 部分 | 说明 |
|------|------|
| `{{#*` | 块开始标记 |
| `inline` | 内联块关键字 |
| `"page"` | 块名称（必须与父模板占位符匹配） |
| `...` | 块内容 |
| `{{/inline}}` | 块结束标记 |

**`{{~> (parent)~}}`** - 继承父模板

```handlebars
{{~> (parent)~}}
```

| 符号 | 说明 |
|------|------|
| `~` | 修剪空白 |
| `>` | 部分模板调用 |
| `(parent)` | 父模板引用 |

---

## 4. 各子模板继承分析

### 4.1 index.hbs - 首页

```handlebars
{{#*inline "page"}}
{{{posts}}}
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

**继承效果**：

```
layout.hbs                    index.hbs 注入
    │                              │
    ├─ <header>...</header>        │
    │                              │
    ├─ <main>                      │
    │     └─ {{~> page}} ←─────────┼── {{{posts}}}
    │                              │
    ├─ <aside>                     │
    │     └─ {{~> sidebar}} ←──────┼── (空)
    │                              │
    └─ <footer>...</footer>        │
```

**最终 HTML**：

```html
<!DOCTYPE html>
<html>
<head>...</head>
<body>
    <header>
        <h1><a href="/">My Blog</a></h1>
        <nav>...</nav>
    </header>

    <main>
        <!-- index.hbs 的 page 块内容 -->
        <article class="post-preview">
            <h2><a href="...">Post Title</a></h2>
            ...
        </article>
    </main>

    <aside>
        <!-- sidebar 为空 -->
    </aside>

    <footer>...</footer>
</body>
</html>
```

### 4.2 post.hbs - 文章页

```handlebars
{{TITLE}}
{{date_time}}
{{{tags}}}
{{{content}}}
```

**特殊性**：post.hbs 不使用内联块，直接输出变量，依赖 layout.hbs 的结构。

**渲染流程**：

```
[1] 渲染 post.hbs
    │
    ├─ {{TITLE}} → "My Post"
    ├─ {{date_time}} → "2026-03-21"
    ├─ {{{tags}}} → "<a>zig</a>"
    └─ {{{content}}} → "<p>...</p>"
    │
    ▼
[2] setPageContent() 设置渲染结果
    │
    ▼
[3] 渲染 layout.hbs
    │
    └─ 注入 page 块（post.hbs 的内容）
```

### 4.3 about.hbs - 关于页

```handlebars
{{#*inline "page"}}
{{{content}}}
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

**特点**：

| 特点 | 说明 |
|------|------|
| 简单 | 只输出 content |
| 空 sidebar | 关于页不需要侧边栏 |
| 无 frontmatter | about.markdown 没有 frontmatter |

### 4.4 tag.hbs - 标签页

```handlebars
{{#*inline "page"}}

POSTS TAGGED WITH "{{TAG_NAME}}"

{{post_count}} post(s) found

{{{posts}}}

{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

**特点**：

| 元素 | 说明 |
|------|------|
| 标题 | 显示标签名称 |
| 统计 | 显示文章数量 |
| 文章列表 | 显示该标签下的所有文章 |
| 空 sidebar | 标签页不需要侧边栏 |

---

## 5. 继承机制实现

### 5.1 zig-handlebars 工作原理

```
[1] 加载子模板 (如 index.hbs)
    │
    ▼
[2] 解析 {{#*inline}} 定义
    │
    ├─ 注册 "page" 块内容
    ├─ 注册 "sidebar" 块内容
    │
    ▼
[3] 遇到 {{~> (parent)~}}
    │
    ▼
[4] 加载父模板 (layout.hbs)
    │
    ▼
[5] 渲染父模板
    │
    ├─ 遇到 {{~> page}} → 注入步骤 2 注册的 page 内容
    ├─ 遇到 {{~> sidebar}} → 注入步骤 2 注册的 sidebar 内容
    │
    ▼
[6] 返回完整 HTML
```

### 5.2 项目中的渲染流程

```zig
// generatePostPage() 中的模板渲染
fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] 准备上下文
    var ctx = Context.init(self.allocator);
    defer ctx.deinit();
    try ctx.set("title", post.title);
    try ctx.set("content", html_content);
    // ...

    // [2] 渲染 post.hbs（子模板）
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);

    // [3] 设置页面内容
    self.template_engine.setPageContent(page_html);

    // [4] 渲染 layout.hbs（父模板）
    const html = try self.template_engine.render("layout.hbs", &ctx);
    defer self.allocator.free(html);

    // [5] 写入文件
    try self.writeHtmlFileWithDate(post.filename, post.date_time, html);
}
```

---

## 6. 块的作用域

### 6.1 块名称匹配

```handlebars
<!-- layout.hbs 定义占位符 -->
<main>{{~> page}}</main>
<aside>{{~> sidebar}}</aside>

<!-- index.hbs 定义块 -->
{{#*inline "page"}}...{{/inline}}
{{#*inline "sidebar"}}...{{/inline}}
```

**名称必须匹配**：

```handlebars
<!-- 错误：名称不匹配 -->
{{#*inline "content"}}...{{/inline}}
<!-- layout.hbs 中没有 {{~> content}} 占位符 -->

<!-- 正确 -->
{{#*inline "page"}}...{{/inline}}
<!-- 匹配 layout.hbs 中的 {{~> page}} -->
```

### 6.2 块的可选性

```handlebars
<!-- sidebar 块是可选的 -->
{{#*inline "sidebar"}}{{/inline}}
<!-- 空块也是有效的 -->
```

**如果未定义块**：

```handlebars
<!-- 如果子模板未定义 sidebar -->
<aside>
    {{~> sidebar}}
    <!-- 输出为空 -->
</aside>
```

---

## 7. 继承模式变体

### 7.1 单层继承

```
layout.hbs
    │
    └─ index.hbs
```

**项目使用**：所有子模板直接继承 layout.hbs

### 7.2 多层继承（项目未使用）

```
base.hbs
    │
    └─ layout.hbs
            │
            └─ index.hbs
```

**zig-handlebars 限制**：当前实现可能不支持多层继承。

---

## 8. 与其他模式对比

### 8.1 模板组合 vs 模板继承

**模板组合**：

```handlebars
{{> header}}
{{> content}}
{{> footer}}
```

**模板继承**：

```handlebars
{{#*inline "page"}}...{{/inline}}
{{~> (parent)~}}
```

**项目选择**：使用继承模式，更简洁

### 8.2 包含模式

```handlebars
<!-- 包含其他模板 -->
{{> navigation}}
{{> footer}}
```

**项目未使用**：项目使用继承而非包含

---

## 9. 最佳实践

### 9.1 保持一致性

```handlebars
<!-- 所有子模板使用相同结构 -->
{{#*inline "page"}}
<!-- 内容 -->
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}

{{~> (parent)~}}
```

### 9.2 命名清晰

```handlebars
<!-- 好的命名 -->
{{#*inline "page"}}
{{#*inline "sidebar"}}

<!-- 避免模糊命名 -->
{{#*inline "content"}}
{{#*inline "stuff"}}
```

### 9.3 最小化子模板

```handlebars
<!-- 子模板只定义必要内容 -->
{{#*inline "page"}}
{{{posts}}}
{{/inline}}

<!-- 布局细节留给父模板 -->
```

---

## 10. 注意事项

### 10.1 空白处理

```handlebars
<!-- 使用 ~ 修剪空白 -->
{{~> page}}
{{~> (parent)~}}

<!-- 否则可能产生多余空白 -->
{{> page}}
```

### 10.2 变量作用域

```handlebars
<!-- 父模板和子模板共享上下文变量 -->
<!-- layout.hbs -->
<title>{{title}}</title>

<!-- index.hbs -->
{{#*inline "page"}}
{{{posts}}}
{{/inline}}
<!-- 都可以访问 ctx.set() 设置的变量 -->
```

### 10.3 循环引用

```handlebars
<!-- 错误：循环引用 -->
<!-- layout.hbs -->
{{~> (child)~}}

<!-- index.hbs -->
{{~> (parent)~}}
<!-- 会导致无限循环 -->
```

---

## 相关文档

- [10-1 架构设计](./10-1-architecture.md) - 模板引擎架构
- [10-2 模板语法](./10-2-template-syntax.md) - Handlebars 语法
- [10-4 模板详解](./10-4-templates-detail.md) - 每个模板详细分析
