# 13.4 性能优化指南

> **优化目标**: 构建速度、页面加载速度、运行时性能  
> **关键指标**: FCP < 1s, LCP < 2.5s, TTI < 3s

## 13.4.1 性能优化层次

```
┌─────────────────────────────────────────────────────────┐
│                  性能优化金字塔                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│                    ╱╲                                   │
│                   ╱  ╲                                  │
│                  ╱ 第 ╲                                 │
│                 ╱ 三  ╲        运行时优化               │
│                ╱ 层   ╲       - 懒加载                  │
│               ╱────────╲      - 防抖节流                │
│              ╱  第二   ╲     - 虚拟滚动                │
│             ╱   层     ╲    构建优化                   │
│            ╱───────────╲   - 增量构建                 │
│           ╱   第一      ╲  - 并行处理                 │
│          ╱   层         ╲ - 缓存复用                 │
│         ╱───────────────╲                            │
│        ╱   架构优化      ╲                           │
│       ╱   - 资源压缩      ╲                          │
│      ╱   - CDN 分发        ╲                         │
│     ╱─────────────────────╲                          │
│    ╱                       ╲                         │
│   ╱═════════════════════════╲                        │
│  基础优化 → 中级优化 → 高级优化                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 优化层次说明

| 层次 | 优化项 | 影响范围 | 实施难度 |
|------|--------|---------|---------|
| 第一层（基础） | 资源压缩、图片优化 | 所有页面 | ⭐ |
| 第二层（中级） | 增量构建、缓存 | 构建流程 | ⭐⭐ |
| 第三层（高级） | 并行处理、预渲染 | 架构层面 | ⭐⭐⭐ |

## 13.4.2 构建性能优化

### 1. 增量构建

```zig
// 在 Blogger.zig 中实现增量构建
pub fn generateIncremental(self: *Blogger) !void {
    const manifest_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, ".build-manifest.json" });
    defer self.allocator.free(manifest_path);
    
    // 读取旧的构建清单
    var old_manifest = try self.loadBuildManifest(manifest_path);
    defer old_manifest.deinit();
    
    // 检测变更的文件
    const changed_posts = try self.detectChangedPosts(old_manifest);
    defer self.allocator.free(changed_posts);
    
    if (changed_posts.len == 0) {
        std.debug.print("No changes detected. Skipping build.\n", .{});
        return;
    }
    
    std.debug.print("Detected {d} changed posts. Rebuilding...\n", .{changed_posts.len});
    
    // 只重新生成变更的文章
    for (changed_posts) |post| {
        try self.generatePostPage(post);
    }
    
    // 更新清单
    try self.saveBuildManifest(manifest_path);
}

// 检测变更
fn detectChangedPosts(self: *Blogger, old_manifest: *BuildManifest) ![]Post {
    var changed = std.ArrayList(Post).init(self.allocator);
    
    const posts = try self.loadPosts();
    for (posts) |post| {
        const old_hash = old_manifest.getHash(post.filename);
        const new_hash = try self.computeFileHash(post.filename);
        
        if (old_hash == null or old_hash.? != new_hash) {
            try changed.append(post);
        }
    }
    
    return changed.toOwnedSlice();
}

// 计算文件哈希
fn computeFileHash(self: *Blogger, path: []const u8) !u64 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();
    
    var hasher = std.hash.Wyhash.init(0);
    try std.io.copy(std.io.AnyWriter, std.io.AnyReader, hasher.writer(), file.reader());
    return hasher.final();
}
```

### 2. 并行处理

```zig
// 使用 Zig 的线程池并行生成页面
pub fn generateParallel(self: *Blogger, posts: []Post) !void {
    var thread_pool = try std.Thread.Pool.init(.{
        .allocator = self.allocator,
        .n_jobs = 4,  // 4 个工作线程
    });
    defer thread_pool.deinit();
    
    var wait_group = std.Thread.WaitGroup.init();
    defer wait_group.wait();
    
    // 并行生成文章页面
    for (posts) |post| {
        wait_group.start();
        try thread_pool.spawn(generatePostPageTask, .{ self, post, &wait_group });
    }
    
    wait_group.wait();
}

