---
title: linux内核调度算法（3）--多核系统的负载均衡
tags:
  - kernel
id: '166'
categories:
  - - 算法
date: 2015-01-27 16:52:00
---

多核CPU现在很常见，那么问题来了，一个程序在运行时，只在一个CPU核上运行？还是交替在多个CPU核上运行呢？[Linux](http://lib.csdn.net/base/linux "Linux知识库")内核是如何在多核间调度进程的呢？又是内核又是CPU核，两个核有点绕，下面称CPU处理器来代替CPU核。 实际上，如果你没有对你的进程做过特殊处理的话，LINUX内核是有可能把它放到多个CPU处理器上运行的，这是内核的负载均衡。上文说过，每个处理器上有一个runqueue队列，表示这颗处理器上处于run状态的进程链表，在多处理器的内核中，就会有多个runqueue，而如果他们的大小很不均衡，就会触发内核的load\_balance函数。这个函数会把某个CPU处理器上过多的进程移到runqueue元素相对少的CPU处理器上。 举个例子来简单说明这个过程吧。当我们刚fork出一个子进程时，子进程也还在当前CPU处理器的runqueue里，它与父进程均分父进程的时间片。当然，时间片与多处理器间的负载均衡没有关系。假设我们的系统是双核的，父进程运行在cpu0上，那么这个fork出来的进程也是在cpu0的runqueue中。 那么，什么时候会发生负载均衡呢？ 1、当cpu1上的runqueue里一个可运行进程都没有的时候。这点很好理解，cpu1无事可作了，这时在cpu1上会调用load\_balance，发现在cpu0上还有许多进程等待运行，那么它会从cpu0上的可运行进程里找到优先级最高的进程，拿到自己的runqueue里开始执行。 2、第1种情形不适用于运行队列一直不为空的情况。例如，cpu0上一直有10个可运行进程，cpu1上一直有1个可运行进程，显然，cpu0上的进程们得到了不公平的对待，它们拿到cpu的时间要小得多，第1种情形下的load\_balance也一直不会调用。所以，实际上，每经过一个时钟节拍，内核会调用scheduler\_tick函数，而这个函数会做许多事，例如减少当前正在执行的进程的时间片，在函数结尾处则会调用rebalance\_tick函数。rebalance\_tick函数决定以什么样的频率执行负载均衡。

```
static void rebalance_tick(int this_cpu, runqueue_t *this_rq,  
               enum idle_type idle)  
{  
    unsigned long old_load, this_load;  
    unsigned long j = jiffies + CPU_OFFSET(this_cpu);  
    struct sched_domain *sd;  
  
    /* Update our load */  
    old_load = this_rq->cpu_load;  
    this_load = this_rq->nr_running * SCHED_LOAD_SCALE;  
    /* 
     * Round up the averaging division if load is increasing. This 
     * prevents us from getting stuck on 9 if the load is 10, for 
     * example. 
     */  
    if (this_load > old_load)  
        old_load++;  
    this_rq->cpu_load = (old_load + this_load) / 2;  
  
    for_each_domain(this_cpu, sd) {  
        unsigned long interval;  
  
        if (!(sd->flags & SD_LOAD_BALANCE))  
            continue;  
  
        interval = sd->balance_interval;  
        if (idle != SCHED_IDLE)  
            interval *= sd->busy_factor;  
  
        /* scale ms to jiffies */  
        interval = msecs_to_jiffies(interval);  
        if (unlikely(!interval))  
            interval = 1;  
  
        if (j - sd->last_balance >= interval) {  
            if (load_balance(this_cpu, this_rq, sd, idle)) {  
                /* We've pulled tasks over so no longer idle */  
                idle = NOT_IDLE;  
            }  
            sd->last_balance += interval;  
        }  
    }  
}  
```

  当idle标志位是SCHED\_IDLE时，表示当前CPU处理器空闲，就会以很高的频繁来调用load\_balance（1、2个时钟节拍），反之表示当前CPU并不空闲，会以很低的频繁调用load\_balance（10-100ms）。具体的数值要看上面的interval了。 当然，多核CPU也有许多种，例如INTEL的超线程技术，而LINUX内核对一个INTEL超线程CPU会看成多个不同的CPU处理器。 上面说过，如果你没有对你的进程做过特殊处理的话，LINUX内核是有可能把它放到多个CPU处理器上运行的，但是，有时我们如果希望我们的进程一直运行在某个CPU处理器上，可以做到吗？内核提供了这样的系统调用。系统调用sched\_getaffinity会返回当前进程使用的cpu掩码，而sched\_setaffinity则可以设定该进程只能在哪几颗cpu处理器上执行。当我们对某些进程有强烈的期待，或者想自己来考虑CPU间的负载均衡，可以这么试试哈。