# 13.2 添加 RSS 订阅

> **RSS 版本**: RSS 2.0  
> **输出文件**: `zig-out/blog/rss.xml`  
> **MIME 类型**: `application/rss+xml`

## 13.2.1 RSS 概述

### 什么是 RSS？

RSS（Really Simple Syndication）是一种用于发布经常更新内容的标准化格式，如博客文章、新闻标题、音频和视频。

### RSS 的作用

| 功能 | 说明 |
|------|------|
| 内容聚合 | 读者可通过 RSS 阅读器订阅博客更新 |
| 自动推送 | 新文章发布时自动通知订阅者 |
| 跨平台 | 支持各种 RSS 阅读器和聚合服务 |
| SEO 优化 | 搜索引擎可抓取 RSS 内容 |

### RSS 阅读器示例

- **桌面端**: Feedly, Inoreader, NetNewsWire
- **移动端**: Reeder, Feedly, Inoreader
- **Web 端**: Feedly, Inoreader, The Old Reader

## 13.2.2 RSS 2.0 标准格式

### 完整 RSS 文件结构

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <!-- 频道元数据 -->
    <title>YongchaoLi's Blog</title>
    <link>https://example.com</link>
    <description>A blog about Zig and systems programming</description>
    <language>en-us</language>
    <lastBuildDate>Mon, 15 Jan 2024 10:30:00 GMT</lastBuildDate>
    <atom:link href="https://example.com/rss.xml" rel="self" type="application/rss+xml"/>
    
    <!-- 文章条目 -->
    <item>
      <title>我的第一篇博客文章</title>
      <link>https://example.com/2024/01/15/my-first-post.html</link>
      <description>这是我的第一篇博客文章，使用 Zig 编写的静态博客生成器生成。</description>
      <pubDate>Mon, 15 Jan 2024 10:30:00 GMT</pubDate>
      <guid isPermaLink="true">https://example.com/2024/01/15/my-first-post.html</guid>
      <category>zig</category>
      <category>tutorial</category>
    </item>
    
    <item>
      <title>Zig 基础语法</title>
      <link>https://example.com/2024/01/20/zig-basics.html</link>
      <description>介绍 Zig 语言的基础语法和特性。</description>
      <pubDate>Sat, 20 Jan 2024 14:00:00 GMT</pubDate>
      <guid isPermaLink="true">https://example.com/2024/01/20/zig-basics.html</guid>
      <category>zig</category>
      <category>programming</category>
    </item>
  </channel>
