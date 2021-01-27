---
title: 阿里云搭建wordpress生产级CMS网站实践
tags:
  - docker
  - wordpress
  - 高可用
id: '23'
categories:
  - - web
date: 2017-01-27 16:47:41
---

搭建cms内容站点时，wordpress是一个很好的选择，不用做任何开发就可以通过配置、插件获得丰富的功能。用[Docker](http://lib.csdn.net/base/docker "Docker知识库")容器技术部署运维都非常简单，特别是对于wordpress这种我们无需做任何开发的组件。而出于低成本考虑，公有云都是一个最佳选择，这里我选择阿里云。为了提速，wordpress前会有一个nginx作为负载均衡和web加速服务器，将静态内容都由nginx处理。出于高可靠性，我选用阿里云的容器服务（目前免费），由它来管理所有容器。而多容器间的磁盘目录共享，阿里云提供了oss映射和nas盘，oss盘的速度太慢，而nas盘IO与云盘相当，可以作为第一选择。[数据库](http://lib.csdn.net/base/mysql "MySQL知识库")为了可靠性，如果没有独立的DB运维人员，就选择rds [MySQL](http://lib.csdn.net/base/mysql "MySQL知识库")好了（目前单机版RDS刚上线，相比双机mysql便宜很多）。 一、最初我们验证方案时，总是先从最简单的做起，功能满足要求即可。 1、我们购买好ECS（如果是经典网络一定要含公网带宽，否则接下来用到阿里云容器服务时你会悲催的发现，目前阿里云容器要求集群内的经典网络节点必须含有公网带宽， 否则该ECS无法加入集群中。VPC网络就没有这个问题。在可见的一段时间内，阿里云可能都不会修复这个问题。），[操作系统](http://lib.csdn.net/base/operatingsystem "操作系统知识库")如果是centos7.0或者7.2，高内核版本支持docker就更简单了。 2、数据库在centos7.0以上版本时，mariadb比较方便（出于性能考虑，docker容器的db不推荐为生产环境下的数据库）。yum install这个数据库，修改/etc/my.cnf，使其支持utf-8，例如：

```
[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

用service mariadb start启动数据库。

2、用create user命令创建好用户，create database创建好db，grant命令赋予权限，例如：

```
create database yourdb;
CREATE USER youruser IDENTIFIED BY 'yourpass';
GRANT ALL PRIVILEGES ON yourdb.* TO youruser;
flush privileges;
```

3、用docker pull wordpress命令拉下官方最新版本的image，然后用docker run将其启动。-p端口映射到主机的80端口上。注意，很多image都会通过提供环境变量来修改配置，而wordpress也是如此，这里我们主要是修改其连接哪个数据库，通常这四个环境变量配置好即可。

```
-e WORDPRESS_DB_HOST=...
-e WORDPRESS_DB_USER=...
-e WORDPRESS_DB_PASSWORD=...
-e WORDPRESS_DB_NAME=...
```

4、接下来我们将购买的域名在阿里云的云解析页面上，配置到该公网IP上。（wordpress里需要配置域名，直接使用域名比IP方便）。

5、在页面上打开域名，显示wordpress的安装站点页面。根据提示傻瓜化的安装这个wordpress，接下来我们就可以体验下wordpress强大的CMS功能了。 二、接下来，我们需要一个更美观的站点，而wordpress提供主题自定义功能。我们可以在google上找到很多免费或者收费的主题。接下来美化这个站点，使之符合功能需求。 1、首先找到符合要求的主题，下载后一般是一个zip文件。 2、在wordpress的/wp-admin页面下进入管理界面，由外观->主题->添加->上传主题页面里，点击选择文件，将zip主题包上传。 通常在这个步骤中，我们可能会看到上传失败的结果。提示上传文件过大（特别是你下载的zip主题包达到几十M的时候）。这是[PHP](http://lib.csdn.net/base/php "PHP知识库")服务默认允许的上传文件过小所致。要想解决，首先得改image的配置。此时，我们首先要重新运行docker，把容器的/var/www/html目录映射到主机目录中（用-v 主机目录：容器目录，这个volumes命令可以把容器的磁盘内容映射到主机目录中供我们修改）。我们在映射目录下创建.htaccess文件，在此文件中输入以下配置项：

```
php_value upload_max_filesize 64M
php_value post_max_size 64M
php_value max_execution_time 300
php_value max_input_time 300
```

这里把上传文件的大小增加到64M。再启动docker容器，上传并安装新主题即可。

三、接着，多半会发现这个站点太慢了，体验很差，我们希望网站速度更快一点。 1、用浏览器的debug模式可以发现，最慢的请求从url看都是获取gravatar头像，请求之所以慢与墙有关（提供头像服务的机器网络不稳定）。最简单的解决办法是在wp-content/themes/your-theme-using目录下，在functions.php文件的结尾加上以下几行（参见[http://www.dmeng.net/wordpress-replace-gravatar-host.html](http://www.dmeng.net/wordpress-replace-gravatar-host.html)）：

```
function dmeng_get_https_avatar($avatar){
    $avatar = str_replace(array("www.gravatar.com", "0.gravatar.com", "1.gravatar.com", "2.gravatar.com"), "secure.gravatar.com", $avatar);
    return $avatar;
}
add_filter('get_avatar', 'dmeng_get_https_avatar');
```

2、增加一台nginx，域名直接指向nginx所在的机器，由nginx将动态请求反向代理给wordpress容器。而静态内容由nginx处理。

此时我们可能会遇到网站无法访问的情况，在nginx的日志里可以看到是301重定向过多导致。解决办法还是在上面的functions.php文件的结尾加上一行：

```
remove_filter('template_redirect', 'redirect_canonical');
```

我们还可能遇到上传新的主题文件时得到413错误，这是nginx拒绝所致，记得在nginx.conf里加上client\_max\_body\_size 60M; 3、nginx还可以配成http2，但后面得用阿里云的slb防单点。 四、一台访问速度和功能都满足我们需求的CMS站点出现了，接下来，我们开始解决可靠性问题。首先，我们需要把单点数据库改为RDS数据库（如果你的数据库有人全职维护那就不需要）。购买一个rds实例，建议与你的ECS在同一个可用区（同一机房内，带宽高又稳定）。目前RDS单实例对于使用wordpress作为站点的公司来说应该够了吧？ 1、在web站点上初始化root用户密码（注意，阿里云的RDS用户权限比自建的小了很多!）。 2、加上访问白名单。通常我们是ECS内网访问数据库（便宜安全），所以将ECS内网IP加入白名单。 3、迁移数据。这里我悲催了，阿里云RDS提供的迁移工具只能是mysql对mysql迁移，而我之前用的是相似的mariadb，迁移工具执行失败，提工单后售后反馈暂时不支持。 不同种类数据库的迁移这种情况下，用sql导入肯定是可行的。于是准备用mysqldump把mariadb中数据库的数据导出sql文件，出现错误： mysqldump: Error: Binlogging on server not active 在my.cnf上加入

```
log_bin=mysql-bin
```

再执行：

```
mysqldump --databases yourdb --user=youruser -hyourhost --password --master-data > transfer.sql
```

先用root登上RDS，建数据库建用户。再导入时发现还是不行：ERROR 1227 (42000) at line 22: Access denied; you need (at least one of) the SUPER privilege(s) for this operation 这是生成的sql文件里有RDS不支持的操作。将其中最上面的两行删掉：

```
CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=191500;
CREATE DATABASE /*!32312 IF NOT EXISTS*/ `yourdb` /*!40100 DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci */;
```

再执行：

```
mysql -u youruser -h yourhost yourdb -p < transfer.sql
```

即完成数据库的迁移。4、最后将docker wordpress容器通过环境变量指向RDS数据库。 五、数据库单点解决掉后，再来解决wordpress [Container](http://lib.csdn.net/base/docker "Docker知识库")容器的单点与运维。这里有个待解决的问题是wordpress容器如果有多个实例，且跨ECS主机，那么就需要共享磁盘去映射容器中的/var/www/html目录（特别是其中包括uploads上传文件的目录）。 1、首先，我们要利用阿里云容器将dockoer容器运维起来的优势，将wordpress放在容器服务里运行。 2、其次，阿里云容器支持OSS或者NAS盘映射到集群内每一台ECS机器上某个目录。即，如果我们配置数据卷支持OSS和NAS（当然需要先购买这两种服务），集群内每个ECS都将自动的多出/mnt/acs\_mnt/nas和/mnt/acs\_mnt/ossfs目录，方便我们每个contain容器进行映射。 这里需要注意，如果是oss映射为磁盘，必须在其他参数里增加“-o umask=000"，否则docker容器每次新生成的文件其权限是有问题的，无法访问，这个BUG阿里云可能以后会解决吧。 另外，大家会强烈的感受到用oss去映射/var/www/html目录网站就会非常慢，这是因为oss本身的时延就高，而改成nas盘就会好多了。（但oss盘比nas盘便宜很多） 当然，用oss可以非常方便的使用cdn，nas就没有这么便利了。 ![](http://taohui.tech/wp-content/uploads/2017/01/unnamed-file.jpg) 最终[架构](http://lib.csdn.net/base/architecture "大型网站架构知识库")如上所示。