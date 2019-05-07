# iOS中的多线程技术

## 进程与线程

### 线程
线程（thread）是组成进程的子单元，操作系统的调度器可以对线程进行单独的调度。**实际上，所有的并发编程 API 都是构建于线程之上的 —— 包括 GCD 和操作队列（operation queues）**。

多线程可以在单核 CPU 上同时（或者至少看作同时）运行。操作系统将小的时间片分配给每一个线程，这样就能够让用户感觉到有多个任务在同时进行。如果 CPU 是多核的，那么线程就可以真正的以并发方式被执行，从而减少了完成某项操作所需要的总时间。

进程是一个正在执行的程序实例.
和其他抢占式的多任务操作系统一样，进程是一个正在执行的程序实例。早期的 UNIX 和 Linux 被设计为多进程操作系统。进程是 CPU 调度的基本单位。CPU 通过时间片轮转等调度算法在多个进程之间快速切换，制造了多个进程并发运行的假象，从而实现了多任务。在 N 个 CPU 密集型进程并发执行的情况下，每个进程得到了真实 CPU 速度的 1/N。在多进程操作系统中，开发者编写顺序程序。从 main 函数开始，进程的执行是串行的。当遇到 I/O 操作阻塞时就会放弃进程时间片。这对性能有很大的影响，因为进程上下文切换的开销很大。因此线程应运而生。

在现代的操作系统中，线程才是 CPU 调度的基本单位。而进程作为线程的容器，是资源管理的单位。线程的执行也是串行的，采用与进程相同的调度算法使其并发执行，每个 CPU 密集型线程同样得到真实 CPU 速度的 1/N，但线程使进程的执行分割为多个子任务，因此线程也被称为轻量级进程。它的好处是当某个线程因为 I/O 操作阻塞时，还可以去执行其他线程从而最大化利用进程时间片。

每个进程都拥有独立且受保护的内存空间，用来存放程序正文和数据以及其打开的文件、子进程、即将发生的报警、信号处理程序、账号信息等。线程只拥有程序计数器、寄存器、堆栈等少量资源，但与其他线程共享该进程的整个内存空间。因此线程切换速度比进程快 10 到 100 倍。

进程分为前台进程和后台进程。iOS 中的后台进程受到了极大的限制。后台进程只可以存在短暂的一段时间就会被系统置为 Suspended 状态。在这种状态下，进程将不能得到 CPU 时间片。当收到内存警告时，系统就会将处在 Suspended 状态后台进程从内存中移除。

## 并发与并行

并行可以在计算机的多个抽象层次上运用，这里仅讨论任务级并行（程序设计层面），不讨论指令级并行等。

并发指能够让多个任务在逻辑上同时执行的程序设计，而并行则是指在物理上真正的同时执行。并行是并发的子集，属于并发的一种实现方式。通过时间片轮转实现的多任务同时执行是通过调度算法实现逻辑上的同步执行，属于并发，他们不是真正物理上的同时执行，不属于并行。当通过多核 CPU 实现并发时，多任务是真正物理上的同时执行，才属于并行。

## 为什么使用多线程

上述关于进程和线程的讨论都建立在单核 CPU 系统中。而现代的计算机系统都拥有多核 CPU，真正实现了多线程同时运行。所以多线程对现代计算机系统来说拥有更加重要的意义，它可以让 CPU 的每个核心得到充分利用，从而真正提高程序的运行效率。

> iPhone4 的 A4 就是单核处理器。从 A4 之后苹果一直坚守在双核阵营。直到现在的 iPhone7 才用上了四核处理器。在家用 PC 领域，运用超线程技术的 Intel Core i7 6950k 已经达到了 10 核 20 线程，可以允许 20 个线程同时运行。

《深入理解计算机系统》中进行过一个实验。将程序运行在一个有四个处理器核的系统上，对一个n=2^31个元素的序列求和。最终得到如下图所示的数据。随着线程数的增加，运行时间下降，直到增加到四个线程，运行时间趋于平稳，甚至开始有点增加。


如果某个进程内都是 CPU 密集型线程，那么多线程对效率的提升没有半点好处，反而会因为线程上下文的频繁切换增大 CPU 开销。这也是上述实验在线程数大于 CPU 数时处理时间增加的原因。

