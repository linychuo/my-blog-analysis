# 12.3 前端功能集成

> **核心依赖**: highlight.js 11.9.0（语法高亮）, KaTeX 0.12.0（数学公式）, Google Fonts（字体）

## 12.3.1 前端资源加载策略

### CDN 资源清单

| 资源类型 | 提供商 | 版本 | 用途 |
|---------|--------|------|------|
| Inter Font | Google Fonts | - | 正文字体 |
| JetBrains Mono | Google Fonts | - | 代码字体 |
| highlight.js CSS | Cloudflare | 11.9.0 | 语法高亮样式 |
| highlight.js JS | Cloudflare | 11.9.0 | 语法高亮引擎 |
| KaTeX CSS | jsDelivr | 0.12.0 | 数学公式样式 |
| KaTeX JS | jsDelivr | 0.12.0 | 数学公式渲染 |

### 加载顺序优化

```html
<head>
    <!-- 1. 字体预连接（提前建立连接） -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    
    <!-- 2. 字体加载（阻塞渲染但必要） -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500;600&display=swap" rel="stylesheet">
    
    <!-- 3. highlight.js（先 CSS 后 JS） -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css" id="hljs-theme">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js"></script>
    
    <!-- 4. KaTeX（defer 延迟执行） -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.css">
    <script defer src="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.js"></script>
    
    <!-- 5. 本地样式（最后加载，可覆盖 CDN 样式） -->
    <link rel="stylesheet" href="/style.css">
</head>
```

## 12.3.2 语法高亮（highlight.js）

### 自动高亮机制

```javascript
// highlight.js 自动检测并高亮代码块
hljs.highlightAll();

// 在页面加载完成后调用
document.addEventListener('DOMContentLoaded', () => {
    hljs.highlightAll();
});
```

### Markdown 代码块渲染

```markdown
<!-- 输入 Markdown -->
```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, {s}!\n", .{"World"});
}
```

<!-- 输出 HTML -->
<pre><code class="language-zig">
<span class="hljs-keyword">const</span> std = <span class="hljs-built_in">@import</span>(<span class="hljs-string">"std"</span>);

<span class="hljs-keyword">pub</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>() <span class="hljs-type">void</span> {
    std.debug.<span class="hljs-title">print</span>(<span class="hljs-string">"Hello, {s}!\n"</span>, .{<span class="hljs-string">"World"</span>});
}
</code></pre>
```

### 支持的语言

my-blog 的 highlight.js 配置支持自动检测以下语言：

| 语言类别 | 支持语言 |
|---------|---------|
| 系统编程 | Zig, C, C++, Rust |
| Web 开发 | JavaScript, TypeScript, HTML, CSS |
| 脚本语言 | Python, Ruby, Bash, PowerShell |
| 数据 | JSON, YAML, XML, SQL |
| 其他 | Go, Java, Kotlin, Swift |

### 手动指定语言

```markdown
<!-- 推荐：明确指定语言 -->
```python
def hello():
    print("Hello, World!")
```

<!-- 不推荐：依赖自动检测 -->
```
def hello():
    print("Hello, World!")
```
```

### 高亮主题切换

```javascript
// 主题切换时同步更新 highlight.js 样式
function toggleTheme() {
    const newTheme = document.documentElement.getAttribute('data-theme');
    const hljsLink = document.getElementById('hljs-theme');
    
    if (newTheme === 'light') {
        hljsLink.href = 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css';
    } else {
        hljsLink.href = 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css';
    }
}
```

## 12.3.3 数学公式（KaTeX）

### 公式语法

my-blog 使用 `$$` 包裹数学公式，支持行内公式和块级公式：

```markdown
<!-- 行内公式 -->
质能方程是 $E = mc^2$。

<!-- 块级公式（独占一行） -->
$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

<!-- 多行公式 -->
$$
\begin{aligned}
f(x) &= x^2 + 2x + 1 \\
     &= (x + 1)^2
\end{aligned}
$$
```

### Markdown 解析器集成

在 `Blogger.zig` 中，Markdown 解析器识别 `$$` 包裹的内容并转换为 KaTeX 可渲染的格式：

```zig
// 伪代码：识别数学公式块
if (content.startsWith("$$") and content.endsWith("$$")) {
    const tex = content[2..content.len - 2];  // 提取 LaTeX 内容
    return fmt"<pre class=\"math-block\">{s}</pre>", .{tex};
}
```