fn generatePostPageTask(self: *Blogger, post: Post, wg: *std.Thread.WaitGroup) void {
    defer wg.finish();
    self.generatePostPage(post) catch |err| {
        std.debug.print("Failed to generate {s}: {}\n", .{ post.filename, err });
    };
}
```

### 3. 缓存模板解析

```zig
// 缓存已解析的模板
const TemplateCache = std.StringHashMap(Template);

pub const Blogger = struct {
    template_cache: TemplateCache,
    
    pub fn init(allocator: Allocator) Blogger {
        return Blogger{
            .template_cache = TemplateCache.init(allocator),
        };
    }
    
    pub fn loadTemplate(self: *Blogger, name: []const u8) !Template {
        // 先查缓存
        if (self.template_cache.get(name)) |cached| {
            return cached;
        }
        
        // 缓存未命中，从文件加载
        const template = try self.loadTemplateFromFile(name);
        
        // 存入缓存
        try self.template_cache.put(name, template);
        
        return template;
    }
};
```

## 13.4.3 页面加载性能优化

### 1. 资源压缩

#### CSS 压缩

```bash
# 使用 esbuild 压缩 CSS
esbuild style.css --minify --outfile=style.min.css

# 或使用 clean-css
cleancss -o style.min.css style.css
```

#### 在 build.zig 中集成压缩

```zig
// 在 build.zig 中添加压缩步骤
const minify_css = b.addSystemCommand(&.{ "esbuild", "--minify", "static/style.css", "--outfile=style.min.css" });
minify_css.step.dependOn(&copy_static.step);

const install_minified = b.addInstallFile(minify_css.getOutput(), "blog/style.min.css");
install_minified.step.dependOn(&minify_css.step);
```

### 2. 图片优化

#### 响应式图片

```html
<!-- 使用 srcset 提供多种尺寸 -->
<img srcset="
    image-480w.jpg 480w,
    image-768w.jpg 768w,
    image-1200w.jpg 1200w
" sizes="
    (max-width: 600px) 480px,
    (max-width: 900px) 768px,
    1200px
" src="image-1200w.jpg" alt="描述" loading="lazy">
```

#### 现代图片格式

```markdown
<!-- 在 Markdown 中使用 AVIF/WebP -->
![描述](image.avif)

<!-- 回退方案 -->
<picture>
    <source srcset="image.avif" type="image/avif">
    <source srcset="image.webp" type="image/webp">
    <img src="image.jpg" alt="描述">
</picture>
```

#### 图片压缩工具

| 工具 | 格式 | 压缩率 |
|------|------|-------|
| squoosh.app | AVIF/WebP | 50-80% |
| imagemin | JPEG/PNG | 30-50% |
| tinypng | PNG | 50-70% |

### 3. 懒加载

#### 图片懒加载

```html
<!-- 原生懒加载 -->
<img src="image.jpg" alt="描述" loading="lazy">

<!-- JavaScript 懒加载（兼容旧浏览器） -->
<img data-src="image.jpg" alt="描述" class="lazy">

<script>
    const lazyImages = document.querySelectorAll('img.lazy');
    const imageObserver = new IntersectionObserver((entries, observer) => {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                const img = entry.target;
                img.src = img.dataset.src;
                img.classList.remove('lazy');
                observer.unobserve(img);
            }
        });
    });
    lazyImages.forEach(img => imageObserver.observe(img));
</script>
```

#### 代码分割

```javascript
// 按需加载 highlight.js
if (document.querySelector('pre code')) {
    const hljs = await import('https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js');
    hljs.highlightAll();
}