但真正的程序中总会有 I/O 密集型线程。正在处理 I/O 的线程大部分时间都处在等待状态，它们不占用 CPU 资源。这时候，线程数量只有大于 CPU 数量时才能保证 CPU 高效运行。所以在遇到 I/O 任务时我们最好开启新的线程。这样做还有另一个好处。iOS 中只有在主线程才可以刷新 UI，如果这些 I/O 任务放在主线程，就可能会阻塞主线程后续的 UI 刷新任务，使界面产生卡顿。

所以使用多线程可以充分利用现在的多核 CPU、减少 CPU 的等待时间、防止主线程阻塞等。除了性能上的提升，对于批量任务，使用多线程也能使代码逻辑更加清晰。

## 多线程应用实例

YYDispatchQueuePool 通过 CPU 核心数来限制总的线程数量（实际上只是将数量限制在合理的范围内），提高 CPU 利用率的同时又尽量减少线程切换的开销。

SDWebImage 在子线程批量处理从磁盘读取图片的任务。在 I/O 操作频繁的情况下，通过多线程充分利用等待时间，同时防止了主线程的阻塞。

## Pthreads

Pthreads 全称 POSIX Threads。POSIX 全称 Portable Operating System Interface of UNIX，译为“可移植操作系统接口”，是 IEEE 为要在各种 UNIX 操作系统上运行软件定义的 API 标准的总称。Pthreads 是 实现 POSIX 线程标准的一套 API。Linux、MacOS、iOS 和大部分 UNIX 系统都支持此标准。甚至 Windows 也通过第三方库 pthreads-w32 实现了此标准。

Pthreads 定义了 C 语言的接口，拥有超过 100 个 API 用来创建和管理线程，这些 API 全都以 pthread_ 作为前缀。iOS 中 CFRunLoop 就是基于 Pthreads 来管理的。

学习使用 Pthreads 是很有好处的，它拥有极强的可移植性。UNIX、Linux、Android、iOS、MacOS、Windows 等现在的主流操作系统都提供对 Pthreads 的支持。如下代码是一个简单的 Pthreads 从创建到其销毁的过程。


```
- (void)viewDidLoad {
    [super viewDidLoad];
    pthread_t thread = NULL;
    pthread_create(&thread, NULL, operate, "my_param");
}

void *operate(void *param) {
    pthread_setname_np("my_thread");
    char thread_name[10];
    pthread_getname_np(pthread_self(), thread_name, 10);
    printf("an operation running in %s with param \"%s\"", thread_name, param);
    pthread_exit((void *)0);
}
```
上述代码打印出 “an operation running in my_thread with param "my_param"”。如果在 operate 方法中断点，可以看到代码暂停在我们新建的 my_thread 线程中。



## NSThread

苹果对 Pthreads 进行了面向对象的封装 NSTherad（也可以认为 NSTherad 只是用到了 Pthreads）。但在 iOS 开发中苹果为多线程提供了一套不用接触线程概念的 API —— GCD。Objective-C 进一步封装了这套 API，暴露给用户的是 NSOpertion 相关的对象。

在使用 NSThread 之前，这里需要先说明线程与 NSThread 对象的区别。NSThread 对象被创建时并不代表一个真正的线程也随之创建，只要当我们调用 NSThread 的 star 方法时才会创建真正的线程。但在面向对象的层面，NSThread 的对象对我们来说就是线程。所以下文提到的线程在没有特别说明的情况下都指 NSThread 对象。

使用 NSThread 实现多线程需要我们自己管理线程的生命周期。我们可以自己设计线程池，自己派发任务。虽然看起来更加灵活，但对大多数对于需求系统系统的封装（GCD、NSOpreation）都能完全满足需求。

创建线程
使用 NSThread 的指定初始化方法来创建线程是没有意义的。这种方式创建的线程没有可执行的任务，NSThread 也没有提供稍后设置任务的接口，我们也没有办法从外部启动线程的 Runloop，所以也没有办法向线程中派发任务。

`- (instancetype)init;`

所以应该使用如下方法来创建线程并通过 block 或 target-selector 来指定任务。创建的线程需要手动调用 start 方法来启动，线程启动后就会执行在初始化时指定的任务。我们还可以在这次任务中手动启动当前线程的 Runloop 以便让此线程常驻内存并接受后续派发的任务。


