# 10-2 模板语法

> Handlebars 模板语法与项目实例分析

---

## 1. 语法概览

### 1.1 Handlebars 语法类型

zig-handlebars 支持以下语法：

| 语法 | 示例 | 说明 |
|------|------|------|
| 变量插值 | `{{name}}` | 转义输出 |
| 未转义 HTML | `{{{html}}}` | 原始 HTML 输出 |
| 内联块 | `{{#*inline "name"}}` | 定义块 |
| 部分模板 | `{{> partial}}` | 引入部分 |
| 模板继承 | `{{~> (parent)~}}` | 继承父模板 |

---

## 2. 变量插值

### 2.1 转义输出 `{{ }}`

**语法**：

```handlebars
{{variable_name}}
```

**项目实例** - post.hbs：

```handlebars
{{TITLE}}
{{date_time}}
```

**渲染结果**：

```
输入：ctx.set("TITLE", "My <Post>")
输出：My &lt;Post&gt;  (HTML 转义)
```

**转义字符**：

| 字符 | 转义后 |
|------|--------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `&` | `&amp;` |
| `"` | `&quot;` |
| `'` | `&#x27;` |

**用途**：显示纯文本，防止 XSS 攻击

### 2.2 未转义输出 `{{{ }}}`

**语法**：

```handlebars
{{{variable_name}}}
```

**项目实例** - post.hbs：

```handlebars
{{{tags}}}
{{{content}}}
```

**渲染结果**：

```
输入：ctx.set("content", "<p>Hello</p>")
输出：<p>Hello</p>  (原始 HTML)
```

**用途**：渲染 HTML 内容（Markdown 转换后、标签链接等）

**安全注意**：

```zig
// 确保 HTML 内容是可信的
const html_content = try markdown.toHtml(allocator, post.content);
// markdown.toHtml 生成可信 HTML
try ctx.set("content", html_content);
```

---

## 3. 内联块定义

### 3.1 定义内联块

**语法**：

```handlebars
{{#*inline "block_name"}}
<!-- 块内容 -->
{{/inline}}
```

**项目实例** - index.hbs：

```handlebars
{{#*inline "page"}}
{{{posts}}}
{{/inline}}

{{#*inline "sidebar"}}{{/inline}}
```

**说明**：

| 部分 | 说明 |
|------|------|
| `{{#*inline "page"}}` | 定义名为 "page" 的块 |
| `{{{posts}}}` | 块内容 |
| `{{/inline}}` | 结束定义 |
| `{{#*inline "sidebar"}}{{/inline}}` | 定义空块 |

### 3.2 内联块的作用

内联块用于**模板继承**，子模板通过定义块来替换父模板的占位符。

**流程**：

```
index.hbs (子模板)
    │
    ├─ 定义 {{#*inline "page"}}
    │
    ├─ 定义 {{#*inline "sidebar"}}
    │
    └─ {{~> (parent)~}}
            │
            ▼
layout.hbs (父模板)
    │
    ├─ <main>{{~> page}}</main> → 注入 page 块内容
    │
    └─ <aside>{{~> sidebar}}</aside> → 注入 sidebar 块内容
```

---

## 4. 模板继承

### 4.1 继承语法

**语法**：

```handlebars
{{~> (parent)~}}
```

**项目实例** - 所有子模板都以这行结束：

```handlebars
{{#*inline "page"}}
<!-- 内容 -->
{{/inline}}

{{#*inline "sidebar"}}
<!-- 内容 -->
{{/inline}}

{{~> (parent)~}}
```

**符号解析**：

| 符号 | 说明 |
|------|------|
| `~` | 修剪空白（可选） |
| `>` | 部分模板调用 |
| `(parent)` | 父模板引用 |

### 4.2 继承机制

```
渲染流程:

[1] 渲染子模板 (如 index.hbs)
    │
    ├─ 处理 {{#*inline}} 定义
    │
    └─ 遇到 {{~> (parent)~}}
            │
            ▼
[2] 渲染父模板 (layout.hbs)
    │
    ├─ 遇到 {{~> page}} → 注入子模板的 page 块
    │
    └─ 遇到 {{~> sidebar}} → 注入子模板的 sidebar 块
    │
    ▼
[3] 返回完整 HTML
```

---

## 5. 部分模板 (Partials)

### 5.1 部分模板语法

**语法**：

```handlebars
{{> partial_name}}
```

**项目中的使用** - layout.hbs：

```handlebars
<main>
    {{~> page}}
</main>

<aside>
    {{~> sidebar}}
</aside>
```

**说明**：

| 部分 | 说明 |
|------|------|
| `{{~> page}}` | 渲染 "page" 块（由子模板定义） |
| `{{~> sidebar}}` | 渲染 "sidebar" 块（由子模板定义） |
| `~` | 修剪空白 |

### 5.2 动态部分

**语法**：

```handlebars
{{> (dynamic_name)}}
```

**项目实例** - layout.hbs 中的动态父模板：

```handlebars
{{~> (parent)~}}
```

**说明**：

```
(parent) 是动态计算的
    │
    ├─ 子模板渲染时 → (parent) = layout.hbs
    │
    └─ 渲染 layout.hbs 的内容
```

---

## 6. 完整语法示例

### 6.1 layout.hbs 完整分析

