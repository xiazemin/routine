https://github.com/Winnerhust/uthread

协程是一种用户态的轻量级线程。本篇主要研究协程的C/C++的实现。

首先我们可以看看有哪些语言已经具备协程语义：

比较重量级的有C\#、erlang、golang\*轻量级有python、lua、javascript、ruby还有函数式的scala、scheme等。

c/c++不直接支持协程语义，但有不少开源的协程库，如：

Protothreads：一个“蝇量级” C 语言协程库

libco:来自腾讯的开源协程库libco介绍，官网

coroutine:云风的一个C语言同步协程库,详细信息



目前看到大概有四种实现协程的方式：

第一种：利用glibc 的 ucontext组件\(云风的库\)第二种：使用汇编代码来切换上下文\(实现c协程\)第三种：利用C语言语法switch-case的奇淫技巧来实现（Protothreads\)第四种：利用了 C 语言的 setjmp 和 longjmp（ 

一种协程的 C/C++ 实现,要求函数里面使用 static local 的变量来保存协程内部的数据）

本篇主要使用ucontext来实现简单的协程库。



2.ucontext初接触



利用ucontext提供的四个函数getcontext\(\),setcontext\(\),makecontext\(\),swapcontext\(\)可以在一个进程中实现用户级的线程切换。



本节我们先来看ucontext实现的一个简单的例子：

\#include &lt;stdio.h&gt;\#include &lt;ucontext.h&gt;\#include &lt;unistd.h&gt; int main\(int argc, const char \*argv\[\]\){    ucontext\_t context;     getcontext\(&context\);    puts\("Hello world"\);    sleep\(1\);    setcontext\(&context\);    return 0;}





注：示例代码来自维基百科.



保存上述代码到example.c,执行编译命令：

gcc example.c -o example





想想程序运行的结果会是什么样？

cxy@ubuntu:~$ ./example Hello worldHello worldHello worldHello worldHello worldHello worldHello world^Ccxy@ubuntu:~$上面是程序执行的部分输出，不知道是否和你想得一样呢？我们可以看到，程序在输出第一个“Hello world"后并没有退出程序，而是持续不断的输出”Hello world“。其实是程序通过getcontext先保存了一个上下文,然后输出"Hello world",在通过setcontext恢复到getcontext的地方，重新执行代码，所以导致程序不断的输出”Hello world“，在我这个菜鸟的眼里，这简直就是一个神奇的跳转。





那么问题来了，ucontext到底是什么？



3.ucontext组件到底是什么



在类System V环境中,在头文件&lt; ucontext.h &gt; 中定义了两个结构类型，mcontext\_t和ucontext\_t和四个函数getcontext\(\),setcontext\(\),makecontext\(\),swapcontext\(\).利用它们可以在一个进程中实现用户级的线程切换。



mcontext\_t类型与机器相关，并且不透明.ucontext\_t结构体则至少拥有以下几个域:

           typedef struct ucontext {               struct ucontext \*uc\_link;               sigset\_t         uc\_sigmask;               stack\_t          uc\_stack;               mcontext\_t       uc\_mcontext;               ...           } ucontext\_t;





