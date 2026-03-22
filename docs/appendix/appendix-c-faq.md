# 附录 C: 常见问题解答（FAQ）

> **适用范围**: my-blog 项目及 Zig 0.15.2  
> **最后更新**: 2024-03-21

## C.1 入门问题

### Q1: Zig 适合初学者吗？

**A:** Zig 是一门系统编程语言，相比 Python/JavaScript 等高级语言，学习曲线较陡峭。但 Zig 的设计哲学是"简单清晰"，没有隐藏控制流，适合想要理解底层原理的学习者。

**推荐学习路径：**
1. 先学习 C 语言基础（指针、内存）
2. 学习 Zig 基础语法（官方文档）
3. 从小项目开始实践
4. 阅读优秀开源项目代码

### Q2: my-blog 项目需要什么前置知识？

**A:** 建议具备以下知识：
- ✅ 基础编程概念（变量、循环、函数）
- ✅ Markdown 语法
- ✅ 命令行基本操作
- ✅ Git 版本控制（可选）

**不需要：**
- ❌ Zig 编程经验（可以在使用中学习）
- ❌ 系统编程经验

### Q3: 为什么选择 Zig 而不是其他语言？

| 对比项 | Zig | Rust | Go | C |
|--------|-----|------|-----|---|
| 学习曲线 | 中等 | 陡峭 | 平缓 | 中等 |
| 内存安全 | 手动 | 编译期保证 | GC | 手动 |
| 性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 编译速度 | 快 | 慢 | 快 | 快 |
| 生态系统 | 小 | 大 | 大 | 极大 |

**Zig 的优势：**
- 语法简单，没有隐藏控制流
- 编译速度快
- 与 C 语言互操作性极佳
- 编译期代码执行能力强大

## C.2 构建和运行

### Q4: 编译失败，提示 "unknown import"？

**A:** 这通常是依赖项未正确安装。

**解决方案：**
```bash
# 1. 确保子模块已初始化
git submodule update --init --recursive

# 2. 或重新克隆
git clone --recursive https://github.com/linychuo/my-blog.git

# 3. 检查 build.zig.zon 中的依赖
cat build.zig.zon
```

### Q5: 如何切换到不同的 Zig 版本？

**A:** 使用 zigup 管理 Zig 版本：

```bash
# 安装 zigup
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/marler8997/zigup/master/install.sh | sh

# 安装特定版本
zigup 0.15.2

# 切换版本
zigup use 0.15.2

# 查看已安装版本
zigup list
```

### Q6: 如何调试构建错误？

**A:** 使用详细输出模式：

```bash
# 显示详细编译信息
zig build --verbose

# 显示编译步骤
zig build --verbose-cimport

# 只编译不运行
zig build --prefix ./zig-out
```

### Q7: 如何自定义输出目录？

**A:** 使用命令行参数：

```bash
# 默认输出到 zig-out/blog
zig build run

# 自定义输出目录
zig build run -- --output ./public

# 或使用 build 选项
zig build -Doutput-dir=./public
```

## C.3 内容创作

### Q8: 如何添加新文章？

**A:** 按照以下步骤：

```bash
# 1. 创建文章文件
touch posts/2024-03-21-my-new-post.markdown

# 2. 编辑 frontmatter 和内容
cat > posts/2024-03-21-my-new-post.markdown << EOF
---
title: 我的新文章
date_time: 2024-03-21 10:00
tags:
  - zig
  - tutorial
---

# 正文内容

这里是文章内容...
EOF

# 3. 生成博客
zig build run

# 4. 查看输出
open zig-out/blog/2024/03/21/my-new-post.html
```

### Q9: Frontmatter 格式错误怎么办？

**A:** 检查以下常见问题：

```yaml
# ✅ 正确格式
---
title: 文章标题
date_time: 2024-03-21 10:00
tags:
  - tag1
  - tag2
---

# ❌ 常见错误
---
title: 文章标题
date_time: 2024/03/21  # 错误：分隔符应该是 -
tags: tag1, tag2       # 错误：应该是数组格式
---
```

**调试方法：**
```bash
# 查看详细错误信息
zig build run 2>&1 | grep -A 5 "frontmatter"
```

### Q10: 如何添加代码高亮？

**A:** my-blog 使用 highlight.js，支持自动语言检测：

````markdown
<!-- 推荐：明确指定语言 -->
```zig
const std = @import("std");
```

<!-- 也支持：自动检测 -->
```
fn hello() {
    print("Hello!");
}
```
````

**支持的语言：** Zig, C, C++, Rust, Python, JavaScript, Go, Java 等。

### Q11: 如何插入数学公式？

**A:** 使用 `$$` 包裹 LaTeX 公式：

