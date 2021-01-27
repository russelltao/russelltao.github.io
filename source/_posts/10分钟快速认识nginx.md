---
title: 10分钟快速认识Nginx
tags:
  - configure
  - man
  - nginx
id: '1201'
categories:
  - - nginx
  - - web
date: 2020-12-17 10:14:06
---

Nginx是当下最流行的Web服务器，通过官方以及第三方C模块，以及在Nginx上构建出的Openresty，或者在Openresty上构建出的Kong，你可以使用Nginx生态满足任何复杂Web场景下的需求。Nginx的性能也极其优秀，它可以轻松支持百万、千万级的并发连接，也可以高效的处理磁盘IO，因而通过静态资源或者缓存，能够为Tomcat、Django等性能不佳的Web应用扛住绝大部分外部流量。   但是，很多刚接触Nginx的同学，对它的理解往往失之偏颇，不太清楚Nginx的能力范围。比如：

*   你可能清楚Nginx对上游应用支持Google的gRPC协议，但对下游的客户端是否支持gRPC协议呢？
*   Openresty中的Nginx版本是单号的，而Nginx官网中的stable稳定版本则是双号的，我们到底该选择哪个版本的Nginx呢？
*   安装Nginx时，下载Nginx docker镜像，或者用yum/apt-get安装，都比下载源代码再编译出可执行文件要简单许多，那到底有必要基于源码安装Nginx吗？
*   当你下载完Nginx源码后，你清楚每个目录与文件的意义吗？

  本文是《从头搭建1个静态资源服务器》系列文章中的第1篇，也是我在6月4日晚直播内容的文字总结，在这篇文章中我将向你演示：Nginx有什么特点，它的能力上限在哪，该如何获取Nginx，Nginx源代码中各目录的意义又是什么。  
<!-- more -->
## Nginx到底是什么？

Nginx是一个集**静态资源**、**负载均衡**于一身的**Web**服务器，这里有3个关键词，我们一一来分析。

*   Web