```
- (instancetype)initWithTarget:(id)target selector:(SEL)selector object:(nullable id)argument;
- (instancetype)initWithBlock:(void (^)(void))block;
```
也可以使用如下类方法来创建线程。这样创建完线程后还会自动调用 start 方法启动该线程。由于这两个方法没有返回值，我们无法在外部拿到线程对象，但我们也不用自己去管理线程对象的生命周期。


```
+ (void)detachNewThreadWithBlock:(void (^)(void))block;
+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(nullable id)argument;
```
除了创建自己的子线程外，我们也可以获取已存在的线程。NSThread 只提供了两个方法分别用来获取当前线程和主线程。


```
@property (class, readonly, strong) NSThread *currentThread;
@property (class, readonly, strong) NSThread *mainThread;
```
启动线程
创建的线程对象后需要手动调用 start 方法才能启动线程。如果我们在创建线程时有制定任务，start 方法就会调用线程的入口方法 - mian。main 方法的默认实现会执行初始化时的 block 或 selector。main 方法只能通过子类重写，不能直接调用。子类化 NSThread 重写 main 方法时不比调用 super，我们可以按照自己的逻辑实现入口方法。


```
- (void)start;
- (void)main;
```
派发任务
主线程在 App 运行期间始终存在，如果想向主线程的 Runloop 派发任务，可以使用 NSObject + NSThreadPerformAdditions 类别中的performSelectorOnMainThread:~两个方法。它们还可以指定携带的参数以及是否阻塞当前线程（派发此任务的线程），还可以派发到指定的 Runloop mode。

如果想向子线程派发任务，则需要先手动启动子线程的 Runloop。然后使用performSelector:onThread:~两个方法向指定线程派发任务。当然如果你指定的线程是主线程，它们的效果就与前两个相同了。

最后我们也可以使用performSelectorInBackground:~向系统默认的后台线程派发任务。相比之下我们不需要自己管理子线程的生命周期，省去了需要不必要的麻烦。


```
@interface NSObject (NSThreadPerformAdditions)

// 在主线程同步或异步执行任务
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array;
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;

// 在指定线程和指定 Runloop Mode 同步或异步执行任务
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array ;
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait ;

// 在后台线程执行任务
- (void)performSelectorInBackground:(SEL)aSelector withObject:(nullable id)arg ;

@end
```
优先级
线程优先级决定了任务开始执后系统资源分配的优先级，例如 CPU 时间片, 网络资源, 硬盘资源等。NSThread 的优先级通过浮点数变量 threadPriority 来控制，它的范围从 0 到 1，默认为 0.5。但这个属性在 iOS8 之后已经被废弃，取而代之的是枚举 NSQualityOfService（QoS）。


```
+ (double)threadPriority;
+ (BOOL)setThreadPriority:(double)p;
@property double threadPriority;
```
QoS 有五种优先级，默认为 NSQualityOfServiceDefault。它的出现统一了 Cocoa 中所有多线程技术的优先级。在此之前，NSOperation 和 NSThread 都通过 threadPriority 来指定优先级，而 GCD 则是根据 DISPATCH_QUEUE_PRIORITY_DEFAULT 等宏定义的整形数来指定优先级。正确的使用新的 QoS 来指定线程或任务优先级可以让 iOS 更加智能的分配硬件资源，以便于提高执行效率和控制电量。


```
@property NSQualityOfService qualityOfService;

typedef NS_ENUM(NSInteger, NSQualityOfService) {
    NSQualityOfServiceUserInteractive = 0x21,
    NSQualityOfServiceUserInitiated = 0x19,
    NSQualityOfServiceUtility = 0x11,
    NSQualityOfServiceBackground = 0x09,
    NSQualityOfServiceDefault = -1
};

```苹果对于 QoS 五种优先级的使用有如下建议。开发者应该严格遵守这样的规则。

