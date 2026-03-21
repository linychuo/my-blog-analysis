# 11-1 解析器架构

> zig-markdown Markdown 解析器设计与实现分析

---

## 1. 解析器概览

### 1.1 什么是 zig-markdown

`zig-markdown` 是 my-blog 项目使用的 Markdown 解析器，它是用 Zig 语言实现的 Markdown 到 HTML 的转换工具。

**核心特性**：

| 特性 | 说明 |
|------|------|
| CommonMark 支持 | 完整支持 CommonMark 规范 |
| 块级元素 | 标题、段落、列表、代码块、引用、表格 |
| 内联元素 | 粗体、斜体、删除线、链接、图片、行内代码 |
| 扩展功能 | 数学公式 (KaTeX)、自动换行 |

### 1.2 项目中的使用

```zig
// Blogger.zig 中的使用
const markdown = @import("zig-markdown");

fn generatePostPage(self: *Blogger, post: Post) !void {
    // Markdown → HTML 转换
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);
    
    // 设置到模板上下文
    try ctx.set("content", html_content);
}
```

**调用流程**：

```
post.content (Markdown 文本)
    │
    ▼
markdown.toHtml(allocator, content)
    │
    ▼
解析 Markdown 语法
    │
    ├─ 识别块级元素
    ├─ 识别内联元素
    └─ 处理扩展语法
    │
    ▼
生成 HTML 代码
    │
    ▼
返回 HTML 字符串
```

---

## 2. 解析器架构

### 2.1 两阶段解析

Markdown 解析通常分为两个阶段：

```
原始 Markdown 文本
    │
    ▼
[阶段 1] 块级解析 (Block Parsing)
    │
    ├─ 识别段落
    ├─ 识别标题
    ├─ 识别列表
    ├─ 识别代码块
    └─ 识别引用块
    │
    ▼
块级元素树 (AST)
    │
    ▼
[阶段 2] 内联解析 (Inline Parsing)
    │
    ├─ 解析粗体 **text**
    ├─ 解析斜体 *text*
    ├─ 解析链接 [text](url)
    ├─ 解析图片 ![alt](src)
    └─ 解析行内代码 `code`
    │
    ▼
完整 AST
    │
    ▼
[阶段 3] HTML 生成
    │
    └─ 遍历 AST 生成 HTML
```

### 2.2 块级元素类型

| 元素 | Markdown 语法 | HTML 输出 |
|------|-------------|----------|
| 标题 | `# Heading` | `<h1>Heading</h1>` |
| 段落 | 普通文本 | `<p>...</p>` |
| 引用 | `> quote` | `<blockquote>...</blockquote>` |
| 代码块 | ` ``` ` | `<pre><code>...</code></pre>` |
| 列表 | `- item` | `<ul><li>...</li></ul>` |
| 有序列表 | `1. item` | `<ol><li>...</li></ol>` |
| 表格 | `\| col \|` | `<table>...</table>` |

### 2.3 内联元素类型

| 元素 | Markdown 语法 | HTML 输出 |
|------|-------------|----------|
| 粗体 | `**text**` | `<strong>text</strong>` |
| 斜体 | `*text*` | `<em>text</em>` |
| 删除线 | `~~text~~` | `<del>text</del>` |
| 链接 | `[text](url)` | `<a href="url">text</a>` |
| 图片 | `![alt](src)` | `<img src="src" alt="alt">` |
| 行内代码 | `` `code` `` | `<code>code</code>` |

---

## 3. 解析流程分析

### 3.1 块级解析

```
输入 Markdown:
# 标题

这是段落。

- 列表项 1
- 列表项 2

``` 代码块
code
```

解析过程:
    │
    ├─ 行 1: "# 标题" → 识别为标题 (ATX Heading)
    │
    ├─ 行 3: "这是段落。" → 识别为段落
    │
    ├─ 行 5-6: "- 列表项" → 识别为无序列表
    │
    └─ 行 8-10: "```..." → 识别为代码块
```

### 3.2 内联解析

```
段落文本:
这是 **粗体** 和 *斜体* 还有 `代码`。

解析过程:
    │
    ├─ "这是 " → 普通文本
    │
    ├─ "**粗体**" → 识别为粗体 → <strong>粗体</strong>
    │
    ├─ " 和 " → 普通文本
    │
    ├─ "*斜体*" → 识别为斜体 → <em>斜体</em>
    │
    ├─ " 还有 " → 普通文本
    │
    └─ "`代码`" → 识别为行内代码 → <code>代码</code>
```

---

## 4. 扩展功能

### 4.1 数学公式 (KaTeX)

**语法**：

```markdown
$$
E = mc^2
$$

行内公式：$x^2$
```

**HTML 输出**：

```html
<!-- 块级公式 -->
<div class="katex-display">
    <span class="katex">...</span>
</div>

<!-- 行内公式 -->
<span class="katex">...</span>
```

**项目配置**：

```html
<!-- layout.hbs 中引入 KaTeX -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.css">
<script src="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.js"></script>
```

### 4.2 自动换行

**语法**：

```markdown
这是第一行\
这是第二行
```

**HTML 输出**：

```html
<p>这是第一行<br>这是第二行</p>
```

### 4.3 表格

**语法**：

```markdown
| 列 1 | 列 2 | 列 3 |
|------|------|------|
| 值 1 | 值 2 | 值 3 |
```

**HTML 输出**：

```html
<table>
    <thead>
        <tr>
            <th>列 1</th>
            <th>列 2</th>
            <th>列 3</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>值 1</td>
            <td>值 2</td>
            <td>值 3</td>
        </tr>
    </tbody>
