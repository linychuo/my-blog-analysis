# 11-4 HTML 生成

> Markdown 到 HTML 的渲染流程与代码高亮

---

## 1. HTML 生成流程

### 1.1 渲染管道

```
Markdown 文本
    │
    ▼
[1] 词法分析 (Lexer)
    │
    ├─ 识别标记符号
    ├─ 分割文本块
    └─ 生成 Token 流
    │
    ▼
[2] 语法分析 (Parser)
    │
    ├─ 构建 AST (抽象语法树)
    ├─ 处理嵌套关系
    └─ 验证语法正确性
    │
    ▼
[3] HTML 生成 (Renderer)
    │
    ├─ 遍历 AST
    ├─ 生成 HTML 标签
    └─ 处理属性和转义
    │
    ▼
[4] 后处理 (Post-processing)
    │
    ├─ 代码高亮
    ├─ 公式渲染
    └─ 优化输出
    │
    ▼
最终 HTML
```

### 1.2 项目中的渲染调用

```zig
// Blogger.zig
const markdown = @import("zig-markdown");

fn generatePostPage(self: *Blogger, post: Post) !void {
    // [1] 调用 markdown.toHtml
    const html_content = try markdown.toHtml(self.allocator, post.content);
    defer self.allocator.free(html_content);

    // [2] 设置到模板上下文
    var ctx = Context.init(self.allocator);
    try ctx.set("content", html_content);
    
    // [3] 渲染模板
    const page_html = try self.template_engine.render("post.hbs", &ctx);
    defer self.allocator.free(page_html);
    
    // [4] 生成最终页面
    // ...
}
```

---

## 2. 词法分析 (Lexer)

### 2.1 Token 识别

**输入**：

```markdown
# 标题

这是**粗体**文本。
```

**Token 流**：

```
[HEADING_1] "标题"
[NEWLINE]
[PARAGRAPH_START]
[TEXT] "这是"
[BOLD_START]
[TEXT] "粗体"
[BOLD_END]
[TEXT] "文本。"
[PARAGRAPH_END]
```

### 2.2 词法分析规则

```zig
// 简化的词法分析逻辑
fn tokenize(allocator: Allocator, input: []const u8) ![]Token {
    var tokens = std.ArrayList(Token).init(allocator);
    
    var i: usize = 0;
    while (i < input.len) {
        switch (input[i]) {
            '#' => try tokens.append(try parseHeading(input, &i)),
            '*' => try tokens.append(try parseBold(input, &i)),
            '`' => try tokens.append(try parseCode(input, &i)),
            // ... 其他标记
            else => try tokens.append(Token{ .type = .TEXT, .value = input[i..i+1] }),
        }
        i += 1;
    }
    
    return tokens.toOwnedSlice();
}
```

---

## 3. 语法分析 (Parser)

### 3.1 构建 AST

**Markdown 输入**：

```markdown
# 标题

段落**粗体**文本。

- 列表 1
- 列表 2
```

**AST 结构**：

```
Document
├── Heading (level=1)
│   └── Text: "标题"
├── Paragraph
│   ├── Text: "段落"
│   ├── Bold
│   │   └── Text: "粗体"
│   └── Text: "文本。"
└── UnorderedList
    ├── ListItem
    │   └── Text: "列表 1"
    └── ListItem
        └── Text: "列表 2"
```

### 3.2 AST 节点类型

```zig
// 简化的 AST 节点定义
const Node = union(enum) {
    heading: Heading,
    paragraph: Paragraph,
    bold: Bold,
    italic: Italic,
    code: Code,
    link: Link,
    list: List,
    list_item: ListItem,
    text: []const u8,
};

const Heading = struct {
    level: u8,
    children: []Node,
};

const Paragraph = struct {
    children: []Node,
};

