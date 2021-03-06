上下文，指的是程序执行的某一个状态，通常我们会通过调用栈来表示这个状态，栈记载了每个调用层级执行到了哪里，以及执行时的环境情况等所有有关的信息



上下文切换，表达的是从一个上下文切换到另一个上下文的技术。“调度”指的是哪个上下文可以获得接下来CPU时间的方法。



进程

进程是一种古老而经典的上下文系统，每个进程有独立的地址空间，资源句柄，他们之间互不发生干扰。



每个进程在内核中会有一个数据结构进行描述，我们称其为进程描述符。这些描述符包含了系统管理进程所需的信息，并且放在一个叫做任务队列的队列里面。



很显然，当新建进程时，我们需要分配新的进程描述符，并且分配新的地址空间\(和父地址空间的映射保持一致，但是两者同时进入COW状态\)。这些过程需要一定的开销。



进程状态

忽略去linux内核复杂的状态转移表，我们实际上可以把进程状态归结为三个最主要的状态：就绪态，运行态，睡眠态。这就是任何一本系统书上都有的三态转换图。



就绪和执行可以互相转换，基本这就是调度的过程。而当执行态程序需要等待某些条件\(最典型就是IO\)时，就会陷入睡眠态。而条件达成后，一般会自动进入就绪。



阻塞

当进程需要在某个文件句柄上做IO，这个fd又没有数据给他的时候，就会发生阻塞。具体来说，就是记录XX进程阻塞在了XX fd上，然后将进程标记为睡眠态，并调度出去。当fd上有数据时\(例如对端发送的数据到达\)，就会唤醒阻塞在fd上的进程。进程会随后进入就绪队列，等待合适的时间被调度。



阻塞后的唤醒也是一个很有意思的话题。当多个上下文阻塞在一个fd上\(虽然不多见，但是后面可以看到一个例子\)，而且fd就绪时，应该唤醒多少个上下文呢？传统上应当唤醒所有上下文，因为如果仅唤醒一个，而这个上下文又不能消费所有数据时，就会使得其他上下文处于无谓的死锁中。



线程

线程是一种轻量进程，实际上在linux内核中，两者几乎没有差别，除了一点——线程并不产生新的地址空间和资源描述符表，而是复用父进程的。 但是无论如何，线程的调度和进程一样，必须陷入内核态。



二、传统的网络模型

进程模型

为每个客户分配一个进程。优点是业务隔离，在一个进程中出现的错误不至于影响整个系统，甚至其他进程。Oracle传统上就是进程模型。缺点是进程的分配和释放有非常高的成本。因此Oracle需要连接池来保持连接减少新建和释放，同时尽量复用连接而不是随意的新建连接。



线程模型

为每客户分配一个线程。优点是更轻量，建立和释放速度更快，而且多个上下文间的通讯速度非常快。缺点是一个线程出现问题容易将整个系统搞崩溃。



三、C10K问题

进程模型的问题

在C10K的时候，启动和关闭这么多进程是不可接受的开销。事实上单纯的进程fork模型在C1K时就应当抛弃了。



Apache的prefork模型，是使用预先分配\(pre\)的进程池。这些进程是被复用的。但即便是复用，本文所描述的很多问题仍不可避免。



线程模型的问题

从任何测试都可以表明，线程模式比进程模式更耐久一些，性能更好。但是在面对C10K还是力不从心的。线程模型的问题在于切换成本高。



熟悉linux内核的应该知道，近代linux调度器经过几个阶段的发展。



linux2.4的调度器

O\(1\)调度器

CFS

实际上直到O\(1\)，调度器的调度复杂度才和队列长度无关。在此之前，过多的线程会使得开销随着线程数增长\(不保证线性\)。



O\(1\)调度器看起来似乎是完全不随着线程的影响。但是这个调度器有显著的缺点——难于理解和维护，并且在一些情况下会导致交互式程序响应缓慢。 CFS使用红黑树管理就绪队列。每次调度，上下文状态转换，都会查询或者变更红黑树。红黑树的开销大约是O\(logm\)，其中m大约为活跃上下文数\(准确的说是同优先级上下文数\)，大约和活跃的客户数相当。



因此，每当线程试图读写网络，并遇到阻塞时，都会发生O\(logm\)级别的开销。而且每次收到报文，唤醒阻塞在fd上的上下文时，同样要付出O\(logm\)级别的开销。



分析