* NSQualityOfServiceUserInteractive：用来处理用户操作，例如界面刷新、动画等。优先级最高，即时执行。
* NSQualityOfServiceUserInitiated：处理初始化任务，为将来的用户操作作准备。例如加载文件或 Email 等。基本即时执行，最多几秒延迟。
* NSQualityOfServiceUtility：用户不需要立即结果的操作，一般伴随进度条。例如下载、数据导入、周期性的内容更新等。几秒到几分钟延迟。
* NSQualityOfServiceBackground：用于用户不可见的操作。例如简历索引、预加载、同步等。几分钟到数小时延迟。
* NSQualityOfServiceDefault：默认的 QoS 用来表示缺省值。当有可能通过其它途径推断出可能的 QoS 信息时，则使用推断出的 Qos。如果不能推断，则使用 UserInitiated 和 Utility 之间的 QoS。
* Utility 及以下的优先级会受到 iOS9 中低电量模式的控制。另外，在没有用户操作时，90% 任务的优先级都应该在 Utility 之下。更多关于电量模式与 QoS 的关系可以参考 官方文档

## 休眠线程

在线程的运行期间我们可以调用如下方法使执行此行代码的线程进入休眠。调用的时候都需要指定休眠的时间，sleepUntilDate指定的是休眠到某个时间，sleepForTimeInterval指定的是休眠的秒数。线程休眠时 Runloop 不会被事件唤醒。

```
+ (void)sleepUntilDate:(NSDate *)date;
+ (void)sleepForTimeInterval:(NSTimeInterval)ti
```;
线程信息
线程在运行期间有多种状态。可以用如下三个属性来判断线程的运行状态。

```
@property (readonly, getter=isExecuting) BOOL executing; // 正在运行
@property (readonly, getter=isFinished) BOOL finished; // 已经完成
@property (readonly, getter=isCancelled) BOOL cancelled; // 已经取消
```
在线程初始化完成后，通过如下方法和属性，可以设置线程的名字，还可以在线程 start 之前设置线程堆栈的大小（单位 byte，最小 16KB 且必须为 4KB 的整数倍）。在线程运行期间可以动态的获取线程的调用堆栈以及调用堆栈的返回地址，还可以判断当前线程是否为主线程。

```
// 调用堆栈的返回地址
@property (class, readonly, copy) NSArray<NSNumber *> *callStackReturnAddresses;

// 调用堆栈的回溯
@property (class, readonly, copy) NSArray<NSString *> *callStackSymbols;

// 线程名字
@property (nullable, copy) NSString *name;

// 线程堆栈大小
@property NSUInteger stackSize;
```
如果想在线程的运行之前储存一些线程依赖的数据，可以使用 threadDictionary 属性。它被定义为只读的可变字典类型，防止直接的指针赋值等“误操作”替换掉系统储存的数据。NSThread 本身没有使用到这个属性，但是 Cocoa 中的其他类可能会使用它。例如，Foundation 用它来存储线程默认的 NSConnection 和 NSAssertionHandler 实例。所以我们在用它储存数据时要避免与系统的 Key 重名。

```
@property (readonly, retain) NSMutableDictionary *threadDictionary;
使用如下方法可以判断当前线程是否为主线程。

// 接受此消息的线程是否为主线程
@property (readonly) BOOL isMainThread;

// 执行此行代码的线程是否为主线程
@property (class, readonly) BOOL isMainThread;
```
## 多线程状态

当有任意一个线程从主线程分离出去时，App 就被认为是多线程的。我们可以通过 isMultiThreaded 方法判断 App 当前是否处在多线程状态（Pthread 等非 Cocoa API 创建的线程不算）。只要某个子线程被创建后（这里的线程是正真的线程，不是 NSThread 对象），不需要正在运行，就认为是多线程状态。

当 App 将要变为多线程时我们还可以通过 NSWillBecomeMultiThreadedNotification 通知来监听此状态。另外 NSThread 的接口中还有 NSDidBecomeSingleThreadedNotification 这个通知，但这个通知苹果并没有实现，放在这里逗你玩而已。


```
+ (BOOL)isMultiThreaded;

FOUNDATION_EXPORT NSNotificationName const NSWillBecomeMultiThreadedNotification;

FOUNDATION_EXPORT NSNotificationName const NSDidBecomeSingleThreadedNotification;
```
现在的 App 也只有在 main 函数里是单线程状态了。就算我们自己不新建线程，系统也会新建线程去执行任务。