当当前上下文\(如使用makecontext创建的上下文）运行终止时系统会恢复uc\_link指向的上下文；uc\_sigmask为该上下文中的阻塞信号集合；uc\_stack为该上下文中使用的栈；uc\_mcontext保存的上下文的特定机器表示，包括调用线程的特定寄存器等。



下面详细介绍四个函数：

int getcontext\(ucontext\_t \*ucp\);





初始化ucp结构体，将当前的上下文保存到ucp中

int setcontext\(const ucontext\_t \*ucp\);





设置当前的上下文为ucp，setcontext的上下文ucp应该通过getcontext或者makecontext取得，如果调用成功则不返回。如果上下文是通过调用getcontext\(\)取得,程序会继续执行这个调用。如果上下文是通过调用makecontext取得,程序会调用makecontext函数的第二个参数指向的函数，如果func函数返回,则恢复makecontext第一个参数指向的上下文第一个参数指向的上下文context\_t中指向的uc\_link.如果uc\_link为NULL,则线程退出。

void makecontext\(ucontext\_t \*ucp, void \(\*func\)\(\), int argc, ...\);





makecontext修改通过getcontext取得的上下文ucp\(这意味着调用makecontext前必须先调用getcontext\)。然后给该上下文指定一个栈空间ucp-&gt;stack，设置后继的上下文ucp-&gt;uc\_link.



当上下文通过setcontext或者swapcontext激活后，执行func函数，argc为func的参数个数，后面是func的参数序列。当func执行返回后，继承的上下文被激活，如果继承上下文为NULL时，线程退出。

int swapcontext\(ucontext\_t \*oucp, ucontext\_t \*ucp\);





保存当前上下文到oucp结构体中，然后激活upc上下文。 



如果执行成功，getcontext返回0，setcontext和swapcontext不返回；如果执行失败，getcontext,setcontext,swapcontext返回-1，并设置对于的errno.



简单说来， 

getcontext获取当前上下文，setcontext设置当前上下文，swapcontext切换上下文，makecontext创建一个新的上下文。



4.小试牛刀-使用ucontext组件实现线程切换



虽然我们称协程是一个用户态的轻量级线程，但实际上多个协程同属一个线程。任意一个时刻，同一个线程不可能同时运行两个协程。如果我们将协程的调度简化为：主函数调用协程1，运行协程1直到协程1返回主函数，主函数在调用协程2，运行协程2直到协程2返回主函数。示意步骤如下：

    执行主函数    切换：主函数 --&gt; 协程1    执行协程1    切换：协程1  --&gt; 主函数    执行主函数    切换：主函数 --&gt; 协程2    执行协程2    切换协程2  --&gt; 主函数    执行主函数    ...这种设计的关键在于实现主函数到一个协程的切换，然后从协程返回主函数。这样无论是一个协程还是多个协程都能够完成与主函数的切换，从而实现协程的调度。





实现用户线程的过程是：

我们首先调用getcontext获得当前上下文修改当前上下文ucontext\_t来指定新的上下文，如指定栈空间极其大小，设置用户线程执行完后返回的后继上下文（即主函数的上下文）等调用makecontext创建上下文，并指定用户线程中要执行的函数切换到用户线程上下文去执行用户线程（如果设置的后继上下文为主函数，则用户线程执行完后会自动返回主函数）。

下面代码context\_test函数完成了上面的要求。

\#include &lt;ucontext.h&gt;\#include &lt;stdio.h&gt; void func1\(void \* arg\){    puts\("1"\);    puts\("11"\);    puts\("111"\);    puts\("1111"\); }void context\_test\(\){    char stack\[1024\*128\];    ucontext\_t child,main;     getcontext\(&child\); //获取当前上下文    child.uc\_stack.ss\_sp = stack;//指定栈空间    child.uc\_stack.ss\_size = sizeof\(stack\);//指定栈空间大小    child.uc\_stack.ss\_flags = 0;    child.uc\_link = &main;//设置后继上下文     makecontext\(&child,\(void \(\*\)\(void\)\)func1,0\);//修改上下文指向func1函数     swapcontext\(&main,&child\);//切换到child上下文，保存当前上下文到main    puts\("main"\);//如果设置了后继上下文，func1函数指向完后会返回此处} int main\(\){    context\_test\(\);     return 0;}



在context\_test中，创建了一个用户线程child,其运行的函数为func1.指定后继上下文为main

func1返回后激活后继上下文，继续执行主函数。



保存上面代码到example-switch.cpp.运行编译命令:

g++ example-switch.cpp -o example-switch





执行程序结果如下

cxy@ubuntu:~$ ./example-switch1111111111maincxy@ubuntu:~$你也可以通过修改后继上下文的设置，来观察程序的行为。如修改代码



child.uc\_link = &main;





为

child.uc\_link = NULL;





再重新编译执行，其执行结果为：

cxy@ubuntu:~$ ./example-switch1111111111cxy@ubuntu:~$可以发现程序没有打印"main"，执行为func1后直接退出，而没有返回主函数。可见，如果要实现主函数到线程的切换并返回，指定后继上下文是非常重要的。



5.使用ucontext实现自己的线程库



掌握了上一节从主函数到协程的切换的关键，我们就可以开始考虑实现自己的协程了。

定义一个协程的结构体如下：

typedef void \(\*Fun\)\(void \*arg\); typedef struct uthread\_t{    ucontext\_t ctx;    Fun func;    void \*arg;    enum ThreadState state;    char stack\[DEFAULT\_STACK\_SZIE\];}uthread\_t;ctx保存协程的上下文，stack为协程的栈，栈大小默认为DEFAULT\_STACK\_SZIE=128Kb.你可以根据自己的需求更改栈的大小。func为协程执行的用户函数，arg为func的参数，state表示协程的运行状态，包括FREE,RUNNABLE,RUNING,SUSPEND,分别表示空闲，就绪，正在执行和挂起四种状态。





在定义一个调度器的结构体

typedef std::vector&lt;uthread\_t&gt; Thread\_vector; typedef struct schedule\_t{    ucontext\_t main;    int running\_thread;    Thread\_vector threads;     schedule\_t\(\):running\_thread\(-1\){}}schedule\_t;调度器包括主函数的上下文main,包含当前调度器拥有的所有协程的vector类型的threads，以及指向当前正在执行的协程的编号running\_thread.如果当前没有正在执行的协程时，running\_thread=-1.





接下来，在定义几个使用函数uthread\_create,uthread\_yield,uthread\_resume函数已经辅助函数schedule\_finished.就可以了。

int  uthread\_create\(schedule\_t &schedule,Fun func,void \*arg\);





创建一个协程，该协程的会加入到schedule的协程序列中，func为其执行的函数，arg为func的执行函数。返回创建的线程在schedule中的编号。

void uthread\_yield\(schedule\_t &schedule\);





挂起调度器schedule中当前正在执行的协程，切换到主函数。

void uthread\_resume\(schedule\_t &schedule,int id\);





恢复运行调度器schedule中编号为id的协程

int  schedule\_finished\(const schedule\_t &schedule\);





判断schedule中所有的协程是否都执行完毕，是返回1，否则返回0.注意：如果有协程处于挂起状态时算作未全部执行完毕，返回0.



代码就不全贴出来了，我们来看看两个关键的函数的具体实现。首先是uthread\_resume函数：

void uthread\_resume\(schedule\_t &schedule , int id\){    if\(id &lt; 0 \|\| id &gt;= schedule.threads.size\(\)\){        return;    }     uthread\_t \*t = &\(schedule.threads\[id\]\);     switch\(t-&gt;state\){        case RUNNABLE:            getcontext\(&\(t-&gt;ctx\)\);             t-&gt;ctx.uc\_stack.ss\_sp = t-&gt;stack;            t-&gt;ctx.uc\_stack.ss\_size = DEFAULT\_STACK\_SZIE;            t-&gt;ctx.uc\_stack.ss\_flags = 0;            t-&gt;ctx.uc\_link = &\(schedule.main\);            t-&gt;state = RUNNING;             schedule.running\_thread = id;             makecontext\(&\(t-&gt;ctx\),\(void \(\*\)\(void\)\)\(uthread\_body\),1,&schedule\);             /\* !! note : Here does not need to break \*/         case SUSPEND:             swapcontext\(&\(schedule.main\),&\(t-&gt;ctx\)\);             break;        default: ;    }}如果指定的协程是首次运行，处于RUNNABLE状态，则创建一个上下文，然后切换到该上下文。如果指定的协程已经运行过，处于SUSPEND状态，则直接切换到该上下文即可。代码中需要注意RUNNBALE状态的地方不需要break.





void uthread\_yield\(schedule\_t &schedule\){    if\(schedule.running\_thread != -1 \){        uthread\_t \*t = &\(schedule.threads\[schedule.running\_thread\]\);        t-&gt;state = SUSPEND;        schedule.running\_thread = -1;         swapcontext\(&\(t-&gt;ctx\),&\(schedule.main\)\);    }}uthread\_yield挂起当前正在运行的协程。首先是将running\_thread置为-1，将正在运行的协程的状态置为SUSPEND，最后切换到主函数上下文。





更具体的代码我已经放到github上,点击这里。



6.最后一步-使用我们自己的协程库



保存下面代码到example-uthread.cpp.

\#include "uthread.h"\#include &lt;stdio.h&gt; void func2\(void \* arg\){    puts\("22"\);    puts\("22"\);    uthread\_yield\(\*\(schedule\_t \*\)arg\);    puts\("22"\);    puts\("22"\);} void func3\(void \*arg\){    puts\("3333"\);    puts\("3333"\);    uthread\_yield\(\*\(schedule\_t \*\)arg\);    puts\("3333"\);    puts\("3333"\); } void schedule\_test\(\){    schedule\_t s;     int id1 = uthread\_create\(s,func3,&s\);    int id2 = uthread\_create\(s,func2,&s\);     while\(!schedule\_finished\(s\)\){        uthread\_resume\(s,id2\);        uthread\_resume\(s,id1\);    }    puts\("main over"\); }int main\(\){    schedule\_test\(\);     return 0;}



执行编译命令并运行：

g++ example-uthread.cpp -o example-uthread./example-uthread



运行结果如下：

cxy@ubuntu:~/mythread$./example-uthread222233333333222233333333main overcxy@ubuntu:~/mythread$可以看到，程序协程func2，然后切换到主函数,在执行协程func3，再切换到主函数，又切换到func2,在切换到主函数，再切换到func3,最后切换到主函数结束。





总结一下，我们利用getcontext和makecontext创建上下文，设置后继的上下文到主函数，设置每个协程的栈空间。在利用swapcontext在主函数和协程之间进行切换。



到此，使用ucontext做一个自己的协程库就到此结束了。相信你也可以自己完成自己的协程库了。



