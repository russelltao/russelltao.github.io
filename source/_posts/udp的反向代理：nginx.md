---
title: Udp的反向代理：nginx
tags:
  - nginx
  - proxy
  - session
  - udp
id: '439'
categories:
  - - nginx
  - - 高并发
date: 2018-04-08 20:04:56
---

在实时性要求较高的特殊场景下，简单的UDP协议仍然是我们的主要手段。UDP协议没有重传机制，还适用于同时向多台主机广播，因此在诸如多人会议、实时竞技游戏、DNS查询等场景里很适用，视频、音频每一帧可以允许丢失但绝对不能重传，网络不好时用户可以容忍黑一下或者声音嘟一下，如果突然把几秒前的视频帧或者声音重播一次就乱套了。使用UDP协议作为信息承载的传输层协议时，就要面临反向代理如何选择的挑战。通常我们有数台企业内网的服务器向客户端提供服务，此时需要在下游用户前有一台反向代理服务器做UDP包的转发、依据各服务器的实时状态做负载均衡，而关于UDP反向代理服务器的使用介绍网上并不多见。本文将讲述udp协议的会话机制原理，以及基于nginx如何配置udp协议的反向代理，包括如何维持住session、透传客户端ip到上游应用服务的3种方案等。

# UDP协议简介

许多人眼中的udp协议是没有反向代理、负载均衡这个概念的。毕竟，udp只是在IP包上加了个仅仅8个字节的包头，这区区8个字节又如何能把session会话这个特性描述出来呢？   
![](/2018/04/UDP报文的协议分层-1.png) 
图1 UDP报文的协议分层 在TCP/IP或者 OSI网络七层模型中，每层的任务都是如此明确：

*   物理层专注于提供物理的、机械的、电子的数据传输，但这是有可能出现差错的；
*   数据链路层在物理层的基础上通过差错的检测、控制来提升传输质量，并可在局域网内使数据报文跨主机可达。这些功能是通过在报文的前后添加Frame头尾部实现的，如上图所示。每个局域网由于技术特性，都会设置报文的最大长度MTU（Maximum Transmission Unit），用netstat -i(linux)命令可以查看MTU的大小：![](/2018/04/MTU大小-1.png)
*   而IP网络层的目标是确保报文可以跨广域网到达目的主机。由于广域网由许多不同的局域网，而每个局域网的MTU不同，当网络设备的IP层发现待发送的数据字节数超过MTU时，将会把数据拆成多个小于MTU的数据块各自组成新的IP报文发送出去，而接收主机则根据IP报头中的Flags和Fragment Offset这两个字段将接收到的无序的多个IP报文，组合成一段有序的初始发送数据。IP报头的格式如下图所示：![](/2018/04/IP报文头部-1.png)

图2 IP报文头部 IP协议头（本文只谈IPv4）里最关键的是Source IP Address发送方的源地址、Destination IP Address目标方的目的地址。这两个地址保证一个报文可以由一台windows主机到达一台linux主机，但并不能决定一个chrome浏览的GET请求可以到达linux上的nginx。 4、传输层主要包括TCP协议和UDP协议。这一层最主要的任务是保证端口可达，因为端口可以归属到某个进程，当chrome的GET请求根据IP层的destination IP到达linux主机时，linux操作系统根据传输层头部的destination port找到了正在listen或者recvfrom的nginx进程。所以传输层无论什么协议其头部都必须有源端口和目的端口。例如下图的UDP头部：
![](/2018/04/UDP的头部-1.png) 
图3 UDP的头部 
TCP的报文头比UDP复杂许多，因为TCP除了实现端口可达外，它还提供了可靠的数据链路，包括流控、有序重组、多路复用等高级功能。由于上文提到的IP层报文拆分与重组是在IP层实现的，而IP层是不可靠的所有数组效率低下，所以TCP层还定义了MSS（Maximum Segment Size）最大报文长度，这个MSS肯定小于链路中所有网络的MTU，因此TCP优先在自己这一层拆成小报文避免的IP层的分包。而UDP协议报文头部太简单了，无法提供这样的功能，所以基于UDP协议开发的程序需要开发人员自行把握不要把过大的数据一次发送。 

