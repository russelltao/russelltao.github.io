---
title: 在阿里云ECS上进行vpn ipsec网络对接
tags:
  - ipsec
  - openswan
  - vpn
id: '350'
categories:
  - - 技术人生
date: 2017-01-28 16:48:42
---

当我们需要与一些安全级别要求很高的服务对接时，服务提供商的网络提供方式可能是使用ipsec点对点对接网络。如果我们不是使用公有云，而是有自己的机房和路由器，这些就只是按照服务方的参数要求配置下路由器的小事情。但对公有云来说（例如阿里云），我们没有自己的路由器，当对接方要求我们使用预共享密匙进行ipsec点对点对接，第一反应什么鬼（之前没接触过的朋友可以看下这篇文章http://www.ibm.com/developerworks/cn/[Linux](http://lib.csdn.net/base/linux "Linux知识库")/l-ipsec/，对该概念讲得蛮清楚）？而接下来拥有私有云独立机房的服务提供方则可能要求最简单直接的解决方案：拉条专线接入服务提供者机房（这个开发成本和运维成本都不小，不适合小而美型的敏捷项目！）？或者买个路由器，在办公室找台机器与服务方对接（高可靠性完全无法保障了）？ 本文讲述使用openswan在linux centos 7下进行ipsec vpn网络的对接，由于相对小众中文资料不多，故也是一篇实践总结笔记。
<!-- more -->
进行网络对接前，服务方会要求我们提供对接的公网IP，而这是很容易的：公有云物理服务器一般都有两个网卡，其中一个正是为虚拟机的公网访问准备的。而像阿里云ECS，其公网IP可靠性都是比较高的，ECS服务器挂掉后，同可用区内的迁移并不会更换我们的公网IP。然而服务提供方由于会与多个厂商对接网络，所以为方便管理可能会用规划厂商内网IP地址的方案降低管理成本，例如：要求我们使用私网private subnetA上的机器privateA1访问他们的private subnetB上的服务器privateB1，对接的公网IP中publicA就是我们提供的公网地址，而publicB就是服务方的对外公网地址，例如下图：

![](/2017/01/ipsecvpn-1-1.jpg) 由于规定了我们的私网地址，我的第一反应是使用阿里云vpc网络，在vpc网络里定义ECS的私网IP非常简单：在阿里云控制台上设置一个虚拟路由器、一个虚拟交换机，配置好subnetA网段到虚拟交换机，再把vpc网络的ecs加到虚拟交换机下，更换ecs的私网IP再重启服务器即可。然而，虽然我购买了这台vpc网络ecs的公网带宽，但在该服务器下用ifconfig却是看不到公网IP的。才想道vpc网络也是在linux下虚拟出来的网络。果然用openswan搭起来的vpn根本与服务提供方对接不了。而且vpc网络与经典网络ECS间内网是不通的（哪怕是同一机房内），且网络性能由于多了一层IP隧道，性能、稳定性都要弱些，只能排除。 换位思考，服务提供方规定厂商的私网IP，只是为了方便其管理，他们也完全无法控制我们的网络行为。所以，可以用最简单的方式，虚拟一个私网IP即可。 而ipsec的对接，用软件方案解决肯定是第一选择，目前strongswan和openswan都是广为使用，而我用yum默认拉下的strongswan版本是5.4.0，总是出现莫名其妙的错误（略过）。这里记录下openswan的使用。 首先yum install openswan安装好。接下来我们可以按照提供方的配置参数要求，去配置openswan使其能够打通ipsec网络。首先编辑ipsec.conf配置文件，

```
vim etc/ipsec.conf  
```

在配置文件下方增加我们这条vpn网络通道的配置参数，如预共享密匙和第1、第2阶段的认证参数：

```
conn 网络名称  
        authby=secret  
        auto=start  
        ike=3des-md5  
        keyexchange=ike  
        phase2=esp  
        phase2alg=3des-md5  
        compress=no  
        pfs=yes  
        type=tunnel  
        left=你的公网地址  
        leftsourceip=你的公网地址  
        leftsubnet=服务方要求的私网子网  
        leftnexthop=%defaultroute  
        right=服务方公网地址  
        rightsourceip=服务方公网地址  
        rightsubnet=服务方私网子网  
```

接下来配置预共享密匙：

```
vim /etc/ipsec.secrets  
```

在其中增加该网络的预共享密匙：

```
siteA-public-IP  siteB-public-IP:  PSK  "pre-shared-key"  
```

现在启动ipsec网络：

```
ipsec start  
```

好了，现在我们可以看下ipsec网络是否连通，执行：

```
ipsec status  
```

可以看到：

```
000 IKE SAs: total(1), half-open(0), open(0), authenticated(1), anonymous(0)  
000 IPsec SAs: total(1), authenticated(1), anonymous(0)  
000    
000 #195: "your conn name":500 STATE_QUICK_I2 (sent QI2, IPsec SA established); EVENT_SA_REPLACE in 6599s; newest IPSEC; eroute owner; isakmp#194; idle; import:admin initiate  
```

ipsec网络已经建立。也可以在对方的路由器上查看到网络已经联通。 现在开始解决privateA1访问privateB1的问题。当然现在去ping服务器私网privateB1肯定是不通的。我们先要虚拟出privateA1地址：

```
ifconfig eth0:2 privateA1  
```

现在用ifconfig可以看到多出了一个privateA1地址。再用它去ping对端私网：

```
ping -I privateA1 privateB1  
```

现在可以看到能ping通。如果知道对方web服务的监听端口，可以再用telnet验证下：

```
telnet -b privateA privateB port  
```

如此网络已通。（这样虚拟出的IP重启系统后就没有了，参考该文解决：https://linuxconfig.org/configuring-virtual-network-interfaces-in-linux） 然而每次要指定privateA作为源IP实在是太不方便了，有些工具还不支持这样指定源IP。解决办法是改路由表，设定访问privateB1所在的子网subnetB时，全部使用privateA作为源地址。例如：

```
ip route add privateSubnetB1 via GW dev eth1  src  privateA1  
```

再去直接ping privateB1，就已经通了。