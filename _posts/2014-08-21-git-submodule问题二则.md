---
layout:     post
title:      git submodule问题二则
date:       2014-08-21 09:00:00
summary:    很多时候正是看似等价的东西坑了自己，git submodule的一些概念不清发生的错，记下排错过程
tags:       问题解决
---

前一段时间又把[pro git][1]完整看了一遍，结合平时用到的项目，发现git submodule这东西确实强大，用依赖解决问题貌似是现在主流且优雅的做法：

 - linux各发行版包管理机制
 - perl的CPAN
 - maven项目构建方式
 - ruby的rubygems
 - 等等等

于是考虑把我现有的项目用git submodule重新构建下
当然以上都是题外话了……


----------
将各个子模块测试通了之后推送到远程仓库，然后按照官方的做法，在主模块执行git submodule add xxx 之类的加入子模块，`联调了下，发现ok`

就这样项目放一边，很久之后，在一台全新机器上clone回来，init submodule后，编译没问题，但是编译`依赖此的程序就报错`，于是我按照下面的方式测试：

 - 进入submodule文件夹，切回到没改动的状态，发现还是编译不！
 - 在别的目录，重新从官网clone个版本一样的，一切正常
 - 然后一个一个文件比较，发现confiure.ac里的版本号是unknown

于是想通了为什么`依赖此的程序就报错`了，因为找不到符合条件的版本！
那么为什么之前`联调了下，发现ok`呢？因为在没有构建submodule之前就在那台机器上编译运行过，unknown的版本没有覆盖有版本号的，所以之前的一切正常其实是用的旧的程序版本！

那么接下来就是搞清，为什么版本号会是unknown，通过跟踪，发现版本的生成是由git-version-gen这个脚本，仔细看下这个脚本，开始有句注释：

```bash
# BUILT_SOURCES = $(top_srcdir)/.version
# $(top_srcdir)/.version:
#       echo $(VERSION) > $@-t && mv $@-t $@
# dist-hook:
#       echo $(VERSION) > $(distdir)/.tarball-version
```

脚本大概就是这样做的：如果是tarball，则根据tarball里的.tarball-version来生成version，否则根据.git/下的tag信息生成，既然如此，为什么没有生成版本号呢？

进去submodule看了下，居然没有.git目录！！
后来发现，submodule的仓库全部在主仓库的.git/modules下！
就因为我的一个理所当然的以为submodule的行为和普通仓库的行为差不多，调试了一整天……

上面的当然是问题一了，问题二比较简单，我理所当然的以为init submodule的时候默认clone的是submodule的master分支，当然后面也知道了，仅仅是add的时候的commit

之所以能让我产生上面两个理所当然，归结于[pro git][1]上的“误导”：

```bash
$ git submodule add git://github.com/chneukirchen/rack.git rack
Initialized empty Git repository in /opt/subtest/rack/.git/
remote: Counting objects: 3181, done.
remote: Compressing objects: 100% (1534/1534), done.
remote: Total 3181 (delta 1951), reused 2623 (delta 1603)
Receiving objects: 100% (3181/3181), 675.42 KiB | 422 KiB/s, done.
Resolving deltas: 100% (1951/1951), done.
```

```bash
$ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#      new file:   .gitmodules
#      new file:   rack
#
```

```bash
$ cat .gitmodules
[submodule "rack"]
      path = rack
      url = git://github.com/chneukirchen/rack.git
```

看出来了吗？表象告诉我们，add 一个 submodule只是简单的往.gitmodules里加入了path&url
  [1]: http://git-scm.com/book/
