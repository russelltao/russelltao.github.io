---
title: 基于websocket接口的jmeter自动化测试实践（1）
tags: []
id: '261'
categories:
  - - web
  - - 高并发
date: 2017-03-21 08:28:31
---

自动化测试对于小团队来说非常重要，特别是技术负责人更偏向于用技术解决问题时（习惯用管理解决问题时，可能会用手动+人海方式）。 而在接口测试中，jmeter无疑是一个低成本方案的自动化测试工具。 为什么呢？因为它在整体设计上把业务逻辑、测试框架、测试数据三者分离了。jmeter进程就是测试框架，而通过如csv等文件提供测试数据，jmx提供包含业务逻辑的测试用例。而jmx脚本，则是以可视化的配置方式来编写（且配置时，可以利用内置函数提供多种功能）。这样的方案，无疑是维护成本最低的。 同时，jmeter有大量的第三方插件，得以支持大部分协议。在性能测试方面，jmeter还支持多台机器组成集群对服务器压测，可以部署agent到服务器以拉取服务器指标的监控实时数据，同时还有大量的压测结果分析工具。 从功能测试角度来看，如果jmeter脚本能覆盖大部分接口及组合场景，那么，阅读jmx脚本无疑是最快速了解产品的方法了。
<!-- more -->
1.  对产品经理而言，通过它可以了解产品的落地细节；
2.  对前端而言，既可以看到后端接口的使用方式，也能够获得集成用例场景，还可以借此产生大量数据以验证页面；
3.  对后端而言，可以自动化回归功能，还可以压测得到性能并验证稳定性；
4.  对运维而言，可以得到性能基线数据。

基于此，我选用jmeter来测试后端的websocket接口。 1、环境的准备 1）下载最新版的jdk [http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)，安装。   2）下载最新版的jmeter，例如当前最新版为3.1，可在 [http://mirrors.tuna.tsinghua.edu.cn/apache//jmeter/binaries/apache-jmeter-3.1.zip](http://mirrors.tuna.tsinghua.edu.cn/apache/jmeter/binaries/apache-jmeter-3.1.zip)下载到压缩包，解压到某目录下即可。   3）下载jmeter插件管理器 jmeter要想支持websocket需要安装一堆插件，磨刀不误砍柴功，先装一个插件叫JMeter Plugins Manager，安装方法也很简单，参考https://www.blazemeter.com/blog/how-install-jmeter-plugins-manager文章，只是把jar包放在lib/ext目录下即可。下载地址在 u[https://jmeter-plugins.org/downloads/all/](https://jmeter-plugins.org/downloads/all/)。   4）在options里找到Plugin Manager，在available plugins里找到Websocket protocol support点击选中，安装后jmeter会自动重启。 ![](http://www.taohui.pub/wp-content/uploads/2017/03/jmeter插件管理-1-1.jpg) 从可用插件里即可非常方便的得到新的插件。 ![](http://www.taohui.pub/wp-content/uploads/2017/03/jmeter-plugins-manager可用插件-1-1.jpg) 这个插件可以自动升级，如下： ![](http://www.taohui.pub/wp-content/uploads/2017/03/jmeter-plugins-manager可升级-1-1.jpg) 5）服务基于websocket和json，故点击这两个插件即可获得。   2、使用websocket sampler进行测试 ![](/2017/03/websocket请求设置-1-1.jpg) 需要注意，虽然这里的WebServer下有Server Name or IP配置，但在HTTP Request Defaults里的Server Name or IP是不支持分享给每个case的，这点很不方便后续维护，一个解决方案是：添加User Defined Variable，其中抽象出Server Name or IP，再把变量testserver放到每个case上！ 另外，backlog表示响应中显示几条message，默认是3。 3、使用json解析响应 测试场景中，协议是以websocket+json格式传递数据，然而，这个websocket插件中却会在response里上面加了一行\[Message n\]这样一个字符串，导致输出不再是标准的json字符串。所以，添加了jmeter json extractor插件后，后置resposne处理器从非标准的response里提取不出值。例如：

```
[Message 2]
{"msg": "成功:登录", "data": {"user_id": 1, "sid": "61875d286b9a1eb329ab5642812216fe"}, "code": 1000, "command": {"path": "employee.consumer.Login"}}
```

这样的结果里，用$.data.sid是取不出sid的值的。当然，用正则表达式肯定是能提取出值的，但如果有大量case，且接口返回格式修改的比较频繁，正则表达式就是一个不大不小的坑，调整修改时效率很低下。 目前我使用的解决方案是，先用正则表达式取出第2行开始的json串（前面的\[Message 2\]信息是插件添加的，非常固定），再把它以jmeter variable的方式传递给json extractor，即可解决。 ![](http://www.taohui.pub/wp-content/uploads/2017/03/json数组取值-1-1.jpg) json返回里会有列表，而列表里取第几个的值，如果序号是固定的当然好办，而如果与某个元素的值有关，则可以用?(@.)这种方式来取，如上图所示。   4、加入内置函数 比如常用的取随机数\_\_Random，或者取当前日期和时间\_\_time，如下所示： ![](http://www.taohui.pub/wp-content/uploads/2017/03/jmeter内置函数-1-1.jpg) 5、加入定时器 ![](http://www.taohui.pub/wp-content/uploads/2017/03/jmeter随机定时器-1-1.jpg) 随机或者固定定时器，都非常有用，模拟各种用户时间尺度上不同的行为。 注意，对单个sampler有效的话，必须把定时器移至sampler的子元素中。 6、加入逻辑控制 ![](https://www.taohui.pub/wp-content/uploads/2017/03/jmeter逻辑判断-2.jpg) 非常好用的逻辑控制器。