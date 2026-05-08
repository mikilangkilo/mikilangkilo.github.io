---
title: "用手机微信 AI 机器人写代码、搭博客、做游戏"
date: 2026-05-09T19:00:00+08:00
draft: false
tags: ["CodeBuddy", "WeChat", "ClawBot", "Termux", "AI", "Android"]
categories: ["技术"]
summary: "一台 Android 手机 + Termux + WeChat ClawBot + CodeBuddy，就能实现随时随地通过微信对话来写代码、管理博客、开发小游戏。本文记录整套技术方案的架构和实战体验。"
---

## 起因

今天我在手机上通过微信跟一个 AI 机器人聊天，让它帮我：

1. 写了 4 个小游戏（打飞机、打坦克、逃离迷宫、人脸识别互动）
2. 搭建并部署了个人博客（Hugo + GitHub Pages）
3. 把博客管理流程整理成了可复用的 Skill
4. 还顺手修了一堆坑

整个过程没有碰电脑——全靠手机上的微信语音/文字对话完成。

## 技术架构

整个系统由以下几个核心组件组成：

```
┌─────────────┐    WebSocket     ┌──────────────┐
│   微信 App   │ ◄──────────────► │  ClawBot     │
│  (发送消息)  │                  │  (微信插件)   │
└─────────────┘                  └──────┬───────┘
                                        │ 长连接
                                        ▼
                              ┌──────────────────┐
                              │   CodeBuddy       │
                              │  (AI 编程助手)     │
                              │                   │
                              │  ┌─────────────┐  │
                              │  │ LLM (GLM-5v) │  │
                              │  └─────────────┘  │
                              │  ┌─────────────┐  │
                              │  │ Tool 执行引擎 │  │
                              │  └─────────────┘  │
                              └─────────┬────────┘
                                        │ Shell / Node.js
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
            ┌───────▼───────┐   ┌──────▼──────┐   ┌──────▼──────┐
            │   Termux      │   │   Git/GitHub │   │ HTTP Server  │
            │               │   │              │   │              │
            │ • 文件读写     │   │ • 克隆仓库    │   │ • 本地游戏    │
            │ • 进程管理     │   │ • 提交推送    │   │   服务        │
            │ • 网络工具     │   │ • SSH 密钥    │   │              │
            │ • 包管理(pkg) │   │              │   │              │
            └───────────────┘   └──────────────┘   └──────────────┘
```

### 核心组件详解

#### 1. Termux — Android 终端神器

Termux 是一个 Android 终端模拟器，提供完整的 Linux 环境：

```bash
# 在手机上安装完整开发环境
pkg install nodejs git hugo python openssh

# 运行任何 Linux 命令
ssh git@github.com          # SSH 连接
node server.js              # 启动服务
hugo --minify                # 构建静态站点
git push origin master      # 推送代码
```

**关键能力**：
- 无需 root 权限运行 Linux 命令
- 支持安装 npm/pip/pkg 包
- 可以运行 Node.js、Python、Go 等运行时
- 支持 SSH/Git 操作

#### 2. CodeBuddy — AI 编程助手

CodeBuddy 是核心大脑，它具备：

- **LLM 对话能力**：使用 GLM-5v-Turbo 模型（支持多模态）
- **工具调用**：能执行文件读写、Shell 命令、Git 操作等
- **Skill 系统**：可扩展的技能模块（类似插件）
- **Gateway 服务**：暴露 HTTP API 供外部调用

#### 3. WeChat ClawBot — 微信桥梁

ClawBot 是微信的一个插件/机器人框架：

- 通过长连接与 CodeBuddy Gateway 通信
- 将微信消息转发给 AI 处理
- 把 AI 的回复发回微信
- 支持文字、语音转文字、图片等多媒体

#### 4. Serveo 隧道 — 公网访问

由于 Android 手机通常在 NAT 后面，外部无法直接访问：

```bash
# SSH 反向隧道，将本地端口暴露到公网
autossh -M 0 -o ServerAliveInterval=30 \
  -R 80:127.0.0.1:33525 serveo.net
```

> 注：cloudflared 在 Termux 上不可用（DNS 解析问题），改用 serveo。

### 网络拓扑

```
手机 (Android/Termux)
├── rmnet_data2: 10.x.x.x (移动数据, 用于对外服务)
├── wlan0: 192.168.110.64 (WiFi)
│
├── CodeBuddy Gateway :46523
│   ├── WeChat ClawBot ←→ WebSocket ←→ 微信
│   ├── HTTP API (/api/v1/runs)
│   └── Web UI
│
├── 游戏服务器 :8899 (Node.js HTTP)
│   └── 访问地址: http://10.x.x.x:8899/
│
├── Stock Bot Daemon (定时任务)
│   └── cron 推送到微信
│
└── Serveo Tunnel → 公网 URL
```

