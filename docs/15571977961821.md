# 我们说的Objective-C是动态运行时语言是什么意思？

**Objective-C 可以通过Runtime 这个运行时机制，在运行时动态的添加变量、方法、类等，所以说Objective-C 是一门动态的语言**

**1.动态类型：如id类型。实际上静态类型因为其固定性和可预知性而使用的特别广泛。静态类型是强类型，动态类型是弱类型，运行时决定接收者。
2.动态绑定：让代码在运行时判断需要调用什么方法，而不是在编译时。与其他面向对象语言一样，方法调用和代码并没有在编译时连接在一起，而是在消息发送时才进行连接。运行时决定调用哪个方法。
3.动态载入。让程序在运行时添加代码模块以及其他资源。用户可以根据需要执行一些可执行代码和资源，而不是在启动时就加载所有组件。可执行代码中可以含有和程序运行时整合的新类。**

什么叫动态静态
静态、动态是相对的，这里动态语言指的是不需要在编译时确定所有的东西，在运行时还可以动态的添加变量、方法和类

其他说法
Objective-C 是C 的超集，在C 语言的基础上添加了面向对象特性，并且利用Runtime 这个运行时机制，为Objective-C 增添了动态的特性。
Objective-C 使用的是 “消息结构” 并非 “函数调用”：使用消息结构的的语言，其运行时所应执行的代码由运行期决定；而使用函数调用的语言，则由编译器决定

（1）动态类型语言：动态类型语言是指在运行期间才去做数据类型检查的语言，也就是说，在用动态类型的语言编程时，永远也不用给任何变量指定数据类型，该语言会在你第一次赋值给变量时，在内部将数据类型记录下来。Python和Ruby就是一种典型的动态类型语言，其他的各种脚本语言如VBScript也多少属于动态类型语言。

（2）静态类型语言：静态类型语言与动态类型语言刚好相反，它的数据类型是在编译其间检查的，也就是说在写程序时要声明所有变量的数据类型，C/C++是静态类型语言的典型代表，其他的静态类型语言还有C#、JAVA等。

因为在运行期可以继续向类中添加方法，所以编译器在编译时还无法确定类中是否有某个方法的实现。对于类无法处理一个消息就会触发消息转发机制

消息转发分为两大阶段：

“动态方法解析”：先征询接收者，所属的类，能否动态添加方法，来处理这个消息，若可以则结束，如不能则继续往下走 
“完整的消息转发机制”： 
请接收者看看有没其他对象能处理这条消息，若有，就把这个消息转发给那个对象然后结束 
运行时系统会把与消息有关细节全部封装到NSInvocation 对象中，再给对象最后一次机会，令其设法解决当前还未处理的这条消息

Objective-c的动态性
Objective-C的动态性，让程序在运行时判断其该有的行为，而不是像c等静态语言在编译构建时就确定下来。它的动态性主要体现在3个方面：

1.动态类型：如id类型。实际上静态类型因为其固定性和可预知性而使用的特别广泛。静态类型是强类型，动态类型是弱类型，运行时决定接收者。
2.动态绑定：让代码在运行时判断需要调用什么方法，而不是在编译时。与其他面向对象语言一样，方法调用和代码并没有在编译时连接在一起，而是在消息发送时才进行连接。运行时决定调用哪个方法。
3.动态载入。让程序在运行时添加代码模块以及其他资源。用户可以根据需要执行一些可执行代码和资源，而不是在启动时就加载所有组件。可执行代码中可以含有和程序运行时整合的新类。

**分两个方面：动态类型(dynamic type)和动态绑定(dynamic binding)**

动态类型
动态类型指的是对象指针类型的动态性,意思就是对象的类型确定将会推迟到运行时。由赋值给它的对象类型决定对象指针的类型。

另外类型确定推迟到运行时之后，可以通过NSObject的isKindOfClass方法动态判断对象最后的类型（动态类型识别）。可以把id修饰的对象理解为动态类型对象，其他在编译器指明类型的为静态类型对象，通常如果不需要涉及到多态的话还是要尽量使用静态类型。因为出错了的话编译器可以提前查出，可读性也好。
举例说明：

`NSString* testObject = [[NSData alloc] init];`

这句话声明了一个NSString指针，但是赋值了一个NSData对象。所以在编译期下面两句话：

```
[testObject stringByAppendingString:@"string"];  //1
[testObject base64EncodedDataWithOptions:NSDataBase64Encoding64CharacterLineLength];
```
 //2
第一句话在编译期可以通过，因为编译期testObject的类型是NSString,所以第二句话会报错，因为那是NSData的方法导致程序跑不起来。
然后如果注释掉第二句话将程序run起来，程序将会在第一句话处crash，因为赋值给testObject的对象是一个NSData对象，而NSData没有stringByAppendingString方法，导致crash。

如果将testObject声明为id类型则两句话都可以编译通过。但是运行时一样会crash在stringByAppendingString，原因同上。

摘抄一段官方文档的说明：