### KaTeX 渲染脚本

```javascript
// 页面加载后渲染所有数学公式块
document.addEventListener('DOMContentLoaded', function() {
    if (typeof katex !== 'undefined') {
        document.querySelectorAll('pre.math-block').forEach(function(el) {
            try {
                const tex = el.textContent.trim();
                const div = document.createElement('div');
                div.className = 'katex-display';
                katex.render(tex, div, {
                    displayMode: true,
                    throwOnError: false  // 错误不抛出异常
                });
                el.parentNode.replaceChild(div, el);
            } catch (e) {
                console.error('KaTeX render error:', e);
            }
        });
    }
});
```

### 支持的数学符号

| 类别 | 示例 | 渲染结果 |
|------|------|---------|
| 上标/下标 | `x^2`, `a_n` | $x^2$, $a_n$ |
| 分数 | `\frac{a}{b}` | $\frac{a}{b}$ |
| 根号 | `\sqrt{x}`, `\sqrt[n]{x}` | $\sqrt{x}$, $\sqrt[n]{x}$ |
| 积分 | `\int_{a}^{b}` | $\int_{a}^{b}$ |
| 求和 | `\sum_{i=1}^{n}` | $\sum_{i=1}^{n}$ |
| 极限 | `\lim_{x \to 0}` | $\lim_{x \to 0}$ |
| 希腊字母 | `\alpha`, `\beta`, `\gamma` | $\alpha$, $\beta$, $\gamma$ |
| 运算符 | `\times`, `\div`, `\pm` | $\times$, $\div$, $\pm$ |
| 关系符 | `\leq`, `\geq`, `\neq` | $\leq$, $\geq$, $\neq$ |
| 矩阵 | `\begin{matrix} a & b \\ c & d \end{matrix}` | 2×2 矩阵 |

### 常见公式示例

```markdown
<!-- 二次方程求根公式 -->
$$
x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
$$

<!-- 欧拉公式 -->
$$
e^{i\pi} + 1 = 0
$$

<!-- 麦克斯韦方程组 -->
$$
\begin{aligned}
\nabla \cdot \mathbf{E} &= \frac{\rho}{\varepsilon_0} \\
\nabla \cdot \mathbf{B} &= 0 \\
\nabla \times \mathbf{E} &= -\frac{\partial \mathbf{B}}{\partial t} \\
\nabla \times \mathbf{B} &= \mu_0\mathbf{J} + \mu_0\varepsilon_0\frac{\partial \mathbf{E}}{\partial t}
\end{aligned}
$$

<!-- 矩阵乘法 -->
$$
\begin{bmatrix}
1 & 2 \\
3 & 4
\end{bmatrix}
\begin{bmatrix}
5 & 6 \\
7 & 8
\end{bmatrix}
=
\begin{bmatrix}
19 & 22 \\
43 & 50
\end{bmatrix}
$$
```

## 12.3.4 字体系统

### Google Fonts 配置

```html
<!-- Inter: 正文字体（多语言支持） -->
<!-- JetBrains Mono: 代码字体（等宽） -->
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500;600&display=swap" rel="stylesheet">
```

### 字体栈定义

```css
/* 正文字体栈 */
body {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

/* 代码字体栈 */
code, pre {
    font-family: 'JetBrains Mono', 'Fira Code', 'Consolas', monospace;
}
```

### 字体特性

| 字体 | 字重 | 用途 |
|------|------|------|
| Inter | 400 | 正文 |
| Inter | 500 | 强调文字 |
| Inter | 600 | 小标题 |
| Inter | 700 | 主标题 |
| Inter | 800 | 特大标题 |
| JetBrains Mono | 400 | 行内代码 |
| JetBrains Mono | 500 | 代码块 |
| JetBrains Mono | 600 | 代码强调 |

### 字体加载优化

```html
<!-- 使用 display=swap 避免 FOIT（Flash of Invisible Text） -->
<link href="https://fonts.googleapis.com/css2?family=Inter&display=swap" rel="stylesheet">

<!-- 预加载关键字体 -->
<link rel="preload" href="https://fonts.gstatic.com/s/inter/v12/UcCO3FwrK3iLTeHuS_fvQtMwCp50KnMw2boKoduKmMEVuLyfAZ9hjp-Ek-_EeA.woff2" as="font" crossorigin>
```

