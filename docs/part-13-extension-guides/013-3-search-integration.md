# 13.3 集成搜索功能

> **实现方案**: 基于 Fuse.js 的客户端全文搜索  
> **索引文件**: `zig-out/blog/search-index.json`  
> **搜索页面**: `zig-out/blog/search.html`

## 13.3.1 搜索方案对比

### 可选方案

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **客户端搜索** (Fuse.js) | 无需服务器，部署简单 | 索引全加载，大数据量慢 | 小型博客（<1000 篇） |
| **服务器搜索** (Algolia) | 速度快，功能强大 | 需要 API 密钥，有费用 | 中大型网站 |
| **静态搜索** (PageFind) | 静态生成，无需 JS | 需要构建步骤 | 中型网站 |
| **Google 搜索** | 免费，准确 | 需要外部服务 | 任何场景 |

### 推荐方案：Fuse.js

my-blog 推荐使用 **Fuse.js** 实现客户端搜索：

| 特性 | 说明 |
|------|------|
| 轻量级 | ~5KB (gzipped) |
| 模糊搜索 | 支持拼写错误容错 |
| 全文搜索 | 搜索标题、内容、标签 |
| 无需服务器 | 纯 JavaScript 实现 |
| 配置灵活 | 可调整权重、阈值 |

## 13.3.2 搜索架构

### 数据流

```
┌─────────────────────────────────────────────────────────┐
│                    搜索功能架构                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 构建时                                               │
│     Blogger.zig 生成 search-index.json                  │
│     ┌──────────────┐                                    │
│     │ 所有文章数据  │ → search-index.json               │
│     └──────────────┘                                    │
│                                                         │
│  2. 页面加载时                                           │
│     search.html 加载 Fuse.js + 索引                     │
│     ┌──────────────┐                                    │
│     │ search-index │ → Fuse.js → 内存索引              │
│     └──────────────┘                                    │
│                                                         │
│  3. 用户搜索时                                           │
│     输入查询 → Fuse.js 搜索 → 显示结果                  │
│     ┌──────────────┐                                    │
│     │ 用户输入     │ → 实时过滤 → 结果列表              │
│     └──────────────┘                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 文件结构

```
zig-out/blog/
├── search.html           # 搜索页面
├── search-index.json     # 搜索索引（自动生成）
├── style.css             # 包含搜索样式
└── ...
```

## 13.3.3 生成搜索索引

### 1. 扩展 Blogger.zig

```zig
// 在 Blogger.zig 中添加搜索索引生成
pub fn generateSearchIndex(self: *Blogger, posts: []Post) !void {
    const index_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, "search-index.json" });
    defer self.allocator.free(index_path);
    
    var index_file = try std.fs.cwd().createFile(index_path, .{});
    defer index_file.close();
    
    const writer = index_file.writer();
    
    // 写入 JSON 数组
    try writer.writeAll("[\n");
    
    for (posts, 0..) |post, i| {
        if (i > 0) try writer.writeAll(",\n");
        
        // 写入文章数据
        try writer.writeAll("  {\n");
        try writer.writeAll("    \"title\": \"{s}\",\n", .{try self.escapeJson(post.title)});
        try writer.writeAll("    \"url\": \"{s}\",\n", .{try self.buildPostUrl(post)});
        try writer.writeAll("    \"date\": \"{s}\",\n", .{post.date_time});
        try writer.writeAll("    \"tags\": [{s}],\n", .{try self.buildTagsJson(post.tags)});
        try writer.writeAll("    \"content\": \"{s}\"\n", .{try self.escapeJson(self.truncateContent(post.content))});
        try writer.writeAll("  }");
    }
    
    try writer.writeAll("\n]\n");
    
    std.debug.print("Generated search index at {s}\n", .{index_path});
}
```

### 2. 辅助函数

```zig
// JSON 转义
fn escapeJson(self: *Blogger, input: []const u8) ![]u8 {
    var result = std.ArrayList(u8).init(self.allocator);
    errdefer result.deinit();
    
    for (input) |char| {
        switch (char) {
            '"' => try result.appendSlice("\\\""),
            '\\' => try result.appendSlice("\\\\"),
            '\n' => try result.appendSlice("\\n"),
            '\r' => try result.appendSlice("\\r"),
            '\t' => try result.appendSlice("\\t"),
            else => try result.append(char),
        }
    }
    
    return result.toOwnedSlice();
}