O\(logm\)的开销看似并不大，但是却是一个无法接受的开销。因为IO阻塞是一个经常发生的事情。每次IO阻塞，都会发生开销。而且决定活跃线程数的是用户，这不是我们可控制的。更糟糕的是，当性能下降，响应速度下降时。同样的用户数下，活跃上下文会上升\(因为响应变慢了\)。这会进一步拉低性能。



问题的关键在于，http服务并不需要对每个用户完全公平，偶尔某个用户的响应时间大大的延长了是可以接受的。在这种情况下，使用红黑树去组织待处理fd列表（其实是上下文列表），并且反复计算和调度，是无谓的开销



四、多路复用

简述

要突破C10K问题，必须减少系统内活跃上下文数\(其实未必，例如换一个调度器，例如使用RT的SCHED\_RR\)，因此就要求一个上下文同时处理多个链接。而要做到这点，就必须在每次系统调用读取或写入数据时立刻返回。否则上下文持续阻塞在调用上，如何能够复用？这要求fd处于非阻塞状态，或者数据就绪。



上文所说的所有IO操作，其实都特指了他的阻塞版本。所谓阻塞，就是上下文在IO调用上等待直到有合适的数据为止。这种模式给人一种“只要读取数据就必定能读到”的感觉。而非阻塞调用，就是上下文立刻返回。如果有数据，带回数据。如果没有数据，带回错误\(EAGAIN\)。因此，“虽然发生错误，但是不代表出错”。



但是即使有了非阻塞模式，依然绕不过就绪通知问题。如果没有合适的就绪通知技术，我们只能在多个fd中盲目的重试，直到碰巧读到一个就绪的fd为止。这个效率之差可想而知。



在就绪通知技术上，有两种大的模式——就绪事件通知和异步IO。其差别简要来说有两点：



就绪通知维护一个状态，由用户读取，而异步IO由系统调用用户的回调函数

就绪通知在数据就绪时就生效，而异步IO直到数据IO完成才发生回调

linux下的主流方案一直是就绪通知，其内核态异步IO方案甚至没有被封装到glibc里去。围绕就绪通知，linux总共提出过三种解决方案（select、poll、epoll）。我们绕过select和poll方案，看看epoll方案的特性。



另外提一点。有趣的是，当使用了epoll后\(更准确说只有在LT模式下\)，fd是否为非阻塞其实已经不重要了。因为epoll保证每次去读取的时候都能读到数据，因此不会阻塞在调用上。



epoll

用户可以新建一个epoll文件句柄，并且将其他fd和这个"epoll fd"关联。此后可以通过epoll fd读取到所有就绪的文件句柄。



epoll有两大模式，ET和LT。LT模式下，每次读取就绪句柄都会读取出完整的就绪句柄。而ET模式下，只给出上次到这次调用间新就绪的句柄。换个说法，如果ET模式下某次读取出了一个句柄，这个句柄从未被读取完过——也就是从没有从就绪变为未就绪。那么这个句柄就永远不会被新的调用返回，哪怕上面其实充满了数据——因为句柄无法经历从非就绪变为就绪的过程。



类似CFS，epoll也使用了红黑树——不过是用于组织加入epoll的所有fd。epoll的就绪列表使用的是双向队列。这方便系统将某个fd加入队列中，或者从队列中解除。



要进一步了解epoll的具体实现，可以参考这篇linux下poll和epoll内核源码剖析。



性能

如果使用非阻塞函数，就不存在阻塞IO导致上下文切换了，而是变为时间片耗尽被抢占（大部分情况下如此），因此读写的额外开销被消除。而epoll的常规操作，都是O\(1\)量级的。而epoll wait的复制动作，则和当前需要返回的fd数有关\(在LT模式下几乎就等同于上面的m，而ET模式下则会大大减少\)。



但是epoll存在一点细节问题。epoll fd的管理使用红黑树，因此在加入和删除时需要O\(logn\)复杂度\(n为总连接数\)，而且关联操作还必须每个fd调用一次。因此在大连接量下频繁建立和关闭连接仍然有一定性能问题\(超短连接\)。不过关联操作调用毕竟比较少。如果确实是超短连接，tcp连接和释放开销就很难接受了，所以对总体性能影响不大。



固有缺陷

原理上说，epoll实现了一个wait\_queue的回调函数，因此原理上可以监听任何能够激活wait\_queue的对象。但是epoll的最大问题是无法用于普通文件，因为普通文件始终是就绪的——虽然在读取的时候不是这样。



这导致基于epoll的各种方案，一旦读到普通文件上下文仍然会阻塞。golang为了解决这个问题，在每次调用syscall的时候，会独立的启动一个线程，在独立的线程中进行调用。因此golang在IO普通文件的时候网络不会阻塞。



