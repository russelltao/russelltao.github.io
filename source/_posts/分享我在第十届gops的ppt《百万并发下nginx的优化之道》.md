---
title: 分享我在第十届GOPS的PPT《百万并发下Nginx的优化之道》
tags:
  - C10M
  - GOPS
  - nginx
  - 高可用
  - 高效运维
id: '729'
categories:
  - - nginx
  - - 技术人生
  - - 高并发
date: 2018-09-19 14:58:55
---

非常荣幸参加已经举办了十届的GOPS。我在架构及高可用专场分享，并担任出品人及主持人。 ![](http://www.taohui.pub/wp-content/uploads/2018/09/unnamed-file-2-4.jpg) 本次会议地点在上海光大会展中心，我在大会场超级五分钟里演讲的内容只是PPT的前5页。这次我想分享的是C10M问题在Nginx上的应用。优化从来都是一个持续过程，所以最需要的是方法论。我会从硬件、集群到单机软件的优化作一个系统的阐述，希望可以把我的心得有效的传达给大家。 PPT我分为四个部分，分别是方法论概述、一个请求在Nginx中是如何被处理的、应用层优化、系统层优化。其中第二部分我更希望用一种感性的方式，为后面的内容作好铺垫。 ![](http://www.taohui.pub/wp-content/uploads/2018/09/unnamed-file-5.jpg) 以下只列出我在专场分享中的PPT正文： ![](http://www.taohui.pub/wp-content/uploads/2018/09/2-3-5.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/3-1-6.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/4-1-6.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/5-1-2.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/6-1-3.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/7-1-2.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/8-1-2.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/9-1-4.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/10-1-6.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/11-1-7.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/12-1-3.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/13-1-8.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/14-1-4.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/15-1-7.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/16-1-8.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/17-1-3.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/18-1-3.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/19-1-6.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/20-1-8.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/21-1-7.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/22-1-2.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/23-1-3.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/24-1-8.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/25-1-5.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/26-1-2.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/27-1-2.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/28-1-5.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/29-1-7.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/30-1-7.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/31-1-3.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/32-1-6.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/33-1-8.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/34-1-4.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/35-1-5.jpg) ![](http://www.taohui.pub/wp-content/uploads/2018/09/36-1-8.jpg) ![](/2018/09/37-1-4.jpg)