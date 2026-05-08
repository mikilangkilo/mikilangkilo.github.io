---
title: "用 Hugo + GitHub Pages 搭建个人博客"
date: 2026-05-09T18:30:00+08:00
draft: false
tags: ["Hugo", "GitHub Pages", "博客", "Termux"]
categories: ["随笔"]
summary: "在 Termux (Android) 环境下使用 Hugo 静态站点生成器搭建个人技术博客，部署到 GitHub Pages。"
---

## 为什么选 Hugo？

在 Android 手机（Termux）上写博客，Hugo 是最佳选择：

- **纯静态**：不需要数据库，不需要服务器运行时
- **构建快**：几百篇文章也能秒级生成
- **主题丰富**：PaperMod 主题美观且功能齐全
- **Markdown 写作**：专注内容，不用管样式

## 技术栈

| 组件 | 选型 |
|------|------|
| 静态生成器 | Hugo 0.161.1 |
| 主题 | PaperMod |
| 托管 | GitHub Pages |
| 域名 | mikilangkilo.github.io |
| 运行环境 | Termux on Android |

## 踩过的坑

### 1. GitHub Pages 404

仓库配置了 `Deploy from branch → master / root`，但一直返回 404。

**原因**：GitHub Pages 默认启用 Jekyll 处理，中文目录名（如 `分类/`、`标签/`）被 Jekyll 过滤掉。

**解决**：在根目录添加 `.nojekyll` 文件：

```bash
echo "" > .nojekyll
git add .nojekyll && git push
```

### 2. GitHub Actions 构建失败

原始 workflow 配置的 Hugo 版本过旧（0.125），安装方式不兼容。

**修复**：
- 升级到 Hugo 0.140.1
- 使用 `peaceiris/actions-hugo@v3` 安装
- Dart Sass 改用 npm 安装

### 3. 用户主页仓库的部署方式

`username.github.io` 这类用户主页仓库比较特殊：
- 如果用 GitHub Actions 部署，需要在 Settings → Pages 中选择 Source 为 **GitHub Actions**
- 如果直接推送静态文件，Source 选择 **Deploy from a branch** → master → `/ (root)`
- 两种方式不能混用，否则容易出问题

## 博客目录结构

```
mikilangkilo.github.io/
├── .nojekyll              # 关键！禁用Jekyll处理
├── hugo.yml               # Hugo配置
├── content/
│   ├── about/index.md     # 关于页面
│   └── posts/             # Markdown文章
├── themes/PaperMod/       # 主题
└── public/                # 构建输出(不推送)
```

## 在 Termux 上管理博客

所有操作都可以在手机上完成：

```bash
# 安装工具链
pkg install hugo git

# 克隆仓库
git clone git@github.com:mikilangkilo/mikilangkilo.github.io.git ~/work/blog

# 写文章（编辑 content/posts/xxx.md）

# 本地预览
cd ~/work/blog && hugo server --bind 0.0.0.0

# 构建并部署
hugo --minify
cp -r public/* .
rm -rf public
git add -A && git commit -m "new post" && git push origin master
```

## 后续计划

- [ ] 配置自定义域名
- [ ] 接入评论系统（Giscus）
- [ ] 添加 Google Analytics
- [ ] 图片懒加载和优化
