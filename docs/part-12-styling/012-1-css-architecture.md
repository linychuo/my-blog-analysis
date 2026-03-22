# 12.1 CSS 架构与设计系统

> **核心文件**: `static/style.css`  
> **设计理念**: GitHub 风格暗色/亮色主题系统，基于 CSS 自定义属性的设计令牌

## 12.1.1 CSS 自定义属性（设计令牌）

my-blog 使用 CSS 自定义属性（CSS Custom Properties）构建了一套完整的设计系统，所有颜色、字体、间距都通过变量定义，便于主题切换和维护。

### 暗色主题变量（默认）

```css
:root {
    /* 背景色 */
    --bg-primary: #0d1117;      /* 主背景 - 深灰黑 */
    --bg-secondary: #161b22;    /* 次级背景 - 卡片/侧边栏 */
    --bg-tertiary: #21262d;     /* 第三级背景 - 代码块 */
    
    /* 文字颜色 */
    --text-primary: #f0f6fc;    /* 主文字 - 接近白色 */
    --text-secondary: #8b949e;  /* 次级文字 - 灰色 */
    --text-muted: #6e7681;      /* 弱化文字 - 深灰色 */
    
    /* 强调色 */
    --accent: #58a6ff;          /* 链接/按钮 - 蓝色 */
    --accent-hover: #79c0ff;    /* 悬停状态 - 亮蓝色 */
    
    /* 边框与阴影 */
    --border: #30363d;          /* 边框颜色 */
    --shadow: rgba(0, 0, 0, 0.3); /* 阴影透明度 */
    
    /* 布局 */
    --max-width: 1200px;        /* 内容最大宽度 */
    --header-height: 70px;      /* 导航栏高度 */
}
```

### 亮色主题变量

```css
[data-theme="light"] {
    --bg-primary: #ffffff;
    --bg-secondary: #f6f8fa;
    --bg-tertiary: #eaeef2;
    --text-primary: #1f2328;
    --text-secondary: #656d76;
    --text-muted: #8c959f;
    --accent: #0969da;
    --accent-hover: #0550ae;
    --border: #d0d7de;
    --shadow: rgba(0, 0, 0, 0.1);
}
```

### 字体系统

```css
/* 字体栈 */
--font-body: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
--font-code: 'JetBrains Mono', monospace;

/* 字体大小比例（9 级） */
--font-xs: 0.75rem;    /* 12px - 标签/注释 */
--font-sm: 0.875rem;   /* 14px - 辅助文字 */
--font-base: 1rem;     /* 16px - 正文 */
--font-md: 1.125rem;   /* 18px - 小标题 */
--font-lg: 1.25rem;    /* 20px - 三级标题 */
--font-xl: 1.5rem;     /* 24px - 二级标题 */
--font-2xl: 1.75rem;   /* 28px - 一级标题 */
--font-3xl: 2rem;      /* 32px - 页面标题 */
--font-4xl: 2.5rem;    /* 40px - 特大标题 */
```

## 12.1.2 布局架构

### 整体布局结构

```
┌─────────────────────────────────────────┐
│           固定导航栏 (70px)              │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────┐  ┌──────────────┐      │
│  │             │  │              │      │
│  │   主内容区   │  │   侧边栏     │      │
│  │   (1fr)     │  │   (280px)    │      │
│  │             │  │              │      │
│  │             │  │  (sticky)    │      │
│  └─────────────┘  └──────────────┘      │
│                                         │
├─────────────────────────────────────────┤
│              页脚                        │
└─────────────────────────────────────────┘
```

### CSS Grid 布局实现

```css
.main-container {
    display: grid;
    grid-template-columns: 1fr 280px;  /* 内容区 + 固定宽度侧边栏 */
    gap: 48px;
    max-width: var(--max-width);
    margin: 0 auto;
    padding: 48px 24px;
}

/* 侧边栏粘性定位 */
.sidebar {
    position: sticky;
    top: calc(var(--header-height) + 24px);
    height: fit-content;
}
```

### 导航栏设计

```css
.navbar {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: var(--header-height);
    background-color: rgba(13, 17, 23, 0.8);  /* 半透明背景 */
    backdrop-filter: blur(10px);              /* 毛玻璃效果 */
    border-bottom: 1px solid var(--border);
    z-index: 1000;
}

.navbar-content {
    max-width: var(--max-width);
    margin: 0 auto;
    display: flex;
    justify-content: space-between;
    align-items: center;
    height: 100%;
    padding: 0 24px;
}
```

## 12.1.3 组件样式系统

### 文章卡片

```css
.post-card {
    background-color: var(--bg-secondary);
    border: 1px solid var(--border);
    border-radius: 8px;
    padding: 24px;
    transition: all 0.3s ease;
}

/* 悬停动画 */
.post-card:hover {
    transform: translateY(-4px);
    border-color: var(--accent);
    box-shadow: 0 8px 24px var(--shadow);
}

/* 顶部强调线动画 */
.post-card::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 3px;
    background: linear-gradient(90deg, var(--accent), var(--accent-hover));
    transform: scaleX(0);
    transition: transform 0.3s ease;
}

.post-card:hover::before {
    transform: scaleX(1);
}
```

