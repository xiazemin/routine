今天看了下云风写的关于 c coroutine博客 \(代码\), 发现 coroutine 实现原理其实还比较简单，就用户态栈切换，只需要几十行汇编，特别轻量级。



具体实现

 1. 创建一个coroutine： 也就是创建一块连续内存，用于存放栈空间，并设置好入口函数所需要的寄存器



　 makecontext glibc c语言实现



 2. resume coroutine:  push保存当前执行上下文的寄存器到栈上，修改%rsp寄存器， jmp 到指定coroutine 执行指令位置，pop 恢复寄存器，开始执行



   swapcontext glibc 汇编实现



 3. yield coroutine： 同resume



   栈切换涉及寄存器操作，得用汇编实现， x86 8个通用寄存器，x64 16个，通过push 保存到栈，pop 恢复到寄存器；比较重要寄存器%rsp 栈顶指针，%rip 指令指针不能直接操作，通过call、jmp 跳转新的Code执行位置。 



  在64汇编中，并不需要对16个寄存器都备份，其中%rax作为返回值、%r10 %r11 被调用方使用前会自己备份.



 参考： X86-64寄存器和栈帧



X86-64寄存器的变化，不仅体现在位数上，更加体现在寄存器数量上。新增加寄存器%r8到%r15。加上x86的原有8个，一共16个寄存器。

刚刚说到，寄存器集成在CPU上，存取速度比存储器快好几个数量级，寄存器多了，GCC就可以更多的使用寄存器，替换之前的存储器堆栈使用，从而大大提升性能。

让寄存器为己所用，就得了解它们的用途，这些用途都涉及函数调用，X86-64有16个64位寄存器，分别是：%rax，%rbx，%rcx，%rdx，%esi，%edi，%rbp，%rsp，%r8，%r9，%r10，%r11，%r12，%r13，%r14，%r15。其中：



%rax 作为函数返回值使用。

%rsp 栈指针寄存器，指向栈顶

%rdi，%rsi，%rdx，%rcx，%r8，%r9 用作函数参数，依次对应第1参数，第2参数。。。

%rbx，%rbp，%r12，%r13，%14，%15 用作数据存储，遵循被调用者使用规则，简单说就是随便用，调用子函数之前要备份它，以防他被修改

%r10，%r11 用作数据存储，遵循调用者使用规则，简单说就是使用之前要先保存原值

 



 cloudwu/coroutine 测试

 测试环境：R620 E5-2620 2.4G



 测试次数：1kw 次yeild操作



 结果： 



time ./main



real 0m7.886s

user 0m4.408s

sys 0m3.447s

分析：



- 单核心每秒1.27M/s yield，每次耗时约 2000 cpu周期

- 因sys占用近一半时间，strace统计 每次yield至少两次 rt\_sigprocmask 系统调用，glibc 还考虑到了sig 设置的切换，其实必要性不大

- 切换时栈需要memcpy，栈因为需要预先分配，一般都在1M左右，但实际使用很少超过10K，如果为每个coroutine 预先分配1M，内存消耗过大。

　　云风实现里面，只分配一个1M的栈，coroutine 切换时才将实际大小的栈memcpy出来。节省内存，但性能消耗也不可忽视

- 如果去掉syscall，性能会有很大提升

 

微信libco协程库

在infoq一个关于微信后端存储视频提到： https://github.com/starjiang/libco

相比：

- 没有使用glibc，只支持linux，但总体和glibc 实现类似，优化掉了 rt\_sigprocmask

- 为每个coroutine 预分配128K的栈

- 包含 epoll 网络库实现

- 单线程版本

- 超时控制

 

就看看socket 的read 实现：

复制代码

ssize\_t read\( int fd, void \*buf, size\_t nbyte \)

{

    ....... 

    int timeout = \( lp-&gt;read\_timeout.tv\_sec \* 1000 \)

                + \( lp-&gt;read\_timeout.tv\_usec / 1000 \);



    struct pollfd pf = { 0 };

    pf.fd = fd;

    pf.events = \( POLLIN \| POLLERR \| POLLHUP \);



    int pollret = poll\( &pf,1,timeout \);  // 此处上下文将yeild， 切换到到epoll\_wait 直到fd可读，当前协程才会被重新resume



    ssize\_t readret = g\_sys\_read\_func\( fd,\(char\*\)buf ,nbyte \); // read系统调用



    if\( readret &lt; 0 \)

    {

        co\_log\_err\("CO\_ERR: read fd %d ret %ld errno %d poll ret %d timeout %d",

    }



    return readret;



}



int poll\(struct pollfd fds\[\], nfds\_t nfds, int timeout\)

{

   ......

    return co\_poll\( co\_get\_epoll\_ct\(\),fds,nfds,timeout \);

}





int co\_poll\( stCoEpoll\_t \*ctx,struct pollfd fds\[\], nfds\_t nfds, int timeout \)

{

....

    for\(nfds\_t i=0;i&lt;nfds;i++\)

    {

        arg.pPollItems\[i\].pSelf = fds + i;

        arg.pPollItems\[i\].pPoll = &arg;



        arg.pPollItems\[i\].pfnPrepare = OnPollPreparePfn;

        struct epoll\_event &ev = arg.pPollItems\[i\].stEvent;



        if\( fds\[i\].fd &gt; -1 \)

        {

            ev.data.ptr = arg.pPollItems + i;

            ev.events = PollEvent2Epoll\( fds\[i\].events \);



            epoll\_ctl\( epfd,EPOLL\_CTL\_ADD, fds\[i\].fd, &ev \);  // 添加epoll监听事件

        }

        //if fail,the timeout would work



    }



    co\_yield\_env\( co\_get\_curr\_thread\_env\(\) \); // yiled 协程，将被切换到epoll\_wait 

....

}





void co\_eventloop\( stCoEpoll\_t \*ctx,pfn\_co\_eventloop\_t pfn,void \*arg \)

{

    epoll\_event \*result = \(epoll\_event\*\)calloc\(1, sizeof\(epoll\_event\) \* stCoEpoll\_t::\_EPOLL\_SIZE \);

     

    for\(;;\)

    {

        int ret = epoll\_wait\( ctx-&gt;iEpollFd,result,stCoEpoll\_t::\_EPOLL\_SIZE, 1 \);

        // 超时时间1ms，而不是一直等待，方便做send timeout 处理

        ....  

        // resume 收到数据fd所在coroutine

}

复制代码

 



总体上，让accept、read、write 等操作网络IO操作，用同步方式来写但实际以NIO方式执行，不阻塞线程，减少代码量同时逻辑更清晰。



 

 

其他语言

Java： 以前公司rpc有通过JavaFlow实现，但没有正式用，貌似有性能和其他一些问题；虚机语言线程上下文较为复杂，不像c那么简单切换栈。



C\#： IEnumerator不怎么完善版本后，4.5 使用语法糖 await 编译器技巧实现类似效果。



Erlang： 原生进程模型就是coroutine，相比上面实现就是玩具；多线程、跨线程任务迁移、私有堆和栈、线程相关内存分配器、消息箱、公平调度等。Erlang 进程栈因为解释执行，栈空间不是由CPU自动管理，不需要连续的，可以动态扩展，没上限可递归到OOM。



Golang：goroutine Erlang的低配版，够用也实用；可惜没有分布式支持，但关键golang执行性能比Erlang高至少一个数量级。