## 12.3.5 返回顶部功能

### 实现逻辑

```javascript
// 1. 监听滚动事件
const backToTopBtn = document.getElementById('backToTop');

window.addEventListener('scroll', () => {
    // 滚动超过 400px 显示按钮
    if (window.scrollY > 400) {
        backToTopBtn.classList.add('visible');
    } else {
        backToTopBtn.classList.remove('visible');
    }
});

// 2. 平滑滚动到顶部
function scrollToTop() {
    window.scrollTo({
        top: 0,
        behavior: 'smooth'  // 平滑滚动
    });
}
```

### 按钮样式

```css
.back-to-top {
    position: fixed;
    bottom: 32px;
    right: 32px;
    width: 48px;
    height: 48px;
    background-color: var(--accent);
    color: #ffffff;
    border: none;
    border-radius: 50%;
    cursor: pointer;
    opacity: 0;
    transform: translateY(20px);
    transition: all 0.3s ease;
    z-index: 999;
}

.back-to-top.visible {
    opacity: 1;
    transform: translateY(0);
}

.back-to-top:hover {
    background-color: var(--accent-hover);
    transform: translateY(-4px);
}
```

## 12.3.6 响应式图片

### 图片加载优化

```html
<!-- 懒加载图片 -->
<img src="image.jpg" alt="描述" loading="lazy">

<!-- 响应式图片（根据屏幕尺寸加载不同版本） -->
<img srcset="image-480.jpg 480w,
             image-768.jpg 768w,
             image-1200.jpg 1200w"
     sizes="(max-width: 600px) 480px,
            (max-width: 900px) 768px,
            1200px"
     src="image-1200.jpg"
     alt="描述">
```

### Markdown 图片语法

```markdown
<!-- 标准图片 -->
![图片描述](image.jpg)

<!-- 带标题的图片 -->
![图片描述](image.jpg "图片标题")

<!-- 响应式图片（需要自定义语法） -->
{{< responsive-img src="image.jpg" alt="描述" >}}
```

## 12.3.7 性能优化

### CDN 选择策略

| CDN | 优势 | 适用资源 |
|-----|------|---------|
| Google Fonts | 全球覆盖，字体专用 | 字体文件 |
| Cloudflare | 速度快，稳定性高 | highlight.js |
| jsDelivr | 支持 npm/GitHub | KaTeX |

### 资源加载优先级

```html
<!-- 高优先级：关键 CSS -->
<link rel="stylesheet" href="style.css">

<!-- 中优先级：非关键 JS -->
<script defer src="app.js"></script>

<!-- 低优先级：懒加载资源 -->
<link rel="preload" href="font.woff2" as="font" crossorigin>
```

### 缓存策略

```html
<!-- 设置长缓存时间（带版本号） -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css">

<!-- 本地资源添加哈希 -->
<link rel="stylesheet" href="/style.a1b2c3d4.css">
```

## 12.3.8 无障碍（a11y）

### ARIA 标签

```html
<!-- 主题切换按钮 -->
<button class="theme-toggle" onclick="toggleTheme()" aria-label="Toggle theme">
    <!-- 图标 -->
</button>

<!-- 返回顶部按钮 -->
<button id="backToTop" class="back-to-top" onclick="scrollToTop()" aria-label="Back to top">
    <!-- 图标 -->
</button>

<!-- 导航菜单 -->
<nav class="navbar" aria-label="Main navigation">
    <!-- 导航链接 -->
</nav>
```

### 键盘导航

```css
/* 焦点可见样式 */
:focus-visible {
    outline: 2px solid var(--accent);
    outline-offset: 2px;
}

/* 跳过导航链接（屏幕阅读器） */
.skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    z-index: 1001;
}

.skip-link:focus {
    top: 0;
}
```

## 相关文档

- [12.1 CSS 架构与设计系统](./12-1-css-architecture.md) - CSS 变量与布局
- [12.2 主题系统实现](./12-2-theme-system.md) - 主题切换机制
- [11.4 HTML 生成](../part-11-markdown-parser/11-4-html-generation.md) - Markdown 转 HTML 流程

---

*本文档介绍了 my-blog 集成的前端功能，包括 highlight.js 语法高亮、KaTeX 数学公式、Google Fonts 字体系统以及性能优化和无障碍设计。*
