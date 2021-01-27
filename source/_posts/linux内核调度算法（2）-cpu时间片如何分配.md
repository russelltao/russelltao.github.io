---
title: linux内核调度算法（2）--CPU时间片如何分配
tags:
  - cpu
  - kernel
  - nice
id: '164'
categories:
  - - 算法
date: 2015-01-27 16:51:01
---

内核在微观上，把CPU的运行时间分成许多分，然后安排给各个进程轮流运行，造成宏观上所有的进程仿佛同时在执行。双核CPU，实际上最多只能有两个进程在同时运行，大家在top、vmstat命令里看到的正在运行的进程，并不是真的在占有着CPU哈。 所以，一些设计良好的高性能进程，比如nginx，都是实际上有几颗CPU，就配几个工作进程，道理就在这。比如你的服务器有8颗CPU，那么nginx worker应当只有8个，当你多于8个时，内核可能会放超过多个nginx worker进程到1个runqueue里，会发生什么呢？就是在这颗CPU上，会比较均匀的把时间分配给这几个nginx worker，每个worker进程运行完一个时间片后，内核需要做进程切换，把正在运行的进程上下文保存下来。假设内核分配的时间片是100ms，做进程切换的时间是5ms，那么进程性能下降还是很明显的，跟你配置的worker有关，越多下降得越厉害。 当然，这是跟nginx的设计有关的。nginx是事件驱动的全异步进程，本身设计上就几乎不存在阻塞和中断，nginx的设计者就希望每一个nginx worker可以独占CPU的几乎全部时间片，这点就是nginx worker数量配置的依据所在。 当然，实际的运行进程里，大部分并不是nginx这种希望独占CPU全部时间片的进程，许多进程，比如vi，它在很多时间是在等待用户输入，这时vi在等待IO中断，是不占用时间片的，内核面对多样化的进程，就需要技巧性的分配CPU时间片了。 内核分配时间片是有策略和倾向性的。换句话说，内核是偏心的，它喜欢的是IO消耗型进程，因为这类进程如果不能及时响应，用户就会很不爽，所以它总会下意识的多分配CPU运行时间给这类进程。而CPU消耗进程内核就不太关心了。这有道理吗？太有了，CPU消耗型慢一点用户感知不出来，电信号和生物信号运转速度差距巨大。虽然内核尽量多的分配时间片给IO消耗型进程，但IO消耗进程常常在睡觉，给它的时间片根本用不掉。很合理吧？ 那么内核具体是怎么实现这种偏心呢？通过动态调整进程的优先级，以及分配不同长短的CPU时间处来实现。先说内核如何决定时间片的长度。 对每一个进程，有一个整型static\_prio表示用户设置的静态优先级，内核里它与nice值是对应的。看看进程描述结构里的static\_prio成员。

```
struct task_struct {  
    int prio, static_prio;  
......}  
```

nice值是什么？其实就是优先级针对用户进程的另一种表示法，nice的取值范围是-20到+19，-20优先级最高，+19最低。上篇曾经说过，内核优先级共有140，而用户能够设置的NICE优先级如何与这140个优先级对应起来呢？看代码：

```
#define MAX_USER_RT_PRIO    100  
#define MAX_RT_PRIO     MAX_USER_RT_PRIO  
#define MAX_PRIO        (MAX_RT_PRIO + 40)  
```

可以看到，MAX\_PRIO就是140，也就是内核定义的最大优先级了。

```
#define USER_PRIO(p)        ((p)-MAX_RT_PRIO)  
#define MAX_USER_PRIO       (USER_PRIO(MAX_PRIO))  
```

而MAX\_USER\_PRIO就是40，意指，普通进程指定的优先级别最多40，就像前面我们讲的那样-20到+19。

```
#define NICE_TO_PRIO(nice)  (MAX_RT_PRIO + (nice) + 20)  
```

nice值是-20表示最高，对应着static\_prio是多少呢？NICE\_TO\_PRIO(0)就是120，NICE\_TO\_PRIO(-20)就是100。 当该进程刚被其父进程fork出来时，是平分其父进程的剩余时间片的。这个时间片执行完后，就会根据它的初始优先级来重新分配时间片，优先级为+19时最低，只分配最小时间片5ms，优先级为0时是100ms，优先级是-20时是最大时间片800ms。我们看看内核是如何计算时间片长度的，大家先看下task\_timeslice时间片计算函数：

