
# iOS 内存管理


## iOS内存管理方案
###   一.tagged pointer
    
> 苹果在 WWDC2013 《Session 404 Advances in Objective-C》中对Tagged Pointer的介绍：[]()https://developer.apple.com/videos/play/wwdc2013/404/

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604112333.png)

苹果主要是为了优化 64 位系统设备对 NSNumber, NSDate, NSString 等小对象的存储,
在引入tagged pointer技术之前,NSNumber 等对象是存储在堆上的,NSNumber 的指针存储的是堆中 NSNumber 对象的地址
![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604122630.png)

在引入 Tagged Pointer 技术之后
NSNumber等对象的值直接存储在了指针中，不必在堆上为其分配内存，节省了很多内存开销。在性能上，有着 3 倍空间效率的提升以及 106 倍创建和销毁速度的提升。

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604122714.png)

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
    
#### Tagged Pointer 的原理
设置环境变量 OBJC_DISABLE_TAG_OBFUSCATION 为 YES， 为关闭 Tagged Pointer 的数据混淆；
    如图:

    ![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604142546.png)
   
 
举例说明 :

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604142731.png)

我们设置环境变量 OBJC_DISABLE_TAG_OBFUSCATION 为 YES，关闭了数据混淆可以看出：number1 的内存为 0xb000000000000012、number2 的内存为 0xb000000000000022、number3 的内存为 0xb000000000000032。并且 number1 的值为 1、number2 的值为 2、number3 的值为 3。



通过观察发现，对象的值 1、2、3 都存储在了对应的指针中，对应 0xb000000000000012 中的 1、0xb000000000000022 中的 2、0xb000000000000032 中的 3。（混淆为苹果对于数据的保护）而 numberFFFF 的值 0xFFFFFFFFFFFFFFFF，由于数据过大，导致无法 1 个指针 8 个字节的内存根本存不下，而申请了堆内存。


我们都知道所有的 oc 对象都有 isa 指针，那么判断一个指针是否是伪指针最重要的证据是其 isa 指针了，我们看下他们对应的 isa 指针，如下图：![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604142945.png)

由上图我们可以看出，number1、number2、number3 指针为 Tagged Pointer 类型，为伪指针，isa 指针为 nil。numberFFFF 的 isa 指针真实存在，在堆内存中分配了空间，不是 Tagged Pointer 类型。

**Tagged Pointer 为 Tag+Data 形式，其中 Data 为内存地址中的 1、2、3 （红色）**，为存储对应着对象的值。（例：0xb000000000000012 中的 1）

以上面例子中的 0xb000000000000012 为例，指针中的 b 代表什么？



b 的二进制为 1011，其中第一位 1 是 Tagged Pointer 标识位，代表这个指针是 Tagged Pointer；后面的 011 是类标识位，对应十进制为 3，表示 NSNumber 类。



指针中的 2 代表什么？



2 代表数据类型（NSNumber 为 short、 int、 long 、 float 、 double 等。NSString 为 string 长度）。



以 iOS 中 NSNumber 为例，我们看下图按照位域操作，Tag 和 Data 分别显示在什么位置、代表什么。


但是内存地址： 0xb000000000000012 中对应的“b” 和 “2”，代表什么？

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604143144.png)

Tagged Pointer 的 Tag 标记，为最高 4 位。其余为 NSNumber 数据。下面会分别对标识位、类标识、数据类型做代码验证。

#####  Tagged Pointer 标识位
如何判断为 Tagged Pointer?



在源码 objc_internal.h 中可以找到判断 Tagged Pointer 标识位的方法，如下代码：

```
static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}


```
将一个指针与 _OBJC_TAG_MASK 掩码 进行按位与操作。这个掩码_OBJC_TAG_MASK 的源码同样在 objc_internal.h 中可以找到：


```
#if (TARGET_OS_OSX || TARGET_OS_IOSMAC) && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1
#endif

#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1UL<<63)
#else
#   define _OBJC_TAG_MASK 1UL
#endif

```
根据源码得知：



