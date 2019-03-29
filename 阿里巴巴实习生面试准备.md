## Android线程
### 为什么要同步
[请看我朋友薛8一篇文章](https://juejin.im/post/5c8610056fb9a049b222b366)
### Android中使用多线程的方法
* 裸new一个Thread(失控线程,不推荐)
* RxJava的调度器(io(优先级比低),密集计算线程(优先级比高,用于执行密集计算任务),安卓主线程,从Looper创建(实际上内部也是创建了Handler))
* Java Executor框架的Executors#newCachedThreadPool(),不会造成资源浪费,60秒没有被使用的线程会被释放
* AsyncTask,内部使用FutureTask实现,通过Handler将结果转发到主线程,默认的Executor是共用的,如果同时执行多个AsyncTask,就可能需要排队,但是可以手动指定Executor解决这个问题,直接new匿名内部类会保存外部类的引用,可能会导致内存泄漏
* Android线程模型提供的Handler和HandlerThread

### 同步机制

* Android线程模型就是消息循环,Looper关联MessageQueue,不断尝试从MessageQueue取出Message来消费,这个过程可能会被阻塞.而Handler最终都调用enqueueMessage(Message,when)入队的,延迟的实现时当前是时间加上延迟时间给消息指定一个执行的时间点,然后在MessageQueue找到插入位置,此时会判断是否需要唤醒线程来消费消息以及更新下次需要暂停的时间
* 信号量要与一个锁结合使用,当前线程要先获得这个锁,然后等待与这个锁相关联的信号量,此时该锁会被解锁,其他线程可以抢到这个锁,如果其他线程抢到了这个锁,那他可以通知这个信号量,然后释放该锁,如果此时第一个线程抢到了该锁,那么它将从等待处继续执行(应用场景,将异步回调操作封装为变为同步操作,避免回调地狱)
* 信号量与锁相比的应用场景不同,锁是服务于共享资源的,而信号量是服务于多个线程间的执行的逻辑顺序的,锁的效率更高一些.

### 其他知识点
* 线程可以设置优先级,优先级越高的越有可能得到CPU
* ThreadGroup,线程组表示一个线程的集合。此外，线程组也可以包含其他线程组。线程组构成一棵树，在树中，除了初始线程组外，每个线程组都有一个父线程组。允许线程访问有关自己的线程组的信息，但是不允许它访问有关其线程组的父线程组或其他任何线程组的信息。
* setUncaughtExceptionHandler,设置该线程由于未捕获到异常而突然终止时调用的处理程序. 
* 不要直接new一个Handler的匿名内部类,会保存外部类的引用,容易内存泄漏(未完成任务的子线程持有Handler,可能间接持有Activity,AsyncTask同理)

## Android进程
### 为什么要做进程间通信?
因为Android没有虚拟内存(磁盘内存),但是内存不够了怎么办?所以就有了OOM Killer.Android对每一个进程可以拥有的Java内存是有限制的,只要超过了就会被杀死(使用JNI的Native堆可以扩大该限制)
### 写时拷贝
fork的进程使用了写时拷贝技术,只有进程空间的内容发生了变化(写入),才将父进程的内容拷贝一份给子进程.

### IPC机制
Android是Linux,Linux有的IPC机制他都有,如信号,Socket,共享内存等
#### Binder
Binder是C/S架构,有效率优势,Socket/管道/消息队列拷贝内存次数都是2次,Binder是1次,共享内存虽然是0次,但是有太底层,不面向对象,不安全等缺点.Android APP是通过Binder作为Client来访问各种系统服务
#### aidl
Android接口定义语言
#### 文件
Windows读写文件会占用该文件,但Android是Linux所以默认无锁,可以用文件做进程间通信,但是多进程下读写不可靠,且受I/O速度限制
#### 管道
在JNI层
#### Handler的Messager
## Android View
### Fragment
[请看我自己总结的私货](https://juejin.im/post/5c921a4d5188252d92095f09)
### View绘制流程

### View点击事件传递

## Android JVM
### 方法数问题
4.4.2及以下使用Google多Dex方案解决,ART虚拟机会将导出一个oat(安装时处理),方法数是因为使用了short作为方法索引的原因,所以不能超过short的长度,也就是65k问题
### GC算法
#### 标记-清除算法：

暂停除了GC线程以外的所有线程,算法分为“标记”和“清除”两个阶段,首先从GC Root开始标记出所有需要回收的对象，在标记完成之后统一回收掉所有被标记的对象。

GC Roots有这些:

* 通过System Class Loader或者Boot Class 
* Loader加载的class对象，通过自定义类加载器加载的class不一定是GC Root
* 处于激活状态的线程
* 栈中的对象
* JNI栈中的对象
* JNI中的全局对象
* 正在被用于同步的各种锁对象
* JVM自身持有的对象，比如系统类加载器等。

缺点:

标记-清除算法的缺点有两个：首先，效率问题，标记和清除效率都不高。其次，标记清除之后会产生大量的不连续的内存碎片，空间碎片太多会导致当程序需要为较大对象分配内存时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

#### 复制算法：

将可用内存按容量分成大小相等的两块，每次只使用其中一块，当这块内存使用完了，就将还存活的对象复制到另一块内存上去，然后把使用过的内存空间一次清理掉。

优点:

这样使得每次都是对其中一块内存进行回收，内存分配时不用考虑内存碎片等复杂情况，只需要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。

缺点:

复制算法的缺点显而易见，可使用的内存降为原来一半。

#### 标记-整理算法：

标记-整理算法在标记-清除算法基础上做了改进，标记阶段是相同的,标记出所有需要回收的对象，在标记完成之后不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，在移动过程中清理掉可回收的对象，这个过程叫做整理。

优点:

标记-整理算法相比标记-清除算法的优点是内存被整理以后不会产生大量不连续内存碎片问题。复制算法在对象存活率高的情况下就要执行较多的复制操作，效率将会变低，而在对象存活率高的情况下使用标记-整理算法效率会大大提高。

#### 分代收集算法：

是java的虚拟机的垃圾回收算法.基于编程中的一个事实,越新的对象的生存期越短,根据内存中对象的存活周期不同，将内存划分为几块，java的虚拟机中一般把内存划分为新生代和年老代，当新创建对象时一般在新生代中分配内存空间，当新生代垃圾收集器回收几次之后仍然存活的对象会被移动到年老代内存中，当大对象在新生代中无法找到足够的连续内存时也直接在年老代中创建。

### ClassLoader
#### 双亲委派机制
简单来说就是先把加载请求转发到父加载器,父加载器失败了,再自己试着加载
### JNI
* 可避免的内存拷贝,直接传递对象,到C层是一个jobject的指针,可以使用jmethodID和jfiledID访问方法和字段,无需进行内存拷贝.
* 无法避免的内存拷贝,基本类型数组,无法避免拷贝,因为JVM不信任C层的任何内存操作,特别是字符串操作,因为Java的字符串与C/C++的字符串所使用的数据类型是不一样的C/C++使用char一个字节(1字节=8位)或wchar_t是四字节.而jstring和jchar使用的是UTF-16编码使用双字节.(Unicode是兼容ASCII,但不兼容GBK,需要自己转换)
* 自己创建的局部引用一定要释放,否则一直持有内存泄漏
* 非局部引用方法返回后就会失效,除非创建全局引用,jclass是一个jobject,方法外围使用时需要创建全局引用,jmethodID和jfiledID不需要.
* JNI是通过Java方法映射到C函数实现的,如果使用这种方法,函数必须以C式接口导出(因为C++会对名字做修饰处理),当然也可以在JNI_OnLoad方法中注册.
* JNIEnv是线程独立的,JNI中使用pthread创建的线程没有JNIEnv,需要AttachCurrentThread来获取JNIEnv,不用时要DetachCurrentThread
* C++智能指针,因为C++有两种语意,智能指针类将一个计数器与类指向的对象相关联，引用计数跟踪该类有多少个对象共享同一指针。每次创建类的新对象时，初始化指针并将引用计数置为1；当对象作为另一对象的副本而创建时，拷贝构造函数拷贝指针并增加与之相应的引用计数；对一个对象进行赋值时，赋值操作符减少左操作数所指对象的引用计数（如果引用计数为减至0，则删除对象），并增加右操作数所指对象的引用计数；调用析构函数时，构造函数减少引用计数（如果引用计数减至0，则删除基础对象）。智能指针就是模拟指针动作的类(它是对象)。所有的智能指针都会重载 -> 和 * 操作符.
* C++的循环引用是在语言层面上是无法避免的,只有在设计时尽可能的避免,比如在可能发生循环引用的地方用weak_ptr代替shared_ptr,weak_ptr(与shared_ptr的区别是他不计引用计数)的lock()可以尝试获取指针,如果存在,则返回一个shared_ptr,否则返回一个保存了nullptr的shared_ptr
* unique_ptr不允许拷贝,删除了有关拷贝语意的运算符和构造函数,但是允许移动
* std::move可以将左值变成右值

## Android 数据结构

### 内存管理
* LRU最近最少使用算法
* 复用,例子Handler的Message.obtain(),使用链表结构做复用


### 红黑树

### HashMap
扩容

### Android中的ArrayMap

### ConcurrentHashMap

### WeakHashMap

### ThreadLocal

## Android Hook

### 启动流程
Application#attachBaseContext->ContentProvider#onCreate->Application#onCreate->启动第一个Activity
### Hook点

### 动态代理

### Activity启动流程

## Android 设计模式/框架

### 组件化

### MVC
传统MVC（Model-View-Controller）架构的设计，是Android最原始的开发模式。以Activity/Fragment作为核心并承担MVC中Controller的角色，但由于Activity/Fragment的功能过于强大，导致最后Activity/Fragment既承担了View的责任，又承担了Controller的责任。所以一般较复杂的页面，Activity/Fragment很容易堆积代码，最终导致Controller混杂了View层和业务逻辑，使对业务逻辑做单元测试变得非常困难,还导致一个非常严重的问题就是不好复用,当你想把某个提供数据功能单独拆出来的时候,你会发现他可能耦合了View。

### MVP
MVP（Model-View-Presenter）架构的设计，是当下最流行的开发模式，目前主要以Google推出的TodoMVP为主，是为了解决MVC中的问题而诞生的，在MVP中引入Presenter的概念，把业务逻辑抽象成Presenter，用它负责View层与Model层的交互，让Activity/Fragment还是Controller但是只是View的Controller属于View层。Presenter只持有View的接口引用。Activity/Fragment只负责UI响应而不负责业务逻辑，业务逻辑全部放到Presenter处理。因为Presenter只是持有View的接口，这样使得对Presenter做单元测试变得可能,且Presenter也比MVC更好复用了.
### MVVM
MVVM（Model-View-ViewModel）架构：MVVM由MVP的演变而来，它使用DataBinding，消息事件让架构更加灵活。其灵活性具体体现在以下3个特点中：
* 数据驱动：ViewModel只关心数据和业务逻辑，基本不需要做UI操作。View层观察ViewModel,当数据发生改变时，Databinding就能自动同步更新UI。UI的修改，又会通过Databinding以及View的Controller自动同步到ViewModel中.ViewModel比Presenter更好复用,因为ViewModel甚至不需要知道它到底是跟哪个View进行交互,只用维护自身状态和数据.
* 松散耦合：ViewModel不涉及任何UI操作和对UI控件的引用，就算把TextView改成EditText，ViewModel也几乎不需要改任何代码.在MVVM中在ViewModel是最为一个处理数据的中间者,因为Model提供的数据可能会很复杂,而View层没法直接显示,经过ViewModel处理后,这些数据能够直接被View接受,且View不知道这些数据是怎么来的,也可对View做单独的单元测试.
* 团队协作：由于View和ViewModel松散耦合，在开发团队中，可以一个人负责开发UI部分（XML + Activity/Fragment），另外一个人负责开发ViewModel与Model对接，即可形成一个可运行的项目 。

### 策略模式

### 装饰者模式

### 依赖注入

### 响应式编程
