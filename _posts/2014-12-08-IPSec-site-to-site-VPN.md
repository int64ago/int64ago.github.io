---
layout:     post
title:      IPSec site-to-site VPN
date:       2014-12-08 09:00:00
summary:    正常点的环境配置一般也花不了多少时间，可是我碰到的问题就折腾我三天，记录下吧，可能有值得继续深入的地方呢
---

正常点的环境配置一般也花不了多少时间，可是我碰到的问题就折腾我三天
记录下吧，可能有值得继续深入的地方呢

----------

### 具体场景
- 网关A：对外IP为192.168.1.3(eth0)，对内IP为10.1.3.1(eth1)
- 网关B：对外IP为192.168.1.4(eth0)，对内IP为10.1.4.1(eth1)
- 网关A下有一个子网10.1.3.0/24，其中有PC-A机10.1.3.2
- 网关B下有一个子网10.1.4.0/24，其中有PC-B机10.1.4.2
- 在网关A和网关B之间建立IPSec VPN，使得10.1.3.0/24和10.1.4.0/24互通

说简单点，就是PC-A和PC-B能相互访问

关于IPSec，一般有传输模式和隧道模式，每种模式还有ESP和AH两种协议
具体可以参考[这个][1]，我觉得说得很明白

我的这个场景就是隧道模式+ESP，用的是openswan
其中一个网关：

`cat /etc/ipsec.conf`

```bash
version 2.0
config setup
        protostack=netkey
        nat_traversal=yes
        virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/16
        oe=off
conn secgw-vpn
        authby=secret
        auto=start
        type=tunnel
        ike=aes256-sha1;modp2048
        phase2=esp
        phase2alg=aes256-sha1;modp2048
        left=192.168.1.3
        leftsubnet=10.1.3.0/24
        right=192.168.1.4
        rightsubnet=10.1.4.0/24
```

`cat /etc/ipsec.secret`

```bash
192.168.1.3 192.168.1.4 : PSK "riest4ever"
```

`cat /etc/sysconfig/iptables`

```bash
#...
*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A POSTROUTING -s 10.1.3.0/24 -d 10.1.4.0/24 -j RETURN
-A POSTROUTING -s 10.1.3.0/24 -j MASQUERADE
#...
```

`cat /etc/sysctl.conf`

```bash
#...
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.log_martians = 0
net.ipv4.conf.default.log_martians = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
```

`service iptables restart && sysctl -p && service ipsec restart`

另外一个网关基本类似，3和4兑换下皆可

正常按照官方文档都不会出什么问题吧，不过我遇到了很奇怪的问题，配置好了后，始终无法工作
`ipsec auto --status`发现握手时成功了的
`ipsec verify`也都是OK，除了
`Checking NAT and MASQUERADEing                  [N/A]`
官方说是配置了转发就可以了，即：
`-A POSTROUTING -s xxxxx/xx -j MASQUERADE`
或者
`-A POSTROUTING -s xxxxx/xx -j SNAT --to-source xxxxx/xx`
可是我这样就是不行，网上几乎清一色也是这样的……
无意中[这里][2]看到了 -j RETURN，加上了之后，居然工作了（虽然`ipsec verify`还是显示的 N/A）

翻了iptables手册后是这样说的

> The RETURN target will cause the current packet to stop traveling through the chain where it hit the rule. If it is the subchain of another chain, the packet will continue to travel through the superior chains as if nothing had happened. If the chain is the main chain, for example the INPUT chain, the packet will have the default policy taken on it. The default policy is normally set to ACCEPT, DROP or similar.
> For example, lets say a packet enters the INPUT chain and then hits a rule that it matches and that tells it to --jump EXAMPLE_CHAIN. The packet will then start traversing the EXAMPLE_CHAIN, and all of a sudden it matches a specific rule which has the --jump RETURN target set. It will then jump back to the INPUT chain. Another example would be if the packet hit a --jump RETURN rule in the INPUT chain. It would then be dropped to the default policy as previously described, and no more actions would be taken in this chain.

中文这样解释：

> 顾名思义，它使包返回上一层，顺序是：子链——>父链——>缺省的策略。具体地说，就是若包在子链 中遇到了RETURN，则返回父链的下一条规则继续进行条件的比较，若是在父链（或称主链，比如INPUT）中 遇到了RETURN，就要被缺省的策略（一般是ACCEPT或DROP）操作了。（译者注：这很象C语言中函数返回值 的情况）
> 我们来举个例子说明一下，假设一个包进入了INPUT链，匹配了某条target为--jump EXAMPLE_CHAIN规则，然后进入了子链EXAMPLE_CHAIN。在子链中又匹配了某条 规则，恰巧target是--jump RETURN，那包就返回INPUT链了。如果在INPUT链里又遇 到了--jump RETURN，那这个包就要交由缺省的策略来处理了。


说白了就是在POSTROUTING上匹配到源、目标IP恰好为两个网关内网段时就返回执行默认策略（这里是ACCEPT）

所以，我上面配置iptables的两句还可以合成一句：
`-A POSTROUTING -s xxxx/xx -d ! oooo/oo -j MASQUERADE`

具体为什么出现这种跟一般情形不一致的情况，还不得而知，毕竟是“国产”的麒麟操作系统！


  [1]: http://www.h3c.com.cn/Service/Channel_Service/Operational_Service/ICG_Technology/201005/675214_30005_0.htm
  [2]: http://comments.gmane.org/gmane.network.openswan.user/17576
