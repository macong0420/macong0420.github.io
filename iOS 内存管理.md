
# iOS 内存管理


## iOS内存管理方案
 * 一.tagged pointer
    
> 苹果在 WWDC2013 《Session 404 Advances in Objective-C》中对Tagged Pointer的介绍：[]()https://developer.apple.com/videos/play/wwdc2013/404/

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604112333.png)

苹果主要是为了优化 64 位系统设备对 NSNumber, NSDate, NSString 等小对象的存储,
在引入tagged pointer技术之前,NSNumber 等对象是存储在堆上的,NSNumber 的指针存储的是堆中 NSNumber 对象的地址
在引入 Tagged Pointer 技术之后
NSNumber等对象的值直接存储在了指针中，不必在堆上为其分配内存，节省了很多内存开销。在性能上，有着 3 倍空间效率的提升以及 106 倍创建和销毁速度的提升。
NSNumber等对象的指针中存储的数据变成了Tag+Data形式（Tag为特殊标记，用于区分NSNumber、NSDate、NSString等对象类型；Data为对象的值）。这样使用一个NSNumber对象只需要 8 个字节指针内存。当指针的 8 个字节不够存储数据时，才会在将对象存储在堆上。

这个怎么验证呢?
![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604113237.png)
给定一个非常大的值 然后查看内存
![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604113332.png)
看到是有值的 说明 存储到了堆上 在堆上分配了地址,
然后我们修改一下 i 的值 给一个很小的值 然后再次查看内存
![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604113514.png)
可以看到 并没有分配堆内存
可见，使用了Tagged Pointer，NSNumber对象的值直接存储在了指针上，不会在堆上申请内存。则使用一个NSNumber对象只需要指针的 8 个字节内存就够了，**大大的节省了内存**占用
从效率上来看
为了使用一个NSNumber对象，需要在堆上为其分配内存，还要维护它的引用计数，管理它的生命周期，实在是影响执行效率。





 * 二、Non-pointer iSA--非指针型iSA
 * 三、SideTables，RefcountMap，weak_table_t

 
 
 
 
 参考地址
 []()https://juejin.cn/post/6844904132940136462