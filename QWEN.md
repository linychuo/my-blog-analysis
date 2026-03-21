# my-blog-analysis

Technical documentation repository for analyzing **my-blog** - a static blog generator written in Zig.

---

## Project Overview

This repository contains comprehensive technical documentation analyzing **my-blog**, a static blog generator built with Zig 0.15.2. The documentation provides in-depth coverage of the project's architecture, implementation details, and usage patterns.

**Source Project**: [github.com/linychuo/my-blog](https://github.com/linychuo/my-blog)

### Key Characteristics

| Aspect | Details |
|--------|---------|
| **Language** | Zig 0.15.2 |
| **Purpose** | Static blog generation from Markdown |
| **Features** | Markdown parsing, Handlebars-style templates, tag system, dark/light themes |
| **Dependencies** | Custom zig-markdown, custom zig-handlebars |
| **Output** | Static HTML files ready for deployment |

---

## Repository Structure

```
my-blog-analysis/
├── docs/                           # Documentation source
│   ├── README.md                   # Main documentation index
│   ├── SUMMARY.md                  # Complete document list
│   ├── part-01-project-overview/   # Project background & architecture
│   └── part-02-quick-start/        # Build & run instructions
└── QWEN.md                         # This file
```

### Documentation Organization

The documentation is organized into 14 parts:

| Part | Status | Content |
|------|--------|---------|
| Part 01 | ✅ Complete | Project overview, tech stack, features, structure |
| Part 02 | ✅ Complete | Requirements, build/run, CLI options, output structure |
| Part 03-14 | ⚪ Pending | Zig basics, control flow, functions, build system, core modules, memory management, error handling, template engine, Markdown parser, styling, extensions, testing |
| Appendix | ⚪ Pending | Std lib API, common patterns, FAQ |

**Progress**: 8/53 documents completed (15%)

---

## The my-blog Project

### Architecture

```
┌─────────────────────────────────────────┐
│              main.zig                   │
│         (CLI entry point)               │
└───────────────────┬─────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
┌──────────────┐ ┌──────────┐ ┌─────────────┐
│  Blogger.zig │ │  std     │ │  Third-party│
│  (core logic)│ │  library │ │  dependencies│
└──────┬───────┘ └──────────┘ └──────┬──────┘
       │                             │
       │              ┌──────────────┼──────────────┐
       ▼              ▼              ▼              ▼
┌─────────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
│zig-markdown │ │zig-       │ │highlight.js│ │  KaTeX    │
│(MD→HTML)    │ │handlebars │ │(syntax HL) │ │(math fmt) │
└─────────────┘ └───────────┘ └───────────┘ └───────────┘
```

### Source Project Structure

```
my-blog/
├── src/
│   ├── main.zig              # Application entry point
│   └── Blogger.zig           # Core blog generation logic
├── templates/
│   ├── layout.hbs            # Main layout template
│   ├── index.hbs             # Homepage template
│   ├── post.hbs              # Article page template
│   ├── about.hbs             # About page template
│   └── tag.hbs               # Tag page template
├── static/
│   ├── style.css             # Main stylesheet
│   └── imgs/                 # Image assets
├── posts/
│   ├── about.markdown        # About page content
│   └── *.markdown            # Blog posts
├── libs/
│   ├── zig-markdown/         # Markdown parser dependency
│   └── zig-handlebars/       # Template engine dependency
├── build.zig                 # Build configuration
├── build.zig.zon             # Package manifest
└── zig-out/blog/             # Generated output
```

### Key Components

#### Blogger.zig
Core struct handling blog generation:
- `Post` struct - Article data structure with frontmatter parsing
- `Blogger` struct - Main generator with methods for loading posts, generating pages, copying assets

#### zig-markdown
Custom Markdown-to-HTML converter supporting:
- CommonMark blocks (headings, paragraphs, lists, code blocks, tables, blockquotes)
- Inline elements (bold, italic, strikethrough, links, images, inline code)
- Extensions (math formulas with `$$`, line breaks with `\`)

#### zig-handlebars
Custom Handlebars-style template engine:
- Variable interpolation: `{{ title }}`
- Unescaped HTML: `{{{ content }}}`
- Inline blocks: `{{#*inline "name"}}...{{/inline}}`
- Template inheritance: `{{~> (parent)~}}`

---

## Building and Running (Source Project)

### Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| Zig | 0.15.2+ | Compiler and runtime |
| Git | 2.0+ | Version control |

### Build Commands

```bash
# Debug build and run
zig build run

# Release build
zig build -Doptimize=ReleaseFast run

# Run tests
zig build test

# Show help
zig build run -- --help
```

### CLI Options

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--posts` | `-p` | `posts` | Source directory for Markdown posts |
| `--output` | `-o` | `build` | Output directory for generated files |
| `--help` | `-h` | - | Show help message |

### Example Usage

```bash
# Default (uses ./posts and ./build)
zig build run

# Custom paths
zig build run -- --posts ./content --output ./public
```

### Generated Output Structure

```
zig-out/blog/
├── index.html              # Homepage
├── about.html              # About page
├── style.css               # Stylesheet (copied)
├── imgs/                   # Images (copied)
├── tags/                   # Tag pages
│   ├── zig.html
│   └── blog.html
└── YYYY/MM/DD/             # Date-based article URLs
    └── post-slug.html
```

---

## Development Conventions

### File Naming

| Type | Convention | Example |
|------|------------|---------|
| Zig modules | PascalCase.zig | `Blogger.zig` |
| Templates | lowercase.hbs | `index.hbs` |
| Documentation | kebab-case.md | `project-structure.md` |
| Blog posts | kebab-case.markdown | `my-first-post.markdown` |

### Documentation Style

- All documentation is written in Chinese
- Each part has a numbered prefix (e.g., `01-1-`, `02-3-`)
- Documents include tables, code examples, and diagrams
- Each section ends with "Related Reading" links

### Testing Practices

- Unit tests using Zig's built-in test framework
- Run with `zig build test`
- Memory leak detection available in debug mode

---

## Key Technologies

### Frontend Dependencies (CDN)

| Resource | Version | Purpose |
|----------|---------|---------|
| Inter Font | - | Body text font |
| JetBrains Mono | - | Code font |
| highlight.js | 11.9.0 | Syntax highlighting |
| KaTeX | 0.12.0 | Math formula rendering |

### Theme System

- Automatic dark/light theme based on time (6AM-6PM)
- Manual toggle with localStorage persistence
- CSS custom properties for theming
- Synchronized code highlighting theme切换

---

## Usage Notes

### When Working with This Repository

1. **Documentation Focus**: This is a documentation repository, not the source code
2. **Source Location**: Actual code is at [github.com/linychuo/my-blog](https://github.com/linychuo/my-blog)
3. **Document Status**: Check `docs/README.md` for current completion status
4. **Adding Content**: Follow existing naming conventions and structure

### Common Tasks

| Task | Location |
|------|----------|
| Add new documentation | `docs/part-XX-category/NNN-title.md` |
| Update document index | `docs/README.md` and `docs/SUMMARY.md` |
| Reference source code | See `docs/part-07-core-modules/` for code analysis |

---

## Related Resources

- **Source Repository**: https://github.com/linychuo/my-blog
- **Zig Documentation**: https://ziglang.org/documentation
- **Zig Standard Library**: https://ziglang.org/documentation/master/std

---

*Last updated: 2026-03-21*
