# DIger's Blog

[![Jekyll](https://img.shields.io/badge/Jekyll-4.4-CC0000?logo=jekyll&logoColor=white)](https://jekyllrb.com/)
[![Chirpy](https://img.shields.io/badge/Theme-Chirpy-159957)](https://github.com/cotes2020/jekyll-theme-chirpy)
[![GitHub Pages](https://img.shields.io/badge/Deployed%20on-GitHub%20Pages-222222?logo=github)](https://pages.github.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

👋 欢迎来到 **DIger's Blog** —— 一个使用 [Jekyll](https://jekyllrb.com/) + [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 主题搭建、托管在 [GitHub Pages](https://pages.github.com/) 上的个人博客。

🌐 在线访问：<https://fewftybet.github.io/>

## ✨ 特性

- 🎨 **Chirpy 主题**：响应式、支持深色 / 浅色切换、PWA 可安装
- 🔍 **全文搜索**、📑 目录、🏷 标签、📂 分类、🗓 归档
- 💬 内置评论系统支持（giscus / utterances / disqus）
- 🚀 **GitHub Actions 自动部署**：push 到 `main` 即触发，约 1 分钟上线
- 📝 Markdown / MathJax / Mermaid / 代码高亮 / 图片懒加载

## 🛠 技术栈

| 组件 | 版本 |
|------|------|
| Jekyll | 4.4 |
| jekyll-theme-chirpy | 7.6 |
| Ruby | 3.2+ (本地) / 3.4 (CI) |
| 部署 | GitHub Actions → GitHub Pages |

## 📂 目录结构

```text
.
├── _config.yml          # 站点配置
├── _data/               # 站点数据（导航、友链等）
├── _includes/           # 公共组件
├── _layouts/            # 模板
├── _posts/              # 博客文章（YYYY-MM-DD-title.md）
├── _tabs/               # 顶部导航页（Home/About/...）
├── _plugins/            # 自定义 Liquid 插件
├── assets/              # 静态资源（含 submodule: chirpy-static-assets）
└── .github/workflows/   # GitHub Actions 配置
```

## ✍️ 写一篇新文章

在 `_posts/` 下新建 Markdown 文件，命名格式 `YYYY-MM-DD-title.md`：

```markdown
---
title: 我的第一篇文章
date: 2026-07-10 12:00:00 +0800
categories: [Tech, Notes]
tags: [jekyll, chirpy]
description: 这是一段描述，会出现在摘要和 SEO meta 中。
---

正文内容...
```

写完后：

```bash
git add -A
git commit -m "post: 我的第一篇文章"
git push origin main
```

约 1 分钟后访问 <https://fewftybet.github.io/> 即可看到。

## 🖥 本地预览

需要先安装 [Ruby 3.2+ with DevKit](https://rubyinstaller.org/)。

```bash
bundle install
bundle exec jekyll serve
```

打开 <http://127.0.0.1:4000/> 即可预览。

## 🙏 致谢

- 主题：[cotes2020/jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)
- 静态资源：[cotes2020/chirpy-static-assets](https://github.com/cotes2020/chirpy-static-assets)
- 引擎：[Jekyll](https://jekyllrb.com/)

## 📬 联系

- Email: 1565118064@qq.com
- GitHub: [@fewftybet](https://github.com/fewftybet)

## 📄 License

本博客内容采用 [MIT](LICENSE) 协议发布，文章内容版权归作者所有。
