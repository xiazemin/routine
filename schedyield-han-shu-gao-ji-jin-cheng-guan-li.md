1、让出处理器



　　Linux提供一个系统调用运行进程主动让出执行权：sched\_yield。进程运行的好好的，为什么需要这个函数呢？有一种情况是用户空间线程的锁定。如果一个线程试图取得另一个线程所持有的锁，则新的线程应该让出处理器知道该锁变为可用。用户空间锁没有内核的支持，这是一个最间单、最有效率的做法。但是现在Linux线程实现引入一个使用futexes的优化解决方案。



　　另一个情况是在有处理器密集型程序可用周期性调用sched\_yield，试图将该进程对系统的冲击减到最小。不管怎么说，如何调度程序应该是系统的事情，而不是进程自己去管。eg：



int main\(\){

    int ret, i;

    ret = sched\_yield\(\);

    if\(ret == -1\){

    printf\("调用sched\_yield失败!\n"\);

    }

    return 0;

}

复制代码

　　那该调用内核是如何实现的？2.6以前的版本sched\_yield所造成的影响非常小，如果存在另一个可以运行的进程，内核就切换到该进程，把进行调用的进程放在可运行进程列表的结尾处。短期内内核会对该进程进行重新调度。这样的话可能出现“乒乓球”现象，也就是两个程序来回运行，直到他们都运行结束。2.6版本中做了一些改变：



如果进程是RR，把它放到可运行进程结尾，返回。

否则，把它从可运行进程列表移除，放到到期进程列表，这样在其他可运行进程时间片用完之前不会再运行该进程。

从可执行进程列表中找到另一个要执行的进程。

2、进程的优先级



　　看过CFS中会看到进程的nice value会决定进程会运行多长时间，或者说是占用的百分比。可以通过系统调用nice来设置、获取进程的nice value。该值的范围是-20～19，越低的值越高的优先级（这个在计算虚拟时间的时候放在分母上），实时进程应该是负数，eg：



int main\(\){

    int ret, i;

    ret = nice\(0\);

    printf\("当前进程的nice value：%d\n", ret\);

    ret = nice\(10\);

    printf\("当前进程的nice value：%d\n", ret\);

    return 0;

}

复制代码

　　因为ret本来就可以是-1，那么在判断是否系统调用失败的时候就要综合ret和errno。还有两个系统调用可以更灵活地设置，getpriority可以获得进程组、用户的任何进程中优先级最高的。setpriority将所指定的所有进程优先级设置为prio，eg：



int main\(\){

    int ret, i;

    ret = getpriority\(PRIO\_PROCESS, 0\);

    printf\("nice value:%d\n", ret\);

    ret = setpriority\(PRIO\_PROCESS, 0, 10\);

    ret = getpriority\(PRIO\_PROCESS, 0\);

    printf\("nice value:%d\n", ret\);

    return 0;

}

复制代码

　　进程有在处理器上执行的优先级，也有传输数据的优先级：I/O优先级。linux有额外的两个系统调用可用显示设置和取得I/O nice value，但是尚未导出：



int ioprio\_get\(int which, int who\);

int ioprio\_set\(int which, int who, int ioprio\);

复制代码

3、处理器亲和性

　　Linux支持具有多个处理器的单一系统。在SMP上，系统要决定每个处理器上要运行那些程序，这里有两项挑战：



调度程序必须想办法充分利用所有的处理器。

切换程序运行的处理器是需要代价的。

　　进程会继承父进程的处理器亲和性，Linux提供两个系统调用用于获取和设定“硬亲和性”。eg：



int main\(\){

    int ret, i;

    cpu\_set\_t set;



    CPU\_ZERO\(&set\);

    ret = sched\_getaffinity\(0, sizeof\(cpu\_set\_t\), &set\);

    if\(ret == -1\)

    printf\("调用失败!\n"\);

    for\(i = 0; i &lt; 10; i++\){

    int cpu = CPU\_ISSET\(i, &set\);

    printf\("cpu=%i is %s\n", i, cpu?"set":"unset"\);

    }



    CPU\_ZERO\(&set\);

    CPU\_SET\(0, &set\);

    CPU\_CLR\(1, &set\);

    ret = sched\_setaffinity\(0, sizeof\(cpu\_set\_t\), &set\);

    if\(ret == -1\)

    printf\("调用失败!\n"\);

    for\(i = 0; i &lt; 10; i++\){

    int cpu = CPU\_ISSET\(i, &set\);

    printf\("cpu=%i is %s\n", i, cpu?"set":"unset"\);

    }

    return 0;

}

复制代码

4、Linux的调度策略与优先级

　　关于Linux系统中对进程的几种调度方法和他们的区别就不在这里说了，这里关注的是如何获取、设置这些值。可以使用sched\_getscheduler来获取进程的调度策略，eg：