// 截断内容（只保留纯文本前 500 字符）
fn truncateContent(self: *Blogger, content: []const u8) []const u8 {
    // 移除 HTML 标签
    var text = self.stripHtmlTags(content);
    
    // 截断
    const max_len = @min(text.len, 500);
    return text[0..max_len];
}

// 构建标签 JSON 数组
fn buildTagsJson(self: *Blogger, tags: []const u8) ![]u8 {
    // 解析逗号分隔的标签
    var result = std.ArrayList(u8).init(self.allocator);
    var iter = std.mem.splitScalar(u8, tags, ',');
    var first = true;
    
    while (iter.next()) |tag| {
        if (!first) try result.appendSlice(", ");
        try result.appendSlice("\"");
        try result.appendSlice(try self.escapeJson(std.mem.trim(u8, tag, " \t\n")));
        try result.appendSlice("\"");
        first = false;
    }
    
    return result.toOwnedSlice();
}
```

### 3. 集成到生成流程

```zig
pub fn generate(self: *Blogger) !void {
    const posts = try self.loadPosts();
    defer self.cleanupPosts(posts);
    
    // 生成各种页面
    try self.generateIndex(posts);
    try self.generatePostPages(posts);
    try self.generateTagPages(posts);
    
    // 生成搜索索引（新增）
    try self.generateSearchIndex(posts);
    
    // 复制搜索页面模板
    try self.copySearchPage();
}
```

## 13.3.4 创建搜索页面

### search.html 模板

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Search - YongchaoLi's Blog</title>
    <link rel="stylesheet" href="/style.css">
    <style>
        /* 搜索页面专用样式 */
        .search-container {
            max-width: 800px;
            margin: 48px auto;
            padding: 0 24px;
        }
        
        .search-box {
            width: 100%;
            padding: 16px 24px;
            font-size: 18px;
            border: 2px solid var(--border);
            border-radius: 8px;
            background-color: var(--bg-secondary);
            color: var(--text-primary);
        }
        
        .search-box:focus {
            outline: none;
            border-color: var(--accent);
        }
        
        .search-results {
            margin-top: 32px;
        }
        
        .search-result-item {
            padding: 24px;
            border-bottom: 1px solid var(--border);
        }
        
        .search-result-item:last-child {
            border-bottom: none;
        }
        
        .search-result-title {
            font-size: var(--font-lg);
            font-weight: 600;
            color: var(--text-primary);
            margin-bottom: 8px;
        }
        
        .search-result-title a {
            color: var(--accent);
            text-decoration: none;
        }
        
        .search-result-title a:hover {
            text-decoration: underline;
        }
        
        .search-result-date {
            font-size: var(--font-sm);
            color: var(--text-secondary);
            margin-bottom: 8px;
        }
        
        .search-result-excerpt {
            color: var(--text-secondary);
            line-height: 1.6;
        }
        
        .search-result-tags {
            margin-top: 12px;
        }
        
        .search-no-results {
            text-align: center;
            padding: 48px 24px;
            color: var(--text-secondary);
        }
        
        .search-loading {
            text-align: center;
            padding: 48px;
            color: var(--text-secondary);
        }
    </style>
</head>
<body>
    <nav class="navbar">
        <div class="navbar-content">
            <a href="/index.html" class="navbar-brand">YongchaoLi's Blog</a>
            <div class="navbar-menu">
                <a href="/about.html">About</a>
            </div>
        </div>
    </nav>
    
    <div class="search-container">
        <h1>Search</h1>
        <input type="text" class="search-box" id="search-input" placeholder="Search articles..." autofocus>
        <div class="search-results" id="search-results">
            <div class="search-loading">Loading search index...</div>
        </div>
    </div>
    
    <!-- 加载 Fuse.js -->
    <script src="https://cdn.jsdelivr.net/npm/fuse.js@7.0.0/dist/fuse.min.js"></script>
    <script>
        // 搜索功能实现
        let fuse = null;
        let allArticles = [];
        
        // 加载搜索索引
        async function loadSearchIndex() {
            try {
                const response = await fetch('/search-index.json');
                allArticles = await response.json();
                
                // 初始化 Fuse.js
                fuse = new Fuse(allArticles, {
                    keys: [
                        { name: 'title', weight: 0.5 },
                        { name: 'content', weight: 0.3 },
                        { name: 'tags', weight: 0.2 }
                    ],
                    threshold: 0.4,  // 模糊搜索阈值（0=精确，1=任意）
                    includeMatches: true,
                    minMatchCharLength: 2,
                    limit: 20
                });
                
                document.getElementById('search-results').innerHTML = '';
            } catch (error) {
                console.error('Failed to load search index:', error);
                document.getElementById('search-results').innerHTML = 
                    '<div class="search-no-results">Failed to load search index.</div>';
            }
        }
        
        // 执行搜索
        function search(query) {
            if (!fuse || !query.trim()) {
                document.getElementById('search-results').innerHTML = '';
                return;
            }
            
            const results = fuse.search(query);
            displayResults(results);
        }
        
        // 显示搜索结果
        function displayResults(results) {
            const container = document.getElementById('search-results');
            
            if (results.length === 0) {
                container.innerHTML = '<div class="search-no-results">No results found.</div>';
                return;
            }
            
            let html = '';
            results.forEach(result => {
                const item = result.item;
                const excerpt = item.content.substring(0, 200) + '...';
                const tags = item.tags.map(tag => `<span class="tag">${tag}</span>`).join('');
                
                html += `
                    <div class="search-result-item">
                        <div class="search-result-title">
                            <a href="${item.url}">${highlightMatch(item.title, '${result.item.title}')}</a>
                        </div>
                        <div class="search-result-date">${item.date}</div>
                        <div class="search-result-excerpt">${excerpt}</div>
                        <div class="search-result-tags">${tags}</div>
                    </div>
                `;
            });
            
            container.innerHTML = html;
        }
        
        // 高亮匹配词
        function highlightMatch(text, original) {
            // 简单实现：返回原文本
            return text;
        }
        
        // 事件监听
        document.addEventListener('DOMContentLoaded', loadSearchIndex);
        
        document.getElementById('search-input').addEventListener('input', (e) => {
            search(e.target.value);
        });
        
        // 支持 URL 参数搜索
        const urlParams = new URLSearchParams(window.location.search);
        const query = urlParams.get('q');
        if (query) {
            document.getElementById('search-input').value = query;
            // 索引加载后会自动搜索
        }
    </script>
</body>
</html>
```

