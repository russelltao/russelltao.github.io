---
title: 基于websocket接口的jmeter自动化测试实践（2）
tags:
  - jmeter
id: '353'
categories:
  - - web
date: 2017-04-05 19:11:28
---

1、通常我们会使用用户自定义变量，把每个用例共用的东西提取出来。然而，当测试环境多起来时，这些写死在jmx脚本里的变量就不那么好用了。例如，对多个环境测试时，难道要复制多个脚本、单独改变量值？ 此时，我们可以使用jmeter**属性**。因为属性是可以通过命令行传递的，例如：
<!-- more -->
```
-Jtestproperty=202
```

`而在需要使用变量的地方直接用${__P(testproperty,)}使用命令行传递的值。` 当然，如果脚本已经大量使用了user defined variable，且可能会有一个默认环境一批默认值，那么，在user defined variable里把变量的值设为${\_\_P(testproperty,30)}携带默认值30即可。 ![](https://www.taohui.pub/wp-content/uploads/2017/03/jmeter属性的使用-2.jpg) 2、我们需要循环使用一系列值用于某个用例，且每个值与循环到第几次有关时，可以在循环中使用计数器。 这时需要注意，如果在thread loop里计数器会一直累加，如果希望在每次thread loop中重新清零，要选择reset。 ![](https://www.taohui.pub/wp-content/uploads/2017/03/jmeter计数器-2.jpg) 3、有时，我们需要构造浮点式的随机数。而jmeter默认的随机数只有整型。此时，可以利用请求中都是字符串，以字符串默认连接组合的方式构造浮点数。 4、当我们需要构造一些测试值，但自带的jmeter函数并不支持时，可以考虑能够直接使用原生java代码生成变量的beanshell。 例如，我们需要构造一个日期为前天，自带的\_\_time只能获取到当前日期。而加入一个beanshell PreProcesser就可以加入java代码得到值。 其中，beanshell里生成的变量，可以调用vars.set(key,value)设置到jmeter上下文中。而想使用已经存在的jmeter上下文中的变量时，则可以使用vars.get(key)。需要注意，返回的value是字符串类型。 ![](https://www.taohui.pub/wp-content/uploads/2017/04/jmeter的beanshell_PreProcesser-2.jpg) 5、做条件判断时，我们很可能会做多个条件组合的判断，而默认的jmeter if controller是不支持的。此时可以这么用：

```
${__javaScript(${count}<60 && ${code}=="5001")}
```