// 按需加载 KaTeX
if (document.querySelector('.math-block')) {
    const katex = await import('https://cdn.jsdelivr.net/npm/katex@0.12.0/dist/katex.min.js');
    // 渲染数学公式
}
```

### 4. 预加载关键资源

```html
<head>
    <!-- 预加载字体 -->
    <link rel="preload" href="https://fonts.gstatic.com/s/inter/v12/UcCO3FwrK3iLTeHuS_fvQtMwCp50KnMw2boKoduKmMEVuLyfAZ9hjp-Ek-_EeA.woff2" as="font" crossorigin>
    
    <!-- 预加载关键 CSS -->
    <link rel="preload" href="/style.css" as="style">
    <link rel="stylesheet" href="/style.css">
    
    <!-- 预加载关键 JS -->
    <link rel="modulepreload" href="/app.js">
    
    <!-- DNS 预解析 -->
    <link rel="dns-prefetch" href="https://fonts.googleapis.com">
    <link rel="dns-prefetch" href="https://cdnjs.cloudflare.com">
</head>
```

## 13.4.4 运行时性能优化

### 1. 防抖和节流

```javascript
// 防抖（debounce）- 用于搜索框
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// 节流（throttle）- 用于滚动事件
function throttle(func, limit) {
    let inThrottle;
    return function(...args) {
        if (!inThrottle) {
            func.apply(this, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

// 使用示例
const searchInput = document.getElementById('search-input');
searchInput.addEventListener('input', debounce((e) => {
    search(e.target.value);
}, 300));

window.addEventListener('scroll', throttle(() => {
    updateBackToTopButton();
}, 100));
```

### 2. 虚拟滚动（长列表优化）

```javascript
// 虚拟滚动实现
class VirtualList {
    constructor(container, items, itemHeight) {
        this.container = container;
        this.items = items;
        this.itemHeight = itemHeight;
        this.visibleCount = Math.ceil(container.clientHeight / itemHeight) + 2;
        
        this.container.style.height = `${items.length * itemHeight}px`;
        this.container.addEventListener('scroll', () => this.render());
        this.render();
    }
    
    render() {
        const scrollTop = this.container.scrollTop;
        const startIndex = Math.floor(scrollTop / this.itemHeight);
        const endIndex = Math.min(startIndex + this.visibleCount, this.items.length);
        
        // 只渲染可见区域的项目
        const visibleItems = this.items.slice(startIndex, endIndex);
        // ... 渲染逻辑
    }
}
```

### 3. Web Workers（后台计算）

```javascript
// 将搜索索引加载移到 Web Worker
// worker.js
self.onmessage = function(e) {
    const { articles, query } = e.data;
    const fuse = new Fuse(articles, { keys: ['title', 'content'] });
    const results = fuse.search(query);
    self.postMessage(results);
};

// 主线程
const worker = new Worker('worker.js');
worker.postMessage({ articles: allArticles, query: 'zig' });
worker.onmessage = (e) => {
    displayResults(e.data);
};
```

## 13.4.5 性能监控

### 1. Core Web Vitals

```javascript
// 使用 web-vitals 库监控核心指标
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

getCLS(console.log);
getFID(console.log);
getFCP(console.log);
getLCP(console.log);
getTTFB(console.log);
```

### 指标说明

| 指标 | 全称 | 目标值 | 说明 |
|------|------|-------|------|
| FCP | First Contentful Paint | < 1.8s | 首次内容绘制 |
| LCP | Largest Contentful Paint | < 2.5s | 最大内容绘制 |
| FID | First Input Delay | < 100ms | 首次输入延迟 |
| TTI | Time to Interactive | < 3.8s | 可交互时间 |
| CLS | Cumulative Layout Shift | < 0.1 | 累积布局偏移 |

### 2. 构建时间监控

```zig
// 在 build.zig 中添加性能监控
const start_time = std.time.milliTimestamp();

// ... 构建逻辑 ...

const end_time = std.time.milliTimestamp();
const duration = end_time - start_time;

std.debug.print("Build completed in {d}ms\n", .{duration});

// 按阶段统计
const load_posts_time = std.time.milliTimestamp() - start_time;
std.debug.print("  - Load posts: {d}ms\n", .{load_posts_time});
```

### 3. 性能预算

```json
// performance-budget.json
{
    "budget": [
        {
            "resourceType": "document",
            "maximumSize": 50
        },
        {
            "resourceType": "stylesheet",
            "maximumSize": 30
        },
        {
            "resourceType": "image",
            "maximumSize": 200
        },
        {
            "resourceType": "script",
            "maximumSize": 100
        },
        {
            "resourceType": "total",
            "maximumSize": 500
        }
    ]
}
```

## 13.4.6 性能检查清单

### 构建时优化

- [ ] 启用增量构建
- [ ] 并行处理独立任务
- [ ] 缓存模板解析结果
- [ ] 压缩 CSS 和 JS
- [ ] 优化图片格式和大小

### 加载时优化

- [ ] 使用 CDN 分发资源
- [ ] 预加载关键资源
- [ ] 懒加载非关键资源
- [ ] 启用 Gzip/Brotli 压缩
- [ ] 设置合理的缓存头

### 运行时优化

- [ ] 防抖/节流频繁事件
- [ ] 虚拟滚动长列表
- [ ] 后台计算使用 Web Workers
- [ ] 避免强制同步布局
- [ ] 使用 requestAnimationFrame 动画

### 监控与测试

- [ ] 监控 Core Web Vitals
- [ ] 使用 Lighthouse 审计
- [ ] 设置性能预算
- [ ] 定期性能回归测试

## 13.4.7 Lighthouse 优化建议

### 常见审计项

| 审计项 | 优化方法 | 预期提升 |
|--------|---------|---------|
| 未压缩图片 | 使用 WebP/AVIF 格式 | -50% 图片大小 |
| 未使用 CSS | 移除未使用样式 | -30% CSS 大小 |
| 渲染阻塞资源 | 异步加载 CSS/JS | +20% FCP |
| 服务器响应慢 | 启用缓存、CDN | -50% TTFB |
| 主线程工作过长 | 代码分割、Web Workers | -40% TTI |

### Lighthouse 运行命令

```bash
# 使用 Chrome DevTools CLI
lighthouse http://localhost:8000 --output=html --output-path=report.html

# 使用 GitHub Actions
# .github/workflows/lighthouse.yml
```

## 13.4.8 性能优化案例

### 案例 1：构建时间从 10s 降至 2s

```zig
// 优化前：串行处理
for (posts) |post| {
    try self.generatePostPage(post);  // 每篇 100ms，100 篇 = 10s
}

// 优化后：并行处理
var thread_pool = try std.Thread.Pool.init(.{ .n_jobs = 8 });
for (posts) |post| {
    try thread_pool.spawn(generatePostPageTask, .{post});
}
// 8 个线程并行，100 篇 ≈ 1.25s
```

### 案例 2：首屏加载从 3s 降至 1s

```html
<!-- 优化前：阻塞渲染 -->
<link rel="stylesheet" href="style.css">
<script src="app.js"></script>

<!-- 优化后：非阻塞加载 -->
<link rel="preload" href="style.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<script src="app.js" defer></script>
<noscript><link rel="stylesheet" href="style.css"></noscript>
```

## 相关文档

- [13.1 编写博客文章指南](./013-1-writing-posts.md) - 文章格式优化
- [13.3 集成搜索功能](./013-3-search-integration.md) - 搜索性能优化
- [6.2 std.Build API](../part-06-build-system/006-2-std-build-api.md) - 构建系统基础

---

*本文档介绍了 my-blog 的性能优化指南，包括构建性能优化（增量构建、并行处理）、页面加载优化（资源压缩、懒加载）和运行时优化（防抖节流、虚拟滚动）。*