const Bold = struct {
    children: []Node,
};
```

---

## 4. HTML 渲染 (Renderer)

### 4.1 渲染器模式

```zig
// 渲染器接口
const Renderer = struct {
    fn renderHeading(self: *Renderer, heading: Heading) ![]const u8 {
        return try std.fmt.allocPrint(
            self.allocator,
            "<h{d}>{s}</h{d}>",
            .{ heading.level, try self.render(heading.children), heading.level }
        );
    }
    
    fn renderParagraph(self: *Renderer, para: Paragraph) ![]const u8 {
        return try std.fmt.allocPrint(
            self.allocator,
            "<p>{s}</p>",
            .{try self.render(para.children)}
        );
    }
    
    fn renderBold(self: *Renderer, bold: Bold) ![]const u8 {
        return try std.fmt.allocPrint(
            self.allocator,
            "<strong>{s}</strong>",
            .{try self.render(bold.children)}
        );
    }
    
    // ... 其他渲染方法
};
```

### 4.2 递归渲染

```zig
// 递归渲染 AST
fn render(self: *Renderer, nodes: []Node) ![]const u8 {
    var output = std.ArrayList(u8).init(self.allocator);
    
    for (nodes) |node| {
        switch (node) {
            .heading => |h| try output.appendSlice(try self.renderHeading(h)),
            .paragraph => |p| try output.appendSlice(try self.renderParagraph(p)),
            .bold => |b| try output.appendSlice(try self.renderBold(b)),
            .text => |t| try output.appendSlice(try self.escapeHtml(t)),
            // ... 其他节点类型
        }
    }
    
    return output.toOwnedSlice();
}
```

---

## 5. HTML 转义

### 5.1 转义规则

**输入**：

```markdown
使用 <script> 标签。

5 & 3 的比较。
```

**输出**：

```html
<p>使用 &lt;script&gt; 标签。</p>

<p>5 &amp; 3 的比较。</p>
```

### 5.2 转义实现

```zig
// HTML 转义函数
fn escapeHtml(allocator: Allocator, input: []const u8) ![]const u8 {
    var output = std.ArrayList(u8).init(allocator);
    
    for (input) |char| {
        switch (char) {
            '<' => try output.appendSlice("&lt;"),
            '>' => try output.appendSlice("&gt;"),
            '&' => try output.appendSlice("&amp;"),
            '"' => try output.appendSlice("&quot;"),
            '\'' => try output.appendSlice("&#x27;"),
            else => try output.append(char),
        }
    }
    
    return output.toOwnedSlice();
}
```

### 5.3 不转义的情况

```zig
// 内联代码不转义（由代码块处理）
fn renderCode(self: *Renderer, code: Code) ![]const u8 {
    // 代码内容已经过处理，不需要再次转义
    return try std.fmt.allocPrint(
        self.allocator,
        "<code>{s}</code>",
        .{code.content}
    );
}
```

---

## 6. 代码高亮

### 6.1 语言标识

**Markdown 输入**：

````markdown
```zig
const std = @import("std");
```
````

**HTML 输出**：

```html
<pre><code class="language-zig">const std = @import("std");
</code></pre>
```

### 6.2 前端高亮

**layout.hbs 中的配置**：

```html
<!DOCTYPE html>
<html>
<head>
    <!-- highlight.js 样式 -->
    <link rel="stylesheet" 
          href="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/styles/github.min.css">
</head>
<body>
    <!-- 页面内容 -->
    {{{content}}}
    
    <!-- highlight.js 脚本 -->
    <script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/core.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/languages/zig.min.js"></script>
    <script>
        document.addEventListener('DOMContentLoaded', (event) => {
            hljs.highlightAll();
        });
    </script>
</body>
</html>
```

### 6.3 highlight.js 工作原理

```javascript
// highlight.js 高亮流程
hljs.highlightAll = function() {
    // [1] 查找所有 <code> 元素
    const codeBlocks = document.querySelectorAll('pre code');
    
    // [2] 对每个代码块应用高亮
    codeBlocks.forEach((block) => {
        // [3] 检测语言（从 class 或自动检测）
        const language = detectLanguage(block);
        
        // [4] 应用语法高亮
        const highlighted = hljs.highlight(block.textContent, {
            language: language
        });
        
        // [5] 更新 HTML
        block.innerHTML = highlighted.value;
    });
};
```

### 6.4 支持的语言

| 语言 | class 名称 | 说明 |
|------|-----------|------|
| Zig | `language-zig` | Zig 语言 |
| JavaScript | `language-javascript` | JavaScript |
| Python | `language-python` | Python |
| C | `language-c` | C 语言 |
| Rust | `language-rust` | Rust |
| HTML | `language-html` | HTML |

---

## 7. 数学公式渲染

### 7.1 KaTeX 集成

**Markdown 输入**：

```markdown
行内公式：$E = mc^2$

块级公式：

$$
\sum_{i=1}^n i = \frac{n(n+1)}{2}
$$
```

**HTML 输出**：

```html
<p>行内公式：<span class="katex"><span class="katex-mathml">...</span></span></p>

<div class="katex-display">
    <span class="katex"><span class="katex-mathml">...</span></span>
