# GCD 使用中需要注意的坑

1. dispatch_once_t 必须是全局或 static 变量 (其实就是保证 dispatch_once_t 只有一份实例)
2. dispatch_after 是**延迟提交**，不是延迟运行
3. dispatch_suspend != 立即停止队列的运行 

> (在 dispatch_suspend 挂起队列后，第一个 block 还是在运行，并且正常输出。
> 结合文档，我们可以得知，dispatch_suspend 并不会立即暂停正在运行的 block，而是在当前 block 执行完成后，暂停后续的 block 执行。

> )

4. “同步” 的 dispatch_apply 

> dispatch_apply 的作用是在一个队列（串行或并行）上 “运行” 多次 block，其实就是简化了用循环去向队列依次添加 block 任务。但是我个人觉得这个函数就是个“坑”，先看看如下代码运行结果:


```
dispatch_queue_t queue = dispatch_queue_create("me.tutuge.test.gcd", DISPATCH_QUEUE_SERIAL);

dispatch_apply(3, queue, ^(size_t i) {
    NSLog(@"apply loop: %zu", i);
});


NSLog(@"After apply");

```


运行结果:
> 2015-04-01 00:55:40.854 GCDTest[47402:1893289] apply loop: 0
> 2015-04-01 00:55:40.856 GCDTest[47402:1893289] apply loop: 1
> 2015-04-01 00:55:40.856 GCDTest[47402:1893289] apply loop: 2
> 2015-04-01 00:55:40.856 GCDTest[47402:1893289] After apply

    
看，明明是提交到异步的队列去运行，但是 “After apply” 居然在 apply 后打印，也就是说，dispatch_apply 将外面的线程（main 线程）“阻塞” 了！

查看官方文档，dispatch_apply 确实会 “等待” 其所有的循环运行完毕才往下执行 =。=，看来要小心使用了

5. dispatch_sync, dispatch_apply 导致的死锁

   
``` 
dispatch_queue_t queue = dispatch_queue_create("me.tutuge.test.gcd", DISPATCH_QUEUE_SERIAL)

dispatch_apply(3, queue, ^(size_t i) {
	NSLog(@"apply loop: %zu", i)

    //再来一个dispatch_apply！死锁！
	dispatch_apply(3, queue, ^(size_t j) {
		NSLog(@"apply loop inside %zu", j)
	})
})
```
> 这端代码只会输出 “apply loop: 1”。。。就没有然后了 =。=
> 所以，一定要避免 dispatch_apply 的嵌套调用


6. 使用 dispatch_barrier_async,dispatch_barrier_sync 的注意事项

## > ***dispatch_barrier_(a)sync 只在自己创建的并发队列上有效，在全局 (Global) 并发队列、串行队列上，效果跟 dispatch_(a)sync 效果一样。***
> 既然在串行队列上跟 dispatch_(a)sync 效果一样，那就要小心别死锁！


