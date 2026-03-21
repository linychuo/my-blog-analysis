# 11-2 块级元素

> Markdown 块级元素解析与 HTML 生成

---

## 1. 块级元素概览

### 1.1 什么是块级元素

块级元素（Block Elements）是 Markdown 中构成文档结构的基本单元，每个块级元素在 HTML 中对应一个独立的块级标签。

**常见块级元素**：

| 元素 | Markdown 语法 | HTML 标签 |
|------|-------------|----------|
| 标题 | `# Heading` | `<h1>` - `<h6>` |
| 段落 | 普通文本 | `<p>` |
| 引用 | `> quote` | `<blockquote>` |
| 代码块 | ` ``` ` | `<pre><code>` |
| 列表 | `- item` | `<ul><li>` |
| 有序列表 | `1. item` | `<ol><li>` |
| 表格 | `\| col \|` | `<table>` |
| 水平线 | `---` | `<hr>` |

### 1.2 解析顺序

Markdown 解析器按以下顺序识别块级元素：

```
1. 空行检测 → 分隔段落
    │
2. 标题检测 → # 符号
    │
3. 引用检测 → > 符号
    │
4. 代码块检测 → ``` 或缩进
    │
5. 列表检测 → - 或数字
    │
6. 表格检测 → | 符号
    │
7. 水平线检测 → ---
    │
8. 默认 → 段落
```

---

## 2. 标题 (Headings)

### 2.1 ATX 标题语法

**语法**：

```markdown
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```

**HTML 输出**：

```html
<h1>一级标题</h1>
<h2>二级标题</h2>
<h3>三级标题</h3>
<h4>四级标题</h4>
<h5>五级标题</h5>
<h6>六级标题</h6>
```

### 2.2 项目实例

**Markdown 输入**：

```markdown
# 我的博客文章

## 引言

这是文章的引言部分。

## 正文

### 子章节

详细内容...

## 结论

总结全文。
```

**HTML 输出**：

```html
<h1>我的博客文章</h1>

<h2>引言</h2>
<p>这是文章的引言部分。</p>

<h2>正文</h2>

<h3>子章节</h3>
<p>详细内容...</p>

<h2>结论</h2>
<p>总结全文。</p>
```

### 2.3 解析规则

```zig
// 标题解析规则
fn parseHeading(line: []const u8) !Heading {
    // [1] 计算 # 数量
    var level: u8 = 0;
    for (line) |char| {
        if (char == '#') level += 1;
        else break;
    }
    
    // [2] 验证级别 (1-6)
    if (level < 1 or level > 6) {
        return error.InvalidHeading;
    }
    
    // [3] 提取标题文本（跳过 # 和空格）
    const text = line[level + 1..];
    
    return Heading{
        .level = level,
        .text = text,
    };
}
```

---

## 3. 段落 (Paragraphs)

### 3.1 段落语法

**语法**：

```markdown
这是第一段。

这是第二段。
```

**HTML 输出**：

```html
<p>这是第一段。</p>

<p>这是第二段。</p>
```

### 3.2 段落分隔规则

```
规则：
    │
    ├─ 空行分隔 → 新段落
    │
    ├─ 连续文本 → 同一段落
    │
    └─ 行尾换行符 → 忽略（除非使用 \ 扩展）
```

**示例**：

```markdown
这是第一行
这是第二行
（同一段落）

这是新段落
（空行分隔）
```

**HTML 输出**：

```html
<p>这是第一行 这是第二行</p>

<p>这是新段落</p>
```

### 3.3 项目中的段落处理

```zig
// 段落解析简化逻辑
fn parseParagraph(lines: [][]const u8) !Paragraph {
    var text = std.ArrayList(u8).init(allocator);
    defer text.deinit();
    
    for (lines) |line| {
        if (line.len == 0) break;  // 空行结束段落
        try text.appendSlice(line);
        try text.append(' ');  // 行尾添加空格
    }
    
    return Paragraph{
        .content = try text.toOwnedSlice(),
    };
}
```

---

## 4. 引用块 (Blockquotes)

### 4.1 引用语法

**语法**：

```markdown
> 这是引用内容。
> 
> 这是第二段引用。
```

**HTML 输出**：

```html
<blockquote>
    <p>这是引用内容。</p>
    <p>这是第二段引用。</p>
</blockquote>
```

### 4.2 嵌套引用

**语法**：

```markdown
> 第一层引用
> > 第二层引用
> > > 第三层引用
```

**HTML 输出**：

```html
<blockquote>
    <p>第一层引用</p>
    <blockquote>
        <p>第二层引用</p>
        <blockquote>
            <p>第三层引用</p>
        </blockquote>
    </blockquote>
</blockquote>
```

---

## 5. 代码块 (Code Blocks)

### 5.1 围栏代码块 (Fenced Code Blocks)

**语法**：

````markdown
```zig
const std = @import("std");

pub fn main() !void {
    std.debug.print("Hello, Zig!\n", .{});
}
```
````

**HTML 输出**：

```html
<pre><code class="language-zig">const std = @import("std");

pub fn main() !void {
    std.debug.print("Hello, Zig!\n", .{});
}
</code></pre>
```

### 5.2 缩进代码块

**语法**：

```markdown
这是段落。

    // 缩进 4 个空格或 1 个制表符
    const x = 1;
    const y = 2;
```

**HTML 输出**：

```html
<p>这是段落。</p>

<pre><code>// 缩进 4 个空格或 1 个制表符
const x = 1;
const y = 2;
</code></pre>
```