</rss>
```

### 必需元素

| 元素 | 位置 | 说明 |
|------|------|------|
| `<title>` | channel | 博客标题 |
| `<link>` | channel | 博客主页 URL |
| `<description>` | channel | 博客描述 |
| `<title>` | item | 文章标题 |
| `<link>` | item | 文章 URL |
| `<description>` | item | 文章摘要或全文 |
| `<pubDate>` | item | 文章发布日期 |
| `<guid>` | item | 文章唯一标识符 |

### 可选元素

| 元素 | 位置 | 说明 |
|------|------|------|
| `<language>` | channel | 语言代码（如 en-us, zh-cn） |
| `<lastBuildDate>` | channel | RSS 最后更新时间 |
| `<atom:link>` | channel | RSS 自我引用链接 |
| `<category>` | item/channel | 分类/标签 |
| `<author>` | item | 作者邮箱 |
| `<image>` | channel | 博客 Logo |

## 13.2.3 在 my-blog 中实现 RSS

### 实现思路

由于 my-blog 的原始版本可能不包含 RSS 功能，我们需要扩展 `Blogger.zig` 来生成 RSS 文件。

### 1. 添加 RSS 生成函数

```zig
// 在 Blogger.zig 中添加
pub fn generateRss(self: *Blogger, posts: []Post) !void {
    const rss_path = try std.fs.path.join(self.allocator, &.{ self.dest_dir, "rss.xml" });
    defer self.allocator.free(rss_path);
    
    var rss_file = try std.fs.cwd().createFile(rss_path, .{});
    defer rss_file.close();
    
    const writer = rss_file.writer();
    
    // 写入 RSS 头部
    try writer.writeAll("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
    try writer.writeAll("<rss version=\"2.0\" xmlns:atom=\"http://www.w3.org/2005/Atom\">\n");
    try writer.writeAll("  <channel>\n");
    
    // 写入频道信息
    try writer.writeAll("    <title>YongchaoLi's Blog</title>\n");
    try writer.writeAll("    <link>https://example.com</link>\n");
    try writer.writeAll("    <description>A blog about Zig and systems programming</description>\n");
    try writer.writeAll("    <language>en-us</language>\n");
    
    // 写入最后更新时间
    const last_build_date = try self.formatRssDate(std.time.timestamp());
    try writer.writeAll("    <lastBuildDate>{s}</lastBuildDate>\n", .{last_build_date});
    
    // 写入自我引用链接
    try writer.writeAll("    <atom:link href=\"https://example.com/rss.xml\" rel=\"self\" type=\"application/rss+xml\"/>\n");
    
    // 写入文章（最近 20 篇）
    const max_items = @min(posts.len, 20);
    for (posts[0..max_items]) |post| {
        try self.writeRssItem(writer, post);
    }
    
    // 关闭标签
    try writer.writeAll("  </channel>\n");
    try writer.writeAll("</rss>\n");
}
```

### 2. 文章条目生成

```zig
fn writeRssItem(self: *Blogger, writer: anytype, post: Post) !void {
    try writer.writeAll("    <item>\n");
    
    // 标题（需要 XML 转义）
    try writer.writeAll("      <title>{s}</title>\n", .{try self.xmlEscape(post.title)});
    
    // 文章链接
    const post_url = try self.buildPostUrl(post);
    try writer.writeAll("      <link>{s}</link>\n", .{post_url});
    
    // 描述（取前 500 字符）
    const description = post.content[0..@min(post.content.len, 500)];
    try writer.writeAll("      <description>{s}</description>\n", .{try self.xmlEscape(description)});
    
    // 发布日期
    const pub_date = try self.formatRssDate(post.date_time);
    try writer.writeAll("      <pubDate>{s}</pubDate>\n", .{pub_date});
    
    // 唯一标识符
    try writer.writeAll("      <guid isPermaLink=\"true\">{s}</guid>\n", .{post_url});
    
    // 标签
    if (post.tags.len > 0) {
        // 解析并写入每个标签
        // ...
    }
    
    try writer.writeAll("    </item>\n");
}
```

### 3. 日期格式化

```zig
fn formatRssDate(self: *Blogger, timestamp: i64) ![]u8 {
    // RSS 日期格式：Mon, 15 Jan 2024 10:30:00 GMT
    const date = std.time.epoch.EpochSeconds{ .secs = @intCast(timestamp) };
    const epoch_day = date.getEpochDay();
    const year_day = epoch_day.getYearDay();
    const day_of_week = epoch_day.getDayOfWeek();
    
    const days = [_][]const u8{ "Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat" };
    const months = [_][]const u8{ "Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec" };
    
    return try std.fmt.allocPrint(self.allocator, "{s}, {d} {s} {d} 00:00:00 GMT", .{
        days[@intFromEnum(day_of_week)],
        year_day.day_index + 1,
        months[year_day.month.index()],
        year_day.year,
    });
}
```

### 4. XML 转义

```zig
fn xmlEscape(self: *Blogger, input: []const u8) ![]u8 {
    var result = std.ArrayList(u8).init(self.allocator);
    errdefer result.deinit();
    
    for (input) |char| {
        switch (char) {
            '&' => try result.appendSlice("&amp;"),
            '<' => try result.appendSlice("&lt;"),
            '>' => try result.appendSlice("&gt;"),
            '"' => try result.appendSlice("&quot;"),
            '\'' => try result.appendSlice("&apos;"),
            else => try result.append(char),
        }
    }
    
    return result.toOwnedSlice();
}
```

### 5. 集成到主流程

```zig
// 在 main.zig 或 Blogger.zig 的生成流程中
pub fn generate(self: *Blogger) !void {
    // 1. 加载文章
    const posts = try self.loadPosts();
    defer {
        for (posts) |*post| {
            post.deinit(self.allocator);
        }
        self.allocator.free(posts);
    }
    
    // 2. 生成首页
    try self.generateIndex(posts);
    
    // 3. 生成文章页
    try self.generatePostPages(posts);
    
    // 4. 生成标签页
    try self.generateTagPages(posts);
    
    // 5. 生成 RSS（新增）
    try self.generateRss(posts);
    
    std.debug.print("Generated RSS feed at {s}/rss.xml\n", .{self.dest_dir});
}
```

## 13.2.4 在模板中添加 RSS 链接

### 在 layout.hbs 中添加

```html
<head>
    <!-- 其他 meta 标签 -->
    <link rel="alternate" type="application/rss+xml" title="YongchaoLi's Blog" href="/rss.xml">
</head>
```

### 在导航栏添加订阅按钮

```html
<nav class="navbar">
    <div class="navbar-content">
        <a href="/index.html" class="navbar-brand">YongchaoLi's Blog</a>
        <div class="navbar-menu">
            <a href="/about.html">About</a>
            <a href="/rss.xml" class="rss-link" aria-label="RSS Feed">
                <svg viewBox="0 0 24 24" fill="currentColor">
                    <path d="M6 18c0-3.313 2.687-6 6-6s6 2.687 6 6M6 6c6.627 0 12 5.373 12 12M6 12c3.313 0 6 2.687 6 6"/>
                </svg>
            </a>
        </div>
    </div>
</nav>
```

## 13.2.5 RSS 样式优化

### 添加 RSS 图标样式

```css
.rss-link {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    color: var(--accent);
    transition: color 0.2s ease;
}

.rss-link:hover {
    color: var(--accent-hover);
}

.rss-link svg {
    width: 20px;
    height: 20px;
}
```

## 13.2.6 测试 RSS 文件

### 验证 XML 格式

```bash
# 使用 xmllint 验证
xmllint --noout zig-out/blog/rss.xml

# 在线验证工具
# https://validator.w3.org/feed/
# https://www.feedvalidator.org/
```

### 在浏览器中预览

```bash
# 启动本地服务器
cd zig-out/blog
python -m http.server 8000

# 访问 RSS 文件
# http://localhost:8000/rss.xml
```

### 在 RSS 阅读器中测试

1. 复制 `rss.xml` 的 URL
2. 在 RSS 阅读器中添加订阅
3. 验证文章标题、链接、日期正确

## 13.2.7 高级功能

### 全文 vs 摘要

```zig
// 选项 1：输出全文（适合技术博客）
const description = post.content;

// 选项 2：输出摘要（前 500 字符）
const description = post.content[0..@min(post.content.len, 500)];

// 选项 3：使用 frontmatter 中的 description 字段
const description = post.description orelse post.content[0..500];
```

### 包含图片

```xml
<item>
    <title>带图片的文章</title>
    <description>
        &lt;img src="https://example.com/image.jpg" alt="封面图"&gt;
        &lt;p&gt;文章摘要...&lt;/p&gt;
    </description>
    <enclosure url="https://example.com/image.jpg" type="image/jpeg" length="12345"/>
</item>
```

### 支持 Atom 格式

Atom 是 RSS 的替代标准，格式略有不同：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
    <title>YongchaoLi's Blog</title>
    <link href="https://example.com"/>
    <updated>2024-01-15T10:30:00Z</updated>
    
    <entry>
        <title>文章标题</title>
        <link href="https://example.com/post.html"/>
        <updated>2024-01-15T10:30:00Z</updated>
        <summary>文章摘要</summary>
    </entry>
</feed>
```

## 13.2.8 常见问题

### Q: RSS 文件不更新？

确保在每次生成博客时都调用 `generateRss()` 函数。

### Q: 特殊字符显示错误？

确保所有 XML 内容都经过转义：
- `&` → `&amp;`
- `<` → `&lt;`
- `>` → `&gt;`
- `"` → `&quot;`

### Q: 日期格式不正确？

RSS 要求 RFC 822 格式：`Mon, 15 Jan 2024 10:30:00 GMT`

### Q: 如何限制 RSS 中的文章数量？

```zig
// 只输出最近 20 篇
const max_items = @min(posts.len, 20);
for (posts[0..max_items]) |post| {
    try self.writeRssItem(writer, post);
}
```

## 13.2.9 RSS 提交指南

### 提交到搜索引擎

```html
<!-- 在 sitemap.xml 中包含 RSS -->
<sitemap>
    <loc>https://example.com/rss.xml</loc>
</sitemap>
```

### 提交到聚合平台

| 平台 | 提交方式 |
|------|---------|
| Google | 通过 Search Console 自动发现 |
| Feedly | 用户手动添加 URL |
| Inoreader | 用户手动添加 URL |
| Bloglines | 提交到目录 |

## 相关文档

- [13.1 编写博客文章指南](./013-1-writing-posts.md) - 文章格式和 frontmatter
- [13.3 集成搜索功能](./013-3-search-integration.md) - 站内搜索实现
- [7.2 Blogger.zig 核心模块](../part-07-core-modules/007-2-blogger-struct.md) - 核心生成逻辑

---

*本文档介绍了如何在 my-blog 中实现 RSS 订阅功能，包括 RSS 2.0 标准格式、生成逻辑、XML 转义和测试验证。*
