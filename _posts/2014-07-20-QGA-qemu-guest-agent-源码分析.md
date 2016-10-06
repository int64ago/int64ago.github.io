---
layout:     post
title:      QGA(qemu guest agent)源码分析
date:       2014-07-20 09:00:00
summary:    qga目前是qemu的一部分，用于guest和host通信，因为qga在guest里仅仅是一个平常的进程（服务），所以，理论上，任何拥有guest机器**管理员/root**权限的人都可以终止它！因此，最好不要将这个作为关键技术的一部分，不过用于增强geust的友好性体验还是很不错的
---

qga目前是qemu的一部分，用于guest和host通信，因为qga在guest里仅仅是一个平常的进程（服务），所以，理论上，任何拥有guest机器**管理员/root**权限的人都可以终止它！因此，最好不要将这个作为关键技术的一部分，不过用于增强geust的友好性体验还是很不错的。


----------

这里以<2.0>为例，分析下qga的源码及原理，不过，与其说是源码分析，不如说是框架介绍，对于其拓展方面内容也会说点。

`qemu/qga/qapi-schema.json`里，按照json的格式添加需要的变量类型和函数
函数是command开头，变量是enum或者type开头
编译期间由`qemu/scripts/qapi*.py`脚本生成相应的类型的实现
输出目录为`qemu/qga/qapi-generated/`

主要产生规则:
 - 如果是变量类型XXOO，则会生成下面2个类型和5个对应操作实例

```
XXOO     //用于返回单个值
XXOOList     //用于返回值列表
//以下为内部接口需要
visit_type_XXOO
visit_type_XXOOList
qapi_free_XXOO
qapi_free_XXOOList
visit_type_XXOO_fields
```

 - 如果是函数类型，则会生成如下函数:

```
qmp_marshal_output_xxoo
qmp_marshal_input_xxoo
qmp_xxoo
```

其中qmp_xxoo即为增加的结构，需要在`qemu/qga/commands*.c`里人肉实现，`qmp_marshal_output/input_xxoo`与qapi通信

另外，还有些“不太重要”的文件

```
qemu/qga/channel*    //用于建立串口通信链接
qemu/qga/service-win32*    //windows下以服务呈现的实现
qemu/qga/guest-agent*    //qapi框架上层统筹
qemu/qapi/*    //框架核心实现
qemu/qga/vss-win32*    //为windows功能拓展，暂不需关注
```

最后，如何增加一个自定义功能呢？

 - 在`qapi-schema.json`按照json格式增加数据格式和函数的定义
 - 在`commands*.c`实现相关接口
 - 如果增加了源码文件，需要在`Makefile.objs`里增加链接规则
 - 编译，分为linux和windows，具体见[这里][1]，如果切换编译规则，一定记得make clean


  [1]: http://wiki.qemu.org/Features/QAPI/GuestAgent
