# 11-3 内联元素

> Markdown 内联元素解析与 HTML 生成

---

## 1. 内联元素概览

### 1.1 什么是内联元素

内联元素（Inline Elements）是 Markdown 中用于格式化文本的元素，它们不会创建新的块级结构，而是在段落内部应用样式。

**常见内联元素**：

| 元素 | Markdown 语法 | HTML 标签 |
|------|-------------|----------|
| 粗体 | `**text**` | `<strong>` |
| 斜体 | `*text*` | `<em>` |
| 删除线 | `~~text~~` | `<del>` |
| 行内代码 | `` `code` `` | `<code>` |
| 链接 | `[text](url)` | `<a>` |
| 图片 | `![alt](src)` | `<img>` |

### 1.2 解析顺序

内联元素按以下优先级解析：

```
1. 行内代码 `code` → 最高优先级（内容不解析）
    │
2. 图片 ![alt](src)
    │
3. 链接 [text](url)
    │
4. 粗体/斜体 **text** *text*
    │
5. 删除线 ~~text~~
    │
6. 普通文本 → 最低优先级
```

---

## 2. 粗体 (Bold)

### 2.1 语法

**语法**：

```markdown
这是**粗体**文本。
这是__粗体__文本。
```

**HTML 输出**：

```html
<p>这是<strong>粗体</strong>文本。</p>

<p>这是<strong>粗体</strong>文本。</p>
```

### 2.2 嵌套使用

```markdown
这是***粗斜体***文本。
这是**粗体和*斜体*混合**。
```

**HTML 输出**：

```html
<p>这是<strong><em>粗斜体</em></strong>文本。</p>

<p>这是<strong>粗体和<em>斜体</em>混合</strong>。</p>
```

### 2.3 项目实例

**Markdown 输入**：

```markdown
## 重要提示

**注意**：这是一个重要的警告。

请务必**仔细阅读**文档。
```

**HTML 输出**：

```html
<h2>重要提示</h2>

<p><strong>注意</strong>：这是一个重要的警告。</p>

<p>请务必<strong>仔细阅读</strong>文档。</p>
```

---

## 3. 斜体 (Italic)

### 3.1 语法

**语法**：

```markdown
这是*斜体*文本。
这是_斜体_文本。
```

**HTML 输出**：

```html
<p>这是<em>斜体</em>文本。</p>

<p>这是<em>斜体</em>文本。</p>
```

### 3.2 与粗体组合

```markdown
这是***粗斜体***文本。
```

**HTML 输出**：

```html
<p>这是<strong><em>粗斜体</em></strong>文本。</p>
```

---

## 4. 删除线 (Strikethrough)

### 4.1 语法

**语法**：

```markdown
这是~~删除线~~文本。
```

**HTML 输出**：

```html
<p>这是<del>删除线</del>文本。</p>
```

### 4.2 使用场景

```markdown
原价：~~$99~~ 现价：$49

~~已过期~~ 已更新
```

**HTML 输出**：

```html
<p>原价：<del>$99</del> 现价：$49</p>

<p><del>已过期</del> 已更新</p>
```

---

## 5. 行内代码 (Inline Code)

### 5.1 语法

**语法**：

```markdown
使用 `print()` 函数输出内容。

变量 `name` 存储用户名称。
```

**HTML 输出**：

```html
<p>使用 <code>print()</code> 函数输出内容。</p>

<p>变量 <code>name</code> 存储用户名称。</p>
```

### 5.2 代码中的反引号

**语法**：

```markdown
使用 `` ` `` 表示反引号。

使用 ``` `nested` ``` 表示嵌套。
```

**HTML 输出**：

```html
<p>使用 <code>`</code> 表示反引号。</p>

<p>使用 <code>`nested`</code> 表示嵌套。</p>
```

### 5.3 项目实例

**Markdown 输入**：

```markdown
## Zig 代码示例

使用 `@import("std")` 导入标准库。

`std.debug.print()` 用于调试输出。
```

