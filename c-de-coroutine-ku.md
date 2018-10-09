https://github.com/cloudwu/coroutine/

https://github.com/lattera/glibc

今天实现了一个 C 用的 coroutine 库.



我相信这个东西已经被无数 C 程序员实现过了, 但是通过 google 找了许多, 或是接口不让我满意, 或是过于重量.



在 Windows 下, 我们可以通过 fiber 来实现 coroutine , 在 posix 下, 有更简单的选择就是 setcontext 。



我的需求是这样的：



首先我需要一个 asymmetric coroutine 。如果你用过 lua 的 coroutine 就明白我指的是什么。



其次，我不希望使用 coroutine 的人太考虑 stack 大小的问题。就是说，用户在 coroutine 内可以使用的 C stack 大小和主线程一样多。



我需要单个 coroutine 空间占有率不要太高，因为我可能会使用上千个 coroutine ，我不希望每个都占用百万字节的堆栈。



因为，在我的应用场合，coroutine 切换的那一刻，使用的堆栈并不多（它可能调用一些需要大量堆栈的库函数，但那些库函数中并不会发生切换），所以，在切换的时刻做栈拷贝是可以接受的。coroutine 切换并不算频繁，这个切换成本是可控的。



最终，我以我的需求实现了我需要的这个版本。



当然，暂时它不支持 windows 。其实 port 到 windows 平台不算困难，只需要把 setcontext 那组 api 改成 fiber 的即可。