int main\(\){

    int ret, i;

    struct sched\_param sp;

    sp.sched\_priority = 1;

    ret = sched\_setscheduler\(0, SCHED\_RR, &sp\);

    if\(ret == -1\)

    printf\("sched\_setscheduler failed.\n"\);

    if\(errno == EPERM\)

    printf\("Process don't the ability.\n"\);



    ret = sched\_getscheduler\(0\);

    switch\(ret\){

    case SCHED\_OTHER:

    printf\("Policy is normal.\n"\);

    break;

    case SCHED\_RR:

    printf\("Policy is round-robin.\n"\);

    break;

    case SCHED\_FIFO:

    printf\("Policy is first-in, first-out.\n"\);

    break;

    case -1:

    printf\("sched\_getscheduler failed.\n"\);

    break;

    default:

    printf\("Unknow policy\n"\);

    }

    return 0;

}

复制代码

　　sched\_getparam和sched\_setparam接口可用于取得、设定一个已经设定好的策略，这里不只是返回一个策略的ID，eg：



int main\(\){

    int ret, i;

    struct sched\_param sp;



    sp.sched\_priority = 1;

    ret = sched\_setparam\(0, &sp\);    

    if\(ret == -1\)

    printf\("sched\_setparam error.\n"\);



    ret = sched\_getparam\(0, &sp\);

    if\(ret == -1\)

    printf\("sched\_getparam error.\n"\);



    printf\("our priority is %d.\n", sp.sched\_priority\);

    return 0;

}

复制代码

　　Linux提供两个用于取得有效优先值的范围的系统调用，分别返回最大值、最小值，eg：



int main\(\){

    int ret, i;

    struct sched\_param sp;



    ret = sched\_get\_priority\_min\(SCHED\_RR\);

    if\(ret == -1\)

    printf\("sched\_get\_priority\_min error.\n"\);

    printf\("The min nice value is %d.\n", ret\);



    ret = sched\_get\_priority\_max\(SCHED\_RR\);

    if\(ret == -1\)

    printf\("sched\_get\_priority\_max error.\n"\);

    printf\("The mmax nice value is %d.\n", ret\);

    return 0;

}

复制代码

　　关于时间片，这个概念可能在Linux中和传统的在操作系统的课程中学到的还是有很大的区别的，如果感兴趣的化可以看看CFS里面的。通过sched\_rr\_get\_interval可以取到分配给pid的时间片的长度，eg：



int main\(\){

    int ret, i;

    struct timespec tp;

    ret = sched\_rr\_get\_interval\(0, &tp\);

    if\(ret == -1\)

    printf\("sched\_rr\_get\_interval error.\n"\);

    printf\("The time is %ds:%ldns.\n", \(int\)tp.tv\_sec, tp.tv\_nsec\);

    return 0;

}

复制代码

5、实时进程的预防措施

　　由于实时进程的本质，开发者在开发和调试此类程序时应该谨慎行事，如果一个实时进程突然发脾气，系统的反应会突然变慢。任何一个CPU密集型循环在一个实时程序中会继续无止境地运行下去，只要没有优先级更高实时进程变成可运行的。因此设计实时程序的时候要谨慎，这类程序至高无上，可用轻易托跨整个系统，下面是一些要决与注意事项：



因为实时进程会好用系统上一切资源，小心不要让系统其他进程等不到处理时间。

循环可能会一直运行到结束。

小心忙碌等待，也就是实时进程等待一个优先级低的进程所占有的资源。

开发一个实时进程的时候，让一个终端保持开启状态，以更高的优先级来运行该实时进程，发生紧急情况终端机依然会有反应，允许你终止失控的实时进程。

使用chrt设置、取得实时属性。

6、资源限制



　　Linux对进程加上了若干资源限制，这些限制是一个进程所能耗用的内核资源的上限。限制的类型如下：



RLIMIT\_AS:地址空间上限。

RLIMIT\_CORE:core文件大小上限。

RLIMIT\_CPU:可耗用CPU时间上限。

RLIMIT\_DATA:数据段与堆的上限。

RLIMIT\_FSIZE:所能创建文件的大小上限。

RLIMIT\_LOCKS:文件锁数目上限。

RLIMIT\_MEMLOCK:不具备CAP\_SYS\_IPC能力的进程最多将多少个字节锁进内存。

RLIMIT\_MSGQUEUE:可以在消息队列中分配多少字节。

RLIMIT\_NICE:最多可以将自己的友善值调多低。

RLIMIT\_NOFILE:文件描述符数目的上限。

RLIMIT\_NPROC:用户在系统上能运行进程数目上限。

RLIMIT\_RSS:内存中页面的数目的上线。

RLIMIT\_RTPRIO:不具备CAP\_SYS\_NICE能力进程所能请求的实时优先级的上限。

RLIMIT\_SIGPENDING:在队列中信号量的上限，Linux特有的限制。

RLIMIT\_STACK:堆栈大小的上限。

这些就不多说了，到了实际用到的时候再仔细看，eg：



int main\(\){

    int ret, i;

    struct rlimit rlim;



    rlim.rlim\_cur = 32\*1024\*1024;

    rlim.rlim\_max = RLIM\_INFINITY;

    ret = setrlimit\(RLIMIT\_CORE, &rlim\);



    ret = getrlimit\(RLIMIT\_CORE, &rlim\);

    if\(ret == -1\)

    printf\("getrlimit error.\n"\);

    printf\("RLIMIT\_CORE limits: soft=%ld hard=%ld\n", rlim.rlim\_cur, rlim.rlim\_max\);



    return 0;

}

