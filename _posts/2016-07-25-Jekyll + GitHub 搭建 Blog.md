---
layout: post
title: "Jekyll + GitHub 搭建 Blog"
category: Linux 
---

#Jekyll + GitHub 搭建 Blog
##[Jekyll](http://jekyllcn.com/)
###Jekyll是什么
Jekyll 是一个简单的博客形态的静态站点生产机器。它有一个模版目录，其中包含原始文本格式的文档，通过一个转换器（如 Markdown）和我们的 Liquid 渲染器转化成一个完整的可发布的静态网站，你可以发布在任何你喜爱的服务器上。

### Jekyll的目录结构
```
├── _config.yml
├── _drafts
	├── begin-with-the-crazy-ideas.markdown
├── _includes
	├── footer.html
	├── header.html
├── _layouts
	|── default.html
	|── post.html
├── _posts
	|── 2007-10-29-why-every-programmer-should-play-nethack.textile
	|── 2009-04-26-barcamp-boston-4-roundup.textile
├── _site
├── .jekyll-metadata
└── index.html
```

##[Github](https://github.com/)
简单来说，Github是一个免费的代码托管平台。Blog在Git上的使用，详见[GitPage](https://pages.github.com/)。

