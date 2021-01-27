---
title: git仓库代码统计
tags:
  - git
  - git_stats
  - ruby
id: '505'
categories:
  - - 编程语言
date: 2018-04-17 15:52:36
---

虽然以代码行数来衡量项目或者程序员并不是一件靠谱的事，但是从统计角度看趋势对于技术管理人员还是很有帮助的！推荐一个比较好用的git仓库代码统计工具：git\_stats，它用于按git提交人、提交次数、修改文件数、代码行数、注释量在时间维度上进行统计，亦可按各文件类型进行简单的统计，非常方便。实际上，这么多功能通常都是用WEB在多个页面上显示的，git\_stats也是如此，它需要你先安装好ruby以生成基础的页面，再用gem安装好git\_stats，最后用git\_stats一条语句即可生成展示页面。这些静态页面如需共享，那么搭个nginx显示静态页面即可。

<!-- more -->
废话不多说，演示下步骤： 1、首先到ruby官网（[http://www.ruby-lang.org/en/downloads/](http://www.ruby-lang.org/en/downloads/)）上下载最新源码包，例如2.5.1版本，解决后，执行linux下以源码安装习惯用的三招：configure/make/make install。 2、接下来使用gem安装git\_stats命令：

```
gem install git_stats
```

3、最后进入你要统计的git代码仓库根目录下，执行命令：

```
git_stats generate -o stats --language zh_tw
```

这里，-o是指定了html页面的输出目录，而输出目录里共包含了以下页面：

```
├── activity
│   ├── by_date.html  #按日期统计活跃度
│   ├── day_of_week.html
│   ├── hour_of_day.html
│   ├── hour_of_week.html
│   ├── month_of_year.html
│   ├── year.html
│   └── year_month.html
├── assets  #库文件
├── authors  #作者数量，并可按作者进行活跃度统计
│   ├── administrator   #每一个作者一个目录
│   │   ├── activity
│   │   │   ├── by_date.html
│   │   │   ├── day_of_week.html
│   │   │   ├── hour_of_day.html
│   │   │   ├── hour_of_week.html
│   │   │   ├── month_of_year.html
│   │   │   ├── year.html
│   │   │   └── year_month.html
│   │   └── author_details
│   │       ├── changed_lines_by_date.html
│   │       ├── commits_by_date.html  
│   │       ├── deletions_by_date.html
│   │       └── insertions_by_date.html
│   ├── best_authors.html
│   ├── changed_lines_by_author_by_date.html
│   ├── commits_sum_by_author_by_date.html
│   ├── deletions_by_author_by_date.html
│   └── insertions_by_author_by_date.html
├── comments  #注释统计
│   └── by_date.html
├── files    #文件统计
│   ├── by_date.html
│   └── by_extension.html
├── general.html
├── index.html
└── lines   #代码行统计
    ├── by_date.html
    └── by_extension.html
```

4、搭建nginx用以展示页面。实际上仅需要在配置好的location内加个alias指向上一步中-o选项生成的目录即可。 可见，该工具生成的页面有助于我们统计代码库中总体的代码提交趋势，以及每个coder的代码提交趋势，对于技术管理是有意义的。以下是我截取的最好用的两个页面，首先是按日期统计活跃度： ![](http://www.taohui.pub/wp-content/uploads/2018/04/gitstats按日期活跃度统计-2.jpg) 按日期统计代码行数也很好用，虽然代码行数并不能反映出个人的贡献量，但是一些明显不靠谱的事还是能够从这里发现的。 ![](/2018/04/gitstats按日期统计代码行数-2.jpg)