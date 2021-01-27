---
title: 一文解释清楚Google BBR拥塞控制算法原理
tags:
  - BBR
  - BDP
  - BtlBW
  - CDF累积概率分布函数
  - CUBIC
  - pacing_gain
  - RTprop
  - RTT
  - tcp
  - youtube
  - 丢包
  - 快速恢复
  - 快速重传
  - 慢启动
  - 拥塞控制
  - 拥塞避免
  - 滑动窗口
  - 路由器
id: '984'
categories:
  - - web
  - - 高并发
date: 2019-08-07 08:55:28
---

BBR对TCP性能的提升是巨大的，它能更有效的使用当下网络环境，Youtube应用后在吞吐量上有平均4%提升（对于日本这样的网络环境有14%以上的提升）： [![](http://www.taohui.pub/wp-content/uploads/2019/07/Youtube-BBS-vs-CUBIC-Throughput-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/youtube-bbs-vs-cubic-throughput-2/) 报文的往返时延RTT降低了33%，这样如视频这样的大文件传输更快，用户体验更好：
<!-- more -->
 [![](http://www.taohui.pub/wp-content/uploads/2019/07/Youtube-BBS-vs-CUBIC-Max-RTT-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/youtube-bbs-vs-cubic-max-rtt-2/) 不像CUBIC这种基于丢包做拥塞控制，常导致瓶颈路由器大量报文丢失，所以重新缓存的平均间隔时间也有了11%提升： [![](http://www.taohui.pub/wp-content/uploads/2019/07/Youtube-BBS-vs-CUBIC-Rebuffers-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/youtube-bbs-vs-cubic-rebuffers-2/) 在Linux4.19内核中已经将拥塞控制算法从CUBIC（该算法从2.6.19内核就引入Linux了）改为BBR，而即将面世的基于UDP的HTTP3也使用此算法。许多做应用开发的同学可能并不清楚什么是拥塞控制，BBR算法到底在做什么，我在《Web协议详解与抓包实战》这门课程中用了6节课在讲相关内容，这里我尝试下用一篇图片比文字还多的文章把这个事说清楚。 TCP协议是面向字符流的协议，它允许应用层基于read/write方法来发送、读取任意长的字符流： [![](http://www.taohui.pub/wp-content/uploads/2019/07/tcp-socket-readwrite.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/tcp-socket-readwrite/) 但TCP之下的IP层是基于块状的Packet报文来分片发送的，因此，TCP协议需要将应用交付给它的字符流拆分成多个Packet（在TCP传输层被称为Segment）发送，由于网速有变化且接收主机的处理性能有限，TCP还要决定何时发送这些Segment。TCP滑动窗口解决了Client、Server这两台主机的问题，但没有去管连接中大量路由器、交换机转发IP报文的问题，因此当瓶颈路由器的输入流大于其输出流时，便会发生拥塞： [![](http://www.taohui.pub/wp-content/uploads/2019/07/较大管道向较小管道传输引发拥堵.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/%e8%be%83%e5%a4%a7%e7%ae%a1%e9%81%93%e5%90%91%e8%be%83%e5%b0%8f%e7%ae%a1%e9%81%93%e4%bc%a0%e8%be%93%e5%bc%95%e5%8f%91%e6%8b%a5%e5%a0%b5/) 这虽然是IP网络层的事，但如果TCP基于分层原则不去管，互联网上大量主机的TCP程序便会造成网络恶性拥堵。上图中瓶颈路由器已经造成了网速下降，但如果发送方不管不顾，那么瓶颈路由器的缓冲队列填满后便会发生大量丢包，且此时RTT（报文往返时间）由于存在长队列而极高。 [![](http://www.taohui.pub/wp-content/uploads/2019/07/BBR队列丢包-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/bbr%e9%98%9f%e5%88%97%e4%b8%a2%e5%8c%85-2/) 如上图，最好的状态是没有队列，此时RTT最低，而State2中RTT升高，但没有丢包，到State 3队列满时开始发生丢包。 TCP的拥塞控制便用于解决这个问题。在BBR出现前，拥塞控制分为四个部分：慢启动、拥塞避免、快速重传、快速恢复： [![](http://www.taohui.pub/wp-content/uploads/2019/07/Reno拥塞控制示意.jpg)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/reno%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%a4%ba%e6%84%8f/) 慢启动在BBR中仍然保留，它的意义是在不知道连接的瓶颈带宽时，以起始较低的发送速率，以每RTT两倍的速度快速增加发送速率，直到到达一个阈值，对应上图中0-4秒。到该阈值后，进入线性提高发送速率的阶段，该阶段叫做拥塞避免，直到发生丢包，对应上图中8-11秒。丢包后，发速速率大幅下降，针对丢包使用快速重传算法重送发送，同时也使用快速恢复算法把发送速率尽量平滑的升上来。 如果瓶颈路由器的缓存特别大，那么这种以丢包作为探测依据的拥塞算法将会导致严重问题：TCP链路上长时间RTT变大，但吞吐量维持不变。 事实上，我们的传输速度在3个阶段被不同的因素限制：1、应用程序限制阶段，此时RTT不变，随着应用程序开始发送大文件，速率直线上升；2、BDP限制阶段，此时RTT开始不断上升，但吞吐量不变，因为此时瓶颈路由器已经达到上限，缓冲队列正在不断增加；3、瓶颈路由器缓冲队列限制阶段，此时开始大量丢包。如下所示： [![](/2019/07/传输速度、RTT与飞行报文的关系-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/%e4%bc%a0%e8%be%93%e9%80%9f%e5%ba%a6%e3%80%81rtt%e4%b8%8e%e9%a3%9e%e8%a1%8c%e6%8a%a5%e6%96%87%e7%9a%84%e5%85%b3%e7%b3%bb-2/) 如CUBIC这样基于丢包的拥塞控制算法在第2条灰色竖线发生作用，这已经太晚了，更好的作用点是BDP上限开始发挥作用时，也就是第1条灰色竖线。 什么叫做BDP呢？它叫做带宽时延积，例如一条链路的带宽是100Mbps，而RTT是40ms，那么

```
BDP=100Mbps*0.04s=4Mb=0.5MB
```

即平均每秒飞行中的报文应当是0.5MB。因此Linux的接收窗口缓存常参考此设置： [![](http://www.taohui.pub/wp-content/uploads/2017/01/BGP-1.jpg)](http://www.taohui.pub/2016/01/27/%e9%ab%98%e6%80%a7%e8%83%bd%e7%bd%91%e7%bb%9c%e7%bc%96%e7%a8%8b7-tcp%e8%bf%9e%e6%8e%a5%e7%9a%84%e5%86%85%e5%ad%98%e4%bd%bf%e7%94%a8/%e9%ab%98%e6%80%a7%e8%83%bd%e7%bd%91%e7%bb%9c%e7%bc%96%e7%a8%8b-bgp/) 第1条灰色竖线，是瓶颈路由器的缓冲队列刚刚开始积压时的节点。随着内存的不断降价，路由器设备的缓冲队列也会越来越大，CUBIC算法会造成更大的RTT时延！ 而BBR通过检测RTprop和BtlBw来实现拥塞控制。什么是RTprop呢？这是链路的物理时延，因为RTT里含有报文在路由器队列里的排队时间、ACK的延迟确认时间等。什么叫延迟确认呢？TCP每个报文必须被确认，确认动作是通过接收端发送ACK报文实现的，但由于TCP和IP头部有40个字节，如果不携带数据只为发送ACK网络效率过低，所以会让独立的ACK报文等一等，看看有没有数据发的时候顺便带给对方，或者等等看多个ACK一起发。所以，可以用下列公式表示RTT与RTprop的差别： [![](http://www.taohui.pub/wp-content/uploads/2019/07/RTprop计算1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/rtprop%e8%ae%a1%e7%ae%971/) RTT我们可以测量得出，RTprop呢，我们只需要找到瓶颈路由器队列为空时多次RTT测量的最小值即可： [![](http://www.taohui.pub/wp-content/uploads/2019/07/RTprop计算2.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/rtprop%e8%ae%a1%e7%ae%972/) 而BtlBw全称是bottleneck bandwith，即瓶颈带宽，我们可以通过测量已发送但未ACK确认的飞行中字节除以飞行时间deliveryRate来测量： [![](http://www.taohui.pub/wp-content/uploads/2019/07/BtlBw计算.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/btlbw%e8%ae%a1%e7%ae%97/) 早在1979年Leonard Kleinrock就提出了第1条竖线是最好的拥塞控制点，但被Jeffrey M. Jaffe证明不可能实现，因为没有办法判断RTT变化到底是不是因为链路变化了，从而不同的设备瓶颈导致的，还是瓶颈路由器上的其他TCP连接的流量发生了大的变化。但我们有了RTprop和BtlBw后，当RTprop升高时我们便得到了BtlBw，这便找到第1条灰色竖线最好的拥塞控制点，也有了后续发送速率的依据。 基于BBR算法，由于瓶颈路由器的队列为空，最直接的影响就是RTT大幅下降，可以看到下图中CUBIC红色线条的RTT比BBR要高很多： [![](http://www.taohui.pub/wp-content/uploads/2019/07/CUBIC与BBR的RTT对比-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/cubic%e4%b8%8ebbr%e7%9a%84rtt%e5%af%b9%e6%af%94-2/) 而因为没有丢包，BBR传输速率也会有大幅提升，下图中插入的图为CDF累积概率分布函数，从CDF中可以很清晰的看到CUBIC下大部分连接的吞吐量都更低： [![](http://www.taohui.pub/wp-content/uploads/2019/07/BBR带宽提升-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/bbr%e5%b8%a6%e5%ae%bd%e6%8f%90%e5%8d%87-2/) 如果链路发生了切换，新的瓶颈带宽升大或者变小怎么办呢？BBR会尝试周期性的探测新的瓶颈带宽，这个周期值为1.25、0.75、1、1、1、1，如下所示： [![](http://www.taohui.pub/wp-content/uploads/2019/07/BBR运行细节-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/bbr%e8%bf%90%e8%a1%8c%e7%bb%86%e8%8a%82-2/) 1.25会使得BBR尝试发送更多的飞行中报文，而如果产生了队列积压，0.75则会释放队列。下图中是先以10Mbps的链路传输TCP，在第20秒网络切换到了更快的40Mbps链路，由于1.25的存在BBR很快发现了更大的带宽，而第40秒又切换回了10Mbps链路，2秒内由于RTT的快速增加BBR调低了发送速率，可以看到由于有了pacing\_gain周期变换BBR工作得很好。 [![](http://www.taohui.pub/wp-content/uploads/2019/07/带宽上升后又下降-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/%e5%b8%a6%e5%ae%bd%e4%b8%8a%e5%8d%87%e5%90%8e%e5%8f%88%e4%b8%8b%e9%99%8d-2/) pacing\_gain周期还有个优点，就是可以使多条初始速度不同的TCP链路快速的平均分享带宽，如下图所示，后启动的连接由于过高估计BDP产生队列积压，早先连接的BBR便会在数个周期内快速降低发送速率，最终由于不产生队列积压下RTT是一致的，故平衡时5条链路均分了带宽： [![](http://www.taohui.pub/wp-content/uploads/2019/07/多个BBR链路快速平分带宽-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/%e5%a4%9a%e4%b8%aabbr%e9%93%be%e8%b7%af%e5%bf%ab%e9%80%9f%e5%b9%b3%e5%88%86%e5%b8%a6%e5%ae%bd-2/) 我们再来看看慢启动阶段，下图网络是10Mbps、40ms，因此未确认的飞行字节数应为10Mbps\*0.04s=0.05MB。红色线条是CUBIC算法下已发送字节数，而蓝色是ACK已确认字节数，绿色则是BBR算法下的已发送字节数。显然，最初CUBIC与BBR算法相同，在0.25秒时飞行字节数显然远超过了0.05MB字节数，大约在 0.1MB字节数也就是2倍BDP： [![](http://www.taohui.pub/wp-content/uploads/2019/07/慢启动下的BBR探测-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/%e6%85%a2%e5%90%af%e5%8a%a8%e4%b8%8b%e7%9a%84bbr%e6%8e%a2%e6%b5%8b-2/) 大约在0.3秒时，CUBIC开始线性增加拥塞窗口，而到了0.5秒后BBR开始降低发送速率，即排空瓶颈路由器的拥塞队列，到0.75秒时飞行字节数调整到了BDP大小，这是最合适的发送速率。 当繁忙的网络出现大幅丢包时，BBR的表现也远好于CUBIC算法。下图中，丢包率从0.001%到50%时，可以看到绿色的BBR远好于红色的CUBIC。大约当丢包率到0.1%时，CUBIC由于不停的触发拥塞算法，所以吞吐量极速降到10Mbps只有原先的1/10，而BBR直到5%丢包率才出现明显的吞吐量下降。 [![](http://www.taohui.pub/wp-content/uploads/2019/07/随机丢包下的吞吐量-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/%e9%9a%8f%e6%9c%ba%e4%b8%a2%e5%8c%85%e4%b8%8b%e7%9a%84%e5%90%9e%e5%90%90%e9%87%8f-2/) CUBIC造成瓶颈路由器的缓冲队列越来越满，RTT时延就会越来越大，而操作系统对三次握手的建立是有最大时间限制的，这导致建CUBIC下的网络极端拥塞时，新连接很难建立成功，如下图中RTT中位数达到 100秒时 Windows便很难建立成功新连接，而200秒时Linux/Android也无法建立成功。 [![](/2019/07/新连接建立困难-1.png)](http://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/%e6%96%b0%e8%bf%9e%e6%8e%a5%e5%bb%ba%e7%ab%8b%e5%9b%b0%e9%9a%be-2/) BBR算法的伪代码如下，这里包括两个流程，收到ACK确认以及发送报文：

```
function onAck(packet) 
  rtt = now - packet.sendtime 
  update_min_filter(RTpropFilter, rtt) 
  delivered += packet.size 
  delivered_time = now 
  deliveryRate = (delivered - packet.delivered) / (delivered_time - packet.delivered_time) 
  if (deliveryRate > BtlBwFilter.currentMax  ! packet.app_limited) 
     update_max_filter(BtlBwFilter, deliveryRate) 
  if (app_limited_until > 0) 
     app_limited_until = app_limited_until - packet.size
```

这里的app\_limited\_until是在允许发送时观察是否有发送任务决定的。发送报文时伪码为：

```
function send(packet) 
  bdp = BtlBwFilter.currentMax × RTpropFilter.currentMin 
  if (inflight >= cwnd_gain × bdp) 
     // wait for ack or retransmission timeout 
     return 
  if (now >= nextSendTime) 
     packet = nextPacketToSend() 
     if (! packet) 
        app_limited_until = inflight 
        return 
     packet.app_limited = (app_limited_until > 0) 
     packet.sendtime = now 
     packet.delivered = delivered 
     packet.delivered_time = delivered_time 
     ship(packet) 
     nextSendTime = now + packet.size / (pacing_gain × BtlBwFilter.currentMax) 
  timerCallbackAt(send, nextSendTime)
```

pacing\_gain便是决定链路速率调整的关键周期数组。 BBR算法对网络世界的拥塞控制有重大意义，尤其未来可以想见路由器的队列一定会越来越大。HTTP3放弃了TCP协议，这意味着它需要在应用层（各框架中间件）中基于BBR算法实现拥塞控制，所以，BBR算法其实离我们很近。理解BBR，我们便能更好的应对网络拥塞导致的性能问题，也会对未来的拥塞控制算法发展脉络更清晰。 我在《Web协议详解与抓包实战》第5部分课程中第15-20课对拥塞控制有更详细的介绍，详见下方课程二维码： [![](/2019/05/poster.jpg)](http://www.taohui.pub/2019/05/06/%e4%b8%ba%e4%bb%80%e4%b9%88%e8%a6%81%e5%87%baweb%e5%8d%8f%e8%ae%ae%e8%bf%99%e9%97%a8%e8%af%be/poster-2/)