# 个人知识库建设方案 (PRD & Tech Spec)

## 1. 项目背景与目标
构建一个基于 **Hugo** 静态网站生成器和 **Congo** 主题的高性能个人知识库。旨在提供一个简洁、快速、易于维护的平台，用于记录技术积累、生活随笔和项目文档。

参考文档：[YWC Tech - 起头並不難：用 Hugo 架站的第一步](https://ywctech.net/tech/start-hugo-site/)

## 2. 产品需求 (PRD)

### 2.1 核心功能需求
1.  **内容管理**
    -   支持标准 Markdown 语法写作。
    -   支持代码高亮 (Syntax Highlighting)。
    -   支持数学公式 (LaTeX/KaTeX)。
    -   支持文章分类 (Categories) 和标签 (Tags) 归档。
2.  **用户体验**
    -   **极简阅读体验**: 无干扰的阅读界面。
    -   **响应式设计**: 完美适配 Desktop, Tablet, Mobile。
    -   **深色/浅色模式**: 根据系统设置自动切换，或用户手动切换。
    -   **站内搜索**: 快速索引和查找文章内容。
3.  **导航与信息架构**
    -   首页：个人简介 + 最近文章列表。
    -   顶部导航：文章、分类、标签、关于我。
    -   底部：版权信息、社交链接。

### 2.2 非功能需求
-   **性能**: 页面加载速度极快 (静态资源)。
-   **SEO**: 良好的搜索引擎优化支持。
-   **维护成本**: 低成本托管 (Cloudflare Pages 免费方案)。

## 3. 技术方案 (Technical Proposal)

### 3.1 技术选型
| 组件 | 选型 | 说明 |
| :--- | :--- | :--- |
| **生成器** | **Hugo** (Extended) | Go 语言编写，构建速度极快，社区活跃。 |
| **主题** | **Congo** (v2) | 基于 Tailwind CSS，轻量、现代、功能丰富。 |
| **版本控制** | **Git** | 源码管理。 |
| **代码托管** | **GitHub** | 存储私有或公开仓库。 |
| **部署托管** | **Cloudflare Pages** | 自动化构建、全球 CDN 加速、免费 SSL。 |
| **开发环境** | Cursor / VS Code | 本地编辑器。 |

### 3.2 系统架构与部署流程
采用 **GitOps** 流程：
1.  **Local**: 本地撰写 Markdown 文件，使用 `hugo server` 预览。
2.  **Push**: 代码提交并推送到 GitHub 仓库 (`git push`).
    -   *Git User Config*: `user.name = "661"` (当前项目特定设置)
3.  **Build**: Cloudflare Pages 检测到 GitHub `main` 分支更新，自动拉取代码。
4.  **Deploy**: Cloudflare 执行 `hugo` 构建命令，将生成的 `public/` 目录发布到 CDN 节点。

### 3.3 目录结构规范
```text
knowledge-base/
├── archetypes/          # 文章模板
├── assets/              # 资源文件 (CSS, JS)
├── content/             # 内容源文件 (Markdown)
│   ├── posts/           # 博客文章
│   └── about.md         # 关于页面
├── layouts/             # 自定义布局 (HTML)
├── static/              # 静态资源 (图片, favicon)
├── themes/              # 主题文件 (Git Submodule 或 Go Module)
├── hugo.toml            # 核心配置文件
└── go.mod               # Go 模块依赖定义
```

### 3.4 关键配置 (hugo.toml)
-   **BaseURL**: 部署后修改为实际域名。
-   **Theme**: `github.com/jpanther/congo/v2`
-   **Language**: `zh-cn`
-   **Author**: 配置个人信息和社交链接。
-   **Menu**: 定义主导航结构。

## 4. 实施计划 (Roadmap)

- [x] **P0: 基础搭建**
    -   环境初始化 (Hugo + Git)。
    -   Congo 主题安装与基础配置。
    -   Git 用户名配置 (`661`)。
- [ ] **P1: 内容填充与定制**
    -   完善首页个人信息 (Profile)。
    -   创建基础页面 (关于、归档)。
    -   迁移/撰写首批文章。
    -   微调主题样式 (如有必要)。
- [ ] **P2: 部署上线**
    -   推送到 GitHub。
    -   配置 Cloudflare Pages。
    -   绑定自定义域名 (如需)。

## 5. 维护规范
-   **提交日志**: 保持清晰的 commit message。
-   **图片管理**: 统一放入 `static/images/` 或使用图床。
-   **FrontMatter**: 每篇文章必须包含 `title`, `date`, `categories`。

