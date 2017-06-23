---
layout:     post
title:      dnscrypt+dnsmasq 构建防污染 DNS 缓存服务器
date:       2014-08-14 09:00:00
summary:    不能正常上网让我整个人都不好了……自定义 DNS 好处多多
---


[什么是DNS污染][1]
解决DNS污染能解决由于DNS污染导致的被墙问题，事例：
[微软OneDrive疑被屏蔽][2]
[雅虎Flickr被屏蔽][3]
[韩国版微信Kakao疑被屏蔽][4]
[之前流行的google hosts “翻墙”][5]
解决DNS污染不能解决：

 - 封IP导致的被墙问题：六月以来Google问题
 - 封关键词/url导致的被墙问题：[Line被关键字屏蔽][6]
 - 开心一刻——>[sinaapp.co躺枪][7]

如果遇到上面问题，请自行走海外线路vpn/proxy等，不过注意安全问题，GoAgent为例：[GoAgent的安全风险][8]


本文下面只介绍如何解决DNS污染投毒，当然个人[个人PC解决方式][9]还是很简单的，于是详细说下局域网如何解决

两个工具：
[dnscrypt-proxy][10]：通过加密线路请求DNS解析
[dnsmasq][11]：DNS缓存服务器

假设局域网机器10.0.0.8，DNS服务器10.0.0.2
其中
dnscrypt-proxy监听127.0.0.1:35535
dnsmasq监听10.0.0.2:53

当我们发起一次DNS请求时，首先经过10.0.0.2:53向dnsmasq请求，如果cache里没有，dnsmasq会通过127.0.0.1:35535向dnscrypt-proxy请求，dnscrypt-proxy通过加密线路向世界各地提供此服务的DNS服务器请求解析

大概流程如上，其中涉及的关键技术官网都有，
下面是些配置及测试(基于archlinux)：
因为dnsmasq依赖dnscrypt-proxy，可以先配置dnscrypt-proxy
`pacman -S dnscrypt-proxy`
`cat /etc/conf.d/dnscrypt-proxy`

```bash
DNSCRYPT_LOCALIP=127.0.0.1
DNSCRYPT_LOCALPORT=35535
DNSCRYPT_USER=nobody
DNSCRYPT_PROVIDER_NAME=2.dnscrypt-cert.opendns.com
DNSCRYPT_PROVIDER_KEY=B735:1140:206F:225D:3E2B:D822:D7FD:691E:A1C3:3CC8:D666:8D0C:BE04:BFAB:CA43:FB79
DNSCRYPT_RESOLVERIP=208.67.220.220
DNSCRYPT_RESOLVERPORT=443
```

`systemctl start dnscrypt-proxy && systemctl enable dnscrypt-proxy`
上面的provider可以[这里][12]查看，不过需要测试，很多不能用，测试：
dig -p 35535 google.com

dnsmasq：
`pacman -S dnsmasq`
`cat /etc/dnsmasq.conf`

```bash
server=127.0.0.1#35535
listen-address=10.0.0.2
```

`cat /etc/resolv.conf`

```
nameserver 127.0.0.1
```

`systemctl start dnsmasq && systemctl enable dnsmasq`

可以在10.0.0.8上测试了，`nslookup google.com`


最后，其实也可以通过unbound等优秀的软件实现，当然有时候dnscrypt也并不是很好用，因此没有洁癖的情况下dnsmasq也够的，可以结合[dnsmasq-china-list][13]


  [1]: http://en.wikipedia.org/wiki/DNS_spoofing
  [2]: http://www.williamlong.info/archives/3905.html
  [3]: http://www.williamlong.info/archives/3906.html
  [4]: http://www.williamlong.info/archives/3907.html
  [5]: https://code.google.com/p/smarthosts/
  [6]: http://www.williamlong.info/archives/3904.html
  [7]: http://s.weibo.com/weibo/sinaapp.co?topnav=1&wvr=5&b=1
  [8]: http://www.williamlong.info/archives/3882.html
  [9]: http://www.williamlong.info/archives/3890.html
  [10]: http://dnscrypt.org/
  [11]: http://www.thekelleys.org.uk/dnsmasq/doc.html
  [12]: https://github.com/jedisct1/dnscrypt-proxy/blob/master/dnscrypt-resolvers.csv
  [13]: https://github.com/felixonmars/dnsmasq-china-list
