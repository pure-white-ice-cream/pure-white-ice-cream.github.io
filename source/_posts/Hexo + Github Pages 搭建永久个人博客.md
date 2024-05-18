---
title: Hexo + Github Pages 搭建永久个人博客
date: 2024/5/18 15:00:00
---

# 环境搭建

- [git](https://git-scm.com/)
- [nodejs](https://nodejs.org/)

# 部署

注册一个 [GitHub](https://github.com/) 账号

按照官方文档操作: [在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages)

# 常见问题(官方文档没提及的)

- 默认分组(main 或 master)不要改名, 否则 pages 部署 时会出现 `Branch "main" is not allowed to deploy to github-pages due to environment protection rules.` 问题, 最简单的解决方法是删库重新创建

- npm 下载慢 或 下载失败, 参考 [nrm 教程](https://zhuanlan.zhihu.com/p/636455443), 换国内镜像源

# 进阶

- [nvm - 一个 nodejs 版本管理工具](https://nvm.uihtm.com/): 跟上面的 nodejs 二选一, 如果已经安装 nodejs, 先卸载原来的 nodejs
- [pnpm - 速度快、节省磁盘空间的软件包管理器](https://www.pnpm.cn/): 代替 npm

本项目包管理器使用的是 pnpm, workflow 里的 pages.yml 是 pnpm 版
