---
layout: post
title: Objective-C部分总结
tags: 
  - Objective-C
  - 基础
---

## 简述
这是一篇对OC一些比较基础的归纳。
[参考文章](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8A%EF%BC%89.md#%E4%BC%98%E5%8C%96%E9%83%A8%E5%88%86)
---
## 1.什么情况使用weak关键字，相比assign有什么不同？
使用weak关键字的情况：
1. 在ARC中，会出现循环引用的情况，这个时候就需要使用weak关键字来修饰，比如delegate属性.
2. 自身引用已经对该属性进行过一次强引用了，所以就不需要再使用强引用，如view/IBOutlet控件一般也是用weak
weak和assign的不同点：
1. weak此特质表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。 而assign的“设置方法”只会执行针对“纯量类型” (scalar type，例如 CGFloat 或 NSlnteger 等)的简单赋值操作。
2. assign可以用非oc对象，但weak必须用于oc对象。
---
## 2.怎么用copy关键字
1. NSString、NSArray、NSDictionary就一般使用copy关键字，因为他们都有对应的可变类型：
因为这几个类有对应的可变类型的，如果使用strong修饰，那么有可能在不知情的情况下改变值，所以这里需要用copy来修饰。
2. block一般也使用copy关键字：
block 使用 copy 是从 MRC 遗留下来的“传统”,在 MRC 中,方法内部的 block 是在栈区的,使用 copy 可以把它放到堆区.在 ARC 中写不写都行：对于 block 使用 copy 还是 strong 效果是一样的，但写上 copy 也无伤大雅，还能时刻提醒我们：编译器自动对 block 进行了 copy 操作。如果不写 copy ，该类的调用者有可能会忘记或者根本不知道“编译器会自动对 block 进行了 copy 操作”，他们有可能会在调用之前自行拷贝属性值。
---
## 3.如何让自己的类用 copy 修饰符？如何重写带 copy 关键字的 setter？
若想令自己所写的对象具有拷贝功能，则需实现 NSCopying 协议。如果自定义的对象分为可变版本与不可变版本，那么就要同时实现NSCopying与NSMutableCopying协议。
具体步骤：
1. 需声明该类遵从 NSCopying 协议
2. 实现 NSCopying 协议。该协议只有一个方法:
```objc
- (id)copyWithZone:(NSZone *)zone;
```
---
## 4.@property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的
@property的本质: *ivar+getter+setter*
“属性” (property)作为 Objective-C 的一项特性，主要的作用就在于封装对象中的数据。 Objective-C 对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过“存取方法”(access method)来访问。其中，“获取方法” (getter)用于读取变量值，而“设置方法” (setter)用于写入变量值。这个概念已经定型，并且经由“属性”这一特性而成为Objective-C 2.0的一部分。 而在正规的 Objective-C 编码风格中，存取方法有着严格的命名规范。 正因为有了这种严格的命名规范，所以 Objective-C 这门语言才能根据名称自动创建出存取方法。
*property*在runtime中的定义*objc_property_t*:
```c
typedef struct objc_property *objc_property_t;
```
而*objc_prperty*是一个结构体：
```objc
struct property_t {
const char *name;
const char *attributes;
};
```
*attributes*在本质上是*objc_property_attribute_t*：
```objc
/// Defines a property attribute
typedef struct {
    const char *name;           /**< The name of the attribute */
    const char *value;          /**< The value of the attribute (usually empty) */
} objc_property_attribute_t;
```
而*attributes*的具体内容是什么呢？其实，包括：类型，原子性，内存语义和对应的实例变量。
例如：我们定义一个*string*的property *@property (nonatomic, copy) NSString *string;*，通过*property_getAttributes(property)*获取到*attributes*并打印出来之后的结果为*T@“NSString”,C,N,V_string*
其中T就代表类型，可参阅 [Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1) ，C就代表Copy，N代表nonatomic，V就代表对于的实例变量。

添加了一个属性之后相关的代码会大致生成一下这些内容:
1. *OBJC_IVAR_$*类名*$*属性名称：该属性的“偏移量” (offset)，这个偏移量是“硬编码” (hardcode)，表示该变量距离存放对象的内存区域的起始地址有多远。
2. *setter* 与 *getter* 方法对应的实现函数
3. *ivar_list*：成员变量列表
4. *method_list*：方法列表
5. *prop_list*：属性列表
在每次新增加一个属性，系统都会在*ivar_list*中添加一个成员变量的描述，在*method_list*中添加setter和getter方法的描述，在*prop_list*添加一个属性的描述，然后计算出该属性在对象中的偏移量，然后给出setter和getter方法对应的实现，在setter方法中从偏移量的位置开始赋值，在getter方法中从偏移量开始取值，为了能够读取正确的字节数，系统对象偏移量的指针类型进行了类型强转。
---
## 5.runtime实现weak属性
weak属性特点:
weak 此特质表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同 assign 类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。

runtime是如何实现weak变量的自动置为nil?
runtime 对注册的类， 会进行布局，对于 weak 对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为 key，当此对象的引用计数为0的时候会 dealloc，假如 weak 指向的对象内存地址是a，那么就会以a为键， 在这个 weak 表中搜索，找到所有以a为键的 weak 对象，从而设置为 nil。
---
## 6.unrecognized selector异常
这个异常出现的原因很简单:
`当调用该对象的上的某个方法时，而该对象没有实现该方法，那么就会报这个错误。`
如何解决:
::objc是动态语言，每个方法在运行时都会被动态转为消息转发。objc_msgSend(receiver, selector)::ocjc在向一个对象发送消息时，runtime库会根据对象的ISA指针找到该对象实际所属的类，然后在该类的方法列表以及父类方法列表中寻找方法运行，如果在最顶层的父类方法列表中都没有找到相应的方法是，程序就会抛出异常。
而objc的运行时给出了三次的拯救机会，现在很多[防止崩溃的库](https://neyoufan.github.io/2017/01/13/ios/BayMax_HTSafetyGuard)都是使用这个机制来防止这个异常的出现：
1. Method resolution：
objc运行时会调用`+resloveInstanceMethod:`或者`+resolveClassMethod:`，让你有机会提供一个函数实现。如果你添加了函数，那么运行时系统将会重新启动一次消息发送过程，否则，将会移动下一步消息转发(Message Forwarding)。
2. Fast forwarding
如果目标对象实现了`-forwardingTargetForSelector:`，那么runtime在这个时候就会调用这个方法，给你把这个消息转发给其他对象的机会。只要返回的不是nil/self，那么整个消息转发机智就会被重启，而转发的对象就是返回的对象。否则将会继续进行下一步Normal Forwarding。
3. Normal forwarding
这是runtime给你挽救的最后一次机会。首先它会发送`-methodSignatureForSelector:`消息获得函数的参数和返回值类型。如果`-methodSignatureForSelector:`返回nil，Runtime则会发出`-doesNotRecognizeSelector:`消息，程序这时也就挂掉了。如果返回了一个函数签名，Runtime就会创建一个NSInvocation对象并发送`-forwardInvocation:`消息给目标对象。
```objc
// 调用顺序:
// 1.
+ (BOOL)resolveClassMethod:(SEL)sel;     // 这是class方法
+ (BOOL)resloveInstanceMethod:(SEL)sel; // 这是实例方法
// 2.
- (id)forwardingTargetForSelector:(SEL)aSelector; // 转发给别的class
// 3.
- (void)forwardInvocation:(NSInvocation *)anInvocation; // 生成一个invocation对象
```
---
## 7.一个obj对象如何进行内存布局？（考虑有父类的情况）
* 所以父类成员变量和自己的成员变量都会存放在该对象所对应的储存空间中。
* 每一个对象内部都有一个isa指针指向他的类对象，类对象中存放着本对象的：
1. 对象方法列表（对象能够接收的消息列表，保存在它对应的类对象中）
2. 成员变量的列表
3. 属性列表
它内部也有一个isa指针指向元对象(meta class),元对象内部存放的是类方法列表,类对象内部还有一个superclass的指针,指向他的父类对象。
每个 Objective-C 对象都有相同的结构，如下图所示：
        *Objective-C 对象的结构图*
                ISA指针
                根类的实例变量
                倒数第二层父类的实例变量
                …
                父类的实例变量
                类的实例变量
* 根对象就是NSObject，它的superclass指针指向nil
* 类对象既然称为对象，那它也是一个实例。类对象中也有一个isa指针指向它的元类(meta class)，即类对象是元类的实例。元类内部存放的是类方法列表，根元类的isa指针指向自己，superclass指针指向NSObject类。
![isa指针的示意图](https://raw.githubusercontent.com/HQL-yunyunyun/hql-yunyunyun.github.io/master/post_image/objective-c部分总结_isa指针示意图.png "isa指针的示意图")
---
## 8.objc内存销毁
1. 调用`-release:` ： 引用计数变为0
* 对象正在被销毁，生命周期即将结束。
* 不能再有新的 *__weak*弱引用，否则将指向nil。
* 调用 `[self dealloc]`。
2. 子类调用 `-dealloc`：
* 继承关系中最底层的子类调用`-dealloc`。
* 如果是*MRC*代码则会手动释放实例变量们(iVars)。
* 继承关系中每一层的父类都在调用`-dealloc`。
3. *NSObject*调用`-dealloc`：
* 只做一件事：调用*Objective-C runtime*中的`object_dispose()`方法。
4. 调用`object_dispose()`:
* 为*C++*的实体变量们*iVars*调用`desturctors`
* 为*ARC*状态下的实体变量们*iVars*调用`-release`
* 解除所有*__weak*引用
* 调用`free()`
- - - -
## 9._objc_msg_Forward做了什么操作？
对*objc-runtime-new.mm* 文件里与*_objc_msgForward*有关的三个函数使用伪代码展示下：
```c
id objc_msgSend(id self, SEL op, ...) {
    if (!self) return nil;
    IMP imp = class_getMethodImplementation(self->isa, SEL op);
    imp(self, op, ...); //调用这个函数，伪代码...
}

//查找IMP
IMP class_getMethodImplementation(Class cls, SEL sel) {
    if (!cls || !sel) return nil;
    IMP imp = lookUpImpOrNil(cls, sel);
    if (!imp) return _objc_msgForward; //_objc_msgForward 用于消息转发
    return imp;
}

IMP lookUpImpOrNil(Class cls, SEL sel) {
    if (!cls->initialize()) {
        _class_initialize(cls);
    }

    Class curClass = cls;
    IMP imp = nil;
    do { //先查缓存,缓存没有时重建,仍旧没有则向父类查询
        if (!curClass) break;
        if (!curClass->cache) fill_cache(cls, curClass);
        imp = cache_getImp(curClass, sel);
        if (imp) break;
    } while (curClass = curClass->superclass);

    return imp;
}
```
而根据第六个问题中的回答可以得知方法调用的一些步骤如下:
```c
/*
结合 《NSObject官方文档》]，排除掉 NSObject 做的事，剩下的就是_objc_msgForward消息转发做的几件事：
1. 调用resolveInstanceMethod:方法 (或resolveClassMethod:)。允许用户在此时为该 Class 动态添加实现。如果有实现了，则调用并返回YES，那么重新开始objc_msgSend流程。这一次对象会响应这个选择器，一般是因为它已经调用过class_addMethod。如果仍没实现，继续下面的动作。
2. 调用forwardingTargetForSelector:方法，尝试找到一个能响应该消息的对象。如果获取到，则直接把消息转发给它，返回非 nil 对象。否则返回 nil ，继续下面的动作。注意，这里不要返回 self ，否则会形成死循环。
3. 调用methodSignatureForSelector:方法，尝试获得一个方法签名。如果获取不到，则直接调用doesNotRecognizeSelector抛出异常。如果能获取，则返回非nil：创建一个 NSlnvocation 并传给forwardInvocation:。
4. 调用forwardInvocation:方法，将第3步获取到的方法签名包装成 Invocation 传入，如何处理就在这里面了，并返回非ni。
5. 调用doesNotRecognizeSelector:，默认的实现是抛出异常。如果第3步没能获得一个方法签名，执行该步骤。

上面前4个方法均是模板方法，开发者可以override，由 runtime 来调用。最常见的实现消息转发：就是重写方法3和4，吞掉一个消息或者代理给其他对象都是没问题的
也就是说_objc_msgForward在进行消息转发的过程中会涉及以下这几个方法：
1. resolveInstanceMethod:方法 (或resolveClassMethod:)。
2. forwardingTargetForSelector:方法
3. methodSignatureForSelector:方法
4. forwardInvocation:方法
5. doesNotRecognizeSelector:方法

正如前文所说：
_objc_msgForward是 IMP 类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，_objc_msgForward会尝试做消息转发。
*/
```
---
## 10.runloop和线程的关系
runloop和线程是紧密相连的，可以说runloop是为了线程而生的。Run loops是线程的基础架构部分， Cocoa 和 CoreFundation 都提供了 run loop 对象方便配置和管理线程的 run loop （以下都以 Cocoa 为例）。每个线程，包括程序的主线程（ main thread ）都有与之相应的 run loop 对象。
runloop和线程的关系:
1. 主线程的runloop是默认开启的。
在iOS程序中，程序启动后会有一个如下的函数
```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
`UIApplicationMain()`函数就是为*main thread*设置一个*NSRunloop*对象。
2. 对于其他线程来说，runloop是默认不开启的。
3. 在任何一个cocoa程序的线程中都可以通过以下代码来获得当前线程的runloop:`NSRunLoop *runloop = [NSRunLoop currentRunLoop];`
---
## 11.runloop详解
runloop有一个mode属性，这个属性的主要作用是指定事件在运行循环中的优先级：
* NSDefaultRunLoopMode（kCFRunLoopDefaultMode）：默认，空闲状态
* UITrackingRunLoopMode：ScrollView滑动时
* UIInitializationRunLoopMode：启动时
* NSRunLoopCommonModes（kCFRunLoopCommonModes）：Mode集合
苹果公开提供的 Mode 有两个：
* NSDefaultRunLoopMode（kCFRunLoopDefaultMode）
* NSRunLoopCommonModes（kCFRunLoopCommonModes）

而我们有时候在调用NSTimer时，在滑动页面的时候Timer会暂停回调也是因为runloop的mode属性的原因：
runloop只能运行在一种mode下，如果要换mode，当前的loop也需要停下重启成新的。利用这个机制，ScrollView滚动过程中NSDefaultRunLoopMode（kCFRunLoopDefaultMode）的mode会切换到UITrackingRunLoopMode来保证ScrollView的流畅滑动：只能在NSDefaultRunLoopMode模式下处理的事件会影响ScrollView的滑动。
如果我们把一个NSTimer对象以NSDefaultRunLoopMode（kCFRunLoopDefaultMode）添加到主运行循环中的时候, ScrollView滚动过程中会因为mode的切换，而导致NSTimer将不再被调度。
同时因为mode还是可定制的，所以：
Timer计时会被scrollView的滑动影响的问题可以通过将timer添加到NSRunLoopCommonModes（kCFRunLoopCommonModes）来解决。

一般来说，一个线程一次只能执行一个任务，执行完成后线程就会退出，这就是runloop要做的事情。如果我们需要一个机制，让线程能随时处理事件但并不退出，通常的代码逻辑 是这样的：
``` c
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
在objc中，对象通常是通过retainCount的机制来决定是否释放的，在每次runloop的时候，都会检查一遍对象的retainCount，如果为0则将对象释放。
---
## 12.不手动指定autoreleasepool的前提下，一个autorealese对象在什么时刻释放？（比如在一个vc的viewDidLoad中创建)
分两种情况:
1. 手动干预释放时机 — 指定*autoreleasepool*就是所谓的：当前作用域大括号结束时释放；
2. 系统自动去释放 — 不手动指定*autoreleasepool*。
*autorelease*对象出了作用域之后，会被添加到最近一次创建的自动释放池中，并会在当前的*runloop*迭代结束时释放。
释放时机可以总结为以下图示：
![释放时机](https://raw.githubusercontent.com/HQL-yunyunyun/hql-yunyunyun.github.io/master/post_image/objective-c部分总结_释放时机.png "释放时机"){: .center-image}

从程序启动到加载完成是一个完整的运行循环，然后会停下来，等待用户交互，用户的每一次交互都会启动一次循环，来处理用户所有的点击事件、触摸事件。
我们都知道：**所有autorelease的对象，在出了作用域之后，会被自动添加到最近创建的自动释放吃中**。
但是如果每次都放进应用程序的*main.m*中的*autoreleasepool*中，迟早会被撑满。在这个过程中必定有一个释放的动作，何时？
`在一次完整的运行循环结束之前，会被销毁。`
那什么时间会创建自动释放池？
`运行循环检测到事件并启动后，就会创建自动释放池。`
子线程的runloop默认是不工作的，无法主动创建，必须得手动创建。
自定义的*NSOperation*和*NSThread*需要手动创建自动释放池。比如： 自定义的*NSOperation*类中的*main*方法里就必须添加自动释放池。否则出了作用域后，自动释放对象会因为没有自动释放池去处理它，而造成内存泄露。
但对于*blockOperation*和*invocationOperation*这种默认的*Operation*，系统已经帮我们封装好了，不需要手动创建自动释放池。
*@autoreleasepool*当自动释放池被销毁或者耗尽时，会向自动释放池中的所有对象发送*release*消息，释放自动释放池中的所有对象。
如果在一个*vc*的*viewDidLoad*中创建一个*Autorelease*对象，那么该对象会在*viewDidAppear*方法执行前就被销毁了。
[黑幕背后的Autorelease · sunnyxx的技术博客](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
---
## 13.block的使用注意点
循环引用，就是一个对象a强引用了对象b，而b又强引用了对象a，这样就会造成循环引用，尤其在使用block的时候得注意这个问题。
如果对象a持有了一个block，而又在block中强引用了对象a(一般是调用self、或者是属性)，那么就会形成循环引用，为什么呢？
* 因为block都会对block内的对象进行捕获，并会强引用[iOS底层原理总结 - 探寻block的本质（一） - 掘金](https://juejin.im/post/5b0181e15188254270643e88#heading-12)。

block内如何修改block外部变量？为什么？
可以使用*__block* 去修饰外部变量。
我们都知道：*Block*不允许修改外部变量的值，这里所说的外部变量的值，指的是栈中指针的内存地址。*__block*所起到的作用就是只要观察到该变量被 *block* 所持有，就将“外部变量”在栈中的内存地址放到了堆中。进而在*block*内部也可以修改外部变量的值。

#代码/总结
