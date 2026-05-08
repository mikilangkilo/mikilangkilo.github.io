---
name: blog-manager
description: Hugo博客管理 — 写文章、构建、部署到GitHub Pages (mikilangkilo.github.io)
type: user
---

# 博客管理 Skill

## 基本信息

- **博客地址**: https://mikilangkilo.github.io
- **仓库**: git@github.com:mikilangkilo/mikilangkilo.github.io.git
- **本地路径**: ~/work/blog/
- **框架**: Hugo + PaperMod 主题
- **作者**: 殷鹏程
- **语言**: 中文 (zh-cn)
- **部署方式**: GitHub Pages (master 分支, root 目录)
- **重要**: 必须保留 `.nojekyll` 文件（否则中文目录名导致404）

## 目录结构

```
~/work/blog/              # 本地仓库根目录
├── .nojekyll             # 必须！禁用Jekyll
├── .gitignore
├── .github/workflows/hugo.yml  # GitHub Actions（备用）
├── hugo.yml              # Hugo配置文件
├── content/              # 文章内容
│   ├── about/            # 关于页面
│   │   └── index.md
│   └── posts/            # 文章目录
│       ├── my-first-post.md
│       └── 新文章.md     # 新文章放这里
├── themes/PaperMod/      # 主题（git submodule）
└── public/               # 构建输出（.gitignore忽略，不推送）
```

## 写新文章流程

### 1. 创建文章

```bash
cd ~/work/blog
```

文章放在 `content/posts/` 下，格式如下：

```markdown
---
title: "文章标题"
date: YYYY-MM-DDTHH:mm:ss+08:00
draft: false          # false=发布, true=草稿(不显示)
tags: ["标签1", "标签2"]
categories: ["分类"]
summary: "摘要文字"    # 可选，首页显示
---

# 文章正文

用 Markdown 写内容。
```

**注意**: `date` 格式必须带时区，例如 `2026-05-09T18:00:00+08:00`

### 2. 本地预览

```bash
cd ~/work/blog && hugo server -D --bind 0.0.0.0 --baseURL http://10.57.106.247:1313
```
然后用手机浏览器打开 `http://<rmnet_data2的IP>:1313` 预览。

### 3. 构建并部署

```bash
cd ~/work/blog

# 构建
hugo --minify

# 将静态文件复制到根目录
cp -r public/* .
rm -rf public

# 提交推送
git add -A
git commit -m "new post: 文章标题"
git push origin master

# 等待30秒后验证
sleep 30 && curl -s -o /dev/null -w "%{http_code}" https://mikilangkilo.github.io
```

### 4. 部署检查清单

- [ ] `.nojekyll` 文件存在（关键！）
- [ ] 根目录有 `index.html`
- [ ] `hugo.yml` 中 baseURL 是 `https://mikilangkilo.github.io/`
- [ ] 中文分类/标签页面能正常访问

## 快捷命令速查

| 操作 | 命令 |
|------|------|
| 创建新文章 | 编辑 `content/posts/xxx.md` |
| 本地预览 | `hugo server -D --bind 0.0.0.0 --baseURL http://IP:端口` |
| 构建生成 | `hugo --minify` |
| 部署上线 | 复制public→root → git push |
| 查看状态 | `curl -sI https://mikilangkilo.github.io \| head -5` |
| 清理缓存 | `hugo --gc` |

## PaperMod 主题常用功能

### Front Matter 参数

```yaml
---
title: ""           # 必填：标题
date: ""            # 必填：日期（带时区）
draft: false        # 是否草稿
tags: []            # 标签
categories: []       # 分类
summary: ""         # 自定义摘要
cover:
  image: ""         # 封面图（相对路径 content/posts/images/xxx.jpg）
  alt: ""
  caption: ""
  hidden: true      # 是否在首页隐藏封面
params:
  ShowToc: true     # 显示目录
  comments: true    # 开启评论
series: []          # 系列文章
---

### 短代码用法

PaperMod 主题支持以下短代码（在文章 Markdown 中使用）：

- **折叠内容**: `{{</* details "标题" */>}}内容{{</* /details */>}}`
- **提示框**: notice / warning / note
- **图片**: figure 短代码

具体语法参考 PaperMod 官方文档。

## 常见问题

### Q: 推送后还是404？
A: 检查根目录是否有 `.nojekyll` 文件和 `index.html`

### Q: 新文章没显示？
A: 检查 `draft: false`，日期不能是未来时间

### Q: 中文链接404？
A: `.nojekyll` 文件丢失了，重新添加并推送

### Q: 图片怎么加？
A: 放在 `content/posts/images/` 目录下，Markdown中用相对路径引用

## 运行日志

> 以下记录博客每次变更操作

| 日期 | 操作 | 说明 |
|------|------|------|
| 2026-05-09 | 初始化 | 从 GitHub 克隆仓库，安装 Hugo 0.161.1，修复 Actions workflow(Hugo版本0.125→0.140.1)，解决Jekyll中文路径404问题(.nojekyll)，成功部署上线 |
| 2026-05-09 | 创建 Skill | 将博客管理流程整理为 CodeBuddy Skill |