```
#define SCALE_PRIO(x, prio) \  
    max(x * (MAX_PRIO - prio) / (MAX_USER_PRIO/2), MIN_TIMESLICE)  
  
static unsigned int task_timeslice(task_t *p)  
{  
    if (p->static_prio < NICE_TO_PRIO(0))  
        return SCALE_PRIO(DEF_TIMESLICE*4, p->static_prio);  
    else  
        return SCALE_PRIO(DEF_TIMESLICE, p->static_prio);  
}  
```

这里有一堆宏，我们把宏依次列出看看它们的值：

```
# define HZ     1000      
#define DEF_TIMESLICE       (100 * HZ / 1000)  
```

所以，DEF\_TIMESLICE是100。假设nice值是-20，那么static\_prio就是100，那么SCALE\_PRIO(100\*4, 100)就等于800，意味着最高优先级-20情形下，可以分到时间片是800ms，如果nice值是+19，则只能分到最小时间片5ms，nice值是默认的0则能分到100ms。

貌似时间片只与nice值有关系。实际上，内核会对初始的nice值有一个-5到+5的动态调整。这个动态调整的依据是什么呢？很简单，如果CPU用得多的进程，就把nice值调高点，等价于优先级调低点。CPU用得少的进程，认为它是交互性为主的进程，则会把nice值调低点，也就是优先级调高点。这样的好处很明显，因为1、一个进程的初始优先值的设定未必是准确的，内核根据该进程的实时表现来调整它的执行情况。2、进程的表现不是始终如一的，比如一开始只是监听80端口，此时进程大部分时间在sleep，时间片用得少，于是nice值动态降低来提高优先级。这时有client接入80端口后，进程开始大量使用CPU，这以后nice值会动态增加来降低优先级。 思想明白了，代码实现上，优先级的动态补偿到底依据什么呢？effective\_prio返回动态补偿后的优先级，注释非常详细，大家先看下。

```
/* 
 * effective_prio - return the priority that is based on the static 
 * priority but is modified by bonuses/penalties. 
 * 
 * We scale the actual sleep average [0 .... MAX_SLEEP_AVG] 
 * into the -5 ... 0 ... +5 bonus/penalty range. 
 * 
 * We use 25% of the full 0...39 priority range so that: 
 * 
 * 1) nice +19 interactive tasks do not preempt nice 0 CPU hogs. 
 * 2) nice -20 CPU hogs do not get preempted by nice 0 tasks. 
 * 
 * Both properties are important to certain workloads. 
 */  
static int effective_prio(task_t *p)  
{  
    int bonus, prio;  
  
    if (rt_task(p))  
        return p->prio;  
  
    bonus = CURRENT_BONUS(p) - MAX_BONUS / 2;  
  
    prio = p->static_prio - bonus;  
    if (prio < MAX_RT_PRIO)  
        prio = MAX_RT_PRIO;  
    if (prio > MAX_PRIO-1)  
        prio = MAX_PRIO-1;  
    return prio;  
}  
```

  可以看到bonus会对初始优先级做补偿。怎么计算出这个BONUS的呢？

```
#define CURRENT_BONUS(p) \  
    (NS_TO_JIFFIES((p)->sleep_avg) * MAX_BONUS / \  
        MAX_SLEEP_AVG)  
```

可以看到，进程描述符里还有个sleep\_avg，动态补偿完全是根据它的值来运作的。sleep\_avg就是关键了，它表示进程睡眠和运行的时间，当进程由休眠转到运行时，sleep\_avg会加上这次休眠用的时间。在运行时，每运行一个时钟节拍sleep\_avg就递减直到0为止。所以，sleep\_avg越大，那么就会给到越大的动态优先级补偿，达到MAX\_SLEEP\_AVG时会有nice值-5的补偿。 内核就是这么偏爱交互型进程，从上面的优先级和时间片分配上都能看出来。实际上，内核还有方法对交互型进程搞优待。上篇说过，runqueue里的active和expired队列，一般的进程时间片用完后进expired队列，而对IO消耗的交互型进程来说，则会直接进入active队列中，保证高灵敏的响应，可见什么叫万千宠爱于一身了。