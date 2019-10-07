---
title: Git笔记
date: 2016-11-27 22:01:11
tags: 
 - git
categories:
 - 开发工具 
---
因为项目中一直使用Svn作为版本控制工具，接触Git的机会不多，所以每次使用，都要去查询命令，这里记录下一些常用的Git命令，方便查询。

<!-- more -->

从阮一峰老师的博客中看到一幅关于Git命令的图片，感觉很赞！
![](http://image.beekka.com/blog/2014/bg2014061202.jpg)

## Git远程操作相关
### git clone
完整的命令是：
``` git
git clone -o <远程仓库名> <版本库地址> <本地目录名>
```
该命令会从远程主机克隆一个版本库，默认远程仓库名为`origin`，但是我们可以通过`-o`参数指定想要的仓库名。

默认情况下，通过clone方法仅仅克隆了远程仓库的master分支，若我们想克隆远程仓库的其他分支，可以通过：
``` git
git checkout -b <本地分支名> <远程仓库名>/<远程分支名>
```
表示在本地创建和远程分支对应的分支，本地和远程分支的名称最好一致。

### git remote
git remote命令主要用于管理远程仓库。

``` git
git remote : 列出所有远程仓库名
git remote -v : 同时列出远程仓库的地址
git remote show <远程仓库名> : 查看远程仓库的详细信息
git remote add <远程仓库名> <远程仓库地址> : 添加远程仓库
git remote rm <远程仓库名> : 删除已有的远程仓库
git remote rename <原远程仓库名> <新远程仓库名> : 修改远程仓库名
git remote set-url --push [远程仓库名] [newServerUrl] : 修改远程仓库的地址
```

### git pull
完整的命令是：
``` git
git pull <远程仓库> <远程分支名>:<本地分支名>
```
表示把远程仓库某个分支的更新拉取到本地，然后和本地分支合并(merge)

若省略本地分支名，则表示拉取下来的远程分支和当前分支合并。
例如：
``` git
git pull origin dev
```
该命令表示拉取`origin/dev`分支，然后与当前分支合并。实际上，这等同于先做`git fetch`，再做`git merge`。

在Git中，可以在远程分支和本地分支之间建立一种追踪关系（tracking）。例如：执行`git clone`的时候，默认所有本地分支与远程仓库的同名分支,自动建立这种追踪关系。即：本地的master分支自动"追踪`origin/master`分支。

当然，我们也可以手动建立追踪关系：
``` git
git branch --set-upstream <本地分支名> <远程仓库/<远程分支名>
```

如果本地分支和远程分支已经建立了追踪关系，那么git pull就可以省略远程分支名了，即：
``` git
git pull origin
```
该命令表示，本地的当前分支自动与对应的origin远程仓库的追踪分支（remote-tracking branch）进行合并。

若当前分支只有一个追踪分支，那么连远程仓库名都可以省了。即：
``` git
git pull
```

### git push
完整的命令是：
``` git
git push <远程仓库> <本地分支名>:<远程分支名>
```
表示把本地分支的更新，推送到远程仓库的某个远程分支。

若省略远程分支名，则表示把本地分支的更新推送到与本地分支同名的远程分支，若远程仓库不存在和本地分支同名的分支，则会创建该远程分支。
``` git 
git push origin dev
```
该命令表示把本地dev分支推动到origin仓库的dev分支，如果后者不存在，则会被创建。

若省略了本地分支名，则表示删除指定的远程分支，这等同于推送一个空的本地分支到远程分支。即：
``` git
git push origin :dev
git push origin --delete dev
//这两者都表示删除origin仓库dev分支
```

若远程仓库存在和本地分支同名的分支，那么可以省略本地分支名，即：
``` git
git push origin
```
该命令表示把本地分支推动到origin仓库同名的分支。

若本地仓库仅关联了一个远程仓库，那么连远程仓库名都可以省略。

最后，git push不会推送标签（tag），除非使用--tags选项，即：
``` git
git push origin --tags
```
### git fetch
完整的命令是：
``` git
git fetch <远程仓库名> <远程分支名>
```
表示取回远程仓库某个分支的更新到本地，与git pull不同，取回的远程分支的更新不会影响到本地代码。

若省略远程分支名，则表示取回远程仓库所有分支的更新。取回的远程分支，在本地通过 <远程仓库名>/<远程分支名>来访问。

例如：把取回的远程分支合并到本地当前分支：
``` git
git merge origin/dev
```
该命令表示把origin仓库的dev分支合并到当前本地分支。

可见`git fetch`+`git merge`正好完成了`git pull`的工作。

#### 同步一个fork
一般情况下，我们fork一个仓库后，需要经常从原仓库同步最新的代码过来，这里记录下同步一个fork的步骤：

1. 把原远端仓库添加为本地仓库的远端仓库，即：
``` git
git remote add <远端仓库名> https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git
```
2. 拉取原远端仓库的分支到本地，即：
``` git
git fetch <远端仓库名>
```
3. 把原远端分支合并到本地分支，即
``` git
git merge <远端仓库名>/<远端分支名>
```
4. 推送本地分支到自己fork的远端仓库，即
``` git
git push <fork的远程仓库名> [远端分支名]
```

## git 分支相关

1. git branch <本地分支名> : 创建一个新的分支
2. git branch -d <本地分支名> : 删除某个分支
3. git branch [-a|-r] : 查看分支
4. git branch <本地分支名> : 创建分支
5. git checkout <本地分支名> : 切换分支
6. git checkout -b <本地分支名> : 创建+切换分支
7. git merge xxx : 合并某分支到当前分支
8. git checkout -b <本地分支名> <远程仓库>/<远程分支名> : 在本地创建和远程分支对应的分支

## Git 日志和版本管理

1. git log : 查看commit记录
2. git reflog : 查看每一次git操作
3. git reset --hard commitId  : 回退到某个版本（HEAD表示当前版本，上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100）
4. git checkout -- file  :  丢弃工作区的修改
5. git reset HEAD file : 把暂存区的修改撤销掉（unstage），重新放回工作区（应该是把指定的file文件恢复到HEAD版本的样子，把缓存区file文件和HEAD版本file文件作对比，把差异恢复到工作区，同时清除了缓存区对file文件的修改）。就到了上一步了，视情况来看是否需要清除工作区修改。
6. git reset --hard commitId : 回退到某个版本，即丢弃某次已提交到版本库的修改（会同时重置工作区和缓存区）。

## Git 标签

1. git tag <tagname>用于新建一个标签，默认为HEAD，也可以指定一个commit id
2. git tag -a <tagname> -m "xxx"用于指定标签信息
3. git tag : 查看所有标签
4. git show <tagname> 查看某个标签
5. git push origin <tagname> : 推送一个本地标签到origin仓库
6. git push origin --tags : 推送全部未推送过的本地标签到origin仓库
7. git tag -d <tagname> : 删除一个仓库
8. git push origin :refs/tags/<tagname> : 删除一个远程标签
9. git pull origin —tagst : 合并远程origin仓库的标签到本地

## Git 配置
完整命令：
``` git
git config [--global] key value
```
其中`--global`可以省略，若省略则表示配置只对当前仓库起作用，否则则对所有仓库起作用。

仓库配置文件位置： 本地仓库地址/.git/config文件
全局配置文件位置：~/.gitconfig文件

例如：
``` git
//配置用户名和邮箱
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
//配置别名 git st == git status
git config --global alias.st status
```
