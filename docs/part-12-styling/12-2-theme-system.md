# 12.2 主题系统实现

> **核心机制**: 基于 `data-theme` 属性的 CSS 变量切换 + localStorage 持久化 + 自动时间检测

## 12.2.1 主题系统架构

my-blog 采用双层主题切换机制：

```
┌─────────────────────────────────────────────────────┐
│                  主题初始化流程                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  页面加载 → 执行内联脚本 → 检测 localStorage        │
│                    ↓                                │
│         ┌───────┬───────────────┬─────────┐         │
│         │       │               │         │         │
│      有缓存   无缓存         无缓存   →  默认     │
│         │       │               │         │         │
│         ↓       ↓               ↓         ↓         │
│     使用缓存  检测时间      检测时间   暗色主题    │
│     主题     6AM-6PM       其他时间   (fallback)  │
│                │               │                   │
│                ↓               ↓                   │
│           亮色主题         暗色主题                │
│                │               │                   │
│                └───────┬───────┘                   │
│                        ↓                           │
│            设置 data-theme 属性                    │
│                        ↓                           │
│            CSS 变量自动应用                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 12.2.2 主题检测脚本

### 内联脚本（防止闪烁）

主题检测脚本放置在 `<head>` 中，在页面渲染前执行，避免主题闪烁（FOUC - Flash of Unstyled Content）：

```html
<script>
!function(){
    const t = localStorage.getItem('theme');
    if (t) {
        // 1. 优先使用用户缓存的主题
        document.documentElement.setAttribute('data-theme', t);
    } else {
        // 2. 无缓存时根据时间自动选择
        const hours = new Date().getHours();
        const autoTheme = (hours >= 6 && hours < 18) ? 'light' : 'dark';
        document.documentElement.setAttribute('data-theme', autoTheme);
    }
}();
</script>
```

### 执行时机说明

| 位置 | 执行时机 | 目的 |
|------|---------|------|
| `<head>` 内联 | DOM 解析前 | 防止主题闪烁 |
| 外部 JS 文件 | DOM 加载后 | 处理用户交互 |

## 12.2.3 主题切换逻辑

### 切换按钮实现

```html
<button class="theme-toggle" onclick="toggleTheme()" aria-label="Toggle theme">
    <!-- 太阳图标（亮色主题显示） -->
    <svg class="sun-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="12" cy="12" r="5"/>
        <path d="M12 1v2M12 21v2M4.22 4.22l1.42 1.42M18.36 18.36l1.42 1.42M1 12h2M21 12h2M4.22 19.78l1.42-1.42M18.36 5.64l1.42-1.42"/>
    </svg>
    <!-- 月亮图标（暗色主题显示） -->
    <svg class="moon-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
    </svg>
</button>
```

### JavaScript 切换函数

```javascript
function toggleTheme() {
    const html = document.documentElement;
    const currentTheme = html.getAttribute('data-theme');
    const newTheme = currentTheme === 'light' ? 'dark' : 'light';
    
    // 1. 更新 DOM 属性
    html.setAttribute('data-theme', newTheme);
    
    // 2. 持久化到 localStorage
    localStorage.setItem('theme', newTheme);
    
    // 3. 切换 highlight.js 主题
    const hljsTheme = newTheme === 'light' 
        ? 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css'
        : 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css';
    document.getElementById('hljs-theme').href = hljsTheme;
}
```

### 图标显示逻辑

```css
/* 默认状态（暗色主题） */
.theme-toggle .sun-icon { 
    display: none;  /* 隐藏太阳 */
}
.theme-toggle .moon-icon { 
    display: block; /* 显示月亮 */
}

/* 亮色主题状态 */
[data-theme="light"] .theme-toggle .sun-icon { 
    display: block; /* 显示太阳 */
}
[data-theme="light"] .theme-toggle .moon-icon { 
    display: none;  /* 隐藏月亮 */
}
```

## 12.2.4 CSS 变量切换机制

### 变量定义与覆盖

```css
/* 基础定义（暗色主题默认值） */
:root {
    --bg-primary: #0d1117;
    --text-primary: #f0f6fc;
    --accent: #58a6ff;
}

/* 亮色主题覆盖 */
[data-theme="light"] {
    --bg-primary: #ffffff;
    --text-primary: #1f2328;
    --accent: #0969da;
}

/* 使用变量（自动应用当前主题） */
body {
    background-color: var(--bg-primary);
    color: var(--text-primary);
}
```

### 级联覆盖规则

```css
/* 优先级：元素选择器 > 属性选择器 > :root */

