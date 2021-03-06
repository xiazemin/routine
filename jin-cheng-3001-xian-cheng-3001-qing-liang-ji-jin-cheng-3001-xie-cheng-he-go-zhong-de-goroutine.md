虽然用python时候在Eurasia和eventlet里了解过协程，但自己对协程的概念也就是轻量级线程，还有一个很通俗的红绿灯说法：线程要守规则，协程看到红灯但是没有车仍可以通行。现在总结各个资料，从个人理解上说明下 进程 线程 轻量级进程 协程 go中的goroutine 那些事儿。



一、进程



操作系统中最核心的概念是进程，分布式系统中最重要的问题是进程间通信。



进程是“程序执行的一个实例” ，担当分配系统资源的实体。进程创建必须分配一个完整的独立地址空间。



进程切换只发生在内核态，两步：1 切换页全局目录以安装一个新的地址空间 2 切换内核态堆栈和硬件上下文。  另一种说法类似：1 保存CPU环境（寄存器值、程序计数器、堆栈指针）2修改内存管理单元MMU的寄存器 3 转换后备缓冲器TLB中的地址转换缓存内容标记为无效。



二、线程



书中的定义：线程是进程的一个执行流，独立执行它自己的程序代码。



维基百科：线程（英语：thread）是操作系统能够进行运算调度的最小单位。



线程上下文一般只包含CPU上下文及其他的线程管理信息。线程创建的开销主要取决于为线程堆栈的建立而分配内存的开销，这些开销并不大。线程上下文切换发生在两个线程需要同步的时候，比如进入共享数据段。切换只CPU寄存器值需要存储，并随后用将要切换到的线程的原先存储的值重新加载到CPU寄存器中去。



用户级线程主要缺点在于对引起阻塞的系统调用的调用会立即阻塞该线程所属的整个进程。内核实现线程则会导致线程上下文切换的开销跟进程一样大，所以折衷的方法是轻量级进程（Lightweight）。在linux中，一个线程组基本上就是实现了多线程应用的一组轻量级进程。我理解为 进程中存在用户线程、轻量级进程、内核线程。



语言层面实现轻量级进程的比较少，stackless python，erlang支持，java并不支持。



三、协程



协程的定义？颜开、许式伟均只说协程是轻量级的线程，一个进程可轻松创建数十万计的协程。仔细研究下，个人感觉这些都是忽悠人的说法。从维基百科上看，从Knuth老爷子的基本算法卷上看“子程序其实是协程的特例”。子程序是什么？子程序（英语：Subroutine, procedure, function, routine, method, subprogram），就是函数嘛！所以协程也没什么了不起的，就是种更一般意义的程序组件，那你内存空间够大，创建多少个函数还不是随你么？



协程可以通过yield来调用其它协程。通过yield方式转移执行权的协程之间不是调用者与被调用者的关系，而是彼此对称、平等的。协程的起始处是第一个入口点，在协程里，返回点之后是接下来的入口点。子例程的生命期遵循后进先出（最后一个被调用的子例程最先返回）；相反，协程的生命期完全由他们的使用的需要决定。



线程和协程的区别：



一旦创建完线程，你就无法决定他什么时候获得时间片，什么时候让出时间片了，你把它交给了内核。而协程编写者可以有一是可控的切换时机，二是很小的切换代价。从操作系统有没有调度权上看，协程就是因为不需要进行内核态的切换，所以会使用它，会有这么个东西。赖永浩和dccmx 这个定义我觉得相对准确  协程－用户态的轻量级的线程。（http://blog.dccmx.com/2011/04/coroutine-concept/）



为什么要用协程：



协程有助于实现：



状态机：在一个子例程里实现状态机，这里状态由该过程当前的出口／入口点确定；这可以产生可读性更高的代码。

角色模型：并行的角色模型，例如计算机游戏。每个角色有自己的过程（这又在逻辑上分离了代码），但他们自愿地向顺序执行各角色过程的中央调度器交出控制（这是合作式多任务的一种形式）。

产生器：它有助于输入／输出和对数据结构的通用遍历。

 



颜开总结的支持协程的常见的语言和平台，可做参考，但应深入调研下才好。

 



Go语言并发之美

 



四、go中的Goroutine



go中的Goroutine， 普遍认为是协程的go语言实现。《Go语言编程》中说goroutine是轻量级线程\(即协程coroutine, 原书90页\). 在第九章进阶话题中, 作者又一次提到, "从根本上来说, goroutine就是一种go语言版本的协程\(coroutine\)" \(原书204页\). 但作者Rob Pike并不这么说。



“一个Goroutine是一个与其他goroutines 并发运行在同一地址空间的Go函数或方法。一个运行的程序由一个或更多个goroutine组成。它与线程、协程、进程等不同。它是一个goroutine。”



在栈实现上，它的编译器分支下的实现gccgo是线程pthread，6g上是多路复用的threads（6g/8g/5g分别代表64位、32位及Arm架构编译器）



infoQ一篇文章介绍特性也说道： goroutine是Go语言运行库的功能，不是操作系统提供的功能，goroutine不是用线程实现的。具体可参见Go语言源码里的pkg/runtime/proc.c



老赵认为goroutine就是把类库功能放进了语言里。



goroutine的并发问题：goroutine在共享内存中运行，通信网络可能死锁，多线程问题的调试糟糕透顶等等。一个比较好的建议规则：不要通过共享内存通信，相反，通过通信共享内存。



并行 并发区别：



并行是指程序的运行状态，要有两个线程正在执行才能算是Parallelism；并发指程序的逻辑结构，Concurrency则只要有两个以上线程还在执行过程中即可。简单地说，Parallelism要在多核或者多处理器情况下才能做到，而Concurrency则不需要。（http://stackoverflow.com/questions/1050222/concurrency-vs-parallelism-what-is-the-difference）



 



参考资料：



《现代操作系统》《分布式系统原理与范型》《深入理解linux内核》《go程序设计语言》



赖勇浩 协程三篇之仅一篇 http://blog.csdn.net/lanphaday/article/details/5397038



颜开 http://qing.blog.sina.com.cn/tj/88ca09aa33002ele.html



go程序设计语言中文 http://tonybai.com/2012/08/28/the-go-programming-language-tutorial-part3/  （中文翻译定义中漏了个 并发）



go程序设计语言英文http://go.googlecode.com/hg-history/release-branch.r60/doc/GoCourseDay3.pdf



go语言初体验 http://blog.dccmx.com/2011/01/go-taste/



https://zh.wikipedia.org/wiki/Go



https://zh.wikipedia.org/wiki/进程



https://zh.wikipedia.org/wiki/线程



http://stackoverflow.com/questions/1050222/concurrency-vs-parallelism-what-is-the-difference



http://www.infoq.com/cn/articles/knowledge-behind-goroutine



go语言编程书评：http://book.douban.com/review/5726587/



为什么我认为goroutine和channel是把别的平台上类库的功能内置在语言里 

http://blog.zhaojie.me/2013/04/why-channel-and-goroutine-in-golang-are-buildin-libraries-for-other-platforms.html

