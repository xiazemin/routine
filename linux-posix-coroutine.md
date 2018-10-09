开发同学在设计高性能后台服务时会优先考虑异步的执行方式，而目前异步执行方式主要有基于事件驱动的状态机模型、coroutine协程模型、Future/Promise模型。很多现代编程语言都提供了原生的coroutine特性（同步代码，异步执行）。本文总结下Linux使用POSIXsetcontext系列调用以实现coroutine语义的方法。

1 背景介绍

在代码里使用context需要包含ucontext.h（/usr/include/ucontext.h）系统头文件。





此头文件列出了POSIX下关于context operation的interface。并且可以看到此头文件还会引用另一个头文件/usr/include/sys/ucontext.h。

补充说明：

\(1\) setcontext function transfers controlto the context in ucp. Execution continues from the point at which the contextwas stored in ucp. setcontext does not return.

\(2\) getcontext saves current context intoucp. This function returns in two possible cases: after the initial call, orwhen a thread switches to the context in ucp via setcontext or swapcontext. Thegetcontext function does not provide a return value to distinguish

 the cases\(its return value is used solely to signal error\), so the programmer must usean explicit flag variable, which must not be a register variable and must bedeclared volatile to avoid constant propagation or other compiler optimizations.

\(3\) makecontext function sets up analternate thread of control in ucp, which has previously been initialized usinggetcontext. The ucp.uc\_stack member should be pointed to an appropriately sizedstack; the constant SIGSTKSZ is commonly used. When ucp is jumped

 to usingsetcontext or swapcontext, execution will begin at the entry point to thefunction pointed to bye func, with argc arguments as specified. When functerminates, control is returned to ucp.uc\_link.

\(4\) swapcontext transfers control to ucpand saves the current execution state into oucp.





头文件/usr/include/sys/ucontext.h主要是定义了相关的Register和DS，用户需要关心的是Userlevel context的ucontext\_t类型。



补充说明：

\(1\) uc\_link points to the context whichwill be resumed when the current context exits, if the context was created withmakecontext \(a secondary context\).

\(2\) uc\_stack is the stack used by thecontext.

\(3\) uc\_mcontext stores execution state,including all registers and CPU flags, the instruction pointer, and the stackpointer. mcontext\_t is an opaque type.

\(4\) uc\_sigmask is used to store the set ofsignals blocked in the context.

2 代码实践

在了解了基础的概念后，下面就可以学习下某开源coroutine的具体实现。代码中添加了一些调试信息和备注。

coroutine的接口定义：



\#ifndef C\_COROUTINE\_H\#define C\_COROUTINE\_H\#define COROUTINE\_DEAD 0\#define COROUTINE\_READY 1\#define COROUTINE\_RUNNING 2\#define COROUTINE\_SUSPEND 3struct schedule;typedef void \(\*coroutine\_func\)\(struct schedule \*, void \*ud\);struct schedule \* coroutine\_open\(void\);void coroutine\_close\(struct schedule \*\);int coroutine\_new\(struct schedule \*, coroutine\_func, void \*ud\);void coroutine\_resume\(struct schedule \*, int id\);int coroutine\_status\(struct schedule \*, int id\);int coroutine\_running\(struct schedule \*\);void coroutine\_yield\(struct schedule \*\);\#endif

 coroutine的实现：