我爱把互联网服务的访问路径，与社会经济中的供应链放在一起做类比，这样很容易理解“上下游”这类比较抽象的词汇。比如，购买小米手机时，实体或者网上店铺是供应链的下游，而高通的CPU则是上游。类似地，**浏览器作为终端自然是下游，关系数据库则是上游**，而Nginx位于源服务器和终端之间，如下图所示： [![](/2020/12/nginx上下游.png)](http://www.taohui.pub/2020/12/17/10%e5%88%86%e9%92%9f%e5%bf%ab%e9%80%9f%e8%ae%a4%e8%af%86nginx/nginx%e4%b8%8a%e4%b8%8b%e6%b8%b8/) 弄明白了上下游的概念后，我们就清楚了“Web服务器”的外延：Nginx的下游协议是Web中的HTTP协议，而上游则可以是任意协议，比如python的网关协议uwsgi，或者C/C++爱用的CGI协议，或者RPC服务常用的gRPC协议，等等。   在Nginx诞生之初，它的下游协议仅支持HTTP/1协议，但随着版本的不断迭代，现在下游还支持HTTP/2、MAIL邮件、TCP协议、UDP协议等等。   Web场景面向的是公网，所以非常强调信息安全。而Nginx对TLS/SSL协议的支持也非常彻底，它可以轻松的对下游或者上游装载、卸载TLS协议，并通过Openssl支持各种安全套件。  

*   静态资源

Web服务器必须能够提供图片、Javascript、CSS、HTML等资源的下载能力，由于它们多数是静态的，所以通常直接存放在磁盘上。Nginx很擅长读取本机磁盘上的文件，并将它们发送至下游客户端！**你可能会觉得，读取文件并通过HTTP协议发送出去，这简直不要太简单，Nginx竟然只是擅长这个？这里可大有文章！**   比如，你可能了解零拷贝技术，Nginx很早就支持它，这使得发送文件时速度可以至少提升一倍！可是，零拷贝对于特大文件很不友好，占用了许多PageCache内存，但使用率却非常低，因此Nginx用Linux的原生异步IO加上直接IO解决了这类问题。再比如，小报文的发送降低了网络传输效率，而Nginx通过Nagle、Cork等算法，以及应用层的postpone\_out指令批量发送小报文，这使得Nginx的性能远远领先于Tomcat、Netty、Apache等竞争对手，因此主流的CDN都是使用Nginx实现的。  

*   负载均衡

在分布式系统中，用加机器扩展系统，是提升可用性的最有效方法。但扩展系统时，需要在应用服务前添加1个负载均衡服务，使它能够将请求流量分发给上游的应用。这一场景中，除了对负载均衡服务的性能有极高的要求外，它还必须能够处理应用层协议。**在OSI网络体系中，IP网络层是第3层，TCP/UDP传输层是第4层，而HTTP等应用层则是第7层，因此，在Web场景中，需求量最大的自然是7层负载均衡**，而Nginx非常擅长应用层的协议处理，这体现在以下4个方面：

1.  通过多路复用、事件驱动等技术，Nginx可以轻松支持C10M级别的并发；
2.  由C语言编写，与操作系统紧密结合的Nginx（紧密结合到什么程度呢？Nginx之父Igor曾经说过，他最后悔的就是让Nginx支持windows操作系统，因为它与类Unix系统差异太大，这使得紧密结合的Nginx必须付出很大代价才能实现），能够充分使用CPU、内存等硬件，极高的效率使它可以同时为几十台上游服务器提供负载均衡功能；
3.  Nginx的架构很灵活，它允许任何第三方以C模块的形式，与官方模块互相协作，给用户提供各类功能。因此，丰富的生态使得Nginx支持多种多样的应用层协议（你可以在Github上搜索到大量的C模块），你也可以直接开发C模块定制Nginx。
4.  Nginx使用了非常开放的2-clause BSD-like license源码许可协议，它意味着你在修改Nginx源码后，还可以作为商业用途发布，TEngine就受益于这一特性。当Lua语言通过C模块注入Nginx后，就诞生了Openresty及一堆Lua语言模块，这比直接开发C语言模块难度下降了很多。而在Lua语言之上，又诞生了Kong这样面向微服务的生态。

  从上述3个关键词的解释，我相信你已经明白了Nginx的能力范围。接下来，我们再来看看如何安装Nginx。  

## 怎样获取Nginx？

Nginx有很多种获取、安装的方式，我把它们分为以下两类：

*   **非定制化安装**

主要指下载编译好的二进制文件，再直接安装在目标系统中，比如：

*   拉取含有Nginx的docker镜像；
*   在操作系统的应用市场中直接安装，比如用apt-get/yum命令直接安装Nginx；
*   获取到网上编译好的Nginx压缩包后，解压后直接运行；

 

*   **定制化安装**

在[http://nginx.org/en/download.html](http://nginx.org/en/download.html)上或者[https://www.nginx-cn.net/product](https://www.nginx-cn.net/product)上下载Nginx源代码，调用configure脚本生成定制化的编译选项后，执行make命令编译生成可执行文件，最后用make install命令安装Nginx。   非定制化安装虽然更加简单，但这样的Nginx默认缺失以下功能：

*   不支持更有效率的HTTP2协议；
*   不支持TCP/UDP协议，不能充当4层负载均衡；
*   不支持TLS/SSL协议，无法跨越公网保障网络安全；
*   未安装stub\_status模块，无法实时监控Nginx连接状态；

你可以通过configure –help命令给出的--with-XXX-module说明，找到Nginx默认不安装的官方模块，例如：（dynamic是动态模块，在后续文章中我会演示其用法）

```
--with-http_ssl_module             enable ngx_http_ssl_module
--with-http_v2_module              enable ngx_http_v2_module
--with-http_realip_module          enable ngx_http_realip_module
--with-http_addition_module        enable ngx_http_addition_module
--with-http_xslt_module            enable ngx_http_xslt_module
--with-http_xslt_module=dynamic    enable dynamic ngx_http_xslt_module
--with-http_image_filter_module    enable ngx_http_image_filter_module
--with-http_image_filter_module=dynamic enable dynamic ngx_http_image_filter_module
--with-http_geoip_module           enable ngx_http_geoip_module
--with-http_geoip_module=dynamic   enable dynamic ngx_http_geoip_module
--with-http_sub_module             enable ngx_http_sub_module
--with-http_dav_module             enable ngx_http_dav_module
--with-http_flv_module             enable ngx_http_flv_module
--with-http_mp4_module             enable ngx_http_mp4_module
--with-http_gunzip_module          enable ngx_http_gunzip_module
--with-http_gzip_static_module     enable ngx_http_gzip_static_module
--with-http_auth_request_module    enable ngx_http_auth_request_module
--with-http_random_index_module    enable ngx_http_random_index_module
--with-http_secure_link_module     enable ngx_http_secure_link_module
--with-http_degradation_module     enable ngx_http_degradation_module
--with-http_slice_module           enable ngx_http_slice_module
--with-http_stub_status_module     enable ngx_http_stub_status_module
```

    因此，从功能的全面性上来说，我们需要从源码上安装Nginx。   你可能会想，那为什么不索性将所有模块都编译到默认的Nginx中呢？按需编译模块，至少有以下4个优点：

*   执行速度更快。例如，通过配置文件关闭功能，就需要多做一些条件判断。
*   减少nginx可执行文件的大小。
*   有些模块依赖项过多，在非必要时启用它们，会增加编译、运行环境的复杂性。
*   给用户提供强大的自定义功能，比如在configure时设定配置文件、pid文件、可执行文件的路径，根据实际情况重新指定编译时的优化参数等等。

  当然，**最重要的还是可以通过configure --add-module选项任意添加自定义模块，这赋予Nginx无限的可能。**   由于Nginx有许多分支和版本，该如何选择适合自己的版本呢？这有两个技巧，我们先来看mainline和stable版本的区别，在[http://nginx.org/en/download.html](http://nginx.org/en/download.html)上你会看到如下页面： [![](http://www.taohui.pub/wp-content/uploads/2020/12/nginx版本下载.png)](http://www.taohui.pub/2020/12/17/10%e5%88%86%e9%92%9f%e5%bf%ab%e9%80%9f%e8%ae%a4%e8%af%86nginx/nginx%e7%89%88%e6%9c%ac%e4%b8%8b%e8%bd%bd/) 这里，**mainline是含有最新功能的主线版本，它的迭代速度最快**。另外，你可能注意到mainline是单号版本，而Openresty由于更新Nginx的频率较低，所以为了获得最新的Nginx特性，它通常使用mainline版本。   **stable是mainline版本稳定运行一段时间后，将单号大版本转换为双号的稳定版本**，比如1.18.0就是由1.17.10转换而来。   Legacy则是曾经的稳定版本。如果从头开始使用Nginx，那么你只需要选择最新的stable或者mainline版本就可以了。但如果你已经在使用某一个Legacy版本的Nginx，现在是否把它升级到最新版本呢？毕竟在生产环境上升级前要做完整的功能、性能测试，成本并不低。此时，我们要从CHANGES变更文件中，寻找每个版本的变化点。点开CHANGES文件，你会看到如下页面： [![](/2020/12/nginx-changes.png)](http://www.taohui.pub/2020/12/17/10%e5%88%86%e9%92%9f%e5%bf%ab%e9%80%9f%e8%ae%a4%e8%af%86nginx/nginx-changes/) 这里列出了每个版本的发布时间，以及发布时的变更。这些变更共分为以下4类：

*   Feature新功能，比如上图HTTP框架新增的auth\_delay指令。
*   Bugfix问题修复，我们尤其要关注一些重大Bug的修复。
*   Change已知特性的变更，比如之前允许HTTP请求头部中出现多个Host头部，但在17.9这个Change后，就认定这类HTTP请求非法了。
*   Security安全问题的升级，比如15.6版本就修复了CVE-2018-16843等3个安全问题。

  从Feature、Bugfix、Change、Security这4个方面，我们就可以更有针对性的升级Nginx。  

## 认识Nginx的源码目录

当获取到Nginx源码压缩包并解压后，你可能对这些目录一头雾水，这里我对它们做个简单说明。比如1.18.0版本的源代码目录是这样的： [![](/2020/12/nginx源码目录.png)](http://www.taohui.pub/2020/12/17/10%e5%88%86%e9%92%9f%e5%bf%ab%e9%80%9f%e8%ae%a4%e8%af%86nginx/nginx%e6%ba%90%e7%a0%81%e7%9b%ae%e5%bd%95/) 其中包含5个文件和5个目录，我们先来看单个文件的意义：

*   CHANGES：即上面介绍过的版本变更文件。
*   ru：由于Igor是俄罗斯人，所以除了上面的英文版变更文件外，还有个俄文版的变更文件。
*   configure：如同其他Linux源码类软件一样，这是编译前的必须执行的核心脚本，它包含下面4个子功能：
    *   解析configure脚本执行时传入的各种参数，包括定制的第三方模块；
    *   针对操作系统、体系架构、编译器的特性，生成特定的编译参数；
    *   生成Makefile、c等文件；
    *   在屏幕上显示汇总后的执行结果。
*   LICENSE：这个文件描述了Nginx使用的[2-clause BSD-like license](https://opensource.org/licenses/BSD-2-Clause)许可协议。
*   README：它只是告诉你去使用http://nginx.org官网查询各模块的用法。

  再来看各个目录的意义：

*   auto：configure只是一个简单的入口脚本，真正的功能是由auto目录下各个脚本完成的。
*   conf：当安装完Nginx后，conf目录下会有默认的配置文件，这些文件就是从这里的conf目录复制过去的。
*   contrib：包含了Nginx相关的周边小工具，比如下一讲将要介绍vim中如何高亮显示Nginx语法，就依赖于其中的vim子目录。
*   html：安装完Nginx并运行后，会显示默认的欢迎页面，以及出现错误的500页面，这两个页面就是由html目录拷贝到安装目录的。
*   man：目录中仅包含8一个文件，它其实是为Linux系统准备的man帮助文档，使用man -l nginx.8命令，可以看到Nginx命令行的使用方法：

[![](/2020/12/nginx-man.png)](http://www.taohui.pub/2020/12/17/10%e5%88%86%e9%92%9f%e5%bf%ab%e9%80%9f%e8%ae%a4%e8%af%86nginx/nginx-man/)

*   src：放置所有Nginx源代码的目录。关于src下的子目录，后续我分析源码时再详细介绍它们。

  以上就是官方压缩包解压后的内容，当你执行完configure脚本后，还会多出Makefile文件以及objs目录，在下一篇文章中我会介绍它们。  

## 小结

最后，对《从头搭建静态资源服务器》系列第1篇做个总结。   Nginx是集静态资源与负载均衡与一身的Web服务器，它支持C10M级别的并发连接，也通过与操作系统的紧密结合，能够高效的使用系统资源。除性能外，Nginx通过优秀的模块设计，允许第三方的C模块、Lua模块等嵌入到Nginx中运行，这极大丰富了Nginx生态。   下载源码编译安装Nginx，可以获得定制Nginx的能力。这样不仅用助于性能的提升，还通过各类模块扩展了Nginx的功能。   Nginx源代码中有5个文件和5个一级目录，其中configure脚本极为关键，在它执行后，还会生成Makefile文件和objs目录，它们与定制化的模块、系统的高性能参数密切相关，此后才能正式编译Nginx。   下一篇，我们将介绍configure脚本的用法，配置文件的语法格式，以及如何配置出静态资源服务。