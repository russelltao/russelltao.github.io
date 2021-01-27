---
title: linux内核调度算法（1）--快速找到最高优先级进程
tags:
  - kernel
  - linux
  - 调度算法
id: '162'
categories:
  - - 算法
date: 2015-01-27 16:47:01
---

为什么要了解内核的调度策略呢？呵呵，因为它值得我们学习，不算是废话吧。内核调度程序很先进很强大，管理你的[Linux](http://lib.csdn.net/base/linux "Linux知识库")上跑的大量的乱七八糟的进程，同时还保持着对用户操作的高灵敏响应，如果可能，为什么不把这种思想放到自己的应用程序里呢？或者，有没有可能更好的实现自己的应用，使得[操作系统](http://lib.csdn.net/base/operatingsystem "操作系统知识库")能够以自己的意志来分配资源给自己的进程？ 带着这两个问题来看看KERNEL。首先回顾上我们开发应用程序，基本上就两种类型，1、IO消耗型：比如hadoop上的trunk服务，很明显它的消耗主要在IO上，包括网络IO磁盘IO等等。2、CPU消耗型，比如mapreduce或者其他的需要对大量数据进行计算处理的组件，就象对高清视频压缩成适合手机观看分辨率的进程，他们的消耗主要在CPU上。当两类进程都在一台SERVER上运行时，操作系统会如何调度它们呢？现在的服务器都是SMP多核的，那么一个进程在多CPU时会来回切换吗？如果我有一个程序，既有IO消耗又有CPU消耗，怎么让多核更好的调度我的程序呢？ 又多了几个问题。来看看内核调度程序吧，我们先从它的优先队列谈起吧。调度程序代码就在内核源码的kernel/sched.c的schedule函数中。 首先看下面的优先级队列，每一个runqueue都有。runqueue是什么？下面会详细说下，现在大家可以理解为，内核为每一颗CPU分配了一个runqueue，用于维护这颗CPU可以运行的进程。runqueue里，有几个成员是prio\_array类型，这个东东就是优先队列，先看看它的定义：

```
struct prio_array {  
    unsigned int nr_active;    表示等待执行的进程总数  
    unsigned long bitmap[BITMAP_SIZE];    一个unsigned long在内核中只有32位哈，大家要跟64位OS上的C程序中的long区分开，那个是64位的。那么这个bitmap是干什么的呢？它是用位的方式，表示某个优先级上有没有待处理的队列，是实现快速找到最高待处理优先进程的关键。如果我定义了四种优先级，我只需要四位就能表示某个优先级上有没有进程要运行，例如优先级是2和3上有进程，那么就应该是0110.......非常省空间，效率也快，不是吗？  
    struct list_head queue[MAX_PRIO];     与上面的bitmap是对应的，它存储所有等待运行的进程。  
};  
```

看看BITMAP\_SIZE是怎么算出来的：#define BITMAP\_SIZE ((((MAX\_PRIO+1+7)/8)+sizeof(long)-1)/sizeof(long)) 那么，LINUX默认配置（如果你用默认选项编译内核的话）MAX\_PRIO是140，就是说一共内核对进程一共定义了140种优先级。等待某个CPU来处理的进程中，可能包含许多种优先级的进程，但，LINUX是个抢占式调度算法的操作系统，就是说，需要调度时一定是找到最高优先级的进程执行。上面的BITMAP\_SIZE值根据MAX\_PRIO算出来为5，那么bitmap实际是32\*5=160位，这样就包含了MAX\_PRIO的140位。优先级队列是怎么使用的？看2649行代码：idx = sched\_find\_first\_bit(array->bitmap);这个方法就用来快速的找到优先级最高的队列。看看它的实现可以方便我们理解这个优先级位的设计：

```
static inline int sched_find_first_bit(unsigned long *b)  
{  
    if (unlikely(b[0]))  
        return __ffs(b[0]);  
    if (unlikely(b[1]))  
        return __ffs(b[1]) + 32;  
    if (unlikely(b[2]))  
        return __ffs(b[2]) + 64;  
    if (b[3])  
        return __ffs(b[3]) + 96;  
    return __ffs(b[4]) + 128;  
}  
```

那么\_\_ffs是干什么的？

```
static inline int __ffs(int x)  
{  
    int r = 0;  
  
    if (!x)  
        return 0;  
    if (!(x & 0xffff)) {  
        x >>= 16;  
        r += 16;  
    }  
    if (!(x & 0xff)) {  
        x >>= 8;  
        r += 8;  
    }  
    if (!(x & 0xf)) {  
        x >>= 4;  
        r += 4;  
    }  
    if (!(x & 3)) {  
        x >>= 2;  
        r += 2;  
    }  
    if (!(x & 1)) {  
        x >>= 1;  
        r += 1;  
    }  
    return r;  
}  
```

sched\_find\_first\_bit返回值就是最高优先级所在队列的序号，与queue是对应使用的哈，queue = array->queue + idx;这样就取到了要处理的进程队列。这个设计在查找优先级时是非常快的，非常值得我们学习。 好，优先级队列搞明白了，现在来看看runqueue，每个runqueue包含三个优先级队列。

```
struct runqueue {  
    spinlock_t lock;   这是个自旋锁，nginx里解决惊群现象时也是用这个。与普通锁的区别就是，使用普通锁时，你去试图拿一把锁，结果发现已经被别人拿走了，你就在那睡觉，等别人锁用完了叫你起来。所以如果有一个人拿住锁了，一百个人都在门前睡觉等。当之前的人用完锁回来后，会叫醒所有100个等锁的人，然后这些人开始互相抢，抢到的人拿锁进去，其他的人继续等。自旋锁不同，当他去拿锁发现锁被别人拿走了，他在那不睡觉的等，稍打个盹就看看自己主动看看锁有没有还回来。大家比较出优劣了吗？  
  
  
    /* 
     * nr_running and cpu_load should be in the same cacheline because 
     * remote CPUs use both these fields when doing load calculation. 
     */  
    unsigned long nr_running;  
#ifdef CONFIG_SMP  
    unsigned long cpu_load;  
#endif  
    unsigned long long nr_switches;  
  
  
    /* 
     * This is part of a global counter where only the total sum 
     * over all CPUs matters. A task can increase this counter on 
     * one CPU and if it got migrated afterwards it may decrease 
     * it on another CPU. Always updated under the runqueue lock: 
     */  
    unsigned long nr_uninterruptible;  
  
  
    unsigned long expired_timestamp;  
    unsigned long long timestamp_last_tick;  
    task_t *curr, *idle;  
    struct mm_struct *prev_mm;  
    prio_array_t *active, *expired, arrays[2];上面说了半天的优先级队列在这里，但是在runqueue里，为什么不只一个呢？这个在下面讲。  
    int best_expired_prio;  
    atomic_t nr_iowait;  
    ... ...  
};  
```

  LINUX是一个时间多路复用的系统，就是说，通过把CPU执行时间分成许多片，再分配给进程们使用，造成即使单CPU系统，也貌似允许多个任务在同时执行。那么，时间片大小假设为100ms，过短过长，过长了有些不灵敏，过短了，连切换进程时可能都要消耗几毫秒的时间。分给100个进程执行，在所有进程都用完自己的时间片后，需要重新给所有的进程重新分配时间片，怎么分配呢？for循环遍历所有的run状态进程，重设时间片？这个性能无法容忍！太慢了，跟当前系统进程数相关。那么2.6内核怎么做的呢？它用了上面提到的两个优先级队列active和expired，顾名思义，active是还有时间片的进程队列，而expired是时间片耗尽必须重新分配时间片的进程队列。 这么设计的好处就是不用再循环一遍所有进程重设时间片了，看看调度函数是怎么玩的：

```
array = rq->active;  
if (unlikely(!array->nr_active)) {  
    /* 
     * Switch the active and expired arrays. 
     */  
    schedstat_inc(rq, sched_switch);  
    rq->active = rq->expired;  
    rq->expired = array;  
    array = rq->active;  
    rq->expired_timestamp = 0;  
    rq->best_expired_prio = MAX_PRIO;  
} else  
    schedstat_inc(rq, sched_noswitch);  
```

  当所有运行进程的时间片都用完时，就把active和expired队列互换指针，没有遍历哦，而时间片耗尽的进程在出acitve队列入expired队列时，已经单独的重新分配好新时间片了。 再看一下schedule(void)调度函数，当某个进程休眠或者被抢占时，系统就开始调试schedule(void)决定接下来运行哪个进程。上面说过的东东都在这个函数里有体现哈。

```
asmlinkage void __sched schedule(void)  
{  
    long *switch_count;  
    task_t *prev, *next;  
    runqueue_t *rq;  
    prio_array_t *array;  
    struct list_head *queue;  
    unsigned long long now;  
    unsigned long run_time;  
    int cpu, idx;  
  
  
    /* 
     * Test if we are atomic.  Since do_exit() needs to call into 
     * schedule() atomically, we ignore that path for now. 
     * Otherwise, whine if we are scheduling when we should not be. 
     */  
    if (likely(!(current->exit_state & (EXIT_DEAD  EXIT_ZOMBIE)))) {先看看当前运行进程的状态  
        if (unlikely(in_atomic())) {  
            printk(KERN_ERR "scheduling while atomic: "  
                "%s/0x%08x/%d\n",  
                current->comm, preempt_count(), current->pid);  
            dump_stack();  
        }  
    }  
    profile_hit(SCHED_PROFILING, __builtin_return_address(0));  
  
  
need_resched:  
    preempt_disable();  
    prev = current;  
    release_kernel_lock(prev);  
need_resched_nonpreemptible:  
    rq = this_rq();      这行找到这个CPU对应的runqueue，再次强调，每个CPU有一个自己的runqueue  
  
  
    /* 
     * The idle thread is not allowed to schedule! 
     * Remove this check after it has been exercised a bit. 
     */  
    if (unlikely(current == rq->idle) && current->state != TASK_RUNNING) {  
        printk(KERN_ERR "bad: scheduling from the idle thread!\n");  
        dump_stack();  
    }  
  
  
    schedstat_inc(rq, sched_cnt);  
    now = sched_clock();  
    if (likely(now - prev->timestamp < NS_MAX_SLEEP_AVG))  
        run_time = now - prev->timestamp;  
    else  
        run_time = NS_MAX_SLEEP_AVG;  
  
  
    /* 
     * Tasks with interactive credits get charged less run_time 
     * at high sleep_avg to delay them losing their interactive 
     * status 
     */  
    if (HIGH_CREDIT(prev))  
        run_time /= (CURRENT_BONUS(prev) ? : 1);  
  
  
    spin_lock_irq(&rq->lock);  
  
  
    if (unlikely(current->flags & PF_DEAD))  
        current->state = EXIT_DEAD;  
    /* 
     * if entering off of a kernel preemption go straight 
     * to picking the next task. 
     */  
    switch_count = &prev->nivcsw;  
    if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {  
        switch_count = &prev->nvcsw;  
        if (unlikely((prev->state & TASK_INTERRUPTIBLE) &&  
                unlikely(signal_pending(prev))))  
            prev->state = TASK_RUNNING;  
        else {  
            if (prev->state == TASK_UNINTERRUPTIBLE)  
                rq->nr_uninterruptible++;  
            deactivate_task(prev, rq);  
        }  
    }  
  
  
    cpu = smp_processor_id();  
    if (unlikely(!rq->nr_running)) {  
go_idle:  
        idle_balance(cpu, rq);  
        if (!rq->nr_running) {  
            next = rq->idle;  
            rq->expired_timestamp = 0;  
            wake_sleeping_dependent(cpu, rq);  
            /* 
             * wake_sleeping_dependent() might have released 
             * the runqueue, so break out if we got new 
             * tasks meanwhile: 
             */  
            if (!rq->nr_running)  
                goto switch_tasks;  
        }  
    } else {  
        if (dependent_sleeper(cpu, rq)) {  
            next = rq->idle;  
            goto switch_tasks;  
        }  
        /* 
         * dependent_sleeper() releases and reacquires the runqueue 
         * lock, hence go into the idle loop if the rq went 
         * empty meanwhile: 
         */  
        if (unlikely(!rq->nr_running))  
            goto go_idle;  
    }  
  
  
    array = rq->active;  
    if (unlikely(!array->nr_active)) {       上面说过的，需要重新计算时间片时，就用已经计算好的expired队列了  
        /* 
         * Switch the active and expired arrays. 
         */  
        schedstat_inc(rq, sched_switch);  
        rq->active = rq->expired;  
        rq->expired = array;  
        array = rq->active;  
        rq->expired_timestamp = 0;  
        rq->best_expired_prio = MAX_PRIO;  
    } else  
        schedstat_inc(rq, sched_noswitch);  
  
  
    idx = sched_find_first_bit(array->bitmap);         找到优先级最高的队列  
    queue = array->queue + idx;  
    next = list_entry(queue->next, task_t, run_list);  
  
  
    if (!rt_task(next) && next->activated > 0) {  
        unsigned long long delta = now - next->timestamp;  
  
  
        if (next->activated == 1)  
            delta = delta * (ON_RUNQUEUE_WEIGHT * 128 / 100) / 128;  
  
  
        array = next->array;  
        dequeue_task(next, array);  
        recalc_task_prio(next, next->timestamp + delta);  
        enqueue_task(next, array);  
    }  
    next->activated = 0;  
switch_tasks:  
    if (next == rq->idle)  
        schedstat_inc(rq, sched_goidle);  
    prefetch(next);  
    clear_tsk_need_resched(prev);  
    rcu_qsctr_inc(task_cpu(prev));  
  
  
    prev->sleep_avg -= run_time;  
    if ((long)prev->sleep_avg <= 0) {  
        prev->sleep_avg = 0;  
        if (!(HIGH_CREDIT(prev)  LOW_CREDIT(prev)))  
            prev->interactive_credit--;  
    }  
    prev->timestamp = prev->last_ran = now;  
  
  
    sched_info_switch(prev, next);  
    if (likely(prev != next)) {              表面现在正在执行的进程，不是选出来的优先级最高的进程  
        next->timestamp = now;  
        rq->nr_switches++;  
        rq->curr = next;  
        ++*switch_count;  
  
  
        prepare_arch_switch(rq, next);  
        prev = context_switch(rq, prev, next);              所以需要完成进程上下文切换，把之前的进程信息CACHE住  
        barrier();  
  
  
        finish_task_switch(prev);  
    } else  
        spin_unlock_irq(&rq->lock);  
  
  
    prev = current;  
    if (unlikely(reacquire_kernel_lock(prev) < 0))  
        goto need_resched_nonpreemptible;  
    preempt_enable_no_resched();  
    if (unlikely(test_thread_flag(TIF_NEED_RESCHED)))  
        goto need_resched;  
}  
```

当然，在我们程序中，也可以通过执行以下系统调用来改变自己进程的优先级。nice系统调用可以改变某个进程的基本优先级，setpriority可以改变一组进程的优先级。