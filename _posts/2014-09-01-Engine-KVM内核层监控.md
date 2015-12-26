---
layout:     post
title:      Engine<--->KVM内核层监控
date:       2014-09-01 09:00:00
summary:    这里的Engine指的是云管理引擎，如Ovirt/Openstack等；另外，不仅仅为了监控KVM，稍加更改也可应用到Xen等虚拟化框架上
tags:       虚拟化
---

这里的Engine指的是云管理引擎，如Ovirt/Openstack等；另外，不仅仅为了监控KVM，稍加更改也可应用到Xen等虚拟化框架上

虚拟云平台的监控一般思路有：

  - 客户机瘦代理模式，如QGA（之前有说）、Ovirt-Agent等，粒度中等
  - 利用现有接口，如qemu提供的挂载的设备接口等，粒度较粗
  - 如果要更细粒度的监控，就需要在hypervisor层做了

其实即使在hypervisor层可以监控，也存在很多问题，如：

  - 很难区分具体的客户机
  - 内核层很容易影响性能及稳定性
  - 监控的东西意义不好直接反应
  - 实际批量部署困难，因为动了内核，会依赖特定内核版本

这些问题，短时间内恐怕还不能解决，本文也不是为了解决这些问题的，本文只是从如何监控上提供一种较稳定的设计思路

先就一幅图了解下大概结构：

