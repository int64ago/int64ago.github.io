---
layout:     post
title:      SwitchyOmega 规则同步方案
date:       2015-01-22 22:49:00
summary:    针对个人多台设备需要同步科学上网列表的情况
---

[SwitchyOmega][1]是之前掉渣天的[SwitchySharp][2]重构版
其实，这个Switchy的历史还是怪坎坷的：Switchy-->Switchy Plus-->SwitchySharp-->Switchy Omega，都是GFW逼的
除了UI看着舒服了还提供了很多实用的[新特性][3]：

 - 支持需要用户名密码验证的代理服务器
 - 更灵活的代理配置：代理情景模式、多个自动切换模式以及多个规则列表
 - 新增多种切换条件类型，并改进原有的切换条件
 - PAC 脚本生成和切换的性能优化
 - 崭新的选项页面和下拉菜单，用户体验更佳
 - 可使用快捷键(Alt+Shift+O)和鼠标右键菜单
 - 许多错误修复以及改进。测试更充分
 - 等等

但是对于个人有多个设备的情况，很可能会有这种需求：希望多个设备**科学上网列表**同步
当然官方[有讨论过][4]是否支持配置同步，从技术上来说这个不难，但是考虑到有些人不同设备环境不一致，配置同步会混乱，可能要做一个选项（目前还未完工？）
但是，如果不需要同步除**科学上网列表**以外的配置呢？

我的做法是清空手动添加的规则，**仅仅**使用自己制定的在线规则，这样修改只需要修改一个文件即可
这是我托管的个人gfwlist：https://github.com/int64ago/private-gfwlist
配置就这样：
![此处输入图片的描述][5]

每次修改后git push即可，还是很方便的

  [1]: https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif
  [2]: https://chrome.google.com/webstore/detail/proxy-switchysharp/dpplabbmogkhghncfbfdeeokoefdjegm
  [3]: https://github.com/FelisCatus/SwitchyOmega/wiki/SwitchyOmega-%E6%96%B0%E5%8A%9F%E8%83%BD
  [4]: https://github.com/FelisCatus/SwitchyOmega/issues/7
  [5]: https://cdn.int64ago.org/o_19c91n3akuhs1m8aut1qt01upq9.png