### 代码块样式

```css
.article-content pre {
    background-color: var(--bg-tertiary);
    border: 1px solid var(--border);
    border-radius: 6px;
    padding: 16px;
    overflow-x: auto;
}

.article-content code {
    font-family: var(--font-code);
    font-size: 0.9em;
}

/* 行内代码 */
.article-content :not(pre) > code {
    background-color: var(--bg-tertiary);
    padding: 2px 6px;
    border-radius: 4px;
}
```

### 标签系统

```css
.tag {
    display: inline-block;
    background-color: var(--tag-bg);
    color: #ffffff;
    padding: 4px 12px;
    border-radius: 16px;
    font-size: var(--font-sm);
    text-decoration: none;
}

.tag:hover {
    opacity: 0.8;
}
```

## 12.1.4 响应式设计

### 断点定义

| 断点 | 宽度范围 | 布局变化 |
|------|---------|---------|
| Desktop | > 900px | 双栏布局（内容 + 侧边栏） |
| Tablet | 600px - 900px | 单栏布局，侧边栏移至底部 |
| Mobile | < 600px | 紧凑单栏，字体缩小 |

### 媒体查询实现

```css
/* 平板断点：900px */
@media (max-width: 900px) {
    .main-container {
        grid-template-columns: 1fr;  /* 单栏布局 */
        gap: 32px;
    }
    
    .sidebar {
        position: static;  /* 取消粘性定位 */
    }
    
    .article-title, .page-title {
        font-size: var(--font-3xl);  /* 标题缩小 */
    }
}

/* 手机断点：600px */
@media (max-width: 600px) {
    .article-title, .page-title {
        font-size: var(--font-2xl);
    }
    
    .post-title {
        font-size: var(--font-lg);
    }
    
    .navbar-menu {
        gap: 12px;  /* 缩小导航间距 */
    }
    
    .main-container {
        padding: 24px 16px;  /* 减小内边距 */
    }
}
```

## 12.1.5 动画与过渡

### 过渡效果规范

```css
/* 标准过渡（0.2s） */
transition: color 0.2s ease, background-color 0.2s ease;

/* 悬停过渡（0.3s） */
transition: transform 0.3s ease, box-shadow 0.3s ease, border-color 0.3s ease;

/* 主题切换过渡 */
html {
    transition: background-color 0.3s ease, color 0.3s ease;
}
```

### 常用动画类

```css
/* 淡入动画 */
@keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
}

.fade-in {
    animation: fadeIn 0.3s ease;
}

/* 返回顶部按钮 */
.back-to-top {
    position: fixed;
    bottom: 32px;
    right: 32px;
    opacity: 0;
    transform: translateY(20px);
    transition: all 0.3s ease;
}

.back-to-top.visible {
    opacity: 1;
    transform: translateY(0);
}
```

## 12.1.6 可访问性设计

### 焦点状态

```css
/* 全局焦点样式 */
:focus-visible {
    outline: 2px solid var(--accent);
    outline-offset: 2px;
    border-radius: 4px;
}

/* 按钮焦点 */
button:focus-visible {
    box-shadow: 0 0 0 3px rgba(88, 166, 255, 0.4);
}
```

### 对比度要求

| 元素组合 | 对比度 | WCAG 级别 |
|---------|--------|----------|
| 主文字/主背景 | 15.8:1 | AAA |
| 次级文字/主背景 | 7.2:1 | AA |
| 强调色/主背景 | 4.6:1 | AA |

## 12.1.7 CDN 依赖集成

### 字体加载

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500;600&display=swap" rel="stylesheet">
```

### 语法高亮（highlight.js）

```html
<!-- 暗色主题默认 -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css" id="hljs-theme">
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js"></script>
```

### 数学公式（KaTeX）

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.css">
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.js"></script>
```

## 12.1.8 样式优先级与层叠

```css
/* 1. 浏览器默认样式（最低优先级） */

/* 2. 外部样式表 style.css */

/* 3. CSS 自定义属性（可被覆盖） */
[data-theme="light"] { --bg-primary: #ffffff; }

/* 4. 内联样式（高优先级） */
<div style="color: red;">

/* 5. !important 声明（最高优先级，慎用） */
.theme-toggle .sun-icon { display: none !important; }
```

## 相关文档

- [12.2 主题系统实现](./012-2-theme-system.md) - 暗色/亮色主题切换机制
- [12.3 前端功能集成](./012-3-frontend-features.md) - highlight.js 与 KaTeX 使用
- [10.2 模板语法详解](../part-10-template-engine/010-2-template-syntax.md) - Handlebars 模板结构

---

*本文档分析了 my-blog 的 CSS 架构设计，包括设计令牌系统、布局架构、组件样式、响应式设计和可访问性设计。*