退出线程
退出线程有两个方法，cancel 和 exit。cancel 方法将线程置为 cancelled 状态以表明线程将要退出，线程会执行完正在执行的任务才退出。而 exit 方法会立即退出当前线程，正在执行中的任务分配的资源将没有机会得到释放，所以正常情况下最好调用 cancel 方法。在线程将要退出时可以通过 NSThreadWillExitNotification 通知来监听此事件。


```
- (void)cancel;
+ (void)exit;
```

FOUNDATION_EXPORT NSNotificationName const NSThreadWillExitNotification;
至此 NSThread 所有公开的 API 都已经介绍完了，更详细的介绍可以参考 NSThread 的官方 API Reference

## GCD

> GCD 拥有众多 API，这里都不再介绍。本节主要介绍 GCD 引入的编程范式的变化。关于 API 你可以在 官方文档 查看详细信息。

GCD（Grand Central Dispatch） 是苹果宣称用来替换线程的一套基于 C 语言的 API。这套 API 引入了编程范式的变化，使从线程和线程函数的角度思考，变为从任务和队列的角度思考。GCD 使用队列来派发任务（block），队列分为串行队列和并发队列，任务的派发方式分为同步派发和异步派发。GCD 自己维护了一个底层的线程库实现，以支持并发和异步的执行模型。使用 GCD 可以减轻开发者处理并发问题的负担，减少类似于死锁之类的潜在错误，而且能够自动地随着逻辑处理器的个数而扩展。

GCD 虽然带来了使用上的便捷性，却同时带来了理解上的复杂性。除了线程的概念外，还要理解 GCD 带来的抽象概念：派发方式和队列性质。这就导致你共需要理解“在某个线程向某种队列使用某种派发方式派发任务”的 2*2*2=8 种行为，当加上主队列这个特殊队列后，你共需要理解 3*3*2=18 种行为。看到这里，先不要过早的担心，其实我们只需要记住一些定理，就可以分析出所有行为的执行结果。

> 在某个线程：当前队列正在执行的任务所在的线程、非当前队列正在执行的任务所在的线程、主线程

> 向某种队列：串行队列、并发队列、主队列

> 使用某种派发方式：同步派发、异步派发

队列都是先进先出的，所以串行队列和并发队列都是按照顺序执行的。但是串行队列需要等前一个任务执行完成才可以执行下一个任务，所以串行队列只需要一个线程就可以完成任务派发。而并发队列可以允许多个任务同时执行，虽然他们开始执行的时间是按照顺序的，但是执行完成的时间并不确定。并发执行只有通过新建线程来实现。还有一种特殊的串行队列-主队列，主队列的任务只能在主线程执行，并且需要等待主线程 Runloop 空闲时才能派发。

同步派发和异步派发的区别就是是否会阻塞当前线程。同步派发的任务在当前线程执行，并使当前线程等待派发的任务执行完成才能继续执行。异步派发的任务在其他线程执行，不会阻塞当前线程。当然还有一条优先级更高的规则，凡是派发到主队列的任务都会在主线程执行。例如，在子线程同步派发任务到主线程，则不会在当前线程执行。或者在主线程异步派发到主线程，也不会在其他线程执行。当以某种派发方式向某种队列多次派发任务后，执行结果如下表所示。



|  | 同步派发 | 异步派发 |
| --- | --- | --- |
| 串行队列  | 当前线程串行执行，阻塞当前线程 | 新建单个线程串行执行，不阻塞当前线程 |
| 并发队列 | 当前线程并发执行，阻塞当前线程 | 新建多个线程并发执行，不阻塞当前线程 |
| 主队列 | 主线程串行执行，阻塞当前线程 | 主线程串行执行，不阻塞当前线程 |
	
> 对于在并发队列同步派发任务，大部分文章都进行了错误的描述，他们认为这与在串行队列同步派发是相同的，都是串行执行。后边会进行纠错。

根据同步派发和串行队列的特性，你应该意识到，如果在“当前串行队列正在执行的任务所在的线程”继续向当前队列同步派发任务，就会造成死锁。因为串行队列是顺序执行的，后进入的任务必须等待前边的任务执行完成，而同步派发的任务则会阻塞当前线程直到自己执行完成。所以，当前线程如果就是“串行队列正在执行的任务所在的线程”，那么串行队列正在执行的任务就会阻塞，就无法执行后边同步派发的任务，同步派发的任务得不到执行，就不会取消当前线程的阻塞状态，从而造成了死锁。你可以通过如下几个例子来进一步理解这些行为。