\#include "coroutine.h"\#include &lt;stdio.h&gt;\#include &lt;stdlib.h&gt;\#include &lt;ucontext.h&gt;\#include &lt;assert.h&gt;\#include &lt;stddef.h&gt;\#include &lt;string.h&gt;\#include &lt;stdint.h&gt;\#include &lt;unistd.h&gt; \#define STACK\_SIZE \(1024 \* 1024\)\#define DEFAULT\_COROUTINE 16 struct coroutine; // 调度器struct schedule {	char stack\[STACK\_SIZE\];   // 所有协程的public stack	ucontext\_t main;          // 主线程的context	int nco;                  // 当前启用的协程数量	int cap;                  // 支持的协程数量	int running;              // 协程id	struct coroutine \*\*co;    // 协程对象集}; // 协程对象struct coroutine {	coroutine\_func func;      // 每个协程的回调函数	void \*ud;                 // 每个协程的用户数据	ucontext\_t ctx;           // 每个协程的context	struct schedule \*sch;     // 每个协程从属的调度器	ptrdiff\_t cap;            // 每个协程private stack的最大分配空间	ptrdiff\_t size;           // 每个协程private stack的实际分配空间	int status;               // 每个协程的当前运行状态	char \*stack;              // 协程的private stack}; struct coroutine \* \_co\_new\(struct schedule \*S , coroutine\_func func, void \*ud\) {	struct coroutine \* co = malloc\(sizeof\(\*co\)\);	co-&gt;func = func;	co-&gt;ud = ud;	co-&gt;sch = S;	co-&gt;cap = 0;	co-&gt;size = 0;	co-&gt;status = COROUTINE\_READY;	co-&gt;stack = NULL;	return co;} void\_co\_delete\(struct coroutine \*co\) { 	// 只释放协程自己动态分配的空间, 并注意释放顺序	free\(co-&gt;stack\);	free\(co\);} struct schedule \* coroutine\_open\(void\) {	struct schedule \*S = malloc\(sizeof\(\*S\)\);	S-&gt;nco = 0;	S-&gt;cap = DEFAULT\_COROUTINE;	S-&gt;running = -1;	S-&gt;co = malloc\(sizeof\(struct coroutine \*\) \* S-&gt;cap\);	memset\(S-&gt;co, 0, sizeof\(struct coroutine \*\) \* S-&gt;cap\);	return S;} void coroutine\_close\(struct schedule \*S\) {	int i;	for \(i = 0; i &lt; S-&gt;cap; i++\) {		struct coroutine \* co = S-&gt;co\[i\];		if \(co\) {			\_co\_delete\(co\);		}	} 	// 最后释放调度器	free\(S-&gt;co\);	S-&gt;co = NULL;	free\(S\);} int coroutine\_new\(struct schedule \*S, coroutine\_func func, void \*ud\) {	struct coroutine \*co = \_co\_new\(S, func , ud\);	if \(S-&gt;nco &gt;= S-&gt;cap\) {		int id = S-&gt;cap;		S-&gt;co = realloc\(S-&gt;co, S-&gt;cap \* 2 \* sizeof\(struct coroutine \*\)\);		memset\(S-&gt;co + S-&gt;cap , 0 , sizeof\(struct coroutine \*\) \* S-&gt;cap\);		S-&gt;co\[S-&gt;cap\] = co;		S-&gt;cap \*= 2;		++S-&gt;nco;		return id;	} else {		int i;		for \(i = 0; i &lt; S-&gt;cap; i++\) {			int id = \(i+S-&gt;nco\) % S-&gt;cap;			if \(S-&gt;co\[id\] == NULL\) {				S-&gt;co\[id\] = co;				++S-&gt;nco; 				// 返回创建好的协程id				return id;			}		}	}	assert\(0\);	return -1;} static voidmainfunc\(uint32\_t low32, uint32\_t hi32\) {	// resume param	uintptr\_t ptr = \(uintptr\_t\)low32 \| \(\(uintptr\_t\)hi32 &lt;&lt; 32\);	struct schedule \*S = \(struct schedule \*\)ptr;	int id = S-&gt;running; 	// debug	printf\("mainfunc: coroutine id\[%d\]\n", S-&gt;running\); 	struct coroutine \*C = S-&gt;co\[id\];	C-&gt;func\(S,C-&gt;ud\); 	\_co\_delete\(C\); 	S-&gt;co\[id\] = NULL;	--S-&gt;nco;	S-&gt;running = -1;} void coroutine\_resume\(struct schedule \* S, int id\) { 	assert\(S-&gt;running == -1\);	assert\(id &gt;= 0 && id &lt; S-&gt;cap\); 	struct coroutine \*C = S-&gt;co\[id\];	if \(C == NULL\)		return; 	int status = C-&gt;status;	switch\(status\) { 	case COROUTINE\_READY:		getcontext\(&C-&gt;ctx\);		C-&gt;ctx.uc\_stack.ss\_sp = S-&gt;stack;       // stack top起始位置\(SP\)		C-&gt;ctx.uc\_stack.ss\_size = STACK\_SIZE;   // 用于计算stack bottom\(数据从stack bottom开始存放\)		C-&gt;ctx.uc\_link = &S-&gt;main;              // 协程执行完切回的context		S-&gt;running = id;                        // 调度器记录当前准备调度的协程id		C-&gt;status = COROUTINE\_RUNNING;          // 将准备调度的协程状态置为"待运行状态" 		uintptr\_t ptr = \(uintptr\_t\)S;		// 需要考虑跨平台指针大小不同的问题		makecontext\(&C-&gt;ctx, \(void \(\*\)\(void\)\) mainfunc, 2, \(uint32\_t\)ptr, \(uint32\_t\)\(ptr&gt;&gt;32\)\);		// 开始执行mainfunc回调, 执行完继续fall through, 即执行下一个协程		swapcontext\(&S-&gt;main, &C-&gt;ctx\);		// debug		printf\("COROUTINE\_READY: coroutine id\[%d\] return\n", S-&gt;running\);		break; 	case COROUTINE\_SUSPEND:		// stack从高地址向低地址生长, 即从stack bottom向stack top存储数据		memcpy\(S-&gt;stack + STACK\_SIZE - C-&gt;size, C-&gt;stack, C-&gt;size\);		S-&gt;running = id;		C-&gt;status = COROUTINE\_RUNNING;		swapcontext\(&S-&gt;main, &C-&gt;ctx\);		// debug		sleep\(2\);		printf\("COROUTINE\_SUSPEND: coroutine id\[%d\] return\n", S-&gt;running\);		break; 	default:		assert\(0\);	} 	printf\("break switch: coroutine id\[%d\] return\n", S-&gt;running\);} static void\_save\_stack\(struct coroutine \*C, char \*top\) { 	// 在stack上创建一个局部变量, 标识当前栈顶\(SP\)的位置\(低地址\)	char dummy  = 0;	printf\("\_save\_stack: &C\[%p\] top\[%p\] &dummy\[%p\] top - &dummy\[%d\] STACK\_SIZE\[%d\]\n", &C, top, &dummy, top - &dummy, STACK\_SIZE\); 	// 检查stack是否有溢出	assert\(top - &dummy &lt;= STACK\_SIZE\); 	// 按需保存当前协程的stack \(begin\)	// 判断协程栈的空间是否足够, 若不够则重新分配	if \(C-&gt;cap &lt; top - &dummy\) {		free\(C-&gt;stack\);		C-&gt;cap = top - &dummy;		C-&gt;stack = malloc\(C-&gt;cap\);	}	C-&gt;size = top - &dummy;	memcpy\(C-&gt;stack, &dummy, C-&gt;size\);	// 按需保存当前协程的stack \(end\)} voidcoroutine\_yield\(struct schedule \* S\) { 	int id = S-&gt;running;	// debug	printf\("coroutine\_yield: coroutine id\[%d\] into\n", S-&gt;running\);	assert\(id &gt;= 0\);     // 注意makecontext\(\)时指定ss\_sp为S-&gt;stack, 当C-&gt;ctx执行时, 协程的栈是存储在S-&gt;stack上的, 即把堆上分配的一块空间虚拟成栈来使用	struct coroutine \* C = S-&gt;co\[id\]; 	// 与栈顶S-&gt;stack的位置进行比较    printf\("coroutine\_yield: &C\[%p\] S-&gt;stack\[%p\]\n", &C, S-&gt;stack\);	assert\(\(char \*\)&C &gt; S-&gt;stack\); 	// 按需动态保存协程的private stack	\_save\_stack\(C, S-&gt;stack + STACK\_SIZE\); 	C-&gt;status = COROUTINE\_SUSPEND;	S-&gt;running = -1; 	swapcontext\(&C-&gt;ctx, &S-&gt;main\);} int coroutine\_status\(struct schedule \* S, int id\) {	assert\(id &gt;= 0 && id &lt; S-&gt;cap\);	if \(S-&gt;co\[id\] == NULL\) {		return COROUTINE\_DEAD;	}	return S-&gt;co\[id\]-&gt;status;} int coroutine\_running\(struct schedule \* S\) {	return S-&gt;running;}