![](https://dn-getlink.qbox.me/e32076ab-31b2-11e4-a482-f590086d082a.png)

N_KERNEL是通过netlink作为桥梁连接内核和用户空间

```c
#include <linux/module.h>
#include <net/sock.h>
#include <linux/netlink.h>
#include <linux/skbuff.h>
#include <linux/version.h>

#include <linux/timer.h>
#include <linux/sched.h>

#define NETLINK_USER 31
/*for new kernel, the method of create netlink is changed*/
#if LINUX_VERSION_CODE >= KERNEL_VERSION(3,6,0)	
#define NEW_KERNEL
#endif
#define MSG_LEN 2


struct timer_list stimer;
int timeout = 1*HZ;
int time_lock = 0x00;
static void time_handler(void)
{
	time_lock ^= 0x01;
	printk("time_lock:%d", time_lock);
}

struct sock *nl_sk = NULL;
struct nlmsghdr *nlh;
int pid = -1;
int kvm_swh = 0;
EXPORT_SYMBOL(kvm_swh);

static void nl_send_msg(char *msg)
{
	struct sk_buff *skb_out;
	int msg_size;
	int res;
	msg_size = strlen(msg);
	skb_out = nlmsg_new(msg_size, 0);
	if (!skb_out){
		printk(KERN_ERR "Failed to allocate new skb\n");
		return;
	}
	nlh = nlmsg_put(skb_out, 0, 0, NLMSG_DONE, msg_size, 0);
	NETLINK_CB(skb_out).dst_group = 0; /* not in mcast group */
	strncpy(nlmsg_data(nlh), msg, msg_size);
	res = nlmsg_unicast(nl_sk, skb_out, pid);
	if (res < 0){//maybe daemon is exited
		//pid = -1;
		printk(KERN_INFO "Error while sending bak to user\n");
	}
}

void log_kernel(u32 reason){
	if(pid != -1 && kvm_swh == 1 && time_lock == 1){
		char msg[MSG_LEN + 1]={0};
		sprintf(msg, "%02X", reason);
		nl_send_msg(msg);
	}
}
EXPORT_SYMBOL(log_kernel);

static void nl_recv_msg(struct sk_buff *skb)
{
	nlh = (struct nlmsghdr *)skb->data;
	printk(KERN_INFO "recv:%s\n", (char *)nlmsg_data(nlh));
	if(strcmp((char*)nlmsg_data(nlh), "recv") == 0){
		pid = nlh->nlmsg_pid; /*pid of sending process */
		printk("ack for %d\n", pid);
	}else if(strcmp((char*)nlmsg_data(nlh), "on") == 0){
		kvm_swh = 1;	
		printk("switch kvm_log on\n");
	}else{
		kvm_swh = 0;
		printk("switch kvm_log off\n");
	}
}

static int __init nl_init(void)
{
	printk(KERN_INFO "init nl module\n");
#ifdef NEW_KERNEL
	struct netlink_kernel_cfg cfg={
		.input = nl_recv_msg,
	};
	nl_sk = netlink_kernel_create(&init_net, NETLINK_USER, &cfg);
#else
	nl_sk = netlink_kernel_create(&init_net, NETLINK_USER, 0, 
			nl_recv_msg, NULL, THIS_MODULE);
#endif
	if (!nl_sk)
	{
		printk(KERN_ALERT "Error creating socket.\n");
		return -10;
	}
	/*init timer*/
	init_timer(&stimer);
	stimer.expires = jiffies + timeout;
	stimer.function = time_handler;
	add_timer(&stimer);
	return 0;
}

static void __exit nl_exit(void)
{
	printk(KERN_INFO "exit nl module\n");
	netlink_kernel_release(nl_sk);
	del_timer(&stimer);
}

module_init(nl_init); 
module_exit(nl_exit);
MODULE_LICENSE("GPL");
```

这里导出了kvm_swh（开关）和log_kernel（日志记录），在kvm里声明即可使用</br>
在编译加载kvm模块的时候，如果遇到找不到symbol，在模块的的Makefile里试试加入`KBUILD_EXTRA_SYMBOLS += /path/to/N_KERNEL/Module.symvers`

SWITCH这里仅仅是简单开关，接受用户空间的开关命令，一直开着hypervisor层的监控室很耗资源的

```c
#include <sys/socket.h>
#include <linux/netlink.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <unistd.h>
#include <malloc.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#define BUF_LEN 256
#define SERVER_PORT 50000
#define NETLINK_USER 31
#define MAX_PAYLOAD 1024 /* maximum payload size*/
struct msghdr msg;
struct nlmsghdr *nlh = NULL;
struct sockaddr_nl src_addr, dest_addr;
struct iovec iov;
int sock_fd;

int main()
{
	struct sockaddr_in si_me, si_other;
	socklen_t len = sizeof(si_other);
	char buf[BUF_LEN];
	int sock_d = socket(AF_INET, SOCK_DGRAM,0);
	bzero(&si_me,sizeof(si_me));
	si_me.sin_family = AF_INET;
	si_me.sin_addr.s_addr = htonl(INADDR_ANY);
	si_me.sin_port = htons(SERVER_PORT);
	bind(sock_d, (struct sockaddr *)&si_me, sizeof(si_me));
	for (;;)
	{
		if(recvfrom(sock_d, buf, BUF_LEN, 0, 
					(struct sockaddr *)&si_other, &len) != -1){
			sock_fd = socket(PF_NETLINK, SOCK_RAW, NETLINK_USER);
			if (sock_fd < 0) return -1;
			memset(&src_addr, 0, sizeof(src_addr));
			src_addr.nl_family = AF_NETLINK;
			src_addr.nl_pid = getpid(); /* self pid */
			bind(sock_fd, (struct sockaddr *)&src_addr, sizeof(src_addr));

			memset(&dest_addr, 0, sizeof(dest_addr));
			memset(&dest_addr, 0, sizeof(dest_addr));
			dest_addr.nl_family = AF_NETLINK;
			dest_addr.nl_pid = 0; /* For Linux Kernel */
			dest_addr.nl_groups = 0; /* unicast */

			nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PAYLOAD));
			memset(nlh, 0, NLMSG_SPACE(MAX_PAYLOAD));
			nlh->nlmsg_len = NLMSG_SPACE(MAX_PAYLOAD);
			nlh->nlmsg_pid = getpid();
			nlh->nlmsg_flags = 0;
			strcpy(NLMSG_DATA(nlh), buf);
			iov.iov_base = (void *)nlh;
			iov.iov_len = nlh->nlmsg_len;
			msg.msg_name = (void *)&dest_addr;
			msg.msg_namelen = sizeof(dest_addr);
			msg.msg_iov = &iov;
			msg.msg_iovlen = 1;
			sendmsg(sock_fd, &msg, 0);
			close(sock_fd);
		}
	}
	return 0;
}
```

LOG_ENGINE主要是向用户空间发送内核空间采集的数据，这里要保证数据量越小越好，内核是很脆弱的

这里用到了共享内存mmap机制，因为rpc-server是python写的，mmap对于跨语言应用还是简单高效的

```c
#include <sys/socket.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <malloc.h>
#include <string.h>
#include <linux/netlink.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/time.h>
#include <time.h>

#define SET_LEN 9
#define SET_NUM 1024
#define MEM_LEN (SET_LEN * SET_NUM)
#define MSG_LEN 3

#define NETLINK_USER 31
#define MAX_PAYLOAD 1024 /* maximum payload size*/

struct sockaddr_nl src_addr, dest_addr;
struct nlmsghdr *nlh = NULL;
struct iovec iov;
int sock_fd;
struct msghdr msg;
char buf[MSG_LEN];

/*
 * log format: A013344
 */
void push_data(char *mem, char *log)
{
        int i;
        for(i = SET_NUM - 2; i >= 0; --i){
                strncpy(mem + (i+1)*SET_LEN, mem + i*SET_LEN, SET_LEN);
        }
        strncpy(mem, log, SET_LEN);
}

void init_mem(char **mem)
{
        int fd = open ("/tmp/kvm_mmap", O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
        lseek(fd, MEM_LEN + 1, SEEK_SET);
        write(fd, "", 1);
        lseek(fd, 0, SEEK_SET);
        *mem = mmap(0, MEM_LEN, PROT_WRITE, MAP_SHARED, fd, 0);
        memset(*mem, 0, MEM_LEN + 1);
        close(fd);
}

void finit(char **mem)
{
        munmap(*mem, MEM_LEN);
}

int main()
{
	sock_fd = socket(PF_NETLINK, SOCK_RAW, NETLINK_USER);
	if (sock_fd < 0){
		puts("create socket err.\n");
		return -1;
	}
	memset(&src_addr, 0, sizeof(src_addr));
	src_addr.nl_family = AF_NETLINK;
	src_addr.nl_pid = getpid(); /* self pid */
	bind(sock_fd, (struct sockaddr *)&src_addr, sizeof(src_addr));

	memset(&dest_addr, 0, sizeof(dest_addr));
	memset(&dest_addr, 0, sizeof(dest_addr));
	dest_addr.nl_family = AF_NETLINK;
	dest_addr.nl_pid = 0; /* For Linux Kernel */
	dest_addr.nl_groups = 0; /* unicast */

	nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PAYLOAD));
	memset(nlh, 0, NLMSG_SPACE(MAX_PAYLOAD));
	nlh->nlmsg_len = NLMSG_SPACE(MAX_PAYLOAD);
	nlh->nlmsg_pid = getpid();
	nlh->nlmsg_flags = 0;

	strcpy(NLMSG_DATA(nlh), "recv");

	iov.iov_base = (void *)nlh;
	iov.iov_len = nlh->nlmsg_len;
	msg.msg_name = (void *)&dest_addr;
	msg.msg_namelen = sizeof(dest_addr);
	msg.msg_iov = &iov;
	msg.msg_iovlen = 1;

	sendmsg(sock_fd, &msg, 0);
	char *mem;
        init_mem(&mem);
	//daemon(1, 0);
	while(1){
		recvmsg(sock_fd, &msg, 0);
		memset(buf, 0, sizeof(buf));
		time_t b_time;
		struct tm *tim;
		struct timeval tv;
        	struct timezone tz;
        	gettimeofday(&tv, &tz);
		b_time=time(NULL);
		tim=localtime(&b_time);
		sprintf(buf, "%02s%02d%02d%03d", NLMSG_DATA(nlh), tim->tm_min, tim->tm_sec, tv.tv_usec/1000);
		//printf("%s\n", buf);
		push_data(mem, buf);
	}
	close(sock_fd);
	finit_mem(&mem);
	log_finit();
	return 0;
}
```

以上SWITCH和LOG-ENGINE可以做成系统服务

然后是RPC-SERVER了，主要是注册两个方法

```python
def kvmLogSwitchSC(self, value):
    address = ('127.0.0.1', 50000)
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.sendto(value + '\0', address)
    return 'true'

def getKvmLogSC(self):
    with open('/tmp/kvm_mmap', 'r') as f:
        with contextlib.closing(mmap.mmap(f.fileno(), 0,
                                access=mmap.ACCESS_READ)) as m:
            res = m.read(MEM_LEN)
            if res[0] == '\0':
                return 'none'
            return res
```

然后engine就可以通过rpc-client调用了

最后，提醒下，重新编译kvm模块替换时，请使用原版内核树的代码

```bash
cp /boot/xxx.config ./.config
make oldconfig && make prepare
make modules
make M=arch/x86/kvm
```