对报文有所了解后，我们再来看看UDP协议的应用场景。相比TCP而言UDP报文头不过8个字节，所以UDP协议的最大好处是传输成本低（包括协议栈的处理），也没有TCP的拥塞、滑动窗口等导致数据延迟发送、接收的机制。但UDP报文不能保证一定送达到目的主机的目的端口，它没有重传机制。所以，应用UDP协议的程序一定是可以容忍报文丢失、不接受报文重传的。如果某个程序在UDP之上包装的应用层协议支持了重传、乱序重组、多路复用等特性，那么他肯定是选错传输层协议了，这些功能TCP都有，而且TCP还有更多的功能以保证网络通讯质量。因此，通常实时声音、视频的传输使用UDP协议是非常合适的，我可以容忍正在看的视频少了几帧图像，但不能容忍突然几分钟前的几帧图像突然插进来：-）

# UDP协议的会话保持机制

有了上面的知识储备，我们可以来搞清楚UDP是如何维持会话连接的。对话就是会话，A可以对B说话，而B可以针对这句话的内容再回一句，这句可以到达A。如果能够维持这种机制自然就有会话了。UDP可以吗？当然可以。例如客户端（请求发起者）首先监听一个端口Lc，就像他的耳朵，而服务提供者也在主机上监听一个端口Ls，用于接收客户端的请求。客户端任选一个源端口向服务器的Ls端口发送UDP报文，而服务提供者则通过任选一个源端口向客户端的端口Lc发送响应端口，这样会话是可以建立起来的。但是这种机制有哪些问题呢？ 问题一定要结合场景来看。比如：
1. 如果客户端是windows上的chrome浏览器，怎么能让它监听一个端口呢？端口是会冲突的，如果有其他进程占了这个端口，还能不工作了？
2. 如果开了多个chrome窗口，那个第1个窗口发的请求对应的响应被第2个窗口收到怎么办？
3. 如果刚发完一个请求，进程挂了，新启的窗口收到老的响应怎么办？等等。可见这套方案并不适合消费者用户的服务与服务器通讯，所以视频会议等看来是不行。 

有其他办法么？有！如果客户端使用的源端口，同样用于接收服务器发送的响应，那么以上的问题就不存在了。像TCP协议就是如此，其connect方的随机源端口将一直用于连接上的数据传送，直到连接关闭。 这个方案对客户端有以下要求：不要使用sendto这样的方法，几乎任何语言对UDP协议都提供有这样的方法封装。应当先用connect方法获取到socket，再调用send方法把请求发出去。这样做的原因是既可以在内核中保存有5元组（源ip、源port、目的ip、目的端口、UDP协议），以使得该源端口仅接收目的ip和端口发来的UDP报文，又可以反复使用send方法时比sendto每次都上传递目的ip和目的port两个参数。 

对服务器端有以下要求：不要使用recvfrom这样的方法，因为该方法无法获取到客户端的发送源ip和源port，这样就无法向客户端发送响应了。应当使用recvmsg方法（有些编程语言例如python2就没有该方法，但python3有）去接收请求，把获取到的对端ip和port保存下来，而发送响应时可以仍然使用sendto方法。   接下来我们谈谈nginx如何做udp协议的反向代理。 

Nginx的stream系列模块核心就是在传输层上做反向代理，虽然TCP协议的应用场景更多，但UDP协议在Nginx的角度看来也与TCP协议大同小异，比如：nginx向upstream转发请求时仍然是通过connect方法得到的fd句柄，接收upstream的响应时也是通过fd调用recv方法获取消息；nginx接收客户端的消息时则是通过上文提到过的recvmsg方法，同时把获取到的客户端源ip和源port保存下来。我们先看下recvmsg方法的定义：

