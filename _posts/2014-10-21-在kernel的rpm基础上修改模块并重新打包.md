---
layout:     post
title:      在kernel的rpm基础上修改模块并重新打包
date:       2014-10-21 09:00:00
summary:    最近需要修改linux kernel，因为是red hat系列，所以比较正统的方法应该是修改后打包成rpm，然后放到源里。其中，其中会涉及到一些问题，下面说说我的方案。当然，也可以单独编译成ko，写脚本在开机的时候替换下，不过这在生产环境中使用显得不是很合适……
---

最近需要修改linux kernel，因为是red hat系列，所以比较正统的方法应该是修改后打包成rpm，然后放到源里。其中，其中会涉及到一些问题，下面说说我的方案。当然，也可以单独编译成ko，写脚本在开机的时候替换下，不过这在生产环境中使用显得不是很合适……

----------
## 背景

我修改的内核版本是xx.yy_x86_64，需要增加一个自己的模块，并且修改其中一个已存在的模块使其依赖我增加的模块

## 源码

首先搞到对应版本的源码，这里不要去直接下载linux kernel source，因为无论是redhat/fedora...都对内核修改了，并且打了很多补丁，我们需要做的就是在此基础上修改。如果你的源内的内核版本就是你需要的，可以直接通过`yumdownloader kernel-xx.yy`下载到*.src.rpm包（否则，可以去rpmfind之类的网站找）。然后，`rpm -ivh *.src.rpm`会在~/rpmbuild下得到rpm的构建相关目录，当然~/rpmbuild/SOURCES会有kernel源码压缩包，但是我们不能直接修改这个，需要在~/rpmbuild/SPECS目录执行`rpmbuild -bp *.spec`，此过程会解压源码并应用spec里定义的补丁，然后生成到~/rpmbuild/BUILD目录，这里面的就是我们需要的源码！

## 补丁

因为内核的补丁是通过git format-patch 生成的，所以在进行修改前，需要通过`git init && git add . && git commit -m "init"`生成一个初始仓库，待修改完commit后可以方便的生成自己的补丁
具体修改及补丁生成不多说，比较关键一点是补丁应用的位置
生成的补丁应该在%build标签前ApplyPatch，因为spec脚本本身会做什么事情，如果在%prep中的ApplyPatch最后添加自定义补丁，很可能会出问题，具体什么问题在于你的修改

## 内核模块

内科模块的修改官方及网上的资料很多，可以自行参考，这里注意的是，如果模块之间有依赖，务必在Kconfig里写清楚

## 补丁应用

如果仅仅是修改模块，需要将生成的patch放到SOURCES目录，以及<补丁>部分所得spec的修改，最后`rpmbuild -ba --with baseonly --without debuginfo --target=`uname -m` kernel.spec`可在~/rpmbuild/RPMS里生成改好的rpm包，以及~/rpmbuild/SRPMS里对应的*.src.rpm包；
如果添加了模块，则需要在源码的configs目录下修改相应config配置参数指定增加的模块按什么方式编译，剩下的步骤同上

更多内容链接：
[参考1](http://linlog.blogspot.com/2010/01/add-new-config-patch-to-linux-kernel.html) [参考2](https://fedoraproject.org/wiki/Building_a_custom_kernel)

找src.rpm推荐链接：
[参考3](http://arm.koji.fedoraproject.org/koji/packages)
[参考4](http://cbs.centos.org/koji/packages)
[参考5](http://www.rpmfind.net/)
[参考6](http://rpm.pbone.net/)
