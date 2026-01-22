# Bug 修复报告

## 问题描述
1. 社交链接图标（GitHub、Email）在首页个人简介处一直不显示
2. 页脚的深色/浅色模式切换按钮消失
3. 配置文件参数未被正确读取

## 根本原因
**TOML 配置文件语法问题**：之前使用的内联数组写法 `links = [{ icon = 'github', ... }]` 在某些情况下无法被 Hugo 正确解析为嵌套的数组对象。

## 修复措施

### 1. 重写 `hugo.toml`
- **修改前**：使用内联数组语法（不稳定）
  ```toml
  [params.author]
    links = [
      { icon = 'github', href = '...' }
    ]
  ```

- **修改后**：使用标准 TOML 数组表语法（推荐）
  ```toml
  [params.author]
  name = '661'
  
  [[params.author.links]]
  icon = 'github'
  href = 'https://github.com/661616'
  
  [[params.author.links]]
  icon = 'mail'
  href = 'mailto:ygzh0525@gmail.com'
  ```

### 2. 添加 Footer 显式配置
```toml
[params.footer]
showCopyright = true
showThemeAttribution = true
showAppearanceSwitcher = true  # 显式开启深色模式切换
showScrollToTop = true
```

### 3. 清理破坏性修改
- 删除了自定义的 `layouts/partials/footer.html`（它破坏了原有布局）
- 清空了所有缓存目录（`resources/`, `public/`, `_vendor/`）

### 4. 手动添加图标文件
在 `assets/icons/` 下放置了 `github.svg` 和 `mail.svg`，确保图标资源存在。

## 验证方法

请刷新浏览器 `http://localhost:1313/`，检查以下项：

1. **首页个人简介下方**：是否出现 GitHub 和邮箱图标？
2. **页面右下角**：是否出现太阳/月亮图标（深色模式切换器）？
3. **点击切换器**：能否正常切换亮色/暗色主题？

## 后续推送
如果本地验证通过，执行以下命令推送到线上：
```bash
git add .
git commit -m "Fix: Rewrite hugo.toml using standard TOML array syntax"
git push
```

Cloudflare Pages 会自动重新部署，约 1-2 分钟后线上也会生效。

