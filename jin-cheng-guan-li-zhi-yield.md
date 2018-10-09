/\*\* \* yield - yield the current processor to other threads. \* \* This is a shortcut for kernel-space yielding - it marks the \* thread runnable and calls sys\_sched\_yield\(\). \*/void \_\_sched yield\(void\){	set\_current\_state\(TASK\_RUNNING\);	sys\_sched\_yield\(\);}这个yield并没有指定当前进程要将执行权利移交给谁，只是放弃运行权利，至于下面由谁来运行，完全看进程调度schedule\(\);多用于I/O等待时，进程短暂wait，但是并没有退出运行队列。/\*\* \* yield\_to - yield the current processor to another thread in \* your thread group, or accelerate that thread toward the \* processor it's on. \* @p: target task \* @preempt: whether task preemption is allowed or not \* \* It's the caller's job to ensure that the target task struct \* can't go away on us before we can do any checks. \* \* Returns true if we indeed boosted the target task. \*/bool \_\_sched yield\_to\(struct task\_struct \*p, bool preempt\){	struct task\_struct \*curr = current;	struct rq \*rq, \*p\_rq;	unsigned long flags;	bool yielded = 0; 	local\_irq\_save\(flags\);	rq = this\_rq\(\); again:	p\_rq = task\_rq\(p\);	double\_rq\_lock\(rq, p\_rq\);	while \(task\_rq\(p\) != p\_rq\) { //确保两个进程是在同一个rq队列上的。		double\_rq\_unlock\(rq, p\_rq\);		goto again;	} 	if \(!curr-&gt;sched\_class-&gt;yield\_to\_task\)		goto out; 	if \(curr-&gt;sched\_class != p-&gt;sched\_class\)		goto out; 	if \(task\_running\(p\_rq, p\) \|\| p-&gt;state\) 		goto out; 	yielded = curr-&gt;sched\_class-&gt;yield\_to\_task\(rq, p, preempt\);	if \(yielded\) {		schedstat\_inc\(rq, yld\_count\);		/\*		 \* Make p's CPU reschedule; pick\_next\_entity takes care of		 \* fairness.		 \*/		if \(preempt && rq != p\_rq\)			resched\_task\(p\_rq-&gt;curr\);	} out:	double\_rq\_unlock\(rq, p\_rq\);	local\_irq\_restore\(flags\); 	if \(yielded\)		schedule\(\); 	return yielded;}

当前进程将执行的权利移交给运行在同一个CPU上同一线程组的其它线程p。



static bool yield\_to\_task\_fair\(struct rq \*rq, struct task\_struct \*p, bool preempt\){	struct sched\_entity \*se = &p-&gt;se; 	if \(!se-&gt;on\_rq\)		return false; 	/\* Tell the scheduler that we'd really like pse to run next. \*/	set\_next\_buddy\(se\); //将CPU下一个要调度执行的进程设置为p-&gt;se,那么下一次调度的时候pick\_next\_task选到的se将是p-&gt;se。这也体现了cfs\_rq队列中next的作用。 	yield\_task\_fair\(rq\); 	return true;}

经过上面两个函数的介绍可以知道，当前任务由于某些原因（多数是等待IO）自愿放弃（yield）CPU权利的时候，其状态仍然为running， 以便IO完成的时候马上被调度，以防止IO数据的丢失。



