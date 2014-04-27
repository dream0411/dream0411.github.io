---
layout: default
title: Git概览 (TODO)
---

{{ page.title }}
===

对于一个版本控制工具，我希望得到几项主要功能：

- 创建、删除仓库
- 为仓库添加、删除文件或文件夹
- 修改文件内容
- 对比修改前后内容
- 查看历史提交
- 检出历史版本
- 在历史版本或记录搜索关键字或代码
- 多人合作编辑，并能处理冲突

除此之外还有些其他细节需求，这些应该在上面的主要功能中作为辅助。

带着对这些功能的需求，开始对Git的学习，下面使用Github服务器。

### 创建仓库
先登录Github，创建一个新仓库hobby:
![在Github上创建新仓库]({{ site.baseurl }}/img/blog/create-rep.jpg)

然后本地打开命令行，找到创建一个新的目录，输入下列命令：

    mkdir hobby
    cd hobby
    git init
    git remote add origin https://github.com/dream0411/hobby.git
    echo like > riding-bike
    echo like > movies
    echo love > coding
    echo "Sample pro for learing git" > README.md
    git add riding-bike moview coding README.md
    git commit -m "Initial commit"
    git push -u origin master


### 删除仓库
TODO

### 添加文件、文件夹
TODO

### 删除文件、文件夹
TODO

### 修改文件内容
修改已跟踪的文件ff.c后
    git add ff.c
    git commit -m "add ff.c"
    git push

### 查看历史提交
TODO

### 检出历史版本
TODO

### 在历史版本或记录搜索关键字或代码
TODO

### 多人合作编辑，并能处理冲突
TODO