五、事件通知机制下的几种程序设计模型

简述

使用通知机制的一大缺憾就是，用户进行IO操作后会陷入茫然——IO没有完成，所以当前上下文不能继续执行。但是由于复用线程的要求，当前线程还需要接着执行。所以，在如何进行异步编程上，又分化出数种方案。



用户态调度

首先需要知道的一点就是，异步编程大多数情况下都伴随着用户态调度问题——即使不使用上下文技术。



因为系统不会自动根据fd的阻塞状况来唤醒合适的上下文了，所以这个工作必须由其他人——一般就是某种框架——来完成。



你可以想像一个fd映射到对象的大map表，当我们从epoll中得知某个fd就绪后，需要唤醒某种对象，让他处理fd对应的数据。



当然，实际情况会更加复杂一些。原则上所有不占用CPU时间的等待都需要中断执行，陷入睡眠，并且交由某种机构管理，等待合适的机会被唤醒。例如sleep，或是文件IO，还有lock。更精确的说，所有在内核里面涉及到wait\_queue的，在框架里面都需要做这种机制——也就是把内核的调度和等待搬到用户态来。



当然，其实也有反过来的方案——就是把程序扔到内核里面去。其中最著名的实例大概是微软的http服务器了。



这个所谓的“可唤醒可中断对象”，用的最多的就是协程。



协程

协程是一种编程组件，可以在不陷入内核的情况进行上下文切换。如此一来，我们就可以把协程上下文对象关联到fd，让fd就绪后协程恢复执行。 当然，由于当前地址空间和资源描述符的切换无论如何需要内核完成，因此协程所能调度的，只有在同一进程中的不同上下文而已。



如何做到

这是如何做到的呢？



我们在内核里实行上下文切换的时候，其实是将当前所有寄存器保存到内存中，然后从另一块内存中载入另一组已经被保存的寄存器。对于图灵机来说，当前状态寄存器意味着机器状态——也就是整个上下文。其余内容，包括栈上内存，堆上对象，都是直接或者间接的通过寄存器来访问的。



但是请仔细想想，寄存器更换这种事情，似乎不需要进入内核态么。事实上我们在用户态切换的时候，就是用了类似方案。



C coroutine的实现，基本大多是保存现场和恢复之类的过程。python则是保存当前thread的top frame\(greenlet\)。



但是非常悲剧的，纯用户态方案\(setjmp/longjmp\)在多数系统上执行的效率很高，但是并不是为了协程而设计的。setjmp并没有拷贝整个栈\(大多数的coroutine方案也不应该这么做\)，而是只保存了寄存器状态。这导致新的寄存器状态和老寄存器状态共享了同一个栈，从而在执行时互相破坏。而完整的coroutine方案应当在特定时刻新建一个栈。



而比较好的方案\(makecontext/swapcontext\)则需要进入内核\(sigprocmask\)，这导致整个调用的性能非常低。



协程与线程的关系

首先我们可以明确，协程不能调度其他进程中的上下文。而后，每个协程要获得CPU，都必须在线程中执行。因此，协程所能利用的CPU数量，和用于处理协程的线程数量直接相关。



作为推论，在单个线程中执行的协程，可以视为单线程应用。这些协程，在未执行到特定位置\(基本就是阻塞操作\)前，是不会被抢占，也不会和其他CPU上的上下文发生同步问题的。因此，一段协程代码，中间没有可能导致阻塞的调用，执行在单个线程中。那么这段内容可以被视为同步的。



我们经常可以看到某些协程应用，一启动就是数个进程。这并不是跨进程调度协程。一般来说，这是将一大群fd分给多个进程，每个进程自己再做fd-协程对应调度。



基于就绪通知的协程框架

首先需要包装read/write，在发生read的时候检查返回。如果是EAGAIN，那么将当前协程标记为阻塞在对应fd上，然后执行调度函数。

调度函数需要执行epoll\(或者从上次的返回结果缓存中取数据，减少内核陷入次数\)，从中读取一个就绪的fd。如果没有，上下文应当被阻塞到至少有一个fd就绪。

查找这个fd对应的协程上下文对象，并调度过去。

当某个协程被调度到时，他多半应当在调度器返回的路上——也就是read/write读不到数据的时候。因此应当再重试读取，失败的话返回1。

如果读取到数据了，直接返回。

这样，异步的数据读写动作，在我们的想像中就可以变为同步的。而我们知道同步模型会极大降低我们的编程负担。