```
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```

相对于recvfrom方法，多了一个msghdr结构体，如下所示：

```
struct msghdr {    
    void         *msg_name;       /* optional address */    
    socklen_t     msg_namelen;    /* size of address */    
    struct iovec *msg_iov;        /* scatter/gather array */    
    size_t        msg_iovlen;     /* # elements in msg_iov */    
    void         *msg_control;    /* ancillary data, see below */    
    size_t        msg_controllen; /* ancillary data buffer len */    
    int           msg_flags;      /* flags on received message */
};
```

其中msg\_name就是对端的源IP和源端口（指向sockaddr结构体）。以上是C库的定义，其他高级语言类似方法会更简单，例如python里的同名方法是这么定义的：

```
(data, ancdata, msg_flags, address) = socket.recvmsg(bufsize[, ancbufsize[, flags]])
```

其中返回元组的第4个元素就是对端的ip和port。

# 配置nginx为UDP反向代理服务

以上是nginx在udp反向代理上的工作原理。实际配置则很简单：

```
# Load balance UDP-based DNS traffic across two servers
stream {    
    upstream dns_upstreams {        
        server 192.168.136.130:53;        
        server 192.168.136.131:53;    
    }     
    server {        
        listen 53 udp;        
        proxy_pass dns_upstreams;        
        proxy_timeout 1s;        
        proxy_responses 1;        
        error_log logs/dns.log;    
    }
}
```

在listen配置中的udp选项告诉nginx这是udp反向代理。而proxy\_timeout和proxy\_responses则是维持住udp会话机制的主要参数。 UDP协议自身并没有会话保持机制，nginx于是定义了一个非常简单的维持机制：客户端每发出一个UDP报文，通常期待接收回一个报文响应，当然也有可能不响应或者需要多个报文响应一个请求，此时proxy\_responses可配为其他值。而proxy\_timeout则规定了在最长的等待时间内没有响应则断开会话。

# 如何通过nginx向后端服务传递客户真实IP

最后我们来谈一谈经过nginx反向代理后，upstream服务如何才能获取到客户端的地址？如下图所示，nginx不同于IP转发，它事实上建立了新的连接，所以正常情况下upstream无法获取到客户端的地址： ![](/2018/04/nginx反向代理掩盖了客户端的IP-1.png) 图4 nginx反向代理掩盖了客户端的IP 上图虽然是以TCP/HTTP举例，但对UDP而言也一样。而且，在HTTP协议中还可以通过X-Forwarded-For头部传递客户端IP，而TCP与UDP则不行。Proxy protocol本是一个好的解决方案，它通过在传输层header之上添加一层描述对端的ip和port来解决问题，例如： 
![](/2018/04/Proxy-protocol-1.png) 但是，它要求upstream上的服务要支持解析proxy protocol，而这个协议还是有些小众。最关键的是，目前nginx对proxy protocol的支持则仅止于tcp协议，并不支持udp协议，我们可以看下其代码： ![](/2018/04/nginx处理proxy-protocol-1.png) 可见nginx目前并不支持udp协议的proxy protocol（笔者下的nginx版本为1.13.6）。 ![](/2018/04/proxy-protocol实际支持udp-1.png) 虽然proxy protocol是支持udp协议的。怎么办呢？

## 方案1：IP地址透传

可以用IP地址透传的解决方案。如下图所示： ![](/2018/04/nginx作为四层反向代理向upstream展示客户端ip时的ip透传方案-1.png) 图5 nginx作为四层反向代理向upstream展示客户端ip时的ip透传方案 这里在nginx与upstream服务间做了一些hack的行为：

*   nginx向upstream发送包时，必须开启root权限以修改ip包的源地址为client ip，以让upstream上的进程可以直接看到客户端的IP。

```
server {    
     listen 53 udp;
     proxy_responses 1;
     proxy_timeout 1s;
     proxy_bind $remote_addr transparent;     
     proxy_pass dns_upstreams;
}
```

