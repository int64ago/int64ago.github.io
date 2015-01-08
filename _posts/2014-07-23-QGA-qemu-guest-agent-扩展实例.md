---
layout:     post
title:      QGA(qemu guest agent)扩展实例
date:       2014-07-23 09:00:00
summary:    上次讲了[QGA(qemu guest agent)源码分析][1]，这次以一个实例说明下，没有理论的东西，不过有些细节要注意，我们以获取进程列表为例说明
tags:       虚拟化
---

上次讲了[QGA(qemu guest agent)源码分析][1]，这次以一个实例说明下，没有理论的东西，不过有些细节要注意，我们以获取进程列表为例说明


我用的是qemu-2.0.0
`git clone https://github.com/qemu/qemu.git -b v2.0.0`
对qemu的改动不细说，这里以git diff patch的形式给出[diff.patch][2]
打完补丁后具体编译见[QGA(qemu guest agent)源码分析][1]<br>

hosts上采用的是xmlrpc-server的方式和外界通信，以便ovirt/openstack等上层应用能容易获取geust信息
有三个文件：
psyc.py

```python
#!/usr/bin/python2
###By Cody Chan<int64ago@gmail.com> ###
import socket
class QgaEnhanceUtils:
    def __init__(self):
        self.sockpath = '/var/lib/libvirt/qemu/channels/%s.org.qemu.ga.0'
    def getProgListSC(self, uuid):
        return self.ret
        sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        sock.connect(self.sockpath % uuid)
        sock.send('{"execute":"guest-get-progs"}')
        sock.settimeout(1) 
        try:
            rec = sock.recv(10240).strip()
            sock.close()
            return rec
        except socket.timeout:
            sock.close()
            return '{"return": "none"}'
```

其中self.sockpath的值需要根据自己设定的情况选取，具体见[qemu][3]/[libvirt][4]，注意这里串口名称最好不要用org.qemu.guest_agent.0

qga_rpc.py

```python
###xmlrpc server ###
###By Cody Chan<int64ago@gmail.com> ###
import SimpleXMLRPCServer
from pysc import QgaEnhanceUtils
qeu = QgaEnhanceUtils()
server = SimpleXMLRPCServer.SimpleXMLRPCServer(("localhost", 8088))
server.register_instance(qeu)
server.serve_forever()
```

rpc_client.py

```python
import xmlrpclib
server = xmlrpclib.ServerProxy("http://localhost:8088")
res = server.getProgListSC('uuid')
print(res)
```

guest上运行qemu-ga.exe，host上运行qga_rpc.py，然后运行rpc_client.py，结果如下：
![][5]
格式化后的json样式：

```json
{
    "return": [
        {
            "name": "System", 
            "path": "?", 
            "pid": 4
        }, 
        {
            "name": "smss.exe", 
            "path": "\\SystemRoot\\System32\\smss.exe", 
            "pid": 256
        }, 
        {
            "name": "csrss.exe", 
            "path": "C:\\Windows\\system32\\csrss.exe", 
            "pid": 336
        }, 
        ...此处省略...
    ]
}
```

rpc_client.py是python写的xmlrpc-client，当然可以用java等实现，可以用于ovirt/openstack上用于监控
  [1]: http://192.168.1.1:4000/2014/07/20/QGA-qemu-guest-agent-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/
  [2]: http://int64ago.qiniudn.com/diff.patch
  [3]: http://wiki.qemu.org/Features/QAPI/GuestAgent
  [4]: http://wiki.libvirt.org/page/Qemu_guest_agent
  [5]: http://int64ago.qiniudn.com/BaiduShurufa_2014-7-23_14-59-14.png
