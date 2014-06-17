---
layout: default
title: Git常用命令 (TODO)
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
TODO

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
TODO

3. 仅恢复某个文件

### 在历史版本或记录搜索关键字或代码
TODO

### 多人合作编辑，并能处理冲突
需要主要分两种情况：

1. 自己本地没有任何修改，直接从远程更新

	git pull

2. 自己从版本x修改，远程也有人从x版本修改过，并且已经提交了，我们想远程的和自己的合并

	git stash	// 把本地的修改先暂存起来
	git fetch	// 获取远程的新版本y到某个缓存位置(本地缓存的远程仓库，大概是这么个意思)，但还不合并到自己的本地代码中
	git rebase	// 把自己本地代码的基准版本从x重新指向刚刚更新下来的y，并把y更新到本地代码中
	git stash pop	// 把第一步暂存的本地修改自动合并到本地代码中；合并后可能有冲突
	// 如果有冲突文件，手动编辑冲突文件的位置，这些冲突已经被git在源文件中标记出来了
	git add somefile.c // 后面就是把冲突文件按照正常流程add, commit, push了
	git commit -m "update and rebase"
	git push