</table>
```

---

## 5. API 使用

### 5.1 toHtml 函数

**签名**：

```zig
pub fn toHtml(allocator: Allocator, markdown: []const u8) ![]const u8
```

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `allocator` | `Allocator` | 内存分配器 |
| `markdown` | `[]const u8` | Markdown 文本 |

**返回**：

| 类型 | 说明 |
|------|------|
| `![]const u8` | HTML 字符串（需要调用者释放） |

### 5.2 使用示例

```zig
const std = @import("std");
const markdown = @import("zig-markdown");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const md = 
        \\# 标题
        \\
        \\这是 **粗体** 文本。
    ;

    // 转换 Markdown → HTML
    const html = try markdown.toHtml(allocator, md);
    defer allocator.free(html);

    std.debug.print("{s}\n", .{html});
}
```

### 5.3 错误处理

```zig
const html = markdown.toHtml(allocator, content) catch |err| {
    std.debug.print("Markdown 解析失败：{}\n", .{err});
    return error.ParseFailed;
};
defer allocator.free(html);
```

**可能的错误**：

| 错误 | 说明 |
|------|------|
| `error.OutOfMemory` | 内存分配失败 |
| `error.InvalidMarkdown` | Markdown 格式错误 |

---

## 6. 项目中的集成

### 6.1 Blogger.zig 集成

```zig
// Blogger.zig
const std = @import("std");
const markdown = @import("zig-markdown");

pub const Blogger = struct {
    // ...
    
    fn generatePostPage(self: *Blogger, post: Post) !void {
        // [1] Markdown → HTML
        const html_content = try markdown.toHtml(self.allocator, post.content);
        defer self.allocator.free(html_content);

        // [2] 设置模板上下文
        var ctx = Context.init(self.allocator);
        try ctx.set("content", html_content);
        
        // [3] 渲染模板
        // ...
    }
};
```

### 6.2 内存管理

```zig
// 正确的内存管理模式
fn processMarkdown(allocator: Allocator, md: []const u8) !void {
    const html = try markdown.toHtml(allocator, md);
    defer allocator.free(html);  // 必须释放
    
    // 使用 html
    useHtml(html);
}  // defer 在此释放 html
```

---

## 7. 解析器特性对比

### 7.1 zig-markdown vs 其他解析器

| 特性 | zig-markdown | cmark | pulldown-cmark |
|------|-------------|-------|----------------|
| 语言 | Zig | C | Rust |
| CommonMark | ✓ | ✓ | ✓ |
| GFM 扩展 | 部分 | 部分 | ✓ |
| 数学公式 | ✓ (KaTeX) | ✗ | ✗ |
| 自定义扩展 | 有限 | 插件 | 插件 |

### 7.2 项目选择 zig-markdown 的原因

| 原因 | 说明 |
|------|------|
| Zig 生态 | 与项目语言一致 |
| 轻量级 | 适合静态博客生成 |
| 扩展支持 | 支持数学公式 |
| 简单集成 | 直接 @import 使用 |

---

## 8. 完整解析示例

### 8.1 输入 Markdown

```markdown
# 我的文章

这是**粗体**和*斜体*。

## 代码示例

```zig
const std = @import("std");
```

## 列表

- 项目 1
- 项目 2

## 公式

$$
\sum_{i=1}^n i = \frac{n(n+1)}{2}
$$
```

### 8.2 输出 HTML

```html
<h1>我的文章</h1>

<p>这是<strong>粗体</strong>和<em>斜体</em>。</p>

<h2>代码示例</h2>

<pre><code class="language-zig">const std = @import("std");
</code></pre>

<h2>列表</h2>

<ul>
<li>项目 1</li>
<li>项目 2</li>
</ul>

<h2>公式</h2>

<div class="katex-display">
    <span class="katex">
        <span class="katex-html">
            <!-- KaTeX 渲染的公式 -->
        </span>
    </span>
</div>
```

---

## 9. 注意事项

### 9.1 HTML 转义

```markdown
<!-- 输入 -->
使用 <script> 标签

<!-- 输出 -->
<p>使用 &lt;script&gt; 标签</p>
<!-- 自动转义，防止 XSS -->
```

### 9.2 代码高亮

```markdown
```zig
// 指定语言
const x = 1;
```

<!-- 输出 -->
<pre><code class="language-zig">
const x = 1;
</code></pre>
```

**前端高亮**：

```html
<!-- layout.hbs 中引入 highlight.js -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/styles/github.min.css">
<script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/core.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/languages/zig.min.js"></script>
<script>hljs.highlightAll();</script>
```

### 9.3 性能考虑

```zig
// 批量转换时复用分配器
fn convertAll(allocator: Allocator, posts: []Post) !void {
    for (posts) |*post| {
        const html = try markdown.toHtml(allocator, post.content);
        defer allocator.free(html);
        // 处理 html
    }
}
```

---

## 相关文档

- [11-2 块级元素](./11-2-block-elements.md) - 标题、段落、列表等详解
- [11-3 内联元素](./11-3-inline-elements.md) - 粗体、斜体、链接等详解
- [11-4 HTML 生成](./11-4-html-generation.md) - HTML 渲染流程与高亮
