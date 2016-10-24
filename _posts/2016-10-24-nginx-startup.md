---
layout: default
title: nginx启动流程
---

{{ page.title }}
===

我在浏览了nginx源码后，得到图中nginx的大致启动流程图；由于单张图的图元数量限制(免费版的Lucidchart)，部分单个流程图过程包含了多个过程，需要简单的条件判断流程则增加了前缀“IF”及其条件变量(ngx前缀的全局变量或示意的单词)。

[nginx启动流程图]({{ site.baseurl }}/img/blog/nginx-startup.jpg)