MacOS 下采用 LSB（Least Significant Bit，即最低有效位）为 Tagged Pointer 标识位；（define _OBJC_TAG_MASK 1UL）



iOS 下则采用 MSB（Most Significant Bit，即最高有效位）为 Tagged Pointer 标识位。（define _OBJC_TAG_MASK (1UL<<63)）< span="">

如下图，以 NSNumber 为例：

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604143653.png)

在 iOS 中，1 个指针 8 个字节，64 位，最高位为 1，则为 Tagged Pointer。



同理在上面 4.3.1 Tag 解析结果一节中，以 0xb000000000000012 为例：



0xb000000000000012 为 16 进制指针中的最高位 b 的二进制为 1011，最高位为 1，则代表这个指针是 Tagged Pointer。



且_objc_isTaggedPointer 判断 Tagged Pointer 标识位是处处优先判断的。如下面源码（下面源码只展示相关部分）所示：

```
ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    if (isTaggedPointer()) return (id)this;
}

ALWAYS_INLINE bool 
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    if (isTaggedPointer()) return false;
}

inline bool 
objc_object::isTaggedPointer() 
{
    return _objc_isTaggedPointer(this);
}

```

在源码 objc_object.h 中可以找到的 objc_object::rootRetain 方法，该方法为引用计数+1 的方法，在这个方法中，优先判断是否是 Tagged Pointer，Tagged Pointer 为伪指针，不需要记录引用计数。

在源码 objc_object.h 中可以找到的 objc_object::rootRelease 方法，该方法为引用计数-1 的方法，在这个方法中，优先判断是否是 Tagged Pointer，Tagged Pointer 为伪指针，不需要记录引用计数。

objc_msgSend 为汇编代码，但其实里面也优先做了 Tagged Pointer 标识位判断。如果不是 Tagged Pointer 则进行消息转发等流程。

Tagged Pointer 的判断是如此的简单，只是二进制的与运算。

从苹果官方介绍来看， Tagged Pointer 被设计的目的是用来存储较小的对象，例如 NSNumber、NSDate、NSString 等；那么 Tagged Pointer 只是一个伪指针，一个 64 位的二进制，如何来区分是 NSNumber 呢？还是 NSString 等呢？



在源码 objc_internal.h 中可以查看到 NSNumber、NSDate、NSString 等类的标识位


```
{
    // 60-bit payloads
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,
    // 保留位
    OBJC_TAG_RESERVED_7        = 7,
    。。。
}

```

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604145451.png)


根据输出我们可以看到：



NSNumber 指针 0xb000000000000012，b 的二进制为 1011，后面的 011 是类标识位，对应十进制为 3，表示 NSNumber 类；



NSString 指针 0xa000000000000611， a 的二进制为 1010，后面的 010 是类标识位，对应十进制为 2，表示 NSString 类。



如图，类标识位置如下：

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604145542.png)

我们知道了以 NSNumber 为例的地址 0xb000000000000012 的数据数值、Tagged Pointer 标识位、Tagged Pointer 类标识。那么最后一位 2 代表的是什么呢？



16 进制的最后一位（即 2 进制的最后四位）表示数据类型。同样我们举例验证：

```
char a = 1;
short b = 1;
int c = 1;
long d = 1;
float e = 1.0;
double f = 1.00;

NSNumber *number1 = @(a);   // 0xb000000000000010
NSNumber *number2 = @(b);   // 0xb000000000000011
NSNumber *number3 = @(c);   // 0xb000000000000012
NSNumber *number4 = @(d);   // 0xb000000000000013
NSNumber *number5 = @(e);   // 0xb000000000000014
NSNumber *number6 = @(f);   // 0xb000000000000015
```
可以看到，我们都用 NSNumber 类，用不同数据类型做测试，内存地址 16 进制只有最后一位发生了变化。其对应的数据类型分别为：

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604145642.png)


在源码 objc-runtime-new.mm 中有一段注释对 Tagged pointer objects 进行了解释，原文如下：