```handlebars
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{title}}</title>
    <!--    ^^^^^^^^^ 转义变量 -->
    <link rel="stylesheet" href="/style.css">
</head>
<body>
    <header>
        <h1><a href="/">{{siteTitle}}</a></h1>
        <!--            ^^^^^^^^^^^ 转义变量 -->
        <nav>
            <a href="/">Home</a>
            <a href="/about.html">About</a>
        </nav>
    </header>

    <main>
        {{~> page}}
        <!-- ^^^^^^ 渲染 page 块 -->
    </main>

    <aside>
        {{~> sidebar}}
        <!-- ^^^^^^^^^ 渲染 sidebar 块 -->
    </aside>

    <footer>
        <p>Generated with <a href="https://ziglang.org">Zig</a> · © {{year}}</p>
        <!--                                                    ^^^^^^ 转义变量 -->
    </footer>
</body>
</html>
```

### 6.2 index.hbs 完整分析

```handlebars
{{#*inline "page"}}
{{{posts}}}
{{/inline}}
<!--
page 块定义:
- 使用三重括号 {{{ }}} 输出未转义 HTML
- posts 变量包含完整的文章列表 HTML
-->

{{#*inline "sidebar"}}{{/inline}}
<!--
sidebar 块定义:
- 空块，首页不需要侧边栏
-->

{{~> (parent)~}}
<!--
继承父模板:
- 渲染 layout.hbs
- 注入 page 和 sidebar 块
-->
```

### 6.3 post.hbs 完整分析

```handlebars
{{TITLE}}
<!--
转义输出文章标题
-->

{{date_time}}
<!--
转义输出日期
-->

{{{tags}}}
<!--
未转义输出标签 HTML
格式：<a href="/tags/xxx">xxx</a>
-->

{{{content}}}
<!--
未转义输出 Markdown 转换的 HTML
格式：<p>...</p>, <h2>...</h2>, etc.
-->
```

---

## 7. 变量命名约定

### 7.1 项目中的变量

| 变量名 | 用途 | 转义 |
|--------|------|------|
| `title` | 页面标题 | 转义 `{{}}` |
| `TITLE` | 文章标题 | 转义 `{{}}` |
| `date_time` | 发布日期 | 转义 `{{}}` |
| `content` | HTML 内容 | 未转义 `{{{{}}}}` |
| `tags` | 标签 HTML | 未转义 `{{{{}}}}` |
| `posts` | 文章列表 HTML | 未转义 `{{{{}}}}` |
| `page_title` | 完整页面标题 | 转义 `{{}}` |
| `year` | 年份 | 转义 `{{}}` |
| `TAG_NAME` | 标签名称 | 转义 `{{}}` |
| `post_count` | 文章数量 | 转义 `{{}}` |

### 7.2 命名风格

```zig
// 项目使用蛇形命名（snake_case）
ctx.set("page_title", "...");
ctx.set("date_time", "...");
ctx.set("post_count", "...");

// 部分使用大写（可能是历史原因）
ctx.set("TITLE", "...");
ctx.set("TAG_NAME", "...");
```

**推荐**：统一使用蛇形命名

---

## 8. 空白处理

### 8.1 空白修剪符号

| 语法 | 说明 |
|------|------|
| `{{var}}` | 正常输出 |
| `{{~var}}` | 修剪左侧空白 |
| `{{var~}}` | 修剪右侧空白 |
| `{{~var~}}` | 修剪两侧空白 |

### 8.2 项目中的使用

```handlebars
{{~> page}}
<!-- 修剪左侧空白 -->

{{~> sidebar}}
<!-- 修剪左侧空白 -->

{{~> (parent)~}}
<!-- 修剪两侧空白 -->
```

**效果**：

```handlebars
<!-- 没有修剪 -->
<main>
    {{> page}}
</main>
<!-- 可能产生多余空白 -->

<!-- 修剪后 -->
<main>
    {{~> page}}
</main>
<!-- 空白更整洁 -->
```

---

## 9. 语法对比

### 9.1 Handlebars vs zig-handlebars

| 特性 | Handlebars (JS) | zig-handlebars |
|------|-----------------|----------------|
| 变量插值 | `{{name}}` | `{{name}}` |
| 未转义 | `{{{name}}}` | `{{{name}}}` |
| 内联块 | `{{#*inline}}` | `{{#*inline}}` |
| 模板继承 | `{{> (parent)}}` | `{{~> (parent)~}}` |
| 条件 | `{{#if}}` | 不支持 |
| 循环 | `{{#each}}` | 不支持 |
| Helper | `{{helper arg}}` | 不支持 |

**注意**：zig-handlebars 是简化实现，不支持条件、循环等高级特性。

### 9.2 项目中如何处理条件/循环

```zig
// 在 Zig 代码中处理，而不是模板中
// 生成标签 HTML
var tags_html = std.ArrayList(u8){};
for (tags) |tag| {
    try tags_html.appendSlice("<a href=\"/tags/" ++ tag ++ "\">" ++ tag ++ "</a>");
}
try ctx.set("tags", tags_html.items);
// 模板中直接输出：{{{tags}}}
```

---

## 10. 注意事项

### 10.1 转义安全

```handlebars
<!-- 安全：用户输入 -->
{{user_input}}  <!-- 转义 -->

<!-- 不安全：信任的 HTML -->
{{{trusted_html}}}  <!-- 未转义，确保来源可信 -->
```

### 10.2 变量存在性

```zig
// 确保设置的变量在模板中使用
try ctx.set("title", "My Blog");
// 模板中：{{title}}

// 如果变量未设置，模板可能输出空字符串
```

### 10.3 内存管理

```zig
// 模板渲染分配的内存需要释放
const html = try template_engine.render("layout.hbs", &ctx);
defer allocator.free(html);
```

---

## 相关文档

- [10-1 架构设计](./10-1-architecture.md) - 模板引擎架构
- [10-3 模板继承](./10-3-inheritance.md) - 继承机制深入
- [07-3 Blogger 结构体](../part-07-core-modules/07-3-blogger-struct.md) - 模板渲染代码
