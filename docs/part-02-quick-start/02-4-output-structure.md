# 2.4 输出结构

> 了解 my-blog 生成的文件组织和目录结构

---

## 目录

- [输出目录概览](#输出目录概览)
- [根目录文件](#根目录文件)
- [标签页面](#标签页面)
- [文章页面](#文章页面)
- [URL 结构](#url-结构)
- [部署指南](#部署指南)

---

## 输出目录概览

```
build/
├── index.html                  # 首页
├── about.html                  # 关于页面
├── style.css                   # 样式表
├── imgs/                       # 图片资源
│   ├── favicon.ico
│   └── *.jpg, *.png
├── tags/                       # 标签页面目录
│   ├── zig.html
│   ├── blog.html
│   └── ...
└── 2026/                       # 年份目录
    └── 03/                     # 月份目录
        └── 21/                 # 日期目录
            ├── my-first-post.html
            └── zig-intro.html
```

---

## 根目录文件

### index.html（首页）

- 显示所有文章列表
- 文章按日期降序排列
- 每篇文章显示标题、日期、标签

### about.html（关于页面）

- 由 `posts/about.markdown` 生成
- 显示博客作者/网站介绍

### style.css（样式表）

- 从 `static/style.css` 复制
- 包含全站样式和主题变量

---

## 标签页面

```
build/tags/
├── zig.html
├── blog.html
└── tutorial.html
```

每个标签页面显示：
- 标签名称和文章数量
- 所有使用该标签的文章列表
- 按日期降序排列

---

## 文章页面

### URL 结构

```
build/YYYY/MM/DD/slug.html
```

**示例**:
```
build/2026/03/21/my-first-post.html
```

### 页面内容

- 文章标题（来自 frontmatter）
- 发布日期
- 标签链接
- HTML 正文（Markdown 转换）
- 代码高亮
- 数学公式（KaTeX）

---

## URL 结构

| 页面类型 | URL 模式 | 示例 |
|----------|----------|------|
| 首页 | `/` | index.html |
| 关于 | `/about.html` | about.html |
| 文章 | `/YYYY/MM/DD/slug.html` | /2026/03/21/post.html |
| 标签 | `/tags/name.html` | /tags/zig.html |

---

## 部署指南

### 部署到 GitHub Pages

```bash
# 1. 构建
zig build -Doptimize=ReleaseFast run

# 2. 切换到 gh-pages 分支
git checkout gh-pages

# 3. 复制构建产物
rm -rf *
cp -r build/* .

# 4. 提交并推送
git add .
git commit -m "Deploy blog"
git push origin gh-pages
```

### GitHub Actions 自动部署

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [ main ]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Zig
        uses: goto-bus-stop/setup-zig@v2
        with:
          version: 0.15.2
      - name: Build
        run: zig build -Doptimize=ReleaseFast run
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build
```

---

## 小结

**输出目录结构**:
```
build/
├── index.html          # 首页
├── about.html          # 关于页
├── style.css           # 样式表
├── imgs/               # 图片
├── tags/               # 标签页
└── YYYY/MM/DD/         # 文章页
```

**关键特点**:
- 日期为基础的 URL 结构
- 标签独立页面
- 静态文件可直接部署

---

## 相关阅读

- [环境要求](02-1-requirements.md) - 前置条件
- [构建与运行](02-2-build-and-run.md) - 构建项目
- [命令行选项](02-3-cli-options.md) - 自定义参数