## 13.3.5 Fuse.js 配置选项

### 核心配置

```javascript
const fuse = new Fuse(articles, {
    // 搜索键及其权重
    keys: [
        { name: 'title', weight: 0.5 },    // 标题权重最高
        { name: 'content', weight: 0.3 },  // 内容权重中等
        { name: 'tags', weight: 0.2 }      // 标签权重较低
    ],
    
    // 模糊搜索阈值
    threshold: 0.4,  // 0=精确匹配，1=完全模糊
    
    // 最小匹配字符数
    minMatchCharLength: 2,
    
    // 最大结果数
    limit: 20,
    
    // 是否包含匹配位置
    includeMatches: true,
    
    // 是否高亮匹配
    includeScore: true,
    
    // 排序规则
    shouldSort: true,
    
    // 是否区分大小写
    caseSensitive: false
});
```

### 阈值调优

| threshold 值 | 效果 | 适用场景 |
|-------------|------|---------|
| 0.0 | 精确匹配 | 代码搜索、精确查询 |
| 0.3 | 较严格 | 技术术语搜索 |
| 0.4 | 平衡（推荐） | 一般博客搜索 |
| 0.6 | 较宽松 | 容忍拼写错误 |
| 1.0 | 完全模糊 | 模糊记忆搜索 |

## 13.3.6 搜索优化技巧

### 1. 索引大小优化

```zig
// 只索引必要字段
fn generateSearchIndex(self: *Blogger, posts: []Post) !void {
    for (posts) |post| {
        // 截断内容，减少索引大小
        const truncated = self.truncateContent(post.content, 500);
        
        // 移除 HTML 标签
        const plain_text = self.stripHtmlTags(truncated);
        
        // 只存储纯文本
        try self.writeIndexEntry(writer, post.title, plain_text, post.tags);
    }
}
```