**HTML 输出**：

```html
<h2>Zig 代码示例</h2>

<p>使用 <code>@import("std")</code> 导入标准库。</p>

<p><code>std.debug.print()</code> 用于调试输出。</p>
```

---

## 6. 链接 (Links)

### 6.1 基本语法

**语法**：

```markdown
访问 [Zig 官网](https://ziglang.org)。
```

**HTML 输出**：

```html
<p>访问 <a href="https://ziglang.org">Zig 官网</a>。</p>
```

### 6.2 带标题的链接

**语法**：

```markdown
访问 [Zig 官网](https://ziglang.org "Zig 编程语言")。
```

**HTML 输出**：

```html
<p>访问 <a href="https://ziglang.org" title="Zig 编程语言">Zig 官网</a>。</p>
```

### 6.3 项目中的链接

**Markdown 输入**：

```markdown
## 相关资源

- [Zig 文档](https://ziglang.org/documentation)
- [Zig 标准库参考](https://ziglang.org/documentation/master/std)
- [GitHub 仓库](https://github.com/ziglang/zig)
```

**HTML 输出**：

```html
<h2>相关资源</h2>

<ul>
    <li><a href="https://ziglang.org/documentation">Zig 文档</a></li>
    <li><a href="https://ziglang.org/documentation/master/std">Zig 标准库参考</a></li>
    <li><a href="https://github.com/ziglang/zig">GitHub 仓库</a></li>
</ul>
```

---

## 7. 图片 (Images)

### 7.1 基本语法

**语法**：

```markdown
![替代文本](图片 URL)
```

**HTML 输出**：

```html
<p><img src="图片 URL" alt="替代文本"></p>
```

### 7.2 带标题的图片

**语法**：

```markdown
![Zig Logo](https://ziglang.org/zig-logo.png "Zig 标志")
```

**HTML 输出**：

```html
<p><img src="https://ziglang.org/zig-logo.png" alt="Zig Logo" title="Zig 标志"></p>
```

### 7.3 项目中的图片

**Markdown 输入**：

```markdown
## 项目截图

![项目截图](/imgs/screenshot.png "项目截图")

这是项目的屏幕截图。
```

**HTML 输出**：

```html
<h2>项目截图</h2>

<p><img src="/imgs/screenshot.png" alt="项目截图" title="项目截图"></p>

<p>这是项目的屏幕截图。</p>
```

---

## 8. 内联元素组合

### 8.1 复杂组合

**Markdown 输入**：

```markdown
这是**粗体**和*斜体*还有`代码`的组合。

这是[链接](https://example.com)和![图片](image.png)的混合。

这是***粗斜体***文本。
```

**HTML 输出**：

```html
<p>这是<strong>粗体</strong>和<em>斜体</em>还有<code>代码</code>的组合。</p>

<p>这是<a href="https://example.com">链接</a>和<img src="image.png" alt="图片">的混合。</p>

<p>这是<strong><em>粗斜体</em></strong>文本。</p>
```

### 8.2 嵌套规则

```
嵌套规则:
    │
    ├─ 行内代码不能嵌套 → `code `code`` 无效
    │
    ├─ 粗体可以包含斜体 → **bold *italic***
    │
    ├─ 链接不能嵌套链接 → [text [nested](url)](url) 无效
    │
    └─ 图片不能包含其他元素 → ![**bold**](url) 无效
```

---

## 9. 转义字符

### 9.1 特殊字符转义

**语法**：

```markdown
使用 \* 显示星号。

使用 \[ 显示方括号。

使用 \\ 显示反斜杠。
```

**HTML 输出**：

```html
<p>使用 * 显示星号。</p>

<p>使用 [ 显示方括号。</p>

<p>使用 \ 显示反斜杠。</p>
```

### 9.2 可转义的字符