## 实战：今天做了什么

### 小游戏开发

通过微信对话，我让 AI 写了 4 个纯前端游戏：

| 游戏 | 技术点 | 行数 |
|------|--------|------|
| 打飞机 | Canvas 2D、粒子系统、波次递进 | ~300 |
| 打坦克 | Canvas 绘制坦克、虚拟摇杆、碰撞检测、AI 敌人 | ~350 |
| 逃离迷宫 | 递归回溯算法生成迷宫、D-Pad 控制、闯关模式 | ~280 |
| 人脸互动 | getUserMedia 调用摄像头、肤色检测追踪、多游戏模式 | ~400 |

每个游戏的开发流程：

```
用户(微信) → "写个打飞机游戏"
    → CodeBuddy 生成 HTML/CSS/JS
    → 写入 ~/work/plane-game.html
    → 启动 Node.js HTTP 服务器 (:8899)
    → 获取 rmnet_data2 的 IP 地址
    → 回复用户访问链接 http://10.x.x.x:8899/plane-game.html
    → 用户在手机浏览器打开游玩
```

### 博客搭建

从零到上线：

```
1. 安装 Hugo: pkg install hugo
2. 克隆仓库: git clone git@github.com:mikilangkilo/mikilangkilo.github.io.git
3. 本地构建: hugo --minify
4. 推送部署: git push origin master
5. 解决 404: 添加 .nojekyll 文件
6. 修复 Actions: 升级 Hugo 版本到 0.140.1
7. 新增文章: 编辑 content/posts/
8. 创建 Skill: blog-manager.md
```

踩过的坑：

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Pages 404 | Jekyll 过滤中文路径 | `.nojekyll` 文件 |
| Actions 构建失败 | Hugo 版本过旧 | 升级 + 用 peaceiris action |
| 当天文章不显示 | `buildFuture` 默认 false | 配置 `buildFuture: true` |
| 手机按钮点击无效 | `touch-action:none` 阻止事件 | 改用 `touchend` 监听 |

### Skill 化管理

将博客操作封装为 Skill（`blog-manager.md`），包含：

- 完整的目录结构和文件说明
- 写文章的标准模板和流程
- 构建部署的一键命令
- 常见问题 FAQ
- 运行日志记录表

以后只需说"写篇文章"，AI 就按 Skill 流程自动完成全部操作。

## 为什么这个方案有意思

### 1. 纯移动端开发

传统开发需要电脑 + IDE + 浏览器调试。这套方案把整个开发链路搬到了手机上：

```
电脑: IDE → 终端 → 浏览器 → Git → 部署
手机: 微信对话 → CodeBuddy → Termux → 自动化
```

### 2. 自然语言编程

不需要懂具体语法：

- "写个打坦克的游戏" → 生成 350 行游戏代码
- "帮我部署博客" → 执行完整的 CI/CD 流程
- "把这个做成 Skill" → 生成结构化的技能文档

### 3. 多模态交互

支持多种输入方式：

- **文字**: 直接打字或粘贴需求
- **语音**: 微信语音消息自动转文字
- **图片**: 可以发送截图让 AI 分析

## 局限性

当然这套方案也有局限：

| 方面 | 限制 |
|------|------|
| 性能 | 手机 CPU/内存有限，不适合大型项目 |
| 屏幕 | 手机屏幕小，代码审查效率低 |
| 网络 | 移动数据不稳定，SSH 隧道可能断连 |
| 权限 | 非 root 环境，某些系统级操作受限 |
| 时效 | LLM 响应有延迟，不适合实时调试 |

适合的场景：轻量开发、脚本编写、博客维护、小游戏、自动化任务。

## 总结

一台 Android 手机 + Termux + WeChat ClawBot + CodeBuddy，就能构成一个完整的「口袋开发环境」：

- **输入端**：微信（随时随地）
- **处理端**：CodeBuddy AI（理解意图、调用工具）
- **执行端**：Termux（Linux 环境）
- **展示端**：浏览器（游戏）/ GitHub Pages（博客）

这不是替代电脑开发的方案，而是一种**补充**——当你不在电脑前，但有个想法想立刻实现时，掏出手机跟微信里的 AI 聊几句就够了。

---

*本文由 CodeBuddy (GLM-5v-Turbo) 在 Termux 环境下撰写，通过 WeChat ClawBot 发布到博客。*