结果输出：

LINUX SUSE 32: 

main start

mainfunc: coroutine id\[0\]

coroutine 0 : 0

coroutine\_yield: coroutine id\[0\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_READY: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

mainfunc: coroutine id\[1\]

coroutine 1 : 100

coroutine\_yield: coroutine id\[1\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_READY: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 0 : 1

coroutine\_yield: coroutine id\[0\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 1 : 101

coroutine\_yield: coroutine id\[1\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 0 : 2

coroutine\_yield: coroutine id\[0\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 1 : 102

coroutine\_yield: coroutine id\[1\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 0 : 3

coroutine\_yield: coroutine id\[0\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 1 : 103

coroutine\_yield: coroutine id\[1\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 0 : 4

coroutine\_yield: coroutine id\[0\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

coroutine 1 : 104

coroutine\_yield: coroutine id\[1\] into

coroutine\_yield: &C\[0xb7df4f9c\] S-&gt;stack\[0xb7cf5008\]

\_save\_stack: &C\[0xb7df4f7c\] top\[0xb7df5008\] &dummy\[0xb7df4f67\] top -&dummy\[161\] STACK\_SIZE\[1048576\]

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

COROUTINE\_SUSPEND: coroutine id\[-1\] return

break switch: coroutine id\[-1\] return

main end 

 

参考

\[1\] http://en.wikipedia.org/wiki/Setcontext

\[2\] https://github.com/cloudwu/coroutine/



