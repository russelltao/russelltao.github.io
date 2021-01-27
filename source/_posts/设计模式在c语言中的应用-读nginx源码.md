---
title: 设计模式在C语言中的应用--读nginx源码
tags:
  - adapter模式
  - nginx
  - strategy模式
  - 设计模式
id: '151'
categories:
  - - nginx
  - - 编程语言
date: 2013-01-27 16:28:19
---

市面上的“设计模式“书籍文章，皆针对[Java](http://lib.csdn.net/base/javase "Java SE知识库")/C++/C#等面向对象语言，似乎离开了面向对象的种种特性，设计模式就无法实现，没有用武之地了。 是这样吗？设计模式的概念是从建筑领域引入的，本身从没歧视过面向过程编程语言，它只是对一类问题的普遍解决方案而已。面向对象语言因为有类、多态等特点，使得开发者们容易达到：隐藏细节、封装变化，而这与设计模式的目的比较一致，所以大师们爱把设计模式与面向对象语言二位一体的使用。然而，存在即合理，[C语言](http://lib.csdn.net/base/c "C语言知识库")直到今日仍然在大型软件工程中担纲主角，其种种设计方法其实与我们通常见到的设计模式本质是相同的。例如nginx这个纯C语言写就的的高性能WEB服务器，就有许多地方使用到了市面书籍提到的设计模式。下面通过nginx源码来看看C语言是怎么做的。当然，UML图都是我根据代码意图所画，并不准确（C语言真没法画UML），只用于方便理解，呵呵。 strategy模式： 该模式用于客户代码在“无知”状态下，可以使用种种不同的实现。下面我们以nginx对网络IO操作的封装部分来看看C语言的实现吧。 设计模式就是通过封装变化来解耦，所以，我们先要找出网络IO操作的变化点来。nginx是跨平台的，它会支持[Linux](http://lib.csdn.net/base/linux "Linux知识库")、freebsd、solaris等[操作系统](http://lib.csdn.net/base/operatingsystem "操作系统知识库")，而每个操作系统的网络IO操作是不同的，这就是变化点了。 所以，nginx首先定义了ngx\_os\_io\_t来封装这些变化。
<!-- more -->
```
typedef struct {  
    ngx_recv_pt        recv;  
    ngx_recv_chain_pt  recv_chain;  
    ngx_recv_pt        udp_recv;  
    ngx_send_pt        send;  
    ngx_send_chain_pt  send_chain;  
    ngx_uint_t         flags;  
} ngx_os_io_t;  
```

这里有五个函数指针（\*\_pt都是函数指针）和一个变量，用于收发网络数据，我把它理解为OO中的abstract class（每个ngx\_os\_io\_t定义的变量都会重新实现这五个函数）。 拥有函数指针的struct，我通常认为它们是OO中的abstract class，实现它们的文件（一堆函数）要对应到OO上，我则喜欢把它们当做子类来看。对于void\*这样的成员，要根据意图来看了，通常我会转换成聚合加继承的关系。 ![](http://www.taohui.pub/wp-content/uploads/2017/01/0_1328087269KWms-1-1.png) ngx\_io会在相应的ngx\_os\_specific\_init方法中，来策略性的选择到底使用哪个实现。客户代码只需要简单的调用ngx\_io中的方法即可。 adapter模式： 这个模式用以适配接口，通常都是我们已经定义好一种接口了，有一个新的实现却有着不同的接口，接下来adapter就开始发力了。下面我们仍然以nginx对网络IO操作的封装部分来看。 linux平台下可能存在普通的IO或者异步IO方式。我们在最初已经封装好ngx\_os\_io\_t接口了，客户代码都是这么直接使用的。现在linux实现了异步IO，而它的调用方式与普通的读写IO接口完全不同，所以，如果要支持aoi就需要一层adapter来适配ngx\_os\_io\_t，这就是adapter方式了。 ![](http://www.taohui.pub/wp-content/uploads/2013/01/adapter.png) 上图中，ngx\_os\_aio适配了原生的异步IO接口，这样，用户代码仍然像以前一样，只要直接使用ngx\_io中的五个接口方法，当nginx的IO部分支持linux aio后，用户代码不需要修改。 bridge桥模式： 桥模式用于将抽象和实现分离，各自都能独立的变化。下面以nginx的核心概念module举例，虽然有些牵强，因为nginx的代码从来没这么用过：通常都是一个抽象module context只对应着一个实现module来用，但是，毕竟这种结构下还是可以达到抽象与实现分离的目的，桥模式只好对应到这上面了。 nginx是以module的概念贯穿始终的。它有一个基本的抽象层ngx\_core\_module\_t（从意图上判断，context有抽象接口的功能，虽然简单从语法上看不出）。然后，nginx module有三个基本类型，分别是event（处理各种事件模型，如epoll/select等），http（处理各种http协议的事件），mail（处理mail相关的事件）。针对每种类型的module，都有许多个实现，比如event module就有9个实现，这里的每个实现其实也是个子类。 但是，在我们理解桥模式时，这些子类暂时要被看成是event module的实例。代码中看，像ngx\_epoll\_module这样的子类中，还是把一些通用的细节隐藏给ngx\_event\_core\_module来做（管理这个词更合适）了。从这个角度可以认为，通过context接口，把三个基本module实现分开了。来看看类图： ![](http://www.taohui.pub/wp-content/uploads/2013/01/bridge模式.png) nginx自己用时，是以ngx\_module\_t中的type成员来决定使用哪个实现的。目前的nginx代码中，如果用了一种接口就一定会指定相应的type。可是实际上，这也可以用来展示桥模式。以事件module为例来看看： ![](/2013/01/bridge2.png) 由于UML本就是针对OO语言的，所以以上我画的类图都比较牵强，什么是继承？什么是聚合？在C语言中，往往都是通过几个函数指针，或者void\*指针实现各种封装和多态。没有什么语法上的关联，我就只能从代码意图中来判断了。而代码意图这个比较虚，因为不同的角度理解出来都不一样，所以这个确实不好画。太灵活了点，我只能从一个便于说明的角度来看，例如：上面的ngx\_devpoll\_module其实就是一个ngx\_module\_t，呵呵，但是，实际上它最关心的是ngx\_event\_actions\_t的实现，如果完全根据语法来看，根本说不通的。但从代码意图中看，这些module并不关心ngx\_module\_t，所以我认为，它们只是在实现ngx\_event\_module\_t了。 当然以上只是一家之言，不必当真，如果对nginx源码有研究的话，欢迎各位拍砖。 客观的说，C语言确实在封装上很差，就像nginx，如果我们要开发一个处理http协议的module嵌入进nginx进程，必须了解ngx\_http\_module里到底做了什么，真没隐藏啥细节，module开发者们表示很郁闷。上面的这些设计模式，只是做到了代码上的解藕。如果nginx用C++写的话，我相信，现在第三方module都能数以万计了。