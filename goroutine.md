Go语言里面的并发使用的是Goroutine，Goroutine可以看做一种轻量级的线程，或者叫用户级线程。与Java的Thread很像，用法很简单：

go fun\(params\);

相当于Java的

new Thread\(someRunnable\).start\(\);

虽然类似，但是Goroutine与Java Thread有着很大的区别。

Java里的Thread使用的是线程模型的一对一模型，每一个用户线程都对应着一个内核级线程。

![](/assets/javathread.png)

上图有两个CPU，然后有4个Java thread，每个Java thread其实就是一个内核级线程，由内核级线程调度器进行调度，轮流使用两个CPU。内核级线程调度器具有绝对的权力，所以把它放到了下面。内核级线程调度器使用公平的算法让四个线程使用两个CPU。

![](/assets/gothread.png)

Go的Goroutine是用户级的线程。同样是4个Goroutine，可能只对应了两个内核级线程。Goroutine调度器把4个Goroutine分配到两个内核级线程上，而这两个内核级线程对CPU的使用由内核线程调度器来分配。

与内核级线程调度器相比，Goroutine的调度器与Goroutine是平等的，所以把它和Goroutine放到了同一个层次。调度器与被调度者权力相同，那被调度者就可以不听话了。一个Goroutine如果占据了CPU就是不放手，调度器也拿它没办法。

同样是下面一段代码：

void run\(\) {

int a = 1;

while\(1==1\) {

a = 1;

}

}

在Java里，如果起多个这样的线程，它们可以平等的使用CPU。但是在Go里面，如果起多个这样的Goroutine，在启动的内核级线程个数一定情况下（通常与CPU个数相等），那么最先启动的Goroutine会一直占据CPU，其它的Goroutine会starve，饿死，因为它不能主动放弃CPU，不配合别人工作。说到配合工作，那就需要说一下协程（coroutine，可以当做cooperative routine\)，协程需要相互合作，互相协助，才能正常工作，所以叫做协程。

![](/assets/gothreadco.png)

协程并不需要一个调度器，它是完全靠互相之间协调来工作的。协程的定义在学术上很抽象，目前实际应用中，协程通常是使用单个内核级线程，用来把异步编程中使用的难懂的callback方式改成看上去像同步编程的样子。

比如nodejs是异步单线程事件驱动的，在一段代码中如果有多次异步操作，比如先调用一个支付系统，得到结果后再更新数据库，那么可能需要嵌套使用callback。pay函数是一个调用支付系统的操作，异步发出请求后就返回，然后等支付完成的事件后触发第一个回调函数，这个函数是更新数据库，又是一个异步操作，等这个异步操作完成后，再次触发返回更新结果的回调函数。 这里只有两个异步操作，如果多的话，有可能会有很多嵌套。

pay\(amount, callback\(payamount\) {

update\(payamount, callback\(result\) {

return result;

}\)}\);

而使用协程，可以看上去像是同步操作

pay\(amount\){

//异步，立刻返回

//payamount需要操作完成后才能被赋值

payamount = dopay\(amount\);

yeild main;//把控制权返回主routine

//dopay事件完成后，主routine会调起这个routine,

//继续执行doupdate

result=doupdate\(payamount\);

yeild main;  //再次把控制权返回主routine

return result;

}

（以上都是伪代码）

把原来的各种嵌套callback改成协程，那么逻辑就会清晰很多。

Goroutine与Coroutine不一样，开发者并不需要关心Goroutine如何被调起，如何放弃控制权，而是交给Goroutine调度器来管理。开发者不用关心，但是Go语言的编译器会替你把工作做了，因为Goroutine必须主动交出控制权才能由调度器统一管理。首先我们可以认为写上面那种死循环而且不调用任何其他函数的Goroutine是没意义的，如果真在实际应用中写出这样的代码，那开发者不是一个合格的程序员。一个Goroutine总会调用其他函数的，一种调用是开发者自己写的函数，一种是Go语言提供的API。那编译器以及这些API就可以做文章了。

比如

void run\(\) {

int a = 0;

int b = 1;

a = b \* 2;

for\(int i = 0; i &lt; 100; i++\) {

```
a = func1\(a\);
```

}

}

那么编译器可能会在调用其他函数的地方偷偷加上几条语句，比如：

void run\(\) {

int a = 0;

int b = 1;

a = b \* 2;

for\(int i = 0; i &lt; 100; i++\) {

//进入调度器，或者以一定概率进入调度器

schedule\(\);

a = func1\(a\);

}

}

再比如

void run\(\) {

socket = new socket\(\);

while\(buffer = socker.read\(\)\) {

deal\(buffer\);

}

}

socker.read\(\)是Go语言提供的一个系统函数，那么Go语言可能在这里面加点操作，读完数据后，进入调度器，让调度器决定这个Goroutine是否继续跑。

下面这段Go语言代码，把内核级线程设成2个，那么主线程会饿死，而在func1里加一个sleep就可以了，这样func1才有机会放弃控制权。

wKiom1eomsjB8sEdAADH57ysyHw063.png

当然Go语言的调度器要比这复杂的多。Goroutine与协程还是有区别的，实现原理是一样的，但是Goroutine的目的是为了实现并发，在Go语言里，开发者不能创建内核级线程，只能创建Goroutine，而协程的目的如上面所示，目前比较常见的用途就是上面这个。Go语言适合编写高并发的应用，因为创建一个Goroutine的代价很低，而且Goroutine切换上下文开销也很低，与创建内核级线程相比，Goroutine的开销可能只是几十分之一甚至几百分之一，而且它不占内核空间，每个内核级线程都会占很大的内核空间，能创建的线程数最多也就几千个，而Goroutine可以很轻松的创建上万个。

Goroutine底层的实现，在Linux上面是用makecontext，swapcontext，getcontext，setcontext这几个函数实现的，这几个系统调用可以实现用户空间线程上下文的保存和切换。