```
// task 1
dispatch_queue_t my_queue1 = dispatch_queue_create("my_queue", DISPATCH_QUEUE_CONCURRENT);
for (int i = 0; i < 3; i++) {
    dispatch_async(my_queue1, ^{
        NSLog(@"task1-%d-%@", i, [NSThread currentThread]);
    });
}
NSLog(@"task1-%@", [NSThread currentThread]);

// task1-<NSThread: 0x6100000679c0>{number = 1, name = main}
// task1-0-<NSThread: 0x61800006a5c0>{number = 3, name = (null)}
// task1-2-<NSThread: 0x60000006b140>{number = 5, name = (null)}
// task1-1-<NSThread: 0x60800006b7c0>{number = 4, name = (null)}
```
task 1 在并发队列异步派发多个任务，于是新建多个线程并发执行，与表中描述一致。下边将 task 1 中的 dispatch_async 改为 dispatch_sync 执行结果如下所示。

```
// task 1.1

// task1-0-<NSThread: 0x618000078980>{number = 1, name = main}
// task1-1-<NSThread: 0x618000078980>{number = 1, name = main}
// task1-2-<NSThread: 0x618000078980>{number = 1, name = main}
// task1-<NSThread: 0x618000078980>{number = 1, name = main}
```
这与表格中对“在并发队列同步派发多个任务”的描述“在当前线程并发执行”不一致，这个测试就是许多其他文章认为并发队列在同步派发时是串行执行的依据。这是一个明显的错误，之所以能够看起来串行执行，是因为派发任务的代码都在同一个线程运行，当第一个任务同步派发的时候阻塞了当前线程，所以第二个任务并没有立即派发，而是等第一个任务执行完才开始派发，就是说并发队列里任何时候都只有一个任务，那怎么会体现出并发性呢？

基于这样的想法设计如下代码：在两个不同的线程中向同一个并发队列同步派发任务，使并发队列中同时拥有多个任务，以此来证明并发队列在同步派发时的并发性。根据同步派发的性质，所有的任务均在当前线程执行，并阻塞当前线程。所以两个派发任务的“当前线程”都被阻塞，但派发到并发队列里的两个任务会在两个不同的“当前线程”并发执行。

```
// task 2
dispatch_queue_t my_queue2 = dispatch_queue_create("my_queue", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
    dispatch_sync(my_queue2, ^{ // block 1
        sleep(2);
        NSLog(@"1-%@", [NSThread currentThread]);
    });
});
sleep(0.5);
dispatch_sync(my_queue2, ^{ // block 2
    dispatch_async(my_queue2, ^{ // block 3
        NSLog(@"2-%@", [NSThread currentThread]);
    });
    sleep(1);
    NSLog(@"3-%@", [NSThread currentThread]);
});
NSLog(@"4-%@", [NSThread currentThread]);

// 2-<NSThread: 0x61000006ad80>{number = 3, name = (null)}
// 3-<NSThread: 0x600000064400>{number = 1, name = main}
// 4-<NSThread: 0x600000064400>{number = 1, name = main}
// 1-<NSThread: 0x600000066980>{number = 4, name = (null)}
```
实验结果与假设一致。block1、block2 两个任务被分别同步派发到并行队列，他们在不同线程执行，并且后派发的任务先完成，体现了并发队列的并发性。下边将 task 2 中的 concurrent 改为 serial 执行结果如下所示。

```
// task 2.1

2017-03-07 00:14:49.842 Threads[61945:9135694] 1-<NSThread: 0x608000264600>{number = 3, name = (null)}
2017-03-07 00:14:50.899 Threads[61945:9135640] 3-<NSThread: 0x60800007eec0>{number = 1, name = main}
2017-03-07 00:14:50.900 Threads[61945:9135640] 4-<NSThread: 0x60800007eec0>{number = 1, name = main}
2017-03-07 00:14:50.900 Threads[61945:9135694] 2-<NSThread: 0x608000264600>{number = 3, name = (null)}
```
在串行队列中，每个任务都需要等待前一个任务执行完成。同样得到了我们期望的结果。

