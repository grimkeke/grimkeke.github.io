---
layout: post
title:  如何创建自己的github.io站点
date:   2017-03-26 16:59:44 +0800
categories: hello
---

github提供了一种[主页服务][github-page]，当用户在github创建了一个(用户名.github.io)格式命名的项目后，
github会为其生成静态站点。 比如我的用户名为grimkeke，那么当我创建了grimkeke.github.io项目，并提交一个文本文件（如html）后，
访问[https://grimkeke.github.io](https://grimkeke.github.io)即可访问刚才提交的文件，需要注意项目名称必须为自身用户名，否则即使上传了文件，浏览器也会报错404.

对于正式编写blog，推荐使用[Jekyll][jekyll-url]，它支持markdown编写笔记，通过命令生成静态文件，并且与github完全打通，
通过jekyll创建的项目，直接上传至github后即为最终站点效果。

#### 如何使用jekyll
1. Install Jekyll and Bundler gems through RubyGems
```gem install jekyll bundler```

2. Create a new Jekyll site at ./myblog
```jekyll new myblog```

3. Change into your new directory
```cd myblog```

4. Build the site on the preview server
```bundle exec jekyll serve```

5. Now browse to http://localhost:4000

[jekyll-url]: https://jekyllrb.com/docs/quickstart/
[github-page]: https://pages.github.com/