> A variable is dynamically typed when the type of the object it points to is not 
> checked at compile time. Objective-C uses the id data type to represent a variable 
> that is an object without specifying what sort of object it is. This is referred 
> to as dynamic typing.

> Dynamic typing contrasts with static typing, in which the system explicitly 
> identifies the class to which an object belongs at compile time. Static type 
> checking at compile time may ensure stricter data integrity, but in exchange for 
> that integrity, dynamic typing gives your program much greater flexibility. And 
> through object introspection (for example, asking a dynamically typed, anonymous 
> object what its class is), you can still verify the type of an object at runtime
> and thus validate its suitability for a particular operation.
静态类型可以严格的保证数据完整性，动态类型牺牲了这种完整性，但是给予了更多的灵活性，例如id，而且通过对象的introspection特性，开发者依然可以在运行时验证一个对象的类型。

introspection：
1.首先是Class类型：
• Class class = [NSObject class]; // 通过类名得到对应的Class动态类型
• Class class = [obj class]; // 通过实例对象得到对应的Class动态类型
• if([obj1 class] == [obj2 class]) // 判断是不是相同类型的实例
2.Class动态类型和类名字符串的相互转换：
• NSClassFromString(@”NSObject”); // 由类名字符串得到Class动态类型
• NSStringFromClass([NSObject class]); // 由类名的动态类型得到类名字符串
• NSStringFromClass([obj class]); // 由对象的动态类型得到类名字符串
3.判断对象是否属于某种动态类型：
• -(BOOL)isKindOfClass:class // 判断某个对象是否是动态类型class的实例或其子类的实例
• -(BOOL)isMemberOfClass:class // 与isKindOfClass不同的是，这里只判断某个对象是否是class类型的实例，不放宽到其子类
4.判断类中是否有对应的方法：
• -(BOOL)respondsTosSelector:(SEL)selector // 类中是否有这个类方法
• -(BOOL)instancesRespondToSelector:(SEL)selector // 类中是否有这个实例方法
上面两个方法都可以通过类名调用，前者判断类中是否有对应的类方法(通过‘+’修饰定义的方法)，后者判断类中是否有对应的实例方法(通过‘-’修饰定义的方法)。此外，前者respondsTosSelector函数还可以被类的实例对象调用，效果等同于直接用类名调用后者instancesRespondToSelector函数。 例如：


```
[1][Test instancesRespondToSelector:@selector(objFunc)];//YES
[2][Test instancesRespondToSelector:@selector(classFunc)];//NO
[3][Test respondsToSelector:@selector(objFunc)];//NO
[4][Test respondsToSelector:@selector(classFunc)];//YES
[5][test respondsToSelector:@selector(objFunc)];//YES
[6][test respondsToSelector:@selector(classFunc)];//NO
```
5.方法名字符串和SEL类型的转换
编译器会根据方法的名字和参数序列生成唯一标识改方法的ID,这个ID为SEL类型。到了运行时编译器通过SEL类型的ID来查找对应的方法，方法的名字和参数序列相同,那么它们的ID就都是相同的。另外，可以通过@select()指示符获得方法的ID。常用的方法如下：


```
SEL funcID = @select(func)；// 这个注册事件回调时常用，将方法转成SEL类型
SEL funcID = NSSelectorFromString(@"func"); // 根据方法名得到方法标识
NSString *funcName = NSStringFromSelector(funcID); // 根据SEL类型得到方法名字符串
```
动态绑定
动态绑定指的是方法的动态性，先来看官方定义

> Dynamic binding is determining the method to invoke at runtime instead of at 
> compile time. Dynamic binding is also referred to as late binding. In Objective-C, 
> all methods are resolved dynamically at runtime. The exact code executed is 
> determined by both the method name (the selector) and the receiving object.

> Dynamic binding enables polymorphism. For example, consider a collection of
>  objects including Dog, Athlete, and ComputerSimulation. Each object has its own 
>  implementation of a run method. In the following code fragment, the actual code 
>  that should be executed by the expression [anObject run] is determined at 
>  runtime. The runtime system uses the selector for the method run to identify the
>   appropriate method in whatever the class of anObject turns out to be.
动态绑定的意思就是将方法的实现推迟到运行时再决定而不是编译期。在oc中，所有的方法都是运行时决定的，具体需要执行的代码需要通过方法名（selector）和接收者一起决定。

动态绑定实现了面向对象中多态这个特性。举例上面英文说的很详细了就不翻译了…代码如下


```
NSArray *anArray = [NSArray arrayWithObjects:aDog, anAthlete, aComputerSimulation, nil];
id anObject = [anArray objectAtIndex:(random()/pow(2, 31)*3)];
[anObject run];
```
动态绑定是基于动态类型的，在运行时对象的类型确定后，那么对象的属性和方法也就确定了(包括类中原来的属性和方法以及运行时动态新加入的属性和方法)，这也就是所谓的动态绑定了。动态绑定的核心用法就该是在运行时动态的为类添加属性和方法，以及方法的最后处理或转发，主要用到C语言语法，因为涉及到运行时，因此要引入运行时头文件：objc/runtime.h。
消息转发可以参考effective oc 12条。

