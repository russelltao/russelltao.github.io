---
title: 如何configure定制出属于你的Nginx？
tags:
  - configure
  - module
  - nginx
  - 模块化
id: '1241'
categories:
  - - nginx
  - - web
date: 2020-12-22 10:02:19
---

上一篇[文章](https://www.taohui.pub/2020/12/17/10%e5%88%86%e9%92%9f%e5%bf%ab%e9%80%9f%e8%ae%a4%e8%af%86nginx/)中，我介绍了Nginx的特性，如何获取Nginx源代码，以及源代码中各目录的含义。本文将介绍如何定制化编译、安装、运行Nginx。 

当你用yum或者apt-get命令安装、启动Nginx后，通过nginx -t命令你会发现，nginx.conf配置文件可能在/etc/目录中。而运行基于源码安装的Nginx时，nginx.conf文件又可能位于/usr/local/nginx/conf/目录，运行OpenResty时， nginx.conf又被放在了/usr/local/openresty/nginx/conf/目录。**这些奇怪的现象都源于编译Nginx前，configure脚本设置的--prefix或者--conf-path选项**。 
<!-- more -->
Nginx的所有功能都来自于官方及第三方模块，**如果你不知道如何使用configure添加需要的模块，相当于放弃了Nginx诞生16年来累积出的丰富生态**。而且，很多高性能特性默认是关闭的，如果你习惯于使用应用市场中编译好的二进制文件，也无法获得性能最优化的Nginx。 本文将会介绍定制Nginx过程中，configure脚本的用法。其中对于定制模块的选项，会从模块的分类讲起，带你系统的掌握如何添加Nginx模块。同时，也会介绍configure执行后生成的objs目录，以及Makefile文件的用法。这也是Nginx开源社区基础培训系列课程第一季，即6月11日晚第2次视频直播课前半部分的文字总结。  

## configure脚本有哪些选项？

在Linux系统（包括各种类Unix操作系统）下，编译复杂的C语言软件工程是由Makefile文件完成的。**你一定注意到Nginx源码中并没有Makefile文件，这是因为Makefile是由configure脚本即时生成的**。接下来我们看看，configure脚本悄悄的做了哪些事，这些工作又会对Nginx产生哪些影响。   configure脚本支持很多选项，掌握它们就可以灵活的定制Nginx。为了方便理解，从功能上我把它们分为5类：

*   **改变Nginx编译、运行时各类资源的默认存取路径**

configure既可以设置Nginx运行时各类资源的默认访问路径，也可以设置编译期生成的临时文件存放路径。比如：

*   \--error-log-path=定义了运行期出现错误信息时写入log日志文件的路径。
*   \--http-log-path=定义了运行期处理完HTTP请求后，将执行结果写入log日志文件的路径。
*   \--http-client-body-temp-path=定义了运行期Nginx接收客户端的HTTP请求时，存放包体的磁盘路径。
*   \--http-proxy-temp-path=定义了运行期负载均衡使用的Nginx，临时存放上游返回HTTP包体的磁盘路径。
*   \--builddir=定义了编译期生成的脚本、源代码、目标文件存放的路径。

等等。 对于已经编译好的Nginx，可以通过nginx -V命令查看设置的路径。**如果没有显式的设置选项，Nginx便会使用默认值，例如官方Nginx将--prefix的默认值设为/usr/local/nginx，而OpenResty的configure脚本则将--prefix的默认值设为/usr/local/openresty/nginx**。  

*   **改变编译器选项**

Nginx由C语言开发，因此默认使用的C编译器，由于C++向前兼容C语言，如果你使用了C++编写的Nginx模块，可以通过--cc-opt等选项，将C编译器修改为C++编译器，这就可以支持C++语言了。 Nginx编译时使用的优化选项是-O，如果你觉得这样优化还不够，可以调大优化级别，比如OpenResty就将gcc的优化调整为-O2。当某些模块依赖其他软件库才能实现需求时，也可以通过--with-ld-opt选项链接其他库文件。  

*   **修改编译时依赖的中间件**

Nginx执行时，会依赖pcre、openssl、zlib等中间件，实现诸如正则表达式解析、TLS/SSL协议处理、解压缩等功能。通常，编译器会自动寻找系统默认路径中的软件库，但当系统中含有多个版本的中间件时，就可以人为地通过路径来指定版本。比如当我们需要使用最新的TLS1.3时，可以下载最新的openssl源码包，再通过--with-openssl=选项指定源码目录，让Makefile使用它去编译Nginx。  

*   **选择编译进Nginx的模块**

**Nginx是由少量的框架代码、大量的C语言模块构成的**。当你根据业务需求，需要通过某个模块实现相应的功能时，必须先通过configure脚本将它编译进Nginx（Nginx被设计为按需添加模块的架构），之后你才能在nginx.conf配置文件中启用它们。下一小节我会详细介绍这部分内容。  

*   **其他选项**

还有些不属于上述4个类别的选项，包括：

*   定位问题时，最方便的是通过log查看DEBUG级别日志，而打开调试日志的前提，是在configure时加入--with-debug选项。
*   HTTP服务是默认打开的，如果你想禁用HTTP或者HTTP缓存服务，可以使用--without-http和--without-http-cache选项。
*   大文件读写磁盘时，并不适宜使用正常的read/write系统调用，因为文件内容会写入PageCache磁盘高速缓存。由于PageCache空间有限，而大文件会迅速将可能高频命中缓存的小文件淘汰出PageCache，同时大文件自身又很难享受到缓存的好处。因此，在Linux系统中，可以通过异步IO、直接IO来处理文件。但开启Linux原生异步IO的前提，是在configure时加入--with-file-aio选项。
*   开启IPv6功能时，需要加入--with-ipv6选项。
*   生产环境中，需要使用master/worker多进程模式运行Nginx。master是权限更高的管理进程，而worker则是处理请求的工作线程，它的权限相对较低。通过--user=和--group=选项可以指定worker进程所属的用户及用户组，当然，你也可以在conf中通过user和group指令修改它。

  在大致了解configure提供的选项后，下面我们重点看下如何定制Nginx模块。

## 如何添加Nginx模块？

编译Nginx前，我们需要决定添加哪些模块。**在定制化模块前，只有分清了模块的类别才能系统的掌握它们**。Nginx通常可以分为6类模块，包括：

*   核心模块：有限的7个模块定义了Nginx最基本的功能。需要注意，**核心模块并不是默认一定编译进Nginx**，例如只有在configure时加入--with-http\_ssl\_module或者--with-stream\_ssl\_modul选项时，ngx\_openssl\_module核心模块才会编译进Nginx。
*   配置模块：仅包括ngx\_conf\_module这一个模块，负责解析conf配置文件。
*   事件模块：Nginx采用事件驱动的异步框架来处理网络报文，它支持epoll、poll、select等多种事件驱动方式。目前，epoll是主流的事件驱动方式，选择事件驱动针对的是古董操作系统。
*   HTTP模块：作为Web服务器及七层负载均衡，Nginx最复杂的功能都由HTTP模块实现，稍后我们再来看如何定制HTTP模块。
*   STREAM模块：负责实现四层负载功能，默认不会编译进Nginx。你可以通过--with-stream启用STREAM模块。
*   MAIL模块：Nginx也可以作为邮件服务器的负载均衡，通过--with-mail选项启用。

[![](/2020/12/Nginx模块化.png)](http://www.taohui.pub/2020/12/22/%e5%a6%82%e4%bd%95configure%e5%ae%9a%e5%88%b6%e5%87%ba%e5%b1%9e%e4%ba%8e%e4%bd%a0%e7%9a%84nginx%ef%bc%9f/nginx%e6%a8%a1%e5%9d%97%e5%8c%96/) 在上图中，HTTP模块又可以再次细分为3类模块：

*   请求处理模块：Nginx接收、解析完HTTP报文后，会将请求交给各个HTTP模块处理。比如读取磁盘文件并发送到客户端的静态资源功能，就是由ngx\_http\_static\_module模块实现的。为了方便各模块间协同配合，Nginx将HTTP请求的处理过程分为11个阶段，如下图所示：

[![](/2020/12/11个HTTP阶段表.png)](http://www.taohui.pub/2020/12/22/%e5%a6%82%e4%bd%95configure%e5%ae%9a%e5%88%b6%e5%87%ba%e5%b1%9e%e4%ba%8e%e4%bd%a0%e7%9a%84nginx%ef%bc%9f/11%e4%b8%aahttp%e9%98%b6%e6%ae%b5%e8%a1%a8/)

*   响应过滤模块：当上面的请求处理模块生成合法的HTTP响应后，将会由各个响应过滤模块依次对HTTP头部、包体做加工处理。比如返回HTML、JS、CSS等文本文件时，若配置了gzip on;指令，就可以添加content-encoding: gzip头部，并使用zlib库压缩包体。
*   upstream负载均衡模块：当Nginx作为反向代理连接上游服务时，允许各类upstream模块提供不同的路由策略，比如ngx\_http\_upstream\_hash\_module模块提供了哈希路由，而ngx\_http\_upstream\_keepalive\_module模块则允许复用TCP连接，降低握手、慢启动等动作提升的网络时延。

  对于这3类模块，你可以从模块名中识别，比如模块中出现filter和http字样，通常就是过滤模块，比如ngx\_http\_gzip\_filter\_module。如果模块中出现upstream和http字样，就是负载均衡模块。 我们可以使用--with-模块名，将其编译进Nginx，也可以通--without-模块名，从Nginx中移出该模块。需要注意的是，当你通过configure --help帮助中看到--with-打头的选项，都是默认不编译进Nginx的模块，反之，--without-打头的选项，则是默认就编译进Nginx的模块。 还有一些HTTP模块并不属于上述3个类别，比如--with-http\_v2\_module是加入支持HTTP2协议的模块。**当你需要添加第三方模块时，则可以通过--add-module=或者--add-dynamic-module=（动态模块将在后续文章中再介绍）选项指定模块源码目录，这样就可以将它编译进Nginx**。  

## 如何安装并运行Nginx？

当configure脚本根据指定的选项执行时，会自动检测体系架构、系统特性、编译器、依赖软件等环境信息，并基于它们生成编译Nginx工程的Makefile文件。同时，还会生成objs目录，我们先来看看objs目录中有些什么：

```
objs
-- autoconf.err            #configure自动检测环境时的执行纪录
-- Makefile                 #编译C代码用到的脚本
-- ngx_auto_config.h   #以宏的方式，存放configure指定的配置，供安装时使用
-- ngx_auto_headers.h#存放编译时包含的头文件中默认生成的宏
-- ngx_modules.c        #根据configure时加入的模块，生成ngx_modules数组
`-- src                          #存放编译时的目标文件
    -- core                #存放核心模块及框架代码生成的目标文件
    -- event               #存放事件模块生成的目标文件
    -- http                 #存放HTTP模块生成的目标文件
    -- mail                 #存放MAIL模块生成的目标文件
    -- misc                #存放ngx_google_perftools_module模块生成的目标文件
    -- os                   #存放与操作系统关联的源代码生成的目标文件
    `-- stream             #存放STREAM模块生成的目标文件
```

接着，执行make命令就可以基于Makefile文件编译Nginx了。make命令可以携带4种参数：

*   build：编译Nginx，这也是make不携带参数时的默认动作。它会在objs目录中生成可执行的二进制文件nginx。
*   clean：通过删除Makefile文件和objs目录，将configure、make的执行结果清除，方便重新编译。
*   install：将Nginx安装到configure时指定的路径中，注意install只针对从头安装Nginx，如果是升级正在运行的服务，请使用upgrade参数。
*   upgrade：替换可执行文件nginx，同时热升级运行中的Nginx进程。

因此，当我们首次安装Nginx时，只需要先执行make命令编译出可执行文件，再执行make install安装到目标路径即可。 启动Nginx也很简单，进入Nginx目录后（比如/usr/local/nginx），在sbin目录下执行nginx程序即可，Nginx默认会启用Daemon守护者模式（参见daemon on;指令），这样shell命令行不会被nginx程序阻塞。 至此，Nginx已经编译、安装完成，并成功运行。

## 小结

最后做个小结，本文介绍了定制化编译、安装及运行Nginx的方法。   

如果你想定制符合自己业务特点的Nginx，那就必须学会使用configure脚本，它会根据输入选项、系统环境生成差异化的编译环境，最终编译出功能、性能都不一样的Nginx。configure支持的选项分为5类，它允许用户修改资源路径、编译参数、依赖软件等，最重要的是可以选择加入哪些官方及第三方模块。 定制模块前，先要掌握模块的类别。

Nginx模块分为6类，作为Web服务器使用时，其中最复杂、强大的自然就是HTTP模块，它又可以再次细分为3小类：请求处理模块、响应过滤模块、负载均衡模块。我们可以使用--with或者--without选项增删官方模块，也可以通过--add-module或者--add-dynamic-module添加第三方模块。 configure会生成源代码、脚本、存放目标文件的临时目录，以及编译C工程的Makefile文件。其中，Makefile支持4个选项，允许我们编译、安装、升级Nginx。由于Nginx支持Daemon模式，启动它时直接运行程序即可。   

下一篇将会介绍nginx.conf的配置语法，以及使用命令行或者免费的可视化工具分析access.log日志文件的方法。