### 2. 搜索结果高亮

```javascript
// 高亮匹配词
function highlightMatch(text, query) {
    const regex = new RegExp(`(${query})`, 'gi');
    return text.replace(regex, '<mark>$1</mark>');
}

// CSS 样式
mark {
    background-color: rgba(88, 166, 255, 0.3);
    padding: 2px 4px;
    border-radius: 2px;
}
```

### 3. 搜索结果分页

```javascript
const RESULTS_PER_PAGE = 10;
let currentPage = 1;

function search(query, page = 1) {
    const results = fuse.search(query);
    const start = (page - 1) * RESULTS_PER_PAGE;
    const end = start + RESULTS_PER_PAGE;
    displayResults(results.slice(start, end));
    renderPagination(results.length, page);
}
```

### 4. 搜索历史

```javascript
// 保存搜索历史到 localStorage
function saveSearchHistory(query) {
    let history = JSON.parse(localStorage.getItem('searchHistory') || '[]');
    history = [query, ...history.filter(q => q !== query)].slice(0, 10);
    localStorage.setItem('searchHistory', JSON.stringify(history));
}

// 显示搜索历史
function showSearchHistory() {
    const history = JSON.parse(localStorage.getItem('searchHistory') || '[]');
    // 显示为建议列表
}
```

## 13.3.7 搜索页面集成

### 在导航栏添加搜索入口

```html
<nav class="navbar">
    <div class="navbar-content">
        <a href="/index.html" class="navbar-brand">YongchaoLi's Blog</a>
        <div class="navbar-menu">
            <a href="/about.html">About</a>
            <a href="/search.html" class="search-icon" aria-label="Search">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <circle cx="11" cy="11" r="8"/>
                    <path d="M21 21l-4.35-4.35"/>
                </svg>
            </a>
        </div>
    </div>
</nav>
```

### 添加快捷键

```javascript
// 按 '/' 键聚焦搜索框
document.addEventListener('keydown', (e) => {
    if (e.key === '/' && document.activeElement.tagName !== 'INPUT') {
        e.preventDefault();
        document.getElementById('search-input').focus();
    }
});

// 按 Escape 键清除搜索
document.addEventListener('keydown', (e) => {
    if (e.key === 'Escape' && document.activeElement.id === 'search-input') {
        document.getElementById('search-input').value = '';
        document.getElementById('search-results').innerHTML = '';
    }
});
```

## 13.3.8 性能优化

### 索引压缩

```zig
// 使用更紧凑的 JSON 格式
const json = try std.json.stringifyAlloc(allocator, index_data, .{
    .whitespace = .minified,  // 无空格
});
```

### 懒加载索引

```javascript
// 用户聚焦搜索框时才加载索引
let fuse = null;

document.getElementById('search-input').addEventListener('focus', async () => {
    if (!fuse) {
        await loadSearchIndex();
    }
});
```

### 预加载索引

```html
<!-- 在首页预加载提示 -->
<link rel="prefetch" href="/search-index.json">
```

## 13.3.9 测试搜索功能

### 测试用例

| 测试场景 | 预期结果 |
|---------|---------|
| 搜索完整标题 | 显示对应文章 |
| 搜索部分内容 | 显示包含该内容的文章 |
| 搜索标签 | 显示带有该标签的文章 |
| 拼写错误 | 仍能匹配正确结果 |
| 搜索不存在的词 | 显示"无结果" |
| 空搜索 | 清空结果列表 |

### 性能测试

```javascript
// 测试搜索速度
console.time('search');
const results = fuse.search('zig');
console.timeEnd('search');  // 应 < 100ms
```

## 相关文档

- [13.1 编写博客文章指南](./013-1-writing-posts.md) - 文章格式
- [13.2 添加 RSS 订阅](./013-2-rss-feed.md) - RSS 生成
- [13.4 性能优化指南](./013-4-performance-optimization.md) - 构建和加载优化

---

*本文档介绍了如何在 my-blog 中集成客户端搜索功能，使用 Fuse.js 实现全文搜索、模糊匹配和结果高亮。*
