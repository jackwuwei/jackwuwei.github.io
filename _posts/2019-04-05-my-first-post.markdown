---
layout:     post
title:      "就这样开始吧"
subtitle:   " \"Hello World, Github Pages\""
date:       2019-04-05 2:00:00
author:     "Jack"
header-img: "img/post-bg-2015.jpg"
tags:
    - 生活
---
## 前言
我的博客就这样开通了。之前一直使用Wordpress搭建个人博客站点，建站比较复杂，模板难看，很多垃圾评论，迁移服务器特别麻烦，一直知道Github可以托管页面，趁假期有空，发现非常简单，还能绑定自己的域名，通过Let's Encrypt实现HTTPS，非常方便。

## 正文
接下来说说搭建这个博客的技术细节。  

正好之前就有关注过 [GitHub Pages](https://pages.github.com/) + [Jekyll](http://jekyllrb.com/) 快速 Building Blog 的技术方案，非常轻松时尚。

其优点非常明显：

* **Markdown** 带来的优雅写作体验
* 非常熟悉的 Git workflow ，**Git Commit 即 Blog Post**
* 利用 GitHub Pages 的域名和免费无限空间，不用自己折腾主机
	* 如果需要自定义域名，也只需要简单改改 DNS 加个 CNAME 就好了
* Jekyll 的自定制非常容易，基本就是个模版引擎
配置的过程中也没遇到什么坑，基本就是 Git 的流程，相当顺手

Jekyll 主题使用了[Hux的模板](http://huangxuan.me/)。本地调试很方便，大致方法如下：
1. 安装[ruby](https://rubyinstaller.org/downloads/)，注意PATH路径的设置；
2. `gem install github-pages`
3. `cd your_site_directory`
4. `jekyll serve`
