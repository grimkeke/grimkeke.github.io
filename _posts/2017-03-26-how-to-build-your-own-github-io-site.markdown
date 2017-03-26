---
layout: post
title:  如何创建自己的github.io站点
date:   2017-03-26 16:59:44 +0800
categories: hello
---

## 玩法
github提供了一种[主页服务][github-page]，当用户在github创建了一个(用户名.github.io)项目后，github会为其生成github.io的静态站点。
比如我的用户名为grimkeke，那么当我创建了grimkeke.github.io项目后，提交一个文本文件（如html）后，
访问https://grimkeke.github.io即可访问刚才提交的文件，需要注意项目名称必须为自身用户名，否则即使上传了文件，浏览器也会报错404.

对于正式编写blog，推荐使用Jekyll[jekyll-url]，它支持markdown编写笔记，通过命令生成静态文件，并且与github完全打通，
通过jekyll创建的项目，直接上传至github后即为最终站点效果。

[jekyll-url]: https://jekyllrb.com/docs/quickstart/
[github-page]: https://pages.github.com/


