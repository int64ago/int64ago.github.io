---
layout:     post
title:      spice-gtk文件拖拽问题patch四个
date:       2014-08-08 09:00:00
summary:    spice-gtk对于文件拖拽有几个小问题，针对这些问题提供分析及相关patch
tags:       虚拟化
---

spice-gtk对于文件拖拽有几个小问题：

 - 文件名包含空格，拖过去会被截断
 - 文件名包含中文（包括全角字符），拖过去乱码
 - 文件名包含[，无法拖动
 - 空文件拖过去打开会显示被占用

现就~~1、2、3~~这些问题提供分析及相关patch，1、2其实是一个问题~~，4问题暂没解决~~


----------


**中文乱码问题**
-------------

因为1、2是一个问题，就直接按照2说好了
开始一直以为是spice-gtk获取文件名之后编码问题，之前为社区提交了一个[patch][1]，也讨论了一番，最终没有采纳，虽然解决了问题，但是现在想想，其实只是在某种特定情况下才会成功，后面解释

spice-gtk在拖拽文件的时候会把文件相关信息通过glib的`g_key_file_set_string()`转为`GKeyFile`，然后通过`g_key_file_to_data()`序列化为字符串，样子如下：
[vdagent-file-xfer]
name=test.txt
size=10

发送到agent端，agent解析后，获取桌面路径，在桌面创建了name所描述的文件，问题就在于`g_key_file_to_data()`获得的序列是UTF-8编码，而创建函数却用了`CreateFileA()`，然后你懂的，下面分别为中文和英文系统显示的情况：

![](https://dn-getlink.qbox.me/2014/08/10/5d37e86b-1ff2-11e4-8bf4-4f12170170a3.jpg)

跟系统的语言还是有关系的
那么只需要把multichar 转为 widechar即可，
见http://lists.freedesktop.org/archives/spice-devel/2014-August/017179.html
至此问题1也顺带解决了，原因还没仔细分析，反正跟编码有关
现在回想之前的patch为什么能解决问题？因为我在client端把编码转为gbk了，而测试的guest是中文windows，于是成功了

**带[文件名无法拖动**
-----------

按照之前说的，keystring传到agent后，agent会通过`g_key_get_string()`解析得到文件名和大小，虽然glib自带`g_key_get_string()`，但是不知道vd_agent基于什么原因，没有使用glib库，于是自己实现了一个简化版本，就是这个简化版本导致了问题（其实glib自带的有没有问题我也不知道）
简化版本根据字符[判断当前group尾部，如果文件名包含这个的话，尾部就判断错了
于是我考虑可以把[ ] 换成 < >，因为windows不允许文件名包含这些，于是spice-gtk和vd_agent两端都要修改下了
patch如下，分别应用于vd_agent和spice-gtk：
http://lists.freedesktop.org/archives/spice-devel/2014-August/017180.html
http://lists.freedesktop.org/archives/spice-devel/2014-August/017181.html

**空文件被占用**
------------
这个在邮件列表里我说的很清楚了，这里就不多分析了，移步：
http://lists.freedesktop.org/archives/spice-devel/2014-August/017184.html

**其它**
--

因为是交叉编译的，所以只能通过log来调试，这种过程是非常痛苦的，编译、打包、移动、测试非常耗时，算了一下，测试这几个问题总共编译的spice-gtk和vd_agent版本目测有几十个……

  [1]: http://lists.freedesktop.org/archives/spice-devel/2014-February/016156.html