</div>
```

### 7.2 KaTeX 配置

**layout.hbs 中的配置**：

```html
<head>
    <!-- KaTeX 样式 -->
    <link rel="stylesheet" 
          href="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.css">
    
    <!-- KaTeX 脚本 -->
    <script defer 
            src="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.js">
    </script>
    <script defer 
            src="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/contrib/auto-render.min.js">
    </script>
    
    <script>
        document.addEventListener("DOMContentLoaded", function() {
            renderMathInElement(document.body, {
                delimiters: [
                    {left: "$$", right: "$$", display: true},
                    {left: "$", right: "$", display: false}
                ]
            });
        });
    </script>
</head>
```

---

## 8. 完整渲染实例

### 8.1 输入 Markdown

```markdown
# 使用 Zig 编写

这是**粗体**和*斜体*。

```zig
const std = @import("std");
```

这是公式：$x^2$
```

### 8.2 渲染流程

```
[1] 词法分析
    │
    ├─ # → HEADING_1
    ├─ ** → BOLD_START
    ├─ * → ITALIC_START
    ├─ ``` → CODE_BLOCK_START
    ├─ $ → MATH_START
    
[2] 语法分析
    │
    ├─ Document
    │   ├─ Heading
    │   ├─ Paragraph (with Bold, Italic)
    │   ├─ CodeBlock
    │   └─ Paragraph (with Math)
    
[3] HTML 生成
    │
    ├─ <h1>使用 Zig 编写</h1>
    ├─ <p>这是<strong>粗体</strong>和<em>斜体</em>。</p>
    ├─ <pre><code class="language-zig">...</code></pre>
    └─ <p>这是公式：<span class="katex">...</span></p>
    
[4] 后处理
    │
    ├─ highlight.js 高亮代码
    └─ KaTeX 渲染公式
```

### 8.3 最终 HTML

```html
<!DOCTYPE html>
<html>
<head>
    <title>使用 Zig 编写</title>
    <link rel="stylesheet" href="/style.css">
    <link rel="stylesheet" href="highlight.js/github.min.css">
    <link rel="stylesheet" href="katex/katex.min.css">
</head>
<body>
    <header>...</header>
    <main>
        <h1>使用 Zig 编写</h1>
        
        <p>这是<strong>粗体</strong>和<em>斜体</em>。</p>
        
        <pre><code class="language-zig">const std = @import("std");</code></pre>
        
        <p>这是公式：<span class="katex">...</span></p>
    </main>
    <footer>...</footer>
    
    <script src="highlight.js/core.min.js"></script>
    <script src="katex/katex.min.js"></script>
    <script>
        hljs.highlightAll();
        renderMathInElement(document.body);
    </script>
</body>
</html>
```

---

## 9. 性能优化

### 9.1 批量渲染

```zig
// 批量转换多篇文章
fn convertAllPosts(allocator: Allocator, posts: []Post) !void {
    for (posts) |*post| {
        // 复用分配器
        const html = try markdown.toHtml(allocator, post.content);
        defer allocator.free(html);
        
        // 处理 html
        // ...
    }
}
```

### 9.2 缓存渲染结果

```zig
// 缓存 HTML 渲染结果
const HtmlCache = struct {
    map: std.StringHashMap([]const u8),
    
    fn getOrRender(self: *HtmlCache, allocator: Allocator, 
                   markdown: []const u8) ![]const u8 {
        const hash = hashMarkdown(markdown);
        
        // 检查缓存
        if (self.map.get(hash)) |cached| {
            return cached;
        }
        
        // 渲染并缓存
        const html = try markdown.toHtml(allocator, markdown);
        try self.map.put(hash, html);
        return html;
    }
};
```

---

## 10. 注意事项

### 10.1 内存管理

```zig
// 必须释放渲染结果
const html = try markdown.toHtml(allocator, content);
defer allocator.free(html);  // 重要！
```

### 10.2 转义安全

```zig
// 用户输入必须转义
{{title}}  <!-- 模板转义 -->

<!-- Markdown 自动转义 HTML -->
<script>alert('xss')</script>
<!-- 输出：&lt;script&gt;alert('xss')&lt;/script&gt; -->
```

### 10.3 CDN 依赖

```html
<!-- 确保 CDN 可用 -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/...">

<!-- 考虑本地备份 -->
<link rel="stylesheet" href="/vendor/highlight.min.css">
```

---

## 相关文档

- [10-2 模板语法](../part-10-template-engine/10-2-template-syntax.md) - 模板中的 HTML 输出
- [11-1 解析器架构](./11-1-parser-architecture.md) - 解析器整体架构
- [11-2 块级元素](./11-2-block-elements.md) - 块级元素详解