/* 1. 最低优先级 - :root */
:root { --bg-primary: #0d1117; }

/* 2. 中优先级 - 属性选择器 */
[data-theme="light"] { --bg-primary: #ffffff; }

/* 3. 高优先级 - 具体元素 */
.article-content { --bg-primary: #161b22; }

/* 4. 最高优先级 - !important（慎用） */
.critical { --bg-primary: #000000 !important; }
```

## 12.2.5 主题持久化

### localStorage 数据结构

```javascript
// 存储结构
localStorage = {
    'theme': 'light'  // 或 'dark'
}

// 读取示例
const savedTheme = localStorage.getItem('theme');  // "light"

// 删除缓存（恢复自动检测）
localStorage.removeItem('theme');
```

### 持久化流程图

```
用户点击切换按钮
       ↓
toggleTheme() 执行
       ↓
┌──────────────────┐
│ 1. 更新 DOM 属性   │ → [data-theme="light"] 生效
│    setAttribute  │
├──────────────────┤
│ 2. 写入 localStorage │ → localStorage.setItem('theme', 'light')
├──────────────────┤
│ 3. 切换代码高亮主题  │ → 更新 link#hljs-theme.href
└──────────────────┘
       ↓
页面刷新
       ↓
读取 localStorage → 恢复用户偏好
```

## 12.2.6 代码高亮主题同步

### 主题联动机制

```javascript
// highlight.js 主题映射表
const hljsThemes = {
    'light': 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css',
    'dark': 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css'
};

// 切换时同步更新
function syncCodeHighlight(theme) {
    const link = document.getElementById('hljs-theme');
    link.href = hljsThemes[theme];
}
```

### 主题对比表

| 主题模式 | highlight.js 主题 | 配色特点 |
|---------|------------------|---------|
| 暗色 | github-dark | 深灰背景，蓝色/绿色语法色 |
| 亮色 | github | 白色背景，红色/紫色语法色 |

## 12.2.7 时间自动检测逻辑

### 时间段划分

```
┌─────────────────────────────────────────────────────┐
│                  24 小时主题自动切换                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│   00:00 ────── 06:00 ────── 18:00 ────── 24:00     │
│     │            │            │            │        │
│     │   暗色     │   亮色     │   暗色     │        │
│     │  (夜间)   │  (白天)   │  (夜间)   │        │
│     │            │            │            │        │
│     └────────────┴────────────┴────────────┘        │
│                  │            │                     │
│              6AM 切换     6PM 切换                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 实现代码

```javascript
function getAutoTheme() {
    const hours = new Date().getHours();
    // 6:00 - 18:00 使用亮色主题
    if (hours >= 6 && hours < 18) {
        return 'light';
    }
    // 其他时间使用暗色主题
    return 'dark';
}
```

### 时区考虑

```javascript
// 使用本地时间（用户时区）
const localHours = new Date().getHours();

// 如需统一时区（如 UTC）
const utcHours = new Date().getUTCHours();
```

## 12.2.8 主题切换最佳实践

### 避免闪烁（FOUC）

```html
<!-- ✅ 正确：内联脚本放在 <head> 顶部 -->
<head>
    <script>/* 主题检测脚本 */</script>
    <link rel="stylesheet" href="style.css">
</head>

<!-- ❌ 错误：外部脚本或放在 <body> -->
<head>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <script src="theme.js"></script>  <!-- 会闪烁！ -->
</body>
```

### 平滑过渡

```css
/* 启用主题过渡动画 */
html {
    transition: background-color 0.3s ease, color 0.3s ease;
}

/* 排除不需要过渡的元素 */
.navbar {
    transition: background-color 0.3s ease;
}
```

### 无障碍考虑

```html
<!-- 添加 aria-label 说明按钮功能 -->
<button class="theme-toggle" onclick="toggleTheme()" aria-label="Toggle theme">
    <!-- 图标 -->
</button>

<!-- 屏幕阅读器提示 -->
<span class="sr-only">当前主题：{{currentTheme}}</span>
```

## 12.2.9 主题扩展指南

### 添加新主题

```css
/* 1. 定义新主题变量 */
[data-theme="dim"] {
    --bg-primary: #1a1f2e;
    --text-primary: #b0b8c4;
    --accent: #79c0ff;
}

/* 2. 添加切换选项 */
function setTheme(theme) {
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('theme', theme);
}

// 使用
setTheme('dim');  // 切换到"护眼模式"
```

### 主题配置文件

```json
// themes.json
{
    "dark": {
        "name": "暗色主题",
        "icon": "moon",
        "variables": { /* ... */ }
    },
    "light": {
        "name": "亮色主题",
        "icon": "sun",
        "variables": { /* ... */ }
    },
    "dim": {
        "name": "护眼模式",
        "icon": "eye",
        "variables": { /* ... */ }
    }
}
```

## 相关文档

- [12.1 CSS 架构与设计系统](./12-1-css-architecture.md) - CSS 变量定义
- [12.3 前端功能集成](./12-3-frontend-features.md) - highlight.js 与 KaTeX
- [10.1 模板引擎架构](../part-10-template-engine/10-1-architecture.md) - layout.hbs 模板结构

---

*本文档详细分析了 my-blog 的主题系统实现，包括主题检测、切换逻辑、持久化机制和代码高亮同步。*
