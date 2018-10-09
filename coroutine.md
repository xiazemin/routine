Coroutine最近在公司的C/C++后台开发界，莫名其妙就火起来了，话说这货Melvin Conway在1963年的paper就已经提出来了，半个世纪过去了，咋突然冒出那么多粉丝出来，个人猜测与微信后台近期的Coroutine改造不无关系，也许这就是所谓的技术影响力吧，呵呵！



WHY?



首先，强烈推荐大家花点时间读一读《State Threads for Internet Applications》这篇文章，这是State Threads库的入门介绍，对于后台系统的performance、load

 scalability、system scalability等概念进行了非常精彩的论述。这里简单回顾一下常见的后台架构：Multi-Process Architecture（MP）、Multi-Threaded Architecture（MT）、Event-Driven State Machine Architecture（EDSM），这里，MP+EDSM、MT+EDSM等衍生品就不一一赘述了。





 

Load Scalability

System Scalability

Code Readability

MP

Low

High

High

MT

Medium

Medium

Medium

EDSM

High

Low

Low









这些年，随着硬件能力的不断提高，单机能力对于整个分布式系统的重要性不断下降，相反，对于开发者的开发效率、代码可读性等要求越来越高。这时，Coroutine就可以派上用场了，利用Coroutine的可重入特性，只要我们在底层库做适当的封装，上层的业务开发人员就可以像写同步程序那样写异步server了。



WHAT?



"Subroutines are special cases of ... coroutines." –Donald Knuth.



Wiki的定义：协程是一种程序组件，是由子例程（过程、函数、例程、方法、子程序）的概念泛化而来的，子例程只有一个入口点且只返回一次，而协程允许多个入口点，可以在指定位置挂起和恢复执行。



直观的定义：

协程的本地数据在后续调用中始终保持协程在控制离开时暂停执行，当控制再次进入时只能从离开的位置继续执行贴一个Wiki上的生产者-消费者Coroutine版，大家感受一下



var q := new queue    coroutine produce      loop          while q is not full              create some new items              add the items to q          yield to consume    coroutine consume      loop          while q is not empty              remove some items from q              use the items          yield to produce  



C/C++语言原生并不支持Coroutine，但是群众的力量是巨大的，正所谓“八仙过海各显神通”：setjmp and

 longjmp、getcontext, setcontext, makecontext and swapcontext，详细的清单可以参考Implementations

 for C、Implementations for C++。

这里推荐一下云风的C语言版Coroutine实现：https://github.com/cloudwu/coroutine/，有人可能觉得这个实现过于简陋，但是我恰恰欣赏它的简单，不到200行的代码，没有任何多余的feature，非常适合新手入门。



typedef void \(\*coroutine\_func\)\(struct schedule \*, void \*ud\);    struct schedule \* coroutine\_open\(void\);  void coroutine\_close\(struct schedule \*\);    int coroutine\_new\(struct schedule \*, coroutine\_func, void \*ud\);  void coroutine\_resume\(struct schedule \*, int id\);  int coroutine\_status\(struct schedule \*, int id\);  int coroutine\_running\(struct schedule \*\);  void coroutine\_yield\(struct schedule \*\);  



友情提醒：阅读代码时，留心\_save\_stack函数的实现，通过一点小技巧实现了Coroutine的栈区按需保存，对于内存的节省还是比较可观的。







HOW?



现在，假设我们有了Coroutine这个工具，又该如何整合进Server呢？为了解答这个问题，我们可以先问问自己，心目中的Server代码应该长啥样呢？





