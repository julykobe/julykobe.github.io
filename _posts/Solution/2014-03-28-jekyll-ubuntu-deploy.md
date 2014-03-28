---
layout: post
title: jekyll本地环境搭建(Ubuntu 12.04)
category: 解决方案
tags: jekyll
keywords: jekyll,ruby,rvm,gihub pages
description: Jekyll是一个静态站点生成器，它会根据网页源码生成静态文件.
---

最近在github上搭了个博客，用的是github提供的pages功能，这个功能允许用户自定义项目首页来替代默认的源码列表。也就是说，可以用这个功能来实现一个托管在github上的静态网页。github提供模板，允许站内生成网页，但也允许用户自己编写网页，然后上传，经过Jekyll程序的再处理后显示。Jekyll是一个静态站点生成器，它会根据网页源码生成静态文件。它提供了模板、变量、插件等功能，所以实际上可以用来编写整个网站。

之前搭环境总是出错，因此这里记录一下我的搭建过程。


系统是Ubuntu 12.04.

1.安装rvm(ruby version manager)： 

	bash -s stable < <(curl -s https://raw.github.com/wayneeseguin/rvm/master/binscripts/rvm-installer)

2.设置classpath： 

	echo '[[ -s "$HOME/.rvm/scripts/rvm" ]] && . "$HOME/.rvm/scripts/rvm" # Load RVM function' >> ~/.bash_profile  
	source ~/.bash_profile

3.安装ruby：

	rvm install 1.9.2 && rvm use 1.9.2  
	rvm rubygems latest  
	sudo apt-get install ruby1.9.1-dev  
4.安装jekyll：
	sudo gem install jekyll  
5.本地调试： 
在项目目录下
	$ jekyll server 
即可通过http://localhost:4000来访问自己的blog了