## NSOperation

Objective-C 对 GCD 的 API 进行了面向对象的封装，GCD 中的任务对应 NSOpertion 对象，GCD 中的队列则对应 NSOpertionQueue 对象。NSOpertion 和 NSOpertionQueue 还提供判断执行状态、取消任务、控制线程数量等更多任务管理的 API。所以 AFNetworking 与 SDWebImage 等管理大量独立任务的第三方都主要使用 NSOperation 实现多线程。

NSOperation 对“任务”进行了抽象。作为抽象基类，它为子类提供了十分有用且线程安全的方式来建立状态、优先级、依赖等模型。系统提了 NSBlockOperation 和 NSInvocationOperation 两个分别以 Block 和 Invocation 储存任务的具体实现。你也可以自己继承 NSOperation 实现自己特有的储存/执行任务的方式。

NSOperation 中的任务只能执行一次。将 NSOperation 添加到 NSOperationQueue 中后就会在子线程自动执行（直接执行或使用 GCD 间接执行）。如果你不想使用 NSOperationQueue，也可以手动调用 NSOperation 的 start 方法来执行它。调用 start 方法默认会在当前线程执行，而且你必须保证在 NSOperation 的 ready 状态下调用，否则将会抛出异常。如果想让 NSOperation 在子线程执行，需要我们手动将其放在子线程，我们也可以子类化 NSOperation 并重写 main 方法实现这个过程。这显然给我们带了很多的麻烦，所以除非特别需要，最好都使用 NSOperationQueue 来执行 NSOperation。

NSOperation
NSOperation 是一个抽象类，我们应该使用具体的子类 NSBlockOperation 或 NSInvocationOperation 来创建 Operation。

```
@interface NSOperation : NSObject {

// 开始执行任务，默认实现会调用 main 方法、修改状态等。
- (void)start;

// 默认实现为空，子类实现它来执行任务。
- (void)main;

// 判断任务是否取消
@property (readonly, getter=isCancelled) BOOL cancelled;
// 取消任务
- (void)cancel;

// 判断任务是否正在执行
@property (readonly, getter=isExecuting) BOOL executing;
// 判断任务是否执行完成
@property (readonly, getter=isFinished) BOOL finished;
// 是否异步执行，默认为 NO，已经被废弃
@property (readonly, getter=isConcurrent) BOOL concurrent;
// 是否同步执行，默认为 NO
@property (readonly, getter=isAsynchronous) BOOL asynchronous;
// 任务是否可以被执行
@property (readonly, getter=isReady) BOOL ready;

// 添加依赖 operation，使当前 operation 在依赖 operation 执行完成后开始
- (void)addDependency:(NSOperation *)op;
// 移除依赖 operation
- (void)removeDependency:(NSOperation *)op;

// 获取所有依赖的 operation
@property (readonly, copy) NSArray<NSOperation *> *dependencies;

// 在 operation queue 中的优先级，默认为 normal
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
    NSOperationQueuePriorityVeryLow = -8L,
    NSOperationQueuePriorityLow = -4L,
    NSOperationQueuePriorityNormal = 0,
    NSOperationQueuePriorityHigh = 4,
    NSOperationQueuePriorityVeryHigh = 8
};
@property NSOperationQueuePriority queuePriority;

// 在 operation 的 main task 完成时执行
@property (nullable, copy) void (^completionBlock)(void);

// 同步执行，阻塞当前线程直到任务完成
- (void)waitUntilFinished;

// 线程优先级，范围 0-1，默认 0.5，已经被废弃，使用 qualityOfService
@property double threadPriority;

// queuePriority 决定了任务执行的顺序
// qualityOfService 则决定了任务开始执后系统资源分配的优先级，例如 CPU 时间片, 网络资源, 硬盘资源等，默认 NSQualityOfServiceBackground
@property NSQualityOfService qualityOfService;

// 任务名称
@property (nullable, copy) NSString *name;

@end

```
## NSBlockOperation