static int HandlerForCmdX\(AsyncServer \*server, const struct sockaddr\_in &client\_addr, const char \*pkg, size\_t len\){    // do some prepare stuff    server-&gt;AsyncSendRecv\(req\_info\_1, rsp\_info\_1\);    // do some prepare stuff    server-&gt;AsyncSendRecv\(req\_info\_2, rsp\_info\_2\);    // send response to client    server-&gt;AsyncSendOnly\(LISTEN\_CHANNEL, client\_addr, rsp\_buf, rsp\_len\);    return 0;} int main\(int argc, char \*argv\[\]\){    AsyncServer server;	    // do any init stuff    server.AddCommand\(USER\_COMMAND\_X, LISTEN\_CHANNEL, HandlerForCmdX\);    // do any else stuff    server.Run\(\);    exit\(EXIT\_SUCCESS\);}

上面的代码看上去挺美，再也不用纠结于各种状态跳转了，简单的注册命令处理函数HandlerForCmdX，然后如行云流水般添加业务处理逻辑，步骤1-&gt;步骤2-&gt;...-&gt;步骤N，完全符合人类的思维模式，从此PM再也不用担心我的开发效率了。想法其实很简单：server所谓的阻塞操作，基本也就等同于网络IO（这里不讨论文件IO、sleep等，同理可以实现），只要封装出类似AsyncSendRecv这些接口，默默地替上层应用完成coroutine的切换，这样上层应用就可以用同步的方式写异步server了。

下面我以云风的coroutine库为例，简单介绍一下封装方法：一个请求进来之后，server会自动为它分配一个coroutine进行处理（与MP、MT的one-to-one模式类似），该coroutine会执行事先注册好的处理函数，比如前文的HandlerForCmdX，执行到AsyncSendRecv内部时，一旦send成功，该coroutine就主动coroutine\_yield出去，继续执行底层网络事件循环，当收到响应后，框架做一些简单的解包工作，得到其sequence号，通过一定的手段转换为相应的coroutine\_id（coroutine\_new的返回值），然后就可以通过coroutine\_resume回到之前的AsyncSendRecv现场，紧接着控制权又回到了HandlerForCmdX，好像什么事都没有发生一样。"Talk

 is cheap. Show me the code."







int AsyncServer::SendOneReq\(const ReqInfo &req\_info\){    if \(this-&gt;SetPeerAddr\(req\_info.channel\_name, \(struct sockaddr\_in \*\)&req\_info.peer\_addr\) != 0\) return -1;    if \(this-&gt;SendOutData\(req\_info.data\_buf, req\_info.data\_len, req\_info.channel\_name\) != 0\) return -2;    co\_map.insert\(std::pair&lt;uint32\_t, int&gt;\(req\_info.sequence, coroutine\_running\(co\_schedule\)\)\);    return 0;} int AsyncServer::RecvOneRsp\(RspInfo &rsp\_info\){    if \(!decode\_result\) {        rsp\_info.timeout = true;        return 0;    }    rsp\_info.timeout = false;    rsp\_info.Resize\(decode\_result-&gt;m\_uiPkgLen\);    memcpy\(rsp\_info.data\_buf, decode\_result-&gt;m\_cpPkg, decode\_result-&gt;m\_uiPkgLen\);    return 0;} int AsyncServer::AsyncSendRecv\(const ReqInfo &req\_info, RspInfo &rsp\_info\){    if \(SendOneReq\(req\_info\) != 0\) return -1;    coroutine\_yield\(co\_schedule\);    if \(RecvOneRsp\(rsp\_info\) != 0\) return -2;    return 0;}

其中，co\_map用于保存sequence-&gt;coroutine\_id的映射关系，当网络框架的事件循环接收到响应包后，解出sequence，从而得到coroutin\_id，调用coroutine\_resume回到coroutine\_yield处。这部分代码与网络框架息息相关，此处略过，有兴趣的读者可以单独交流。





Limitation



截止目前，一切看上去都挺美好的，似乎用上coroutine，后台开发就解放了，所以不得不提一些coroutine的局限性：





虽然最终代码看起来和同步代码无异，而且单进程单线程，但用户注册的处理函数中不可以使用static变量（处理函数本身调用的函数不受限制）与EDSM类似，所有的coroutine隶属于同一个内核态线程，无法充分利用现代CPU的多核能力（可以考虑MP+Coroutine、MP+EDSM架构）coroutine可以看做用户态线程，属于非抢占式，如果某个coroutine阻塞住了，整个server也就歇菜了（EDSM同样存在这个问题，有点吹毛求疵了）