*   upstream上的路由表需要修改，因为upstream是在内网，它的网关是内网网关，并不知道把目的ip是client ip的包向哪里发。而且，它的源地址端口是upstream的，client也不会认的。所以，需要修改默认网关为nginx所在的机器。

```
# route del default gw 原网关ip
# route add default gw nginx的ip
```

*   nginx的机器上必须修改iptable以使得nginx进程处理目的ip是client的报文。

```
# ip rule add fwmark 1 lookup 100
# ip route add local 0.0.0.0/0 dev lo table 100 
# iptables -t mangle -A PREROUTING -p tcp -s 172.16.0.0/28 --sport 80 -j MARK --set-xmark 0x1/0xffffffff
```

这套方案其实对TCP也是适用的。

## 方案2：DSR（上游服务无公网）

除了上述方案外，还有个Direct Server Return方案，即upstream回包时nginx进程不再介入处理。这种DSR方案又分为两种，第1种假定upstream的机器上没有公网网卡，其解决方案图示如下： ![](/2018/04/nginx做udp反向代理时的DSR方案（upstream无公网）-1.png) 图6 nginx做udp反向代理时的DSR方案（upstream无公网） 这套方案做了以下hack行为： 1、在nginx上同时绑定client的源ip和端口，因为upstream回包后将不再经过nginx进程了。同时，proxy\_responses也需要设为0。

```
server {
    listen 53 udp; proxy_responses 0;
    proxy_bind $remote_addr:$remote_port transparent;
     proxy_pass dns_upstreams;
}
```

2、与第一种方案相同，修改upstream的默认网关为nginx所在机器（任何一台拥有公网的机器都行）。 3、在nginx的主机上修改iptables，使得nginx可以转发upstream发回的响应，同时把源ip和端口由upstream的改为nginx的。例如：

```
# tc qdisc add dev eth0 root handle 10: htb
# tc filter add dev eth0 parent 10: protocol ip prio 10 u32 match ip src 172.16.0.11 match ip sport 53 action nat egress 172.16.0.11 192.168.99.10
# tc filter add dev eth0 parent 10: protocol ip prio 10 u32 match ip src 172.16.0.12 match ip sport 53 action nat egress 172.16.0.12 192.168.99.10
# tc filter add dev eth0 parent 10: protocol ip prio 10 u32 match ip src 172.16.0.13 match ip sport 53 action nat egress 172.16.0.13 192.168.99.10
# tc filter add dev eth0 parent 10: protocol ip prio 10 u32 match ip src 172.16.0.14 match ip sport 53 action nat egress 172.16.0.14 192.168.99.10
```

## 方案3：DSR（上游服务有公网）

DSR的另一套方案是假定upstream上有公网线路，这样upstream的回包可以直接向client发送，如下图所示： ![](/2018/04/nginx做udp反向代理时的DSR方案（upstream有公网）-1.png) 图6 nginx做udp反向代理时的DSR方案（upstream有公网） 这套DSR方案与上一套DSR方案的区别在于：由upstream服务所在主机上修改发送报文的源地址与源端口为nginx的ip和监听端口，以使得client可以接收到报文。例如：

```
# tc qdisc add dev eth0 root handle 10: htb
# tc filter add dev eth0 parent 10: protocol ip prio 10 u32 match ip src 172.16.0.11 match ip sport 53 action nat egress 172.16.0.11 192.168.99.10
```

# 结语

以上三套方案皆可以使用开源版的nginx向后端服务传递客户端真实IP地址，但都需要nginx的worker进程跑在root权限下，这对运维并不友好。从协议层面，可以期待后续版本支持proxy protocol传递客户端ip以解决此问题。在当下的诸多应用场景下，除非业务场景明确无误的拒绝超时重传机制，否则还是应当使用TCP协议，其完善的流量、拥塞控制都是我们必须拥有的能力，如果在UDP层上重新实现这套机制就得不偿失了。