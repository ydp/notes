---
title: Hexo 搭建博客
date: 2016-09-07 12:26:09
categories: 备忘
tags: deprecated
---
## 安装hexo

``` bash
sudo npm install -g hexo --no-optional
```
在mac上安装需要添加 --no-optional

参考: [HEXO+Github,搭建属于自己的博客](http://www.jianshu.com/p/465830080ea9)

## 安装pacman

``` bash
git clone https://github.com/A-limon/pacman.git themes/pacman
```
参考: [Pacman主题介绍](https://yangjian.me/pacman/hello/introducing-pacman-theme/)

## 设置代码高亮

``` javascript
<link rel="stylesheet" href="//cdn.bootcss.com/highlight.js/9.2.0/styles/github.min.css">
<script src="//cdn.bootcss.com/highlight.js/9.2.0/highlight.min.js"></script>

hljs.initHighlightingOnLoad();
```

参考: [Hexo高级教程之代码高亮](http://www.ieclipse.cn/2016/07/18/Web/Hexo-dev-highlight/)

highlight js 高亮主题预览: [highlight demo](https://highlightjs.org/static/demo/)

## 写文章

``` bash
categories:
tags:

<!--more-->
```
