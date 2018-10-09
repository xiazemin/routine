名字

getcontext, setcontext —— 获取或者设置用户上下文

概要

\#include &lt;ucontext.h&gt;

int getcontext\(ucontext\_t \*ucp\);

int setcontext\(const ucontext\_t \*ucp\);

描述

在类System-V环境中，定义在&lt;ucontext.h&gt;头文件中的mcontext\_t和ucontext\_t的两种数据类型，以及getcontext\(\)，setcontext\(\)，makecontext\(\)和swapcontext\(\)四个函数允许在一个进程不同的协程中用户级别的上下文切换。

mcontext\_t数据结构是依赖机器和不透明的。ucontext\_t数据结构至少包含下面的字段：

typedef struct ucontext {

```
struct ucontext \*uc\_link;

sigset\_t         uc\_sigmask;

stack\_t          uc\_stack;

mcontext\_t       uc\_mcontext;

...
```

} ucontext\_t;

sigset\_t和stack\_t定义在&lt;signal.h&gt;头文件中。uc\_link指向当前的上下文结束时要恢复到的上下文（只在当前上下文是由makecontext创建时，个人理解：只有makecontext创建新函数上下文时需要修改），uc\_sigmask表示这个上下文要阻塞的信号集合（参见sigprocmask），uc\_stack是这个上下文使用的栈（个人理解：非makecontext创建的上下文不要修改），uc\_mcontext是机器特定的保存上下文的表示，包括调用协程的机器寄存器。

getcontext\(\)函数初始化ucp所指向的结构体，填充当前有效的上下文。

setcontext\(\)函数恢复用户上下文为ucp所指向的上下文。成功调用不会返回。ucp所指向的上下文应该是getcontext\(\)或者makeontext\(\)产生的。

如果上下文是getcontext\(\)产生的，切换到该上下文，程序的执行在getcontext\(\)后继续执行。

如果上下文被makecontext\(\)产生的，切换到该上下文，程序的执行切换到makecontext\(\)调用所指定的第二个参数的函数上。当该函数返回时，我们继续传入makecontext\(\)中的第一个参数的上下文中uc\_link所指向的上下文。如果是NULL，程序结束。

返回值

成功时，getcontext\(\)返回0，setcontext\(\)不返回。错误时，都返回-1并且赋值合适的errno。

注意

这个机制最早的化身是setjmp/longjmp机制。但是它们没有定义处理信号的上下文，下一步就出了sigsetjmp/siglongjmp。当前这套机制给予了更多的控制权。但是另一方面，没有简单的方法去探明getcontext\(\)的返回是第一次调用还是通过setcontext\(\)调用。用户不得不发明一套他自己的书签的数据，并且当寄存器恢复时，register声明的变量不会恢复（寄存器变量）。

当信号发生时，当前的用户上下文被保存，一个新的内核为信号处理器产生的上下文被创建。不要在信号处理器中使用longjmp:它是未定义的行为。使用siglongjmp\(\)或者setcontext\(\)替代。

名字

makecontext，swapcontext —— 操控用户上下文

概要

\#include &lt;ucontext.h&gt;

void makecontext\(ucontext\_t \*ucp, void \(\*func\)\(void\), int argc, ...\);

int swapcontext\(ucontext\_t \*restrict oucp, const ucontext\_t \*restrict ucp\);

描述

makecontext\(\)函数修改ucp所指向的上下文，ucp是被getcontext\(\)所初始化的上下文。当这个上下文采用swapcontext\(\)或者setcontext\(\)被恢复，程序的执行会切换到func的调用，通过makecontext\(\)调用的argc传递func的参数。

在makecontext\(\)产生一个调用前，应用程序必须确保上下文的栈分配已经被修改。应用程序应该确保argc的值跟传入func的一样（参数都是int值4字节）；否则会发生未定义行为。