| 字符 | 转义 | 说明 |
|------|------|------|
| `\` | `\\` | 反斜杠 |
| | `` \` `` | 反引号 |
| `*` | `\*` | 星号 |
| `_` | `\_` | 下划线 |
| `[` | `\[` | 左方括号 |
| `]` | `\]` | 右方括号 |
| `(` | `\(` | 左圆括号 |
| `)` | `\)` | 右圆括号 |

### 9.3 项目实例

**Markdown 输入**：

```markdown
## 转义示例

文件名：`test\_.md`

使用 \*\* 而不是 **bold**。

URL: `https://example.com/path\?query=1`
```

**HTML 输出**：

```html
<h2>转义示例</h2>

<p>文件名：<code>test_.md</code></p>

<p>使用 ** 而不是 <strong>bold</strong>。</p>

<p>URL: <code>https://example.com/path?query=1</code></p>
```

---

## 10. 自动链接

### 10.1 URL 自动识别

**语法**：

```markdown
访问 https://ziglang.org 获取更多信息。
```

**HTML 输出**：

```html
<p>访问 <a href="https://ziglang.org">https://ziglang.org</a> 获取更多信息。</p>
```

### 10.2 邮箱地址

**语法**：

```markdown
联系 <email@example.com>。
```

**HTML 输出**：

```html
<p>联系 <a href="mailto:email@example.com">email@example.com</a>。</p>
```

---

## 11. HTML 实体

### 11.1 特殊字符

**语法**：

```markdown
&copy; 2026

&amp; 符号

&lt; 和 &gt;
```

**HTML 输出**：

```html
<p>&copy; 2026</p>

<p>&amp; 符号</p>

<p>&lt; 和 &gt;</p>
```

### 11.2 Unicode 字符

```markdown
Unicode: © ® ™ € £ ¥
```

**HTML 输出**：

```html
<p>Unicode: © ® ™ € £ ¥</p>
```

---

## 12. 完整实例

### 12.1 复杂段落

**Markdown 输入**：

```markdown
## 内联元素综合示例

这是**粗体**、*斜体*和~~删除线~~的组合。

使用 `const x = 1;` 定义常量。

访问 [Zig 官网](https://ziglang.org) 获取文档。

![Zig Logo](https://ziglang.org/zig-logo.png)

这是***粗斜体***文本。

这是<https://自动链接.com>。
```

**HTML 输出**：

```html
<h2>内联元素综合示例</h2>

<p>这是<strong>粗体</strong>、<em>斜体</em>和<del>删除线</del>的组合。</p>

<p>使用 <code>const x = 1;</code> 定义常量。</p>

<p>访问 <a href="https://ziglang.org">Zig 官网</a> 获取文档。</p>

<img src="https://ziglang.org/zig-logo.png" alt="Zig Logo">

<p>这是<strong><em>粗斜体</em></strong>文本。</p>

<p>这是<a href="https://自动链接.com">https://自动链接.com</a>。</p>
```

---

## 13. 注意事项

### 13.1 空格处理

```markdown
<!-- 正确：星号紧贴文本 -->
**粗体**

<!-- 错误：星号与文本有空格 -->
** 粗体 **
<!-- 不会被解析为粗体 -->
```

### 13.2 优先级问题

```markdown
<!-- 行内代码优先级最高 -->
`**不是粗体**`
<!-- 输出：<code>**不是粗体**</code> -->

<!-- 粗体中的星号 -->
**这是\*\*星号\*\***
<!-- 输出：<strong>这是**星号**</strong> -->
```

### 13.3 链接中的格式

```markdown
<!-- 链接中可以有格式 -->
[**粗体链接**](url)
<!-- 输出：<a href="url"><strong>粗体链接</strong></a> -->

<!-- 但链接不能嵌套 -->
[[嵌套](url1)](url2)
<!-- 无效 -->
```

---

## 相关文档

- [11-1 解析器架构](./11-1-parser-architecture.md) - 解析器整体架构
- [11-2 块级元素](./11-2-block-elements.md) - 标题、段落、列表等
- [11-4 HTML 生成](./11-4-html-generation.md) - HTML 渲染流程
