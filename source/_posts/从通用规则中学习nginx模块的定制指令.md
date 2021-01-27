---
title: 从通用规则中学习Nginx模块的定制指令
tags:
  - nginx
  - 时间单位
  - 空间单位
  - 配置语法
id: '1212'
categories:
  - - nginx
  - - web
date: 2020-12-23 15:14:21
---

[上一篇文章](https://www.taohui.pub/2020/12/22/%e5%a6%82%e4%bd%95configure%e5%ae%9a%e5%88%b6%e5%87%ba%e5%b1%9e%e4%ba%8e%e4%bd%a0%e7%9a%84nginx%ef%bc%9f/)中，我介绍了如何定制属于你自己的Nginx，本文将介绍nginx.conf文件的配置语法、使用方式，以及如何学习新模块提供的配置指令。

每个Nginx模块都可以定义自己的配置指令，所以这些指令的格式五花八门。比如content\_by\_lua\_block后跟着的是Lua语法，limit\_req\_zone后则跟着以空格、等号、冒号等分隔的多个选项。这些模块有没有必然遵循的通用格式呢？如果有，那么掌握了它，就能快速读懂生产环境复杂的nginx.conf文件。
<!-- more -->
其次，我们又该如何学习个性化十足的模块指令呢？其实，所有Nginx模块在介绍它的配置指令时，都遵循着相同的格式：Syntax、Default、Context、Description，这能降低我们的学习门槛。如果你还不清楚这一套路，那就只能学习其他文章翻译过的二手知识，效率很低。

比如搭建静态资源服务用到的root、alias指令，该如何找到、阅读它的帮助文档？为什么官方更推荐使用root指令？alias指令又适合在哪些场景中使用呢？


nginx.conf配置文件中的语法就像是一门脚本语言，你既可以定义变量（set指令），也可以控制条件分支（if指令），还有作用域的概念（server{}块、location{}块等）。所以，为复杂的业务场景写出正确的配置文件，并不是一件很容易的事。为此，**Nginx特意针对vim编辑器提供了语法高亮功能**，但这需要你手动打开，尤其是include文件散落在磁盘各处时。

本文将会系统地介绍nginx.conf配置文件的用法，并以搭建静态资源服务时用到的root、alias指令为例，看看如何阅读Nginx模块的指令介绍。同时，本文也是Nginx开源社区基础培训系列课程第一季，即6月11日晚第2次视频直播的部分文字总结。

## 快速掌握Nginx配置文件的语法格式

Nginx是由少量框架代码、大量模块构成的，其中，Nginx框架会按照特定的语法，将配置指令读取出来，再交由模块处理。**因此，Nginx框架定义了通用的语法规则，而Nginx模块则定义了每条指令的语法规则，**作为初学者，如果将学习目标定为掌握所有的配置指令，方向就完全错了，而且这是不可能完成的任务。

比如，ngx\_http\_lua\_module模块定义了content\_by\_lua\_block指令，只要它符合框架定义的{}块语法规则，哪怕大括号内是一大串Lua语言代码，框架也会把它交由ngx\_http\_lua\_module模块处理。因此，下面这行指令就是合法的：

```
content_by_lua_block {ngx.say("Hello World ")}
```

再比如，ngx\_http\_limit\_req\_module模块定义了limit\_req\_zone指令，只要它符合指令行语法（以分号;结尾），框架就会将指令后的选项将由模块处理。所以，即使下面这行指令出现了r/s（每秒处理请求数）这样新定义的单位，仍然是合法的：

```
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```

所以，在我看来，只要弄清楚了以下2点，就能快速掌握Nginx配置文件，：

1.  **Nginx框架定义了每条指令的基本格式，这是所有模块必须遵守的规则**，这包括以下5条语法：

*   通过{}大括号作为分隔符的配置块语法。比如http{ }、location{ }、upstream{ }等等，至于配置块中究竟是放置Javascript语言、Lua语言还是字符串、数字，这完全由定义配置块的Nginx模块而定。
*   通过;分号作为分隔符的指令语法。比如root html;就打开了静态资源服务。
*   以#作为关键字的注释语法。比如#pid logs/nginx.pid;指令就是不会生效的。
*   以$作为关键字的变量语法。变量是Nginx模块之间能够互相配合的核心要素，也是Nginx与管理员之间的重要接口，通过$变量名的形式，就可以灵活控制Nginx模块的行为。下一篇文章我会详细介绍Nginx变量。
*   include指令可以将其他配置文件载入到nginx.conf中，这样可以提升配置的可维护性。例如includemime.types;语句，就将Content-Type与文件后缀名的映射关系，放在了独立的mime.types文件中，降低了耦合性。

2.  **Nginx框架为了提高模块解析指令选项的效率，提供了一系列通用的工具函数，绝大多数模块都会使用它们**，毕竟这降低了模块开发的难度以及用户的学习成本。比如，当配置文件中包含字节数时，Nginx框架提供了ngx\_conf\_set\_size\_slot函数， 各模块通过它就可以解析以下单位：


|  空间单位   | 意义  |
|  ----  | ----  |
| k/K  | KB |
| m/M  | MB |
| g/G  | GB |


因此，limit\_req\_zone指令中zone=one:10m中就定义10MB的共享内存，这替代了很不好理解的10485760字节。

再比如，读取时间可以使用以下单位：


|  时间单位   | 意义  |
|  ----  | ----  |
| ms  | 毫秒 |
| s  | 秒 |
| m  | 分钟 |
| h  | 小时 |
| d  | 天 |
| w  | 周 |
| M  | 月 |
| y  | 年 |


这样，ssl\_session\_cache shared:SSL:2h;指令就设置TLS会话信息缓存2小时后过期。

除以上规则外，如果编译了pcre开发库后，你还可以在nginx.conf中使用正则表达式，它们通常以~符号打头。

## 如何使用Nginx配置文件？

掌握了语法规则后，nginx.conf配置文件究竟是放在哪里的呢？

编译Nginx时，configure脚本的--prefix选项可以设置Nginx的运行路径，比如：

```
./configure –prefix=/home/nginx
```

此时，安装后的Nginx将会放在/home/nginx目录，而配置文件就会在/home/nginx/conf目录下。

如果你没有显式的指--prefix选项，默认路径就是/usr/local/nginx。由于OpenResty修改了configure文件，因此它的默认路径是/usr/local/openresty/nginx。在默认路径确定后，nginx.conf配置文件就会放在conf子目录中。当然，通过--conf-path选项，你可以分离它们。

另外在运行Nginx时，你还可以通过nginx -c PATH/nginx.conf选项，指定任意路径作为Nginx的配置文件。

由于配置语法比较复杂，因此Nginx为[vim编辑器](https://zh.wikipedia.org/zh-hans/Vim)准备了语法高亮功能。在Nginx源代码中，你可以看到contrib目录，其中vim子目录提高了语法高亮功能：

```
[contrib]# tree vim
vim
-- ftdetect
 `-- nginx.vim
-- ftplugin
 `-- nginx.vim
-- indent
 `-- nginx.vim
`-- syntax
`-- nginx.vim
```

当你将contrib/vim/\* 复制到~/.vim/目录时（~表示你当前用户的默认路径，如果.vim目录不存在时，请先用mkdir创建），再打开nginx.conf你就会发现指令已经高亮显示了：

[![](/2020/12/nginx-vim语法高亮.png)](/2020/12/nginx-vim语法高亮.png)

出于可读性考虑，你或许会将include文件放在其他路径下，此时再用vim打开这些子配置文件，可能没有语法高亮效果。这是因为contrib/vim/ftdetect/nginx.vim文件定义了仅对4类配置文件使用语法高亮规则：

```
//对所有.nginx后缀的配置文件语法高亮
au BufRead,BufNewFile *.nginx set ft=nginx

//对/etc/nginx/目录下的配置文件语法高亮
au BufRead,BufNewFile */etc/nginx/* set ft=nginx

//对/usr/local/nginx/conf/目录下的配置文件语法高亮
au BufRead,BufNewFile */usr/local/nginx/conf/* set ft=nginx

//对任意路径下，名为nginx.conf的文件语法高亮
au BufRead,BufNewFile nginx.conf set ft=nginx
```

因此，你可以将这类文件的后缀名改为.nginx，或者将它们移入/etc/nginx/、/usr/local/nginx/conf/目录即可。当然，你也可以向ftdetect/nginx.vim添加新的识别目录。

即使拥有语法高亮功能，对于生产环境中长达数百、上千行的nginx.conf，仍然难以避免出现配置错误。这时可以通过nginx -t或者nginx -T命令，检查配置语法是否正确。出现错误时，**Nginx会在屏幕上给出错误级别、原因描述以及到底是哪一行配置出现了错误**。例如：

```
# nginx -t
nginx: [emerg] directive "location" has no opening "{" in /usr/local/nginx/conf/notflowlocation.conf:1281
nginx: configuration file /usr/local/nginx/conf/nginx.conf test failed
```

从上面的错误信息中，我们知道Nginx解析配置文件失败，错误发生在include的子配置文件/usr/local/nginx/conf/notflowlocation.conf的第1281行，从描述上推断是location块的配置出现了错误，可能是缺失了大括号，或者未转义的字符导致无法识别出大括号。

当你修改完配置文件后，可以通过nginx -s reload命令重新载入指令。这一过程不会影响正在服务的TCP连接，在描述Nginx进程架构的文章中，我会详细解释其原因。

## 搭建静态资源服务，root与alias有何不同？

接下来我们以root和alias指令为例，看看如何掌握配置指令的使用方法。

**配置指令的说明，被放置在它所属Nginx模块的帮助文档中**。因此，如果你对某个指令不熟悉，要先找到所属模块的说明文档。对于官方模块，你可以进入nginx.org站点查找。搭建静态资源服务的root/alias指令是由ngx\_http\_core\_module模块实现的，因此，我们可以进入[http://nginx.org/en/docs/http/ngx\_http\_core\_module.html](http://nginx.org/en/docs/http/ngx_http_core_module.html)页面寻找指令介绍，比如root指令的介绍如下所示：

```
Syntax:root path;
Default: root html;
Context:http, server, location, if in location
```

这里Syntax、Default、Context 3个关键信息，是所有Nginx配置指令共有的，下面解释下其含义：

*   Syntax：表示指令语法，包括可以跟几个选项，每个选项的单位、分隔符等。root path指令，可以将URL映射为磁盘访问路径path+URI，比如URL为/img/a.jpg时，磁盘访问路径就是html/img/a.jpg。

注意，这里path既可以是相对路径，也可以是绝对路径。作为相对路径，path的前缀路径是由configure --prefix指定，也可以在运行时由nginx -p path指定。

*   Default：表示选项的默认值，也就是说，即使你没有在nginx.conf中写入root指令，也相当于配置了root html;
*   Context：表示指令允许出现在哪些配置块中。比如root可以出现在server{}中，而alias则只能出现在location{}中。为什么root指令的Context，允许其出现在http{ }、server{ }、location { }、if { }等多个配置块中呢?

这是因为，Nginx允许多个配置块互相嵌套时，相同指令可以向上继承选项值。例如下面两个配置文件是完全等价的：

```
server{
  root html;
    location / {
  }
}

server{
  root html;
  location / {
    root html;
  }
｝
```

这种向上承继机制，可以简化Nginx配置文件。因此，使用root指令后，不用为每个location块重复写入root指令。相反，alias指令仅能放置在location块中，这与它的使用方式有关：

```
Syntax:alias path;
Default:—
Context:location
```

alias的映射关系与其所属的location中匹配的URI前缀有关，比如当HTTP请求的URI为/a/b/c.html时，在如下配置中，实际访问的磁盘路径为/d/b/c.html：

```
location /a {
  alias /d;
}
```

因此，**当URI中含有磁盘路径以外的前缀时，适合使用alias指令。反之，若完整的URI都是磁盘路径的一部分时，则不妨使用root指令**。学习其他指令时，如果你不清楚它属于哪一个模块，还可以**查看以字母表排序的指令索引**[http://nginx.org/en/docs/dirindex.html](http://nginx.org/en/docs/dirindex.html)页面，点击后会进入所属模块的指令介绍页面。

如果是第三方模块，通常在README文件中会有相应的指令介绍。比如OpenResty模块的指令会放在GitHub项目首页的README文件中：

[![](/2020/12/第三方模块说明文档.png)](http://www.taohui.pub/2020/12/23/%e4%bb%8e%e9%80%9a%e7%94%a8%e8%a7%84%e5%88%99%e4%b8%ad%e5%ad%a6%e4%b9%a0nginx%e6%a8%a1%e5%9d%97%e7%9a%84%e5%ae%9a%e5%88%b6%e6%8c%87%e4%bb%a4/%e7%ac%ac%e4%b8%89%e6%96%b9%e6%a8%a1%e5%9d%97%e8%af%b4%e6%98%8e%e6%96%87%e6%a1%a3/)

而TEngine模块的指令介绍则会放在tengine.taobao.org网站上：

[![](/2020/12/tengine说明文档.png)](http://www.taohui.pub/2020/12/23/%e4%bb%8e%e9%80%9a%e7%94%a8%e8%a7%84%e5%88%99%e4%b8%ad%e5%ad%a6%e4%b9%a0nginx%e6%a8%a1%e5%9d%97%e7%9a%84%e5%ae%9a%e5%88%b6%e6%8c%87%e4%bb%a4/tengine%e8%af%b4%e6%98%8e%e6%96%87%e6%a1%a3/)

从这两张截图中可以看到，第三方模块在解释指令的用法时，同样遵循着上文介绍过的方式。

## 小结

本文介绍了Nginx配置文件的使用方法。

学习Nginx的通用语法时，要先掌握Nginx框架解析配置文件的5条基本规则，这样就能读懂nginx.conf的整体结构。其次，当模块指令包含时间、空间单位时，会使用Nginx框架提供的通用解析工具，熟悉这些时、空单位会降低你学习新指令的成本。

配置文件的位置，可以由编译期configure脚本的—prefix、--conf-path选项指定，也可以由运行时的-p选项指定。复杂的配置文件很容易出错，通过nginx -t/T命令可以检测出错误，同时屏幕上会显示出错的文件、行号以及原因，方便你修复Bug。

用vim工具编辑配置文件时，将Nginx源码中contrib/vim/目录复制到~/.vim/目录，就可以打开语法高亮功能。对于子配置文件，只有放置在/etc/nginx或者/usr/local/nginx/conf目录中，或者后缀为.nginx时，才会高亮显示语法。当然，你可以通过ftdetect/nginx.vim文件修改这一规则。

学习模块指令时，要从它的帮助文档中找到指令的语法、默认值、上下文和描述信息。比如，root和alias的语法相似，但alias没有默认值，仅允许出现在location上下文中，这实际上与它必须结合URI前缀来映射磁盘路径有关。

**由于每个Nginx模块都能定义独特的指令，这让nginx.conf变成了复杂的运维界面。在掌握了基本的配置语法，以及第三方模块定义指令时遵循的潜规则后，你就能游刃有余地编写Nginx配置文件。**