```
/***********************************************************************
* Tagged pointer objects.
*
* Tagged pointer objects store the class and the object value in the
* object pointer; the "pointer" does not actually point to anything.
*
* Tagged pointer objects currently use this representation:
* (LSB)
*  1 bit   set if tagged, clear if ordinary object pointer
*  3 bits  tag index
* 60 bits  payload
* (MSB)
* The tag index defines the object's class.
* The payload format is defined by the object's class.
*
* If the tag index is 0b111, the tagged pointer object uses an
* "extended" representation, allowing more classes but with smaller payloads:
* (LSB)
*  1 bit   set if tagged, clear if ordinary object pointer
*  3 bits  0b111
*  8 bits  extended tag index
* 52 bits  payload
* (MSB)
*
* Some architectures reverse the MSB and LSB in these representations.
*
* This representation is subject to change. Representation-agnostic SPI is:
* objc-internal.h for class implementers.
* objc-gdb.h for debuggers.
**********************************************************************/


```


对应注释翻译：



Tagged pointer 指针对象将 class 和对象数据存储在对象指针中；指针实际上不指向任何东西。

Tagged pointer 当前使用此表示形式：

(LSB)(macOS)64 位分布如下：

1 bit 标记是 Tagged Pointer

3 bits 标记类型

60 bits 负载数据容量，（存储对象数据）

(MSB)(iOS)64 位分布如下：

tag index 表示对象所属的 class

负载格式由对象的 class 定义

如果 tag index 是 0b111(7)， tagged pointer 对象使用 “扩展” 表示形式

允许更多类，但 有效载荷 更小

(LSB)(macOS)(带有扩展内容)64 位分布如下：

1 bit 标记是 Tagged Pointer

3 bits 是 0b111

8 bits 扩展标记格式

52 bits 负载数据容量，（存储对象数据）

在这些表示中，某些体系结构反转了 MSB 和 LSB。



从注释中我们得知：



Tagged pointer 存储对象数据目前 分为 60bits 负载容量和 52bits 负载容量。

类标识允许使用扩展形式。



那么如何判断负载容量？类标识的扩展类型有那些？我们来看下全面的 objc_tag_index_t 源码：

```
// objc_tag_index_t
{
    // 60-bit payloads
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,
    // 保留位
    OBJC_TAG_RESERVED_7        = 7,
    // 52-bit payloads
    OBJC_TAG_Photos_1          = 8,
    OBJC_TAG_Photos_2          = 9,
    OBJC_TAG_Photos_3          = 10,
    OBJC_TAG_Photos_4          = 11,
    OBJC_TAG_XPC_1             = 12,
    OBJC_TAG_XPC_2             = 13,
    OBJC_TAG_XPC_3             = 14,
    OBJC_TAG_XPC_4             = 15,
    OBJC_TAG_NSColor           = 16,
    OBJC_TAG_UIColor           = 17,
    OBJC_TAG_CGColor           = 18,
    OBJC_TAG_NSIndexSet        = 19,
    // 前60位负载内容
    OBJC_TAG_First60BitPayload = 0, 
    // 后60位负载内容
    OBJC_TAG_Last60BitPayload  = 6, 
    // 前52位负载内容
    OBJC_TAG_First52BitPayload = 8, 
    // 后52位负载内容
    OBJC_TAG_Last52BitPayload  = 263, 
    // 保留位
    OBJC_TAG_RESERVED_264      = 264
}

```


##### NSString
接下来我们来分析一下Tagged Pointer在NSString中的应用。同NSNumber一样，在64 bit的MacOS下，如果一个NSString对象指针为Tagged Pointer，那么它的后 4 位（0-3）作为标识位，第 4-7 位表示字符串长度，剩余的 56 位就可以用来存储字符串。
示例：

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604151830.png)

从打印结果来看，有三种NSString类型：

| 类型 | 描述 |
| --- | --- |
| __NSCFConstantString | 1. 常量字符串，存储在字符串常量区，继承于 __NSCFString。相同内容的 __NSCFConstantString 对象的地址相同，也就是说常量字符串对象是一种单例，可以通过 == 判断字符串内容是否相同。2. 这种对象一般通过字面值@"..."创建。如果使用 __NSCFConstantString 来初始化一个字符串，那么这个字符串也是相同的 __NSCFConstantString。 |
| __NSCFString | 1. 存储在堆区，需要维护其引用计数，继承于 NSMutableString。 2. 通过stringWithFormat:等方法创建的NSString对象（且字符串值过大无法使用Tagged Pointer存储）一般都是这种类型。 |
| NSTaggedPointerString | Tagged Pointer，字符串的值直接存储在了指针上。 |


