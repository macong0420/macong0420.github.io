# 关于Category的探究

分类（Category）是OC中的特有语法，它是表示一个指向分类的结构体的指针。原则上它只能增加方法，不能增加成员（实例）变量,**因为分类的结构体中根本就不存在实例变量列表,所以没办法增加实例变量,只能通过关联对象的方式增加**

* 分类是用于给原有类添加方法的,因为分类的结构体指针中，没有属性列表，只有方法列表。所以< 原则上讲它只能添加方法, 不能添加属性(成员变量),实际上可以通过其它方式添加属性>。
* 分类中的可以写@property, 但不会生成setter/getter方法, 也不会生成实现以及私有的成员变量（编译时会报警告）。
* 可以在分类中访问原有类中.h中的属性。
* 如果分类中有和原有类同名的方法, **会优先调用分类中的方法**, 就是说会忽略原有类的方法。所以同名方法调用的优先级为 `**分类 > 本类 > 父类**。因此在开发中尽量不要覆盖原有类。
* 如果多个分类中都有和原有类中同名的方法, 那么调用该方法的时候执行谁由编译器决定；编译器会执行最后一个参与编译的分类中的方法


> 关联对象增加实例变量

## Category方法中调用主类的同名原方法
  1. 先给出结论:在Category调用主类同名方法的时候,会直接调用Category中的方法,不会调用主类的,好像就是主类方法被覆盖了一样
  原因:
  其实主类方法并没有被覆盖, **只是Category的方法排在了方法列表最前面**,会优先调用,所以主类不调用,看起来效果像是被覆盖了一样
  2. 多个分类,多个同名方法的调用顺序
    当一个类拥有多个分类的时候,调用同名方法,谁最后编译,谁就会被调用,因为最后编译的那个Category,其方法被放在了方法列表最前面,(不管是实例方法,还是元类的类方法列表的最前面),当调用的时候,objc_msgSend会优先找到他.
    
    
## Runtime 对Category方法顺序的处理
先看一下Category在Runtime中的定义:


```
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
};

```    
可看出，struct category_t包含了Category的名字，所属的主类，实例方法列表，类方法列表，协议列表，实例属性列表。（Category无法添加实例变量，可用关联对象实现） 
其实，main()执行方法之前，dyld加载动态链接库及程序自己的可执行文件/mach-o文件。 
dyld的_dyld_start函数开始，接着调用libdispatch.dylib的_os_object_init函数，接着调用libobjc的_objc_init函数。此时，将控制权交给libobjc。 


### Category对实例方法和类方法列表的调整
Category实际上会对实例方法或者类方法列表进行调整
在main()函数执行之前,dyld加载动态链接库和程序逐级的可执行文件/mach-o文件,在_read_iamges的时候,对调用 remethodizeClass(cls)方法,这个就是对方法列表进行重新排列

![](https://ww1.sinaimg.cn/large/006tNc79ly1g5wwq6iv35j31380titp8.jpg)



第一个remethodizeClass(cls)是对实例方法的方法列表重整。 
第二个remethodizeClass(cls->ISA())传元类，是对类方法的方法列表的重整。




参考:

[https://blog.csdn.net/WOTors/article/details/52576433]()

[https://juejin.im/post/5b6a63f4e51d4534b93f770f]()

