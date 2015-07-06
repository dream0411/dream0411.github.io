---
layout: default
title: Git常用命令
---

{{ page.title }}
===

对于一个版本控制工具，我希望得到几项主要功能：

- 创建、删除仓库
- 检出已有仓库
- 为仓库添加、删除文件或文件夹
- 修改文件内容
- 对比修改前后内容
- 查看历史提交
- 检出历史版本
- 在历史版本或记录搜索关键字或代码
- 多人合作编辑，并能处理冲突

除此之外还有些其他细节需求，这些应该在上面的主要功能中作为辅助。

带着对这些功能的需求，开始记录Git的常用命令。

*后面是我的一些使用记录，只实验过这些方法可行，不一定是最好最合适的方法，请读者仅供参考。*

### 创建仓库
先登录Github，创建一个新仓库hobby:
![在Github上创建新仓库]({{ site.baseurl }}/img/blog/create-rep.jpg)

然后本地打开命令行，找到创建一个新的目录，输入下列命令：

```
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
```

### 检出已有仓库
Git通常将检出叫做clone；
进入一个目录，比如/home/me/gitreps，然后执行：
	git clone https://github.com/dream0411/hobby
然后/home/me/gitreps/hobby目录下就会包含代码库的所有内容了

### 删除仓库
需要登录到Github网站上自己的管理页面中删除代码库，本地的目录自己删除就可以了。

### 添加文件、文件夹
在hobby目录下新增singing文件、language/cpp、language/go后，将这三个文件都添加到代码库中：

```
git add singing language
git commit -m "Add singing and language forder"
git push
```

### 删除文件、文件夹
删除已在仓库中的singing文件

```
git rm singing
git commit -m "forget it"
git push
```

### 修改文件内容
修改已跟踪的文件ff.c后

```
git add ff.c
git commit -m "add ff.c"
git push
```

### 查看历史提交
	git log

### 检出历史版本

1. 恢复到倒数第二次提交的版本
有时候刚提交完，发现问题，需要撤销回去，执行以下命令会撤销刚刚提交的全部内容:

```
git reset --soft HEAD^
git reset --hard
```
然后做git status，会发现这次提交的记录和log都没了。

2. 恢复到任意版本
找到以前的某个版本(commit的id)后，比如c78e2ee3c43245c252fe379652f7ba8545b417e4，我们想更新到这个commit，但是注意，执行操作前，这个目录不能有未提交的修改，如果有未提交的内容，但是又不想现在就提交，可执行下列命令：

```
// 先记下当前的commit id比如d723a99d32f5022fb8b510a31a2ebada9312a44d
git log -1

// 如果我们有未提交的修改，用这个把这些修改暂存起来，用于稍后恢复
git stash

// 将整个代码库恢复到c78e2e
git checkout c78e2e

// 自己想做的一系列操作
...

// 收尾工作，恢复到我们原来的commit id
git checkout d723a

// 恢复我们之前暂存的那些未提交代码
git stash pop
```

3. 仅恢复某个文件
两种方式：

将文件README恢复到历史版本:

	git checkout c78e2e -- README

将文件README历史版本显示出来，可以用文件重定向操作符写至另一个文件：

	git show c78e2e:README > README-old

### 在历史版本或记录搜索关键字或代码

	git grep "SomeCode" $(git rev-list --all)
	
可以用正则、and、or等多种方式匹配，git help grep有多种选项说明。

### 多人合作编辑，并能处理冲突
需要主要分两种情况：

1. 自己本地没有任何修改，直接从远程更新

	git pull

2. 自己从版本x修改，远程也有人从x版本修改过，并且已经提交至版本x+1，我们想获取远程的代码，并将自己的代码合并至x+1，再提交至x+2

```
git stash	// 把本地的修改先暂存起来
git fetch	// 获取远程的新版本y到某个缓存位置(本地缓存的远程仓库，大概是这么个意思)，但还不合并到自己的本地代码中
git rebase	// 把自己本地代码的基准版本从x重新指向刚刚更新下来的y，并把y更新到本地代码中
git stash pop	// 把第一步暂存的本地修改自动合并到本地代码中；合并后可能有冲突
// 如果有冲突文件，手动编辑冲突文件的位置，这些冲突已经被git在源文件中标记出来了
git add somefile.c // 后面就是把冲突文件按照正常流程add, commit, push了
git commit -m "update and rebase"
git push
```

这样提交后，会看到这三个提交是在一条线上的。

### 比较差异

1. *git diff* 用于比较当前未git add的修改与父提交之间的差异
2. *git diff --cached* 用于比较已git add的，下次git commit时的修改内容
3. *git diff HEAD* 比较的是当前所有已修改但为git add的内容，这些内容在git commit -a时会被提交
4. *git diff master..test* 比较两个分支
5. *git diff master...test* 比较master和test的共同父分支与test分支的差异
6. *git diff test* 比较当前分支和另一分支的差异
7. *git diff [commit1] commit2 -- files* 比较两个提交commit1和commit2之间的差异，如果省略一个，则比较的是当前提交与另一提交间的差异；如果指定files，则之比较这些文件的差异