当makecontext\(\)修改过的上下文返回时，uc\_link用来决定上下文是否要被恢复。应用程序需要在调用makecontext\(\)前初始化uc\_link。

swapcontext\(\)函数保存当前的上下文到oucp所指向的数据结构，并且设置到ucp所指向的上下文。

保存了旧值oucp，跳转到ucp所指的地方

返回值

成功完成，swapcontext\(\)返回0。否则返回-1，并赋值合适的errno。

错误

swapcontext\(\)函数可能会因为下面的原因失败：

ENOMEM ucp参数没有足够的栈空间去完成操作。

例子

\#include &lt;stdio.h&gt;

\#include &lt;ucontext.h&gt;

static ucontext\_t ctx\[3\];

static void

f1 \(void\)

{

```
puts\("start f1"\);

swapcontext\(&ctx\[1\], &ctx\[2\]\);

puts\("finish f1"\);
```

}

static void

f2 \(void\)

{

```
puts\("start f2"\);

swapcontext\(&ctx\[2\], &ctx\[1\]\);

puts\("finish f2"\);
```

}

int

main \(void\)

{

```
char st1\[8192\];

char st2\[8192\];





getcontext\(&ctx\[1\]\);

ctx\[1\].uc\_stack.ss\_sp = st1;

ctx\[1\].uc\_stack.ss\_size = sizeof st1;

ctx\[1\].uc\_link = &ctx\[0\];

makecontext\(&ctx\[1\], f1, 0\);





getcontext\(&ctx\[2\]\);

ctx\[2\].uc\_stack.ss\_sp = st2;

ctx\[2\].uc\_stack.ss\_size = sizeof st2;

ctx\[2\].uc\_link = &ctx\[1\];

makecontext\(&ctx\[2\], f2, 0\);





swapcontext\(&ctx\[0\], &ctx\[2\]\);

return 0;
```

}

代码试用总结：

1 makecontext之前必须调用getcontext初始化context，否则会段错误core

2 makecontext之前必须给uc\_stack分配栈空间，否则也会段错误core

3 makecontext之前如果需要上下文恢复到调用前，则必须设置uc\_link以及通过swapcontext进行切换

4 getcontext产生的context为当前整个程序的context，而makecontext切换到的context为新函数独立的context，但setcontext切换到getcontext的context时，getcontext所在的函数退出时，并不需要uc\_link的管理，依赖于该函数是在哪被调用的，整个栈会向调用者层层剥离

5 不产生新函数的上下文切换指需要用到getcontext和setcontext

6 产生新函数的上下文切换需要用到getcontext，makecontext和swapcontext

ucontext性能小试：

运行环境为我的mac下通过虚拟机开启的centos64位系统，不代表一般情况，正常在linux实体机上应该会好很多吧

1 单纯的getcontext:

function\[ getcontext\(&ctx\) \] count\[ 10000000 \]

cost\[ 1394.88 ms\] avg\_cost\[ 0.14 us\]

total CPU time\[ 1380.00 ms\] avg\[ 0.14 us\]

user CPU time\[ 560.00 ms\] avg\[ 0.06 us\]

system CPU time\[ 820.00 ms\] avg\[ 0.08 us\]

2 新函数的协程调用

通过getcontext和对uc\_link以及uc\_stack赋值，未了不增加其他额外开销，uc\_stack为静态字符串数组分配，运行时不申请，makecontext中的函数foo为空函数，调用swapcontext切换协程调用测试

function\[ getcontext\_makecontext\_swapcontext\(\) \] count\[ 1000000 \]

cost\[ 544.55 ms\] avg\_cost\[ 0.54 us\]

total CPU time\[ 550.00 ms\] avg\[ 0.55 us\]

user CPU time\[ 280.00 ms\] avg\[ 0.28 us\]

system CPU time\[ 270.00 ms\] avg\[ 0.27 us\]

每秒百万级别的调用性能。

ucontext协程的实际使用：