打印结果分析：


| NSString 对象 |  类型 | 分析  |
| --- | --- | --- |
|a  |  __NSCFConstantString| 通过字面量@"..."创建 |
| b | __NSCFString | a 的深拷贝，指向不同的内存地址，被拷贝到堆区 |
|  c|__NSCFConstantString| a 的浅拷贝，指向同一块内存地址 |
| d | NSTaggedPointerString |单独对 a 进行 copy（如 c），浅拷贝是指向同一块内存地址，所以不会产生Tagged Pointer；单独对 a 进行 mutableCopy（如 b），复制出来是可变对象，内容大小可以扩展；而Tagged Pointer存储的内容大小有限，因此无法满足可变对象的存储要求。 |
| e | __NSCFConstantString | 使用 __NSCFConstantString 来初始化的字符串 |
| f |NSTaggedPointerString	  | 通过stringWithFormat:方法创建，指针足够存储字符串的值。 |
|string1  |NSTaggedPointerString  |  通过stringWithFormat:方法创建，指针足够存储字符串的值。|
| string2 |NSTaggedPointerString | 通过stringWithFormat:方法创建，指针足够存储字符串的值。 |
| string3 | __NSCFString | 通过stringWithFormat:方法创建，指针不足够存储字符串的值。 |


可以看到，为Tagged Pointer的有d、f、string1、string2指针。它们的指针值分别为
0x6115、0x6615 、0x6766656463626175、0x880e28045a54195。
其中0x61、0x66、0x67666564636261分别对应字符串的 ASCII 码。
最后一位5的二进制为0101，最后一位1是代表这个指针是Tagged Pointer，010对应十进制为2，表示NSString类。
倒数第二位1、1、7、9代表字符串长度。
对于string2的指针值0x880e28045a54195，虽然从指针中看不出来字符串的值，但其也是一个Tagged Pointer。
下图是MacOS下NSString的Tagged Pointer位视图：

![](https://raw.githubusercontent.com/macong0420/Picture/master/img/20210604153313.png)

小结：



区分什么位置为负载内容位



MacOS 下采用 LSB 即 OBJC_TAG_First60BitPayload、OBJC_TAG_First52BitPayload。



iOS 下则采用 MSB 即 OBJC_TAG_Last60BitPayload、OBJC_TAG_Last52BitPayload。



区分负载数据容量



当类标识为 0-6 时，负载数据容量为 60bits。



当类标识为 7 时(对应二进制为 0b111)，负载数据容量为 52bits。



类标识的扩展类型有哪些？



如果 tag index 是 0b111(7)， tagged pointer 对象使用 “扩展” 表示形式



类标识的扩展类型为上面 OBJC_TAG_Photos_1 ～OBJC_TAG_NSIndexSet。



类标识与负载数据容量对应关系



当类标识为 0-6 时，负载数据容量为 60bits。即 OBJC_TAG_First60BitPayload 和 OBJC_TAG_Last60BitPayload，负载数据容量 的取值区间也为 0 - 6。



当类标识为 7 时，负载数据容量为 52bits。即 OBJC_TAG_First52BitPayload 和 OBJC_TAG_Last52BitPayload，负载数据容量的取值区间为 8 - 263。



你品，你细品这里。只要一个 tag，既可以区分负载数据容量，也可以区分类标识，就是这么滴强大～


 * 二、Non-pointer iSA--非指针型iSA
 * 三、SideTables，RefcountMap，weak_table_t

 
 
 
 
 参考地址
 []()https://juejin.cn/post/6844904132940136462
 []()https://segmentfault.com/a/1190000021499221
 []()https://www.infoq.cn/article/r5s0budukwyndafrivh4