### 5.3 项目中的代码高亮

```html
<!-- layout.hbs 中引入 highlight.js -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/styles/github.min.css">
<script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/core.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/languages/zig.min.js"></script>
<script>
    document.addEventListener('DOMContentLoaded', (event) => {
        hljs.highlightAll();
    });
</script>
```

**JavaScript 高亮逻辑**：

```javascript
// highlight.js 自动识别 language-zig 类
// 并应用语法高亮
hljs.highlightAll();
```

---

## 6. 列表 (Lists)

### 6.1 无序列表

**语法**：

```markdown
- 项目 1
- 项目 2
  - 嵌套项目 2.1
  - 嵌套项目 2.2
- 项目 3
```

**HTML 输出**：

```html
<ul>
    <li>项目 1</li>
    <li>
        项目 2
        <ul>
            <li>嵌套项目 2.1</li>
            <li>嵌套项目 2.2</li>
        </ul>
    </li>
    <li>项目 3</li>
</ul>
```

### 6.2 有序列表

**语法**：

```markdown
1. 第一步
2. 第二步
3. 第三步
```

**HTML 输出**：

```html
<ol>
    <li>第一步</li>
    <li>第二步</li>
    <li>第三步</li>
</ol>
```

### 6.3 列表解析规则

```zig
// 列表项解析
fn parseListItem(line: []const u8) !ListItem {
    // [1] 检测列表标记
    const marker = detectMarker(line);  // -, *, +, 或数字
    if (marker == null) return error.NotListItem;
    
    // [2] 提取内容（跳过标记和空格）
    const content = line[marker.len + 1..];
    
    // [3] 检测嵌套（缩进）
    const indent = countLeadingSpaces(line);
    
    return ListItem{
        .content = content,
        .indent = indent,
        .marker = marker.?,
    };
}
```

---

## 7. 表格 (Tables)

### 7.1 表格语法

**语法**：

```markdown
| 列 1 | 列 2 | 列 3 |
|------|------|------|
| 值 1 | 值 2 | 值 3 |
| 值 4 | 值 5 | 值 6 |
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
        <tr>
            <td>值 4</td>
            <td>值 5</td>
            <td>值 6</td>
        </tr>
    </tbody>
</table>
```

### 7.2 对齐方式

**语法**：

```markdown
| 左对齐 | 居中对齐 | 右对齐 |
|:-------|:------:|-------:|
| 值 1 | 值 2 | 值 3 |
```

**HTML 输出**：

```html
<table>
    <thead>
        <tr>
            <th style="text-align: left;">左对齐</th>
            <th style="text-align: center;">居中对齐</th>
            <th style="text-align: right;">右对齐</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td style="text-align: left;">值 1</td>
            <td style="text-align: center;">值 2</td>
            <td style="text-align: right;">值 3</td>
        </tr>
    </tbody>
</table>
```

---

## 8. 水平线 (Horizontal Rules)

### 8.1 水平线语法

**语法**：

```markdown
---
***
___
```

**HTML 输出**：

```html
<hr>
```

### 8.2 解析规则

```
规则：
    │
    ├─ 至少 3 个连续字符 (-, *, _)
    │
    ├─ 可以包含空格
    │
    └─ 不能与其他元素混淆（如标题）
```

**示例**：

```markdown
这是段落。

---

这是新段落。
```

**HTML 输出**：

```html
<p>这是段落。</p>

<hr>

<p>这是新段落。</p>
```

---

## 9. 完整实例分析

### 9.1 完整 Markdown 文章

```markdown
# 使用 Zig 编写博客

## 简介

Zig 是一门系统编程语言。

## 特性

- 简单
- 快速
- 安全

## 代码示例

```zig
const std = @import("std");

pub fn main() !void {
    std.debug.print("Hello!\n", .{});
}
```

## 总结

Zig 是一门优秀的语言。
```

### 9.2 生成的 HTML

```html
<h1>使用 Zig 编写博客</h1>

<h2>简介</h2>
<p>Zig 是一门系统编程语言。</p>

<h2>特性</h2>
<ul>
    <li>简单</li>
    <li>快速</li>
    <li>安全</li>
</ul>

<h2>代码示例</h2>
<pre><code class="language-zig">const std = @import("std");

pub fn main() !void {
    std.debug.print("Hello!\n", .{});
}
</code></pre>

<h2>总结</h2>
<p>Zig 是一门优秀的语言。</p>
```

---

## 10. 注意事项

### 10.1 空行的重要性

```markdown
<!-- 正确：空行分隔 -->
段落 1。

段落 2。

<!-- 错误：缺少空行 -->
段落 1。
段落 2。
<!-- 可能被解析为同一段落 -->
```

### 10.2 列表中的空行

```markdown
<!-- 列表项之间不需要空行 -->
- 项目 1
- 项目 2
- 项目 3

<!-- 添加空行会增加段落间距 -->
- 项目 1

- 项目 2

- 项目 3
```

### 10.3 代码块的语言标识

````markdown
<!-- 推荐：指定语言 -->
```zig
const x = 1;
```

<!-- 不推荐：不指定语言 -->
```
const x = 1;
```
<!-- 无法应用语法高亮 -->
````

---

## 相关文档

- [11-1 解析器架构](./11-1-parser-architecture.md) - 解析器整体架构
- [11-3 内联元素](./11-3-inline-elements.md) - 粗体、斜体、链接等
- [11-4 HTML 生成](./11-4-html-generation.md) - HTML 渲染流程