```markdown
<!-- 行内公式 -->
质能方程是 $E = mc^2$。

<!-- 块级公式 -->
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

### Q12: 如何添加图片？

**A:** 使用标准 Markdown 语法：

```markdown
<!-- 相对路径 -->
![图片描述](../static/imgs/photo.jpg)

<!-- 绝对 URL -->
![图片描述](https://example.com/image.jpg)

<!-- 带标题 -->
![图片描述](image.jpg "图片标题")
```

**最佳实践：**
- 将图片放在 `static/imgs/` 目录
- 使用描述性的 alt 文本
- 考虑使用 WebP/AVIF 格式优化大小

## C.4 主题和样式

### Q13: 如何修改主题颜色？

**A:** 编辑 `static/style.css` 中的 CSS 变量：

```css
/* 暗色主题 */
:root {
    --bg-primary: #0d1117;  /* 修改主背景色 */
    --accent: #58a6ff;      /* 修改强调色 */
    --text-primary: #f0f6fc; /* 修改主文字颜色 */
}

/* 亮色主题 */
[data-theme="light"] {
    --bg-primary: #ffffff;
    --accent: #0969da;
    --text-primary: #1f2328;
}
```

### Q14: 如何禁用自动主题切换？

**A:** 修改 `templates/layout.hbs` 中的脚本：

```html
<!-- 原代码（自动检测） -->
<script>
    const t = localStorage.getItem('theme');
    t ? document.documentElement.setAttribute('data-theme', t) 
      : document.documentElement.setAttribute('data-theme', 
          new Date().getHours() >= 6 && new Date().getHours() < 18 ? 'light' : 'dark');
</script>

<!-- 修改为固定主题 -->
<script>
    document.documentElement.setAttribute('data-theme', 'dark');  // 固定暗色
    // 或
    document.documentElement.setAttribute('data-theme', 'light'); // 固定亮色
</script>
```

### Q15: 如何添加自定义字体？

**A:** 在 `layout.hbs` 的 `<head>` 中添加：

```html
<head>
    <!-- 添加 Google Font -->
    <link href="https://fonts.googleapis.com/css2?family=Fira+Code&display=swap" rel="stylesheet">
</head>
```

然后在 `style.css` 中使用：

```css
code, pre {
    font-family: 'Fira Code', monospace;
}
```

## C.5 部署和发布

### Q16: 如何部署博客？

**A:** 部署生成的静态文件：

```bash
# 1. 生成博客
zig build -Doptimize=ReleaseFast run

# 2. 上传到服务器
rsync -avz zig-out/blog/ user@server:/var/www/html/

# 或使用 SCP
scp -r zig-out/blog/* user@server:/var/www/html/
```

**支持的托管平台：**
- GitHub Pages
- Netlify
- Vercel
- Cloudflare Pages
- 任何静态文件服务器

### Q17: 如何配置自定义域名？

**A:** 以 GitHub Pages 为例：

```bash
# 1. 在仓库根目录创建 CNAME 文件
echo "blog.example.com" > CNAME

# 2. 在 DNS 提供商处添加 CNAME 记录
# blog.example.com CNAME ianychuo.github.io

# 3. 在 GitHub 仓库设置中启用 Pages
```

### Q18: 如何启用 HTTPS？

**A:** 使用 Let's Encrypt 免费证书：

```bash
# 在服务器上
sudo apt-get install certbot

# 获取证书
sudo certbot --nginx -d blog.example.com

# 自动续期（添加到 crontab）
0 0 1 * * certbot renew --quiet
```

## C.6 性能优化

### Q19: 构建速度太慢怎么办？

**A:** 尝试以下优化：

```bash
# 1. 使用增量构建（如果支持）
zig build --cachedir ./zig-cache

# 2. 并行编译
zig build -Dcpu=baseline

# 3. 减少文章数量（分批构建）
# 4. 升级硬件（更多 RAM，SSD）
```

### Q20: 页面加载速度慢？

**A:** 优化建议：

1. **图片优化**
   ```bash
   # 压缩图片
   convert image.jpg -quality 80 image-optimized.jpg
   
   # 转换为 WebP
   cwebp -q 80 image.jpg -o image.webp
   ```

2. **启用 CDN**
   ```html
   <!-- 使用 CDN 加载静态资源 -->
   <link rel="stylesheet" href="https://cdn.example.com/style.css">
   ```

3. **启用缓存**
   ```nginx
   # Nginx 配置
   location ~* \.(css|js|jpg|png|gif)$ {
       expires 30d;
       add_header Cache-Control "public, immutable";
   }
   ```

## C.7 故障排除

### Q21: 文章没有生成？

**A:** 检查以下项目：

```bash
# 1. 检查文件位置
ls posts/*.markdown

# 2. 检查 frontmatter 格式
head -10 posts/your-post.markdown

# 3. 查看详细错误
zig build run 2>&1

# 4. 检查输出目录
ls -la zig-out/blog/
```

### Q22: 样式没有生效？

**A:** 可能原因：

1. **CSS 缓存** - 强制刷新浏览器缓存（Ctrl+F5）
2. **路径错误** - 检查 `<link>` 标签路径
3. **优先级问题** - 使用浏览器开发者工具检查

```bash
# 检查 style.css 是否正确复制
cat zig-out/blog/style.css | head -20
```

### Q23: 标签页没有显示？

**A:** 检查：

```bash
# 1. 文章是否有标签
grep "tags:" posts/*.markdown

# 2. 标签格式是否正确
# 应该是：
tags:
  - zig
  - tutorial

# 而不是：
tags: zig, tutorial  # 可能解析失败
```

### Q24: 内存不足错误？

**A:** 解决方案：

```bash
# 1. 增加 swap 空间（Linux）
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. 分批生成文章
# 3. 使用 ReleaseFast 模式减少内存使用
zig build -Doptimize=ReleaseFast run
```

## C.8 扩展和定制

### Q25: 如何添加 RSS 订阅？

**A:** 参考文档 [13.2 添加 RSS 订阅](../part-13-extension-guides/013-2-rss-feed.md)

简要步骤：
1. 扩展 `Blogger.zig` 生成 `rss.xml`
2. 在 `layout.hbs` 添加 RSS 链接
3. 在导航栏添加 RSS 图标

### Q26: 如何添加搜索功能？

**A:** 参考文档 [13.3 集成搜索功能](../part-13-extension-guides/013-3-search-integration.md)

简要步骤：
1. 生成 `search-index.json`
2. 创建 `search.html` 页面
3. 使用 Fuse.js 实现客户端搜索

### Q27: 如何添加评论系统？

**A:** 使用第三方评论服务：

```html
<!-- Disqus -->
<div id="disqus_thread"></div>
<script>
    var disqus_config = function () {
        this.page.url = PAGE_URL;
        this.page.identifier = PAGE_IDENTIFIER;
    };
    (function() {
        var d = document, s = d.createElement('script');
        s.src = 'https://your-disqus-shortname.disqus.com/embed.js';
        s.setAttribute('data-timestamp', +new Date());
        (d.head || d.body).appendChild(s);
    })();
</script>

<!-- 或 Giscus（基于 GitHub Discussions） -->
<script src="https://giscus.app/client.js"
        data-repo="your-repo"
        data-repo-id="repo-id"
        data-category="Announcements"
        data-category-id="category-id"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="dark"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>
```

### Q28: 如何添加统计分析？

**A:** 添加 Google Analytics 或其他统计代码：

```html
<!-- Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID"></script>
<script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', 'GA_MEASUREMENT_ID');
</script>

<!-- 或使用 Plausible（隐私友好） -->
<script defer data-domain="blog.example.com" src="https://plausible.io/js/script.js"></script>
```

## C.9 其他问题

### Q29: 如何备份博客？

**A:** 定期备份以下内容：

```bash
# 备份脚本
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR="./backups/$DATE"

mkdir -p $BACKUP_DIR
cp -r posts/ $BACKUP_DIR/
cp -r static/ $BACKUP_DIR/
cp -r templates/ $BACKUP_DIR/
cp build.zig build.zig.zon $BACKUP_DIR/

# 压缩
tar -czf backups/$DATE.tar.gz $BACKUP_DIR
rm -rf $BACKUP_DIR

# 上传到云存储（可选）
aws s3 cp backups/$DATE.tar.gz s3://my-backups/
```

### Q30: 如何迁移到其他平台？

**A:** my-blog 生成标准 HTML，迁移简单：

1. **导出内容**
   ```bash
   # 所有文章都在 posts/ 目录
   # 保持 Markdown 源文件
   ```

2. **选择新平台**
   - Hugo
   - Jekyll
   - Hexo
   - WordPress

3. **转换 frontmatter**（如需要）

4. **导入内容**到新平台

## 相关文档

- [附录 A: std 库 API](./appendix-a-std-lib-api.md) - 标准库参考
- [附录 B: 常用代码模式](./appendix-b-common-patterns.md) - 代码模式
- [13.1 编写博客文章指南](../part-13-extension-guides/013-1-writing-posts.md) - 文章创作

---

*本文档汇总了 my-blog 项目的常见问题解答，涵盖入门、构建、创作、主题、部署、性能优化和故障排除等方面。*