NSBlockOperation 通过 Block 的形式管理多个任务并发执行。可以使用 init 创建然后通过 addExecutionBlock 添加多个 Block。也可以直接使用 blockOperationWithBlock: 创建并添加第一个 Block。它管理的 Block是、 可以通过数组 executionBlocks 获取。调用 Start 方法后数组中的所有的 Block 会并发执行，第一个 Block 在当前线程执行，其余会在新建的子线程执行。直到所有 Block 都执行完毕，NSBlockOperation 才算完成。

```
// 系统提供的封装了 Block 的 NSOperation 实体类，默认新建线程异步执行
@interface NSBlockOperation : NSOperation {

// 用 Block 初始化 NSBlockOperation
+ (instancetype)blockOperationWithBlock:(void (^)(void))block;

// 添加多个操作，默认新建线程异步执行
- (void)addExecutionBlock:(void (^)(void))block;

// 获取所有的 execution blocks
@property (readonly, copy) NSArray<void (^)(void)> *executionBlocks;

@end
```
## NSInvocationOperation

NSInvocationOperation 通过 Invocation 的形式管理单个任务的执行。只能通过 initWithTarget:selector:object:的形式在初始化的时候指定 Targrt-Action。它实现了非并发的 Operation。

```
// 系统提供的封装了 Invocation 的 NSOperation 实体类，默认新建线程异步执行
@interface NSInvocationOperation : NSOperation {

// 用 Action-Target 初始化 NSInvocationOperation
- (nullable instancetype)initWithTarget:(id)target selector:(SEL)sel object:(nullable id)arg;

// 用 NSInvocation 初始化 NSInvocationOperation，指定初始化方法
- (instancetype)initWithInvocation:(NSInvocation *)inv NS_DESIGNATED_INITIALIZER;

// 获取 invocation
@property (readonly, retain) NSInvocation *invocation;

// 获取运行结果。如果操作没有返回值、操作被取消或抛出了异常，访问此属性会抛出异常
@property (nullable, readonly, retain) id result;

@end

// 获取 NSInvocationOperation result 时操作被取消或没有返回值时抛出的异常名称
FOUNDATION_EXPORT NSExceptionName const NSInvocationOperationVoidResultException;
FOUNDATION_EXPORT NSExceptionName const NSInvocationOperationCancelledException;
```
## NSOperationQueue

```
// 操作的默认最大并发数，由 NSOperationQueue 基于当前系统状况动态确定
static const NSInteger NSOperationQueueDefaultMaxConcurrentOperationCount = -1;

// NSOperationQueue 通过队列管理多个 NSOperation 的执行
@interface NSOperationQueue : NSObject {

// 添加 Operation
- (void)addOperation:(NSOperation *)op;

// 添加多个 Operation，同步或异步执行
- (void)addOperations:(NSArray<NSOperation *> *)ops waitUntilFinished:(BOOL)wait;

// 通过 Block 添加 Operation
- (void)addOperationWithBlock:(void (^)(void))block;

// 获取所有 Operation
@property (readonly, copy) NSArray<__kindof NSOperation *> *operations;
// 所有 Operation 数量
@property (readonly) NSUInteger operationCount;

// 最大并行数，等于1时代表串行队列
@property NSInteger maxConcurrentOperationCount;

// 暂停/恢复队列任务派发，获取暂停状态
@property (getter=isSuspended) BOOL suspended;

// 队列名称
@property (nullable, copy) NSString *name;

// 队列中 Operation 默认的优先级，如果 Operation 有自己设置的优先级则使用自己的优先级
// 默认 Background，主队列为 UserInteractive 且不能改变
@property NSQualityOfService qualityOfService;

// 用 dispatch_queue_t 派发任务。
// 默认 nil 且不能设置为 dispatch_get_main_queue()。
// 设置时 operationCount 必须为0，否则会抛出异常。
// dispatch_queue_t 的 qualityOfService 会覆盖 OperationQueue 的设置。
@property (nullable, assign /* actually retain */) dispatch_queue_t underlyingQueue;

// 取消所有 Operation
- (void)cancelAllOperations;

// 同步执行，阻塞当前线程直到所有任务完成
- (void)waitUntilAllOperationsAreFinished;

// 获取当前队列
@property (class, readonly, strong, nullable) NSOperationQueue *currentQueue;
// 获取主队列
@property (class, readonly, strong) NSOperationQueue *mainQueue;

@end
```














参考:

[http://xuyafei.cn/post/draft/ios-thread]()