将getcontext，makecontext，swapcontext封装成一个类似于lua的协同式协程，需要代码中主动yield释放出CPU。

协程的栈采用malloc进行堆分配，分配后的空间在64位系统中和栈的使用一致，地址递减使用，uc\_stack.uc\_size设置的大小好像并没有多少实际作用，使用中一旦超过已分配的堆大小，会继续向地址小的方向的堆去使用，这个时候就会造成堆内存的越界使用，更改之前在堆上分配的数据，造成各种不可预测的行为，coredump后也找不到实际原因。

对使用协程函数的栈大小的预估，协程函数中调用其他所有的api的中的局部变量的开销都会分配到申请给协程使用的内存上，会有一些不可预知的变量，比如调用第三方API，第三方API中有非常大的变量，实际使用过程中开始时可以采用mmap分配内存，对分配的内存设置GUARD\_PAGE进行mprotect保护，对于内存溢出，准确判断位置，适当调整需要分配的栈大小。

![](/assets/importucontext.png)

现代Unix系统都在ucontext.h中提供用于上下文切换的函数，这些函数有getcontext, setcontext，swapcontext 和makecontext。其中，getcontext用于保存当前上下文，setcontext用于切换上下文，swapcontext会保存当前上下文并切换到另一个上下文，makecontext创建一个新的上下文。实现用户线程的过程是：我们首先调用getcontext获得当前上下文，然后修改ucontext\_t指定新的上下文。同样的，我们需要开辟栈空间，但是这次实现的线程库要涉及栈生长的方向。然后我们调用makecontext切换上下文，并指定用户线程中要执行的函数。

       在这种实现中还有一个挑战，即一个线程必须可以主动让出CPU给其它线程。swapcontext函数可以完成这个任务，图4展示了一个这种实现的样例程序，child线程和parent线程不断切换以达到多线程的效果。在libfiber-uc.c文件中可以看到完整的实现。

\#include 

\#include 

\#include 

// 64kB stack

\#define FIBER\_STACK 1024\*64

ucontext\_t child, parent;

// The child thread will execute this function

void threadFunction\(\)

{

printf\( "Child fiber yielding to parent" \);

swapcontext\( &child, &parent \);

printf\( "Child thread exiting\n" \);

swapcontext\( &child, &parent \);

}

int main\(\)

{

// Get the current execution context

getcontext\( &child \);

// Modify the context to a new stack

child.uc\_link = 0;

child.uc\_stack.ss\_sp = malloc\( FIBER\_STACK \);

child.uc\_stack.ss\_size = FIBER\_STACK;

child.uc\_stack.ss\_flags = 0; 

if \( child.uc\_stack.ss\_sp == 0 \)

{

perror\( "malloc: Could not allocate stack" \);

exit\( 1 \);

}

// Create the new context

printf\( "Creating child fiber\n" \);

makecontext\( &child, &threadFunction, 0 \);

// Execute the child context

printf\( "Switching to child fiber\n" \);

swapcontext\( &parent, &child \);

printf\( "Switching to child fiber again\n" \);

swapcontext\( &parent, &child \);

// Free the stack

free\( child.uc\_stack.ss\_sp \);

printf\( "Child fiber returned and stack freed\n" \);

return 0;

}

图4用makecontext实现线程

用户级线程的抢占



       抢占实现的一个最重要的因素就是定时触发的计时器中断，它的存在使得我们能够中断当前程序的执行，异步对进程的时间片消耗情况进行统计，并在必要的时候（可能是时间片耗尽，也可能是一个高优先级的程序就绪）从当前进程调度到其它进程去执行。

       对于用户空间程序来说，与内核空间的中断相对应的就是信号，它和中断一样都是异步触发，并能引起执行流的跳转。所以要想实现用户级线程的抢占，我们可以借助定时器信号\(SIGALRM\)，必要时在信号处理程序内部进行上下文的切换



