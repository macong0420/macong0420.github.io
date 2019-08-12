# 关于Category的探究
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

