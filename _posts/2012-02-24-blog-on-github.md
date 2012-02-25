---
layout: post
title: 用github写博客
---

{{ page.title }}
================

<p class="meta">2012-02-14 23:30 - 广州</p>

用github写博客美中不足的是博客生成器只能选jekyll, 而jekyll是ruby实现的, 目前自己又不想学习ruby。

jekyll post文件有两部分组成:

1. YAML格式的jekyll header
2. 文章正文

> 文章正文是一个Liquid template。生成静态文件时，jekyll先解析jekyll 头, 然后使用解析出来的变量把文章正文渲染成纯文本。然后把生成的纯文本再使用markdown,或者textile解析器生成html。


* YAML Front Matter: [https://github.com/mojombo/jekyll/wiki/yaml-front-matter](https://github.com/mojombo/jekyll/wiki/yaml-front-matter)
* Liquid: [http://liquidmarkup.org/](http://liquidmarkup.org/)
