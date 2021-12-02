---
title: EventBus 事件总线框架深入分析
date: 2021-01-31 14:16:55
cover: true
tags: 
    - 观察者模式
    - EventBus
    - 事件总线
    - 框架
category: 
	- 框架
summary: EventBus 事件总线框架从应用场景到原理深入分析





---

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/post_banner_jay.png)

# EventBus 事件总线框架深入分析

## 1. EventBus 关键词云

设计模式、数据结构、内存占用、查找效率、线程安全、对象池、事件继承问题、事件发布器、线程模式、同步、异步

## 2. EventBus 快问快答

EventBus 中运用了那些设计模式

- 观察者、单例、构建者、门面、策略

EventBus 中运用了那些数据结构

- ThreadLocal、CopyOnWriteArrayList、ConcurrentHashMap、HashMap、PendingPostQueue

EventBus 中的对象池

- FindState/FIND_STATE_POOL、PendingPost/pendingPostPool

EventBus 中的锁与线程安全问题

- 并发容器、同步锁、ThreadLocal

如何设计一个最小原型 EventBus 框架需要考虑哪些问题

- 内存占用、查找效率、线程安全

EventBus 中的事件发布时如何实现同步、异步的

- 同步：主线程、Handler；异步：子线程

## 3. EventBus 总结

#### 3.1 编译时APT流程总结

EventBus 中订阅者的收集是通过两种方式，一种是运行时反射收集，另一种是通过注解处理器收集。通过apt收集相比通过运行时反射的方式可以减少因为反射带来的性能开销，但是会影响编译时的时间。综合考量必然还是推荐通过APT方式。

通过APT方式收集订阅者的大体流程如下：

- 在编码阶段添加订阅者方法时我们需要通过 **@Subscribe** 注解类标注订阅者，该注解类提供三个参数用来个性化配置订阅者，三个参数分别是：**enum ThreadMode、boolean sticky，int priority** 。注解处理器所做的工作就是收集订阅者方法以及这些方法所在的类信息。
- EventBus 规定生成的索引类的全类名是由开发者自行定义传入的，所以在编译之前还模块的 build.gradle 中需要配置索引类，否则会编译错误。
- 注解处理器注解处理流程主要分为：**收集订阅者、校验订阅者、生成索引类**三个过程。
  - **收集订阅者**：遍历所有可执行元素 ExecutableElement，校验订阅者方法签名，订阅者方法签名需要满足 **正好只有一个参数的非静态的公开的方法** 的规则。然后将找到的订阅者方法和订阅者类存入 **ListMap<TypeElement, ExecutableElement> methodsByClass **  容器中。该容器的数据结构是：`HashMap<K, List<V>>()` key 存入的是订阅者类，value 是该类中所有订阅者方法的集合。
  - **校验订阅者**（包括订阅者类/订阅者类的父类和订阅者方法）：
    - 校验订阅者类：遍历所有上一步收集到的订阅者类，校验订阅者类，订阅者类需要满足**public/default+索引类和订阅者类的包名一样** 的规则，不满足的订阅者类需要添加到 **Set<TypeElement> classesToSkip** 容器中标记。
    - 校验订阅者类中的方法：满足上一步的订阅者类中任何一个订阅者方法满足以下情况的直接将对应的订阅者类需要添加到 **Set<TypeElement> classesToSkip** 容器中标记。
      - 订阅者方法的参数类型(也就是Event 类型)不是 DeclaredType 或者是 DeclaredType 但不是 TypeElement。
      - 订阅者方法的参数类型是类类型但是对索引类不可见。
  - **生成索引类**：遍历最终收集到的 **methodsByClass** 订阅者信息通过字符串拼接的方式生成索引类。这里换成Javapoet的方式更容易维护，EventBus 可能是为了节省框架包的体积没有采用Javapoet的方式生成java文件。

#### 3.2 初始化EventBus 总结

总的来说 EventBus 的初始化流程不是很复杂， EventBus 对象的创建结合了单例模式和构建者模式。

单例模式常用于构建全局唯一类并提供全局唯一访问点。

构建者模式常用于构建可以通过设置不同的可选参数，定制化地创建一个复杂对象。

EventBus 恰好需要全局唯一且配置复杂，此时就可以结合两个设计模式应对不同的构建场景。

需要全局唯一默认配置的实例直接通过单例获取，需要为前面的单例个性化定制也可以通过构建者模式配置参数。

你自己维护全局唯一或者需要局部唯一的场景也可以通过构建者模式个性化定制。

可能是因为 EventBus  的应用场景多样，他的DCL单例模式并没有显示私有构造方法和静态实例变量，也就是说直接 new EventBus 也是可以的。

#### 3.3 注册订阅者流程总结

注册订阅者分为查找和订阅两个过程

查找过程具体是通过当前订阅者类找出该类中声明的所有订阅者方法。查找逻辑封装在 SubscriberMethodFinder 类中，查找方式有两中，如果使用了注解处理器模块就可以通过生成的索引类查找，这种方式不需要通过反射就能收集到所有订阅者方法，效率较高。

还有一种就是运行时通过反射订阅者类的getMethods 或者getDeclaredMethods这两个方法收集订阅者方法，这种方式效率低，不推荐使用。

订阅过程具体是针对当前订阅者的每一个订阅者方法（查找流程得到）以事件类为范围进行全局排序，收集当前订阅者类的所有事件以及对粘性事件的分发。

#### 3.4 事件发布流程总结

发布事件通过ThreadLocal保证发送时的同步问题，粘性事件就是发布前存到一个map中，当订阅者注册时会遍历该map执行粘性事件的发布，事件发布时默认会考虑事件的继承关系即订阅事件的父类的订阅者也会收到事件。

具体发布过程为：通过单事件，找到所有观察者，遍历所有观察该事件的观察者，执行发布操作，发布时会根据订阅者的线程模型做出不同处理，通过三个发布器 mainThreadPoster,backgroundPoster,asyncPoster  完成四种线程模型的切换，最后通过反射调用观察者。

#### 3.5 注销订阅者流程总结

注销流程就是先通过订阅者类找所有事件，遍历每一个事件找所有订阅者，判断订阅者中的订阅者类是否和当前类一致，一致就将订阅者中的 active 标记为false 用来阻止继续发布事件，并从内存map中移除。



## 4. 概述

> 框架的作用有：隐藏实现细节，降低开发难度，做到代码复用，解耦业务与非业务代码，让程序员聚焦业务开发。

EventBus 翻译为“事件总线”，它提供了实现观察者模式的骨架代码。我们可以基于此框架，非常容易地在自己的业务场景中实现**观察者模式**，从而减少样板代码。其中，[Google Guava EventBus](https://github.com/google/guava/wiki/EventBusExplained)  就是一个比较著名的 EventBus 框架，它不仅仅支持异步非阻塞模式，同时也支持同步阻塞模式。

而我们今天要分析的 [**GreenRobot EventBus**](https://github.com/greenrobot/EventBus) 是同时适用于 Android 和 Java 平台的事件总线框架，它可简化Activities, Fragments, Threads, Services之间的通信且轻量，它的核心设计理念是对观察者模式（Observer Design Pattern）也被称为发布订阅模式（Publish-Subscribe Design Pattern）的封装。传统的事件传递方式包括：Handler、BroadcastReceiver、Interface回调等，相比之下EventBus的优点是代码简洁，使用简单，并将事件发布和订阅充分解耦。

> **观察者模式 **
>
> 定义：定义了对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。一般情况下，被依赖的对象叫作被观察者（Observable），依赖的对象叫作观察者（Observer）。
>
> 作用：解耦观察者和被观察者。
>
> 实现：
>
> - **同步阻塞**是最经典的实现方式，主要是为了代码解耦；观察者和被观察者代码在同一个线程内执行，被观察者发送更新通知后就一直阻塞着，直到所有的观察者代码都执行完成之后，才会执行后续的代码。
>
> - **异步非阻塞**除了能实现代码解耦之外，还能提高代码的执行效率；实现方式是被观察者发送更新通知后启动一个新的线程来执行观察者的回调函数。
>

**EventBus 观察者模式框架 VS 自己实现观察模式**

利用 EventBus 框架实现的观察者模式，跟从零开始编写的观察者模式相比，从大的流程上来说，实现思路大致一样，都需要定义观察者（Observer），并且通过 register() 函数注册Observer，也都需要通过调用某个函数（比如，EventBus 中的 post() 函数）来给 Observer 发送消息（在 EventBus 中消息被称作事件 event）。

但在实现细节方面，它们又有些区别。基于 EventBus，我们不需要定义 Observer 接口，任意类型的对象都可以注册到 EventBus 中，通过 @Subscribe 注解来标明类中哪个函数可以接收被观察者发送的消息。

跟经典的观察者模式的不同之处在于，当我们调用 post() 函数发送消息的时候，并非把消息发送给所有的观察者，而是发送给可匹配的观察者。所谓可匹配指的是，能接收的消息类型是发送消息（post 函数定义中的 event）类型或是其父类。



## 5. 工作机制

![EventBus-Android-Publish-Subscribe](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211107112804.png)

**发布者(Publisher)**：发布者主动生成事件发布事件给指定订阅者。

```java
EventBus.getDefault().post(new MessageEvent());
```

**事件总线(EventBus)**：统筹所有事件的调度工作，如：收集、注册、切换、分发、解注册等操作。

```java
EventBus.getDefault().register(this);
EventBus.getDefault().unregister(this);
```

**订阅者(Subscriber)**：声明订阅方法并通过注释标记，可指定线程模式。

```java
@Subscribe(threadMode = ThreadMode.MAIN)  
public void onMessageEvent(MessageEvent event) {/* Do something */};
```

## 6. EventBus 优势

- 简化组件之间的通信，同进程内随便发，组件化中需要考虑 Event Object 的存放位置。
- 解耦事件的发送者和接收者，仅通过 Event Object 进行链接发送者和接受者。
- 在 UI 组件和后台线程切换中表现良好的性能
- 避免复杂且容易出错的依赖关系和生命周期问题，提供解注册订阅者。
- 很快；专为高性能而优化，很小（~60k jar）
- [在实践中被证明通过应用与1,000,000,000+安装](http://www.appbrain.com/stats/libraries/details/eventbus/greenrobot-eventbus)
- 具有分发指定线程、设置订阅者优先级等功能。

##  7. EventBus 功能

- **简单而强大：** EventBus 是一个小型库，其 API 非常容易学习。然而，通过解耦组件，您的软件架构可能会受益匪浅：订阅者在使用事件时并不了解发送者。
- 实战**测试：** EventBus 是最常用的 Android 库之一：数以千计的应用程序使用 EventBus，包括非常流行的应用程序。超过 10 亿次应用安装不言而喻。
- **高性能：**尤其是在 Android 上，性能很重要。EventBus 进行了大量分析和优化；可能使其成为同类中最快的解决方案。
- **方便的基于注释的 API** （不牺牲性能）**：**只需将 @Subscribe 注释放在您的订阅者方法中。由于注释的构建时索引，EventBus 不需要在您的应用程序运行时进行注释反射，这在 Android 上非常慢。
- **Android 主线程传递：**在与 UI 交互时，EventBus 可以在主线程中传递事件，而不管事件是如何发布的。
- **后台线程传递：**如果您的订阅者执行长时间运行的任务，EventBus 还可以使用后台线程来避免 UI 阻塞。
- **事件和订阅者继承：**在 EventBus 中，面向对象的范式适用于事件和订阅者类。假设事件类 A 是 B 的超类。发布的 B 类事件也将发布给对 A 感兴趣的订阅者。类似地考虑订阅者类的继承。
- **零配置：** 您可以立即开始使用代码中任何位置可用的默认 EventBus 实例。
- **可配置：** 要根据您的要求调整 EventBus，您可以使用构建器模式调整其行为。

## 8. EventBus 应用

- 如果使用 EventBus 的页面比较多，可以在 Acitivity/Fragment  基类里面绑定和解绑，并添加一个默认接收事件。
- 跨界面修改值
  - 你有一个主界面，里面有一些信息可能会修改，但触发源不在该界面，是在其他的界面触发了一些事件后，首页的内容需要做修改。
  - 如果没有EventBus，也有很多的方式可以实现，譬如定义全局静态变量、或者onResume时获取触发源的值修改界面值、或者定义个CallBack接口传出去等。
  - 譬如微信首页你有未读消息3个时，界面会有3个小红点点，当你点开一个未读消息后，进入了下个界面，那么此时未读消息就是2了，但你并不在首页了，你需要在你打开消息并阅读完毕后通知首页改成2.这就是一种跨界面修改值。
- Activity/Fragment 与 Fragment 之间通信
- 注册页面回退逻辑
  - 在注册页面填写了手机号、个人信息，传头像操作后，注册成功了，进入了主界面。此时我们需要在主界面关闭之前的注册的所有页面，此时就可以使用eventbus来通知前几个注册用的activity来关闭自己。这样的目的就是当注册失败时，用户按返回键还是能回到填写信息页。当注册成功后，按返回键就直接退出程序，不再保留注册填信息页了。
- 推送/消息功能
  - 收到推送后需要不同的页面来做处理的。例如：微信PC登录时，手机端的确认登录页面是可以随时随地弹出的，
- 组件化通讯
  - 组件之间的交互，例如：测试环境中环境切换组件，切换后需要重新登录并重置环境信息等。
- EventBus最好的使用方式就是替代某些 BroadcastReceiver 和 Interface；如fragment之间进行通信，用广播和接口都比较麻烦，而用EventBus则比较简单。
- 以下场景可以考虑不用
  - Event 会根据传递的参数给所有接收者都传递消息，这就导致如果你想给指定一个类里发布消息就得自己写一个接口类，要不然就会好多执行者都会执行该方法，所以一般能用Intent组件传值时还是用Intent。
  - EventBus相对于BroadcastReceiver，广播是相对消耗时间、空间最多的一种方式，但是大家都知道，广播是四大组件之一，许多系统级的事件都是通过广播来通知的，比如说网络的变化、电量的变化，短信发送和接收的状态，所以，如果与android系统进行相关的通知，还是要选择本地广播；在BroadcastReceiver的 onReceive方法中，可以获得Context 、intent参数，这两个参数可以调用许多的sdk中的方法，而eventbus获得这两个参数相对比较困难。
  - EventBus相对于handler，可以实现handler的方式，但是也会面对有许多接收者的问题，所以如果是线程回调的话，我觉得还是用handler比较好。

## 9. EventBus 使用

### 9.1 订阅者索引

使用订阅者索引可以避免在运行时使用反射对订阅者方法进行昂贵的查找。EventBus 注释处理器在编译时查找它们。

#### 符合注解收集的要求

- @Subscribe 方法及其类**必须是 public**。
- 事件类**必须是 public**。
- @Subscribe可以**不**被使用**匿名类的内部**。
- 当 EventBus 不能使用索引时，例如不满足上述要求，它会在运行时降级为通过反射查找订阅者。这确保@Subscribe 方法接收事件，即使它们不是索引的一部分。

#### 配置注解处理器

```groovy
//java
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [ eventBusIndex : 'com.example.myapp.MyEventBusIndex' ]
            }
        }
    }
}
 
dependencies {
    def eventbus_version = '3.2.0'
    implementation "org.greenrobot:eventbus:$eventbus_version"
    annotationProcessor "org.greenrobot:eventbus-annotation-processor:$eventbus_version"
}

//kotlin 
apply plugin: 'kotlin-kapt' // ensure kapt plugin is applied

dependencies {
    def eventbus_version = '3.2.0'
    implementation "org.greenrobot:eventbus:$eventbus_version"
    kapt "org.greenrobot:eventbus-annotation-processor:$eventbus_version"
}

kapt {
    arguments {
        arg('eventBusIndex', 'com.example.myapp.MyEventBusIndex')
    }
}
```

#### 使用订阅者索引类

在您的*Application*类中，使用*EventBus.builder().addIndex(indexInstance)*将索引类的实例传递给 EventBus。组件中的索引类也可以通过addIndex方法添加到 EventBus 实例中。

```kotlin
//创建一个新实例并配置索引类
val eventBus = EventBus.builder().addIndex(MyEventBusIndex()).build()
//使用单例模式并配置索引类
EventBus.builder().addIndex(MyEventBusIndex()).installDefaultEventBus()
// Now the default instance uses the given index. Use it like this:
val eventBus = EventBus.getDefault()
```

#### 防止混淆订阅者

```java
-keepattributes *Annotation*
-keepclassmembers class * {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }
 
# And if you use AsyncExecutor:
-keepclassmembers class * extends org.greenrobot.eventbus.util.ThrowableFailureEvent {
    <init>(java.lang.Throwable);
}
```

### 9.2 配置EventBus

EventBusBuilder 类可以配置 EventBus 的各个方面。例如

使用 EventBus.getDefault() 是一种从应用程序中的任何位置获取共享 EventBus 实例的简单方法。EventBusBuilder 还允许使用installDefaultEventBus ( )方法配置此默认实例。可以在 Application 类中在使用 EventBus 之前配置默认 EventBus 实例。

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        EventBus.builder()
            //将此与 BuildConfig.DEBUG 一起使用可让应用程序尽在在 DEBUG 模式下崩溃。默认为false
            // 这样就不会在开发过程中错过异常（Invoking subscriber failed）
            .throwSubscriberException(false)
            //如果发送了没有订阅者的event,是否需要打印提示哪一个 event bean 的log,默认为true
            //提示信息： No subscribers registered for event class org.greenrobot.eventbusperf.jay.bus.SubEvent
            .logNoSubscriberMessages(true)
            .installDefaultEventBus()
    }
}
```

### 9.3 ThreadMode

在 EventBus 中，您可以使用四种 ThreadMode 来指定订阅者方法所在的线程。

- [1 ThreadMode: POSTING](https://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/#ThreadMode_POSTING) ：发布者和订阅者在同一个线程。
  - 这是默认设置。事件传递是同步完成的，需要注意避免阻塞主线程。
  - 避免了线程切换意味着开销较小。
- [2 ThreadMode: MAIN](https://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/#ThreadMode_MAIN) ：订阅者将在 Android 的主线程（UI 线程）中调用。
  - 事件传递是同步完成的，需要注意避免阻塞主线程。
- [3 ThreadMode: MAIN_ORDERED](https://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/#ThreadMode_MAIN_ORDERED) ：订阅者将在 Android 的主线程中被调用，该事件总是通过Handler排队等待稍后传递给订阅者。
  - 为事件处理提供了更严格和更一致的顺序。
  - 如果前一个也是main_ordered 需要等前一个执行完成后才执行。
  - 事件传递是异步完成的。
- [4 ThreadMode: BACKGROUND](https://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/#ThreadMode_BACKGROUND) ：如果发帖线程非主线程则订阅者的处理会在工作线程中执行否则和发布者同一个线程处理。
  - 事件传递是异步完成的。
- [5 ThreadMode: ASYNC](https://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/#ThreadMode_ASYNC) ：无论事件在哪个线程发布，订阅者都会在新建的工作线程中执行。
  - EventBus 使用线程池来有效地重用线程。
  - 事件传递是异步完成的。
  - 如果事件处理程序方法的执行可能需要一些时间，则应使用此模式，例如用于网络访问

```kotlin
 //在主线程发消息
 发布者所在线程:Thread-2, 订阅者所在线程: Thread-2, 订阅者线程模式: BACKGROUND 
 发布者所在线程:Thread-2, 订阅者所在线程: pool-1-thread-1, 订阅者线程模式: ASYNC 
 发布者所在线程:Thread-2, 订阅者所在线程: Thread-2, 订阅者线程模式: POSTING 
 发布者所在线程:Thread-2, 订阅者所在线程: main, 订阅者线程模式: MAIN 
 发布者所在线程:Thread-2, 订阅者所在线程: main, 订阅者线程模式: MAIN_ORDERED 
 //在子线程发消息
 发布者所在线程:main, 订阅者所在线程: main, 订阅者线程模式: MAIN 
 发布者所在线程:main, 订阅者所在线程: pool-1-thread-2, 订阅者线程模式: BACKGROUND 
 发布者所在线程:main, 订阅者所在线程: pool-1-thread-1, 订阅者线程模式: ASYNC 
 发布者所在线程:main, 订阅者所在线程: main, 订阅者线程模式: POSTING 
 发布者所在线程:main, 订阅者所在线程: main, 订阅者线程模式: MAIN_ORDERED 
```

### 9.4 订阅者优先级

订阅者优先级影响事件传递的顺序。 在同一个交付线程 ( ThreadMode ) 中，较高优先级的订阅者将在其他具有较低优先级的订阅者之前收到事件。 默认优先级为 0。注意：优先级不影响具有不同ThreadMode的订阅者之间的传递顺序！

```kotlin
@Subscribe(threadMode = ThreadMode.POSTING, priority = 2)
fun onMessageEvent_POSTING1(event: MessageEvent) {
    showMsg(event, "POSTING1")
}

@Subscribe(threadMode = ThreadMode.POSTING, priority = 4)
fun onMessageEvent_POSTING2(event: MessageEvent) {
    showMsg(event, "POSTING2")
}

@Subscribe(threadMode = ThreadMode.MAIN, priority = 1)
fun onMessageEvent_MAIN1(event: MessageEvent) {
    showMsg(event, "MAIN1")
}

@Subscribe(threadMode = ThreadMode.MAIN, priority = 3)
fun onMessageEvent_MAIN2(event: MessageEvent) {
    showMsg(event, "MAIN2")
}

 //打印结果
 发布者所在线程:main, 订阅者所在线程: main, 订阅者线程模式: POSTING2 
 发布者所在线程:main, 订阅者所在线程: main, 订阅者线程模式: MAIN2 
 发布者所在线程:main, 订阅者所在线程: main, 订阅者线程模式: POSTING1 
 发布者所在线程:main, 订阅者所在线程: main, 订阅者线程模式: MAIN1 
```

**取消事件传递**

您可以通过从订阅者的事件处理方法调用cancelEventDelivery ( Object event ) 来取消事件传递过程。任何进一步的事件传递都将被取消，后续订阅者将不会收到该事件。

```java
// Prevent delivery to other subscribers*
EventBus.getDefault().cancelEventDelivery(event) ;
```

事件通常由更高优先级的订阅者取消。取消仅限于在发布线程 ( ThreadMode . PostThread ) 中运行的事件处理方法。

### 9.5 粘性事件 

普通事件都是需要先注册(register)，再post才能接受到事件；如果你使用 postSticky 发送事件，那么可以不需要先注册，也能接受到事件，也就是一个延迟注册的过程。 

普通的事件我们通过post发送给EventBus，发送过后之后当前已经订阅过的方法可以收到。但是如果有些事件需要所有订阅了该事件的方法都能执行呢？例如一个Activity，要求它管理的所有Fragment都能执行某一个事件，但是当前我只初始化了3个Fragment，如果这时候通过post发送了事件，那么当前的3个Fragment当然能收到。但是这个时候又初始化了2个Fragment，那么我必须重新发送事件，这两个Fragment才能执行到订阅方法。 

粘性事件就是为了解决这个问题，通过 postSticky 发送粘性事件，这个事件不会只被消费一次就消失，而是一直存在系统中，直到被 removeStickyEvent 删除掉。那么只要订阅了该粘性事件的所有方法，只要被register 的时候就会被检测到并且执行。订阅的方法需要添加 sticky = true 属性。

```kotlin
EventBus.getDefault().postSticky(MessageEvent(Thread.currentThread().name))

//消费粘性事件方式一：
val stickyEvent = EventBus.getDefault().getStickyEvent(MessageEvent::class.java)
// 最好检查之前是否实际发布过事件
if (stickyEvent != null) {
    // 消费掉粘性事件
    EventBus.getDefault().removeStickyEvent(stickyEvent)
}
//消费粘性事件方式二：
val stickyEvent2 = EventBus.getDefault().removeStickyEvent(MessageEvent::class.java)
// 最好检查之前是否实际发布过事件
if (stickyEvent2 != null) {
    //已经消费了
}

@Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
fun onMessageEvent_sticky(event: MessageEvent) {
    showMsg(event, "MAIN")
    //消费粘性事件方式三：
    EventBus.getDefault().removeStickyEvent(event)
}
```

### 9.6 异步执行器

AsyncExecutor 就像一个线程池，但具有失败（异常）处理功能。失败会引发异常，AsyncExecutor 会将这些异常包装在一个事件中，该事件会自动发布。

 *AsyncExecutor 是一个非核心实用程序类。它可能会为您节省一些在后台线程中进行错误处理的代码，但它不是核心 EventBus 类。*

调用 AsyncExecutor.create() 来创建一个实例并将其保存在应用程序范围内。然后要执行某些操作，请实现 RunnableEx接口并将其传递给AsyncExecutor的execute方法。与Runnable不同，RunnableEx可能抛出异常。

```kotlin
//AsyncExecutor类似于线程池，但具有失败(异常)处理。失败是抛出异常，AsyncExecutor将把这些异常封装在一个事件中，该事件将自动发布。
AsyncExecutor.create().execute {
    EventBus.getDefault().postSticky(SubEvent<String>())
}

//线程池中发出的时间
@Subscribe(threadMode = ThreadMode.MAIN)
fun handleLoginEvent(event: SubEvent<String>) {
    // do something
}

//线程池中任务异常时发出的时间
@Subscribe(threadMode = ThreadMode.MAIN)
fun handleFailureEvent(event: ThrowableFailureEvent) {
    // do something
}
```



## 10 EventBus 原理

了解框架之前我们先定义几个核心角色用于描述整个流程

- 发布者类：调用 post/postSticky 发布事件的类

- 发布者方法：调用 post/postSticky 发布事件的方法

- 事件类： post/postSticky 方法参数类以及订阅者方法参数类型
- 订阅者类：订阅者方法所在的类
- 订阅者方法：通过注解标注的订阅者方法

![image-20211111000620374](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211111000627.png)



### 10.1 编译时部分-通过 APT 收集订阅者注解并生成索引类

注解类 Subscribe 用于标注订阅者方法

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    //用来指定指定订阅者方法所在的线程。
    ThreadMode threadMode() default ThreadMode.POSTING;
    //如果为 true，则将最近的粘性事件（通过EventBus.postSticky(Object) ）传递给该订阅者。
    boolean sticky() default false;
    //订阅者优先级影响事件传递的顺序。
    // 在同一个交付线程 ( ThreadMode ) 中，较高优先级的订阅者将在其他具有较低优先级的订阅者之前收到事件。 默认优先级为 0。
    // 注意：优先级不会影响具有不同ThreadMode的订阅者之间的传递顺序！
    int priority() default 0;
}
```

收集那些代码元素：订阅者类&订阅者方法

```java
class BusTestActivity : Activity() {
  	//定义一个订阅者方法
    @Subscribe(sticky = true, threadMode = ThreadMode.POSTING, priority = 2)
    fun onMessageEvent_POSTING1(event: MessageEvent) {
        showMsg(event, "POSTING1")
    }
}
```

注解处理器通过继承 Java AbstractProcessor 抽象类并配置注解和选项参数实现

```java
@SupportedAnnotationTypes("org.greenrobot.eventbus.Subscribe")
@SupportedOptions(value = {"eventBusIndex", "verbose"})
@IncrementalAnnotationProcessor(AGGREGATING)
public class EventBusAnnotationProcessor extends AbstractProcessor {
  ...
}
```

- 通过处理器参数获取配置的订阅者索引全类名，没有配置该参数但是却依赖了注解处理组件会抛异常

```java
//设置apt参数
javaCompileOptions {
    annotationProcessorOptions {
        arguments = [
                eventBusIndex: 'org.greenrobot.eventbusperf.MyEventBusIndex',
                verbose: 'true',
        ]
    }
}
//获取apt参数
String index = processingEnv.getOptions().get(OPTION_EVENT_BUS_INDEX);
if (index == null) {
    messager.printMessage(Diagnostic.Kind.ERROR, "No option " + OPTION_EVENT_BUS_INDEX +
            " passed to annotation processor");
    return false;
}

//没有配置的错误信息
> Task :EventBusPerformance:compileDebugJavaWithJavac FAILED
错误: No option eventBusIndex passed to annotation processor
```

- 收集注解流程

```java
private void collectSubscribers(Set<? extends TypeElement> annotations, RoundEnvironment env, Messager messager{
    for (TypeElement annotation : annotations) {
        //获取目标注解标注的所有元素，这里是所有的订阅者方法
        Set<? extends Element> elements = env.getElementsAnnotatedWith(annotation);
        for (Element element : elements) {
            //ExecutableElement 可执行元素指的是方法类型
            if (element instanceof ExecutableElement) {
                ExecutableElement method = (ExecutableElement) element;
                //检查方法:正好只有一个参数的非静态的公开的方法
                if (checkHasNoErrors(method, messager)) {
                    //获取方法所在的类元素
                    TypeElement classElement = (TypeElement) method.getEnclosingElement();
                    //存入容器
                    methodsByClass.putElement(classElement, method);
                }
            } else {
                messager.printMessage(Diagnostic.Kind.ERROR, "@Subscribe is only valid for methods", element);
            }
        }
    }
}
```

- 校验某个类对索引类包是否可访问

```java
private boolean isVisible(String myPackage, TypeElement typeElement) {
    //类的修饰符
    Set<Modifier> modifiers = typeElement.getModifiers();
    boolean visible;
    if (modifiers.contains(Modifier.PUBLIC)) {
        visible = true;
    } else if (modifiers.contains(Modifier.PRIVATE) || modifiers.contains(Modifier.PROTECTED)) {
        visible = false;
    } else {
        //类所在的包
        String subscriberPackage = getPackageElement(typeElement).getQualifiedName().toString();
        //处理器参数没有指定索引类
        if (myPackage == null) {
            //todo 没有包名什么情况
            visible = subscriberPackage.length() == 0;
        } else {
            //索引类和观察者包名一样
            visible = myPackage.equals(subscriberPackage);
        }
    }
    return visible;
}
```

- 校验收集到的注解元素信息是否符合预期

```java
private void checkForSubscribersToSkip(Messager messager, String myPackage) {
    //遍历所有订阅者方法所在的类
    for (TypeElement skipCandidate : methodsByClass.keySet()) {
        //方法所在的类，
        TypeElement subscriberClass = skipCandidate; //循环获取父类
        while (subscriberClass != null) {//所有观察者
            //校验某个类类对索引类包是否可访问
            if (!isVisible(myPackage, subscriberClass)) {
                //索引类访问不到观察者类，跳过
                boolean added = classesToSkip.add(skipCandidate);
                if (added) {//存在不可访问观察者
                    String msg;
                    //由于类不是公开的，所以回退到反射
                    if (subscriberClass.equals(skipCandidate)) { //没有继承关系存在
                        msg = "Falling back to reflection because class is not public";
                    } else { //父类
                        msg = "Falling back to reflection because " + skipCandidate +
                                " has a non-public super class";
                    }
                    messager.printMessage(Diagnostic.Kind.NOTE, msg, subscriberClass);
                }
                break;
            }
            //观察者类中的所有观察方法
            List<ExecutableElement> methods = methodsByClass.get(subscriberClass);
            if (methods != null) {
                for (ExecutableElement method : methods) {
                    String skipReason = null;
                    //方法第一个参数
                    VariableElement param = method.getParameters().get(0);
                    //参数类型
                    TypeMirror typeMirror = getParamTypeMirror(param, messager);
                    //不是类类型报错
                    if (!(typeMirror instanceof DeclaredType) || !(((DeclaredType) typeMirror).asElement() instanceof TypeElement)) {
                        skipReason = "event type cannot be processed";
                    }
                    //是类类型但是对索引类不可见
                    if (skipReason == null) {
                        TypeElement eventTypeElement = (TypeElement) ((DeclaredType) typeMirror).asElement();
                        //参数类对索引类不可见
                        if (!isVisible(myPackage, eventTypeElement)) {
                            skipReason = "event type is not public";
                        }
                    }
                    //存在观察者方法但是不可见先存下来，用于过滤
                    if (skipReason != null) {
                        boolean added = classesToSkip.add(skipCandidate);
                        if (added) {
                            String msg = "Falling back to reflection because " + skipReason;
                            if (!subscriberClass.equals(skipCandidate)) {
                                msg += " (found in super class for " + skipCandidate + ")";
                            }
                            messager.printMessage(Diagnostic.Kind.NOTE, msg, param);
                        }
                        break;
                    }
                }
            }
            //获取观察者类的父类，继续循环
            subscriberClass = getSuperclass(subscriberClass);
        }
    }
}
```

- 将收集到的索引信息写入索引类中的 map 容器中

```java
private void writeIndexLines(BufferedWriter writer, String myPackage) throws IOException {
    for (TypeElement subscriberTypeElement : methodsByClass.keySet()) {
        //只生成可访问的
        if (classesToSkip.contains(subscriberTypeElement)) {
            continue;
        }
        String subscriberClass = getClassString(subscriberTypeElement, myPackage);
        if (isVisible(myPackage, subscriberTypeElement)) {
            writeLine(writer, 2,
                    "putIndex(new SimpleSubscriberInfo(" + subscriberClass + ".class,",
                    "true,", "new SubscriberMethodInfo[] {");
            List<ExecutableElement> methods = methodsByClass.get(subscriberTypeElement);
            writeCreateSubscriberMethods(writer, methods, "new SubscriberMethodInfo", myPackage);
            writer.write("        }));\n\n");
        } else {
            writer.write("        // Subscriber not visible to index: " + subscriberClass + "\n");
        }
    }
}
```

- 生成的 MyEventBusIndex 文件

```java
//通过注释处理创建的生成索引类的接口。
public interface SubscriberInfo {
    Class<?> getSubscriberClass();
    SubscriberMethod[] getSubscriberMethods();
    SubscriberInfo getSuperSubscriberInfo();
    boolean shouldCheckSuperclass();
}

public class MyEventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();
        putIndex(new SimpleSubscriberInfo(org.greenrobot.eventbusperf.testsubject.PerfTestEventBus.SubscriberClassEventBusAsync.class,true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onEventAsync", TestEvent.class, ThreadMode.ASYNC),
        }));
      //其它索引信息......
      }

private static void putIndex(SubscriberInfo info) {
    SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
}
  
@Override
public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
    SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
    if (info != null) {
        return info;
    } else {
        return null;
    }
}
      
```

#### 编译时APT流程总结

EventBus 中订阅者的收集是通过两种方式，一种是运行时反射收集，另一种是通过注解处理器收集。通过apt收集相比通过运行时反射的方式可以减少因为反射带来的性能开销，但是会影响编译时的时间。综合考量必然还是推荐通过APT方式。

通过APT方式收集订阅者的大体流程如下：

- 在编码阶段添加订阅者方法时我们需要通过 **@Subscribe** 注解类标注订阅者，该注解类提供三个参数用来个性化配置订阅者，三个参数分别是：**enum ThreadMode、boolean sticky，int priority** 。注解处理器所做的工作就是收集订阅者方法以及这些方法所在的类信息。
- EventBus 规定生成的索引类的全类名是由开发者自行定义传入的，所以在编译之前还模块的 build.gradle 中需要配置索引类，否则会编译错误。
- 注解处理器注解处理流程主要分为：**收集订阅者、校验订阅者、生成索引类**三个过程。
  - 收集订阅者：遍历所有可执行元素 ExecutableElement，校验订阅者方法签名，订阅者方法签名需要满足 **正好只有一个参数的非静态的公开的方法** 的规则。然后将找到的订阅者方法和订阅者类存入 **ListMap<TypeElement, ExecutableElement> methodsByClass **  容器中。该容器的数据结构是：`HashMap<K, List<V>>()` key 存入的是订阅者类，value 是该类中所有订阅者方法的集合。
  - 校验订阅者（包括订阅者类/订阅者类的父类和订阅者方法）：
    - 校验订阅者类：遍历所有上一步收集到的订阅者类，校验订阅者类，订阅者类需要满足**public/default+索引类和订阅者类的包名一样** 的规则，不满足的订阅者类需要添加到 **Set<TypeElement> classesToSkip** 容器中标记。
    - 校验订阅者类中的方法：满足上一步的订阅者类中任何一个订阅者方法满足以下情况的直接将对应的订阅者类需要添加到 **Set<TypeElement> classesToSkip** 容器中标记。
      - 订阅者方法的参数类型(也就是Event 类型)不是 DeclaredType 或者是 DeclaredType 但不是 TypeElement。
      - 订阅者方法的参数类型是类类型但是对索引类不可见。
  - 生成索引类：遍历最终收集到的 **methodsByClass** 订阅者信息通过字符串拼接的方式生成索引类。这里换成Javapoet的方式更容易维护，EventBus 可能是为了节省框架包的体积没有采用Javapoet的方式生成java文件。

### 10.2 运行时部分-初始化、注册/查找订阅者等

#### 10.2.1 初始化EventBus

构建 EventBus 实例的三种方式：
- EventBus.getDefault() + 默认Builder

- EventBus.builder().installDefaultEventBus() + 自定义配置

- EventBus.builder().build() + 自定义配置

方式一：DCL单例方式创建进程唯一实例

```java
static volatile EventBus defaultInstance;

//使用进程范围的 EventBus 实例的应用程序的便捷单例。
public static EventBus getDefault() {
 	 //通过局部变量中转可节省性能
    EventBus instance = defaultInstance;
    if (instance == null) {
        synchronized (EventBus.class) {
            instance = EventBus.defaultInstance;
            if (instance == null) {
                instance = EventBus.defaultInstance = new EventBus();
            }
        }
    }
    return instance;
}
```

其它两种方式

```java
//EventBus.builder().build() + 自定义配置
EventBus.builder()
    .throwSubscriberException(false)
    .logNoSubscriberMessages(true)
    //添加索引类，减少运行时反射
    .addIndex(MyEventBusIndex())
    .build()
  
public EventBus build() {
    return new EventBus(this);
}

//EventBus.builder().installDefaultEventBus() + 自定义配置
EventBus.builder()
    .throwSubscriberException(false)
    .logNoSubscriberMessages(true)
    //添加索引类，减少运行时反射
    .addIndex(MyEventBusIndex())
    .installDefaultEventBus()
  
public EventBus installDefaultEventBus() {
    synchronized (EventBus.class) {
        if (EventBus.defaultInstance != null) {
            throw new EventBusException("Default instance already exists." +
                    " It may be only set once before it's used the first time to ensure consistent behavior.");
        }
        EventBus.defaultInstance = build();
        return EventBus.defaultInstance;
    }
}
```

**构建EventBus时的默认配置**

```java
EventBus(EventBusBuilder builder) {
    logger = builder.getLogger();
    //通过事件类找所有该事件的订阅者，
    subscriptionsByEventType = new HashMap<>();
    //通过订阅者类找所有Event
    typesBySubscriber = new HashMap<>();
    //通过粘性事件类查找所有粘性事件对象
    stickyEvents = new ConcurrentHashMap<>();
    //构建 AndroidHandlerMainThreadSupport
    mainThreadSupport = builder.getMainThreadSupport();
    //构建 HandlerPoster
    mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
    //在后台发布事件
    backgroundPoster = new BackgroundPoster(this);
    //在后台发布事件
    asyncPoster = new AsyncPoster(this);
    indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
    //通过反射或APT查找订阅者
    subscriberMethodFinder = new SubscriberMethodFinder(
            //添加由 EventBus 的注释预处理器生成的索引。默认空集合
            builder.subscriberInfoIndexes,
            //启用严格的方法验证（默认值：false）
            builder.strictMethodVerification,
            //即使有生成的索引也强制使用反射（默认值：false）
            builder.ignoreGeneratedIndex);
    //无法分发事件时是否打印错误信息
    logSubscriberExceptions = builder.logSubscriberExceptions;
    //没有订阅者注册事件是否打印错误信息
    logNoSubscriberMessages = builder.logNoSubscriberMessages;
    //在调用订阅者时如果发生异常是否 发送一个 SubscriberExceptionEvent 通知订阅者
    sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
    //没有订阅者注册事件是否是否通知订阅者类的父类
    sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
    //在调用订阅者时如果发生异常是否抛出 RuntimeException
    throwSubscriberException = builder.throwSubscriberException;
    //是否通知订阅者类的父类中的订阅者方法
    eventInheritance = builder.eventInheritance;
    //订阅者执行在工作线程时用到的线程池：Executors.newCachedThreadPool()
    executorService = builder.executorService;
}
```

##### 初始化EventBus 总结

总的来说 EventBus 的初始化流程不是很复杂， EventBus 对象的创建结合了单例模式和构建者模式。

单例模式常用于构建全局唯一类并提供全局唯一访问点。

构建者模式常用于构建可以通过设置不同的可选参数，定制化地创建一个复杂对象。

EventBus 恰好需要全局唯一且配置复杂，此时就可以结合两个设计模式应对不同的构建场景。

需要全局唯一默认配置的实例直接通过单例获取，需要为前面的单例个性化定制也可以通过构建者模式配置参数。

你自己维护全局唯一或者需要局部唯一的场景也可以通过构建者模式个性化定制。

可能是因为 EventBus  的应用场景多样，他的DCL单例模式并没有显示私有构造方法和静态实例变量，也就是说直接 new EventBus 也是可以的。

#### 10.2.2 注册订阅者：查找订阅者方法

```java
public override fun onStart() {
    super.onStart()
    EventBus.getDefault().register(this)
}
//注册给定的订阅者以接收事件。 订阅者一旦对接收事件不再感兴趣，须调用 unregister(Object) 。
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    //通过订阅者类找出该类中所有的订阅者方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);//01
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);//02
        }
    }
}
```

01查找订阅者方法流程：通过APT或反射方式查找订阅者方法并内存缓存

```java
//ConcurrentHashMap 内存缓存保证线程安全
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

//通过订阅者类查找订阅者方法
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    //先从内存缓存尝试取，节省查找开销
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    if (ignoreGeneratedIndex) {
        //运行时反射查找
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        //从APT中收集的备选中查找
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        //订阅者类中至少有一个订阅者方法，否则运行时报错
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        //找到后缓存到内存
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

**运行时反射查找** 

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    //准备一个 FindState 实例，如果对象池中没有就new,一个FindState对应一个订阅者类，用于表示查找状态
    FindState findState = prepareFindState();
    //存入订阅者 class
    findState.initForSubscriber(subscriberClass);
    //遍历subscriberClass的超类体系，调用findUsingReflectionInSingleClass查找当前clazz的所有订阅函数
    while (findState.clazz != null) {
        //在订阅者类中通过反射的方式查找订阅者方法
        findUsingReflectionInSingleClass(findState);
        findState.moveToSuperclass();//获取父类继续查找
    }
    //循环结束 findState.subscriberMethods 中保存了这个类中的所有订阅者方法
    return getMethodsAndRelease(findState);
}


```

在订阅者类中通过反射的方式查找订阅者方法

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    //getDeclaredMethods 在某些设备上也会出现 NoClassDefFoundError
    try {
        //getDeclaredMethods 要比 getMethods 快，尤其是当订阅者是像 Activities 这样的胖类时
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        //getMethods 在某些设备上也会出现 NoClassDefFoundError，可能会在 getMethods 周围添加 catch
        try {
            methods = findState.clazz.getMethods();
        } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
            throw new EventBusException(msg, error);
        }
        //clazz.getDeclaredMethods()只返回当前clazz中声明的函数，
        // 而clazz.getMethods()将返回clazz的所有函数(包括继承自父类和接口的函数)，
        // 因此，此时skipSuperClasses被置为true，阻止递归查找父类。
        findState.skipSuperClasses = true;
    }

    //遍历所有方法
    for (Method method : methods) {
        int modifiers = method.getModifiers(); //修饰符
        //MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
        //校验订阅者方法：must be public, non-static, and non-abstract"
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();//参数类型
            if (parameterTypes.length == 1) {//正好一个参数
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    // 恰好一个参数的非静态的公开类并且有Subscribe注解标记
                    Class<?> eventType = parameterTypes[0];// 参数为事件类
                    //检查重名方法（本类或父类之间可能重复）用于控制findState.subscriberMethods是否添加找到的method
                    //如果不校验，如果子类重写订阅者方法会导致执行两次子类的订阅者方法
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        //收集订阅者方法，封装 SubscriberMethod
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

为啥不直接用Class.getMethods直接获取该类的全部方法呢？

如果这个类比较庞大，用getMethods查找所有的方法就显得很笨重了，
如果使用的是getDeclaredMethods（该类声明的方法不包括从父类那里继承来的public方法），速度就会快一些，因为找的方法变少了，没有什么 equals,toString,hashCode等Object类的方法。

Class#getMethods()，不检查方法签名（对于诸如不存在的参数类型之类的东西）。这已更改为 use Class#getDeclaredMethods()，它会检查并在出现问题时抛出异常。



FindState 对象和对象池

```java
private static final int POOL_SIZE = 4;
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    findState.recycle(); //findState 回收数据
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                //将当前用过的 findState 缓存回去，下次注册时不用 new
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    return subscriberMethods;
}

//先从对象池中随便找一个，没有才创建
private FindState prepareFindState() {
    //用的时候隔离开，用完了放回去。
    synchronized (FIND_STATE_POOL) { //FindState[] FIND_STATE_POOL
        for (int i = 0; i < POOL_SIZE; i++) { //POOL_SIZE = 4
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null; //在原来的位置用null占位
                return state; // 遍历FindState对象池，只要找到一个空对象就返回，
            }
        }
    }
    return new FindState(); 
}

static class FindState {
    final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
    final Map<Class, Object> anyMethodByEventType = new HashMap<>();
    final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
    final StringBuilder methodKeyBuilder = new StringBuilder(128);
    Class<?> subscriberClass;
    Class<?> clazz;
    boolean skipSuperClasses;
    SubscriberInfo subscriberInfo;

    void initForSubscriber(Class<?> subscriberClass) {
        this.subscriberClass = clazz = subscriberClass;
        skipSuperClasses = false;
        subscriberInfo = null;
    }

    void recycle() {
        subscriberMethods.clear();
        anyMethodByEventType.clear();
        subscriberClassByMethodKey.clear();
        methodKeyBuilder.setLength(0);
        subscriberClass = null;
        clazz = null;
        skipSuperClasses = false;
        subscriberInfo = null;
    }

    // 检查是否已经添加过这个订阅者方法
    boolean checkAdd(Method method, Class<?> eventType) {
        //2 级检查：仅具有事件类型的第一级（快速），在需要时具有完整签名的第二级。
        // 通常订阅者没有侦听相同事件类型的方法。
        //第一层判断有无method监听此eventType,如果没有则可直接把找到的method加到subscriberMethods中。
        //第二层检查的是从MethodSignature（方法签名）判断能否把找到的method加进去。是为了防止在找父类时覆盖了子类的方法，因为此方法是子类是重写，方法名参数名完全一样（方法签名）；另一个原因是可能是当一个类有多个方法监听同一个event(尽管一般不会这样做)，也能将这些方法加进去。
        Object existing = anyMethodByEventType.put(eventType, method);
        if (existing == null) { //没有添加过，
            //anyMethodByEventType存储<eventType, method>映射关系，
            // 若existing为空，则表示eventType第一次出现。
            // 一般情况下，一个对象只会有一个订阅函数处理特定eventType。
            return true;
        } else {//一个类有多个方法监听同一个事件类型
            if (existing instanceof Method) {
                //处理一个对象有多个订阅函数处理eventType的情况，
                // 此时，anyMethodByEventType中eventType被映射到一个非Method对象(即this)。
                if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                    // Paranoia check
                    throw new IllegalStateException();
                }
                // Put any non-Method object to "consume" the existing Method
                //将任何非 Method 对象“使用”现有的 Method
                anyMethodByEventType.put(eventType, this);
            }
            return checkAddWithMethodSignature(method, eventType);
        }
    }

    //由于存在多个订阅函数处理eventType，此时，单纯使用eventType作为key已经无法满足要求了，
    // 因此，使用method.getName() + ">" + eventType.getName()作为methodKey，
    // 并使用subscriberClassByMethodKey存储<methodKey, methodClass>的映射关系。
    private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
        methodKeyBuilder.setLength(0);
        methodKeyBuilder.append(method.getName());
        methodKeyBuilder.append('>').append(eventType.getName());
        //onEvent>TestEvent
        String methodKey = methodKeyBuilder.toString();
        //getDeclaringClass: 返回表示类或接口的 Class 对象，该类或接口声明了由此对象表示的可执行文件。
        Class<?> methodClass = method.getDeclaringClass();
        //map["onEvent>TestEvent"]=
        //如果methodClassOld或者methodClass是methodClassOld的子类，
        // 则将<methodKey, methodClass>放入，否则不放入。
        // 满足函数名相同、参数类型相同且被@Subscribe修饰的函数，
        // 在一个类中不可能存在两个；考虑类继承体系，若这样的两个函数分别来自父类和子类，
        // 则最终被加入的是子类的函数。
        Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
        //确定此Class对象表示的类或接口是否与指定的Class参数表示的类或接口相同，或者是其超类或超接口。
        if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
            // Only add if not already found in a sub class
            //仅在子类中未找到时才添加
            return true;
        } else {
            // Revert the put, old class is further down the class hierarchy
            //还原放置，旧类在类层次结构中更靠后
            subscriberClassByMethodKey.put(methodKey, methodClassOld);
            return false;
        }
    }

    void moveToSuperclass() {
        if (skipSuperClasses) { //反射方法时是通过getMethod 方式，已经包含父类方法了
            clazz = null;
        } else {
            clazz = clazz.getSuperclass();
            String clazzName = clazz.getName();
            // Skip system classes, this degrades performance.
            // Also we might avoid some ClassNotFoundException (see FAQ for background).
            if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") ||
                    clazzName.startsWith("android.") || clazzName.startsWith("androidx.")) {
                clazz = null;
            }
        }
    }
}
```

**通过APT中收集数据中查找**

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        //通过订阅者类从索引类中查找订阅者方法信息:subscriberInfo
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                // 检查重名方法（本类或父类之间都可能重复
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            //apt 没有正常收集该类，降级为反射方式查找
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}

private SubscriberInfo getSubscriberInfo(FindState findState) {
		//找完子类找父类
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }

    //apt 收集的索引类
    if (subscriberInfoIndexes != null) {
        //通过订阅者类从索引类中查找订阅者方法信息
        for (SubscriberInfoIndex index : subscriberInfoIndexes) {
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    return null;
}
```

**查找流程小结**

注册订阅者分为查找和订阅两个过程

查找过程具体是通过当前订阅者类找出该类中声明的所有订阅者方法。查找逻辑封装在 SubscriberMethodFinder 类中，查找方式有两中，如果使用了注解处理器模块就可以通过生成的索引类查找，这种方式不需要通过反射就能收集到所有订阅者方法，效率较高。

还有一种就是运行时通过反射订阅者类的getMethods 或者getDeclaredMethods这两个方法收集订阅者方法，这种方式效率低，不推荐使用。

亮点整理：

- METHOD_CACHE 缓存了查找过的进程内所有订阅者的键值对(订阅者类，类中的订阅者方法)信息，使用ConcurrentHashMap即保证的查找效率也避免了线程安全问题。

```java
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
```

- FIND_STATE_POOL 是 FindState 对象池的一维静态数组，FindState 对查找的状态值做了一些封装以及对订阅者方法的检查逻辑。
  - 为什么要使用FindState呢？首先是面向对象封装的采用
  - 在JVM系统中频繁地创建对象，是非常消耗资源的，在jvm垃圾回收时候，有可能会出现内存抖动的问题。使用对象池数组就有效的避免了内存抖动的问题。
  - 对 FIND_STATE_POOL  的操作需要考虑线程同步问题，这里使用了`synchronized`关键字来保证线程安全。

```java
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
```



**订阅流程**

查找到当前注册的订阅者类中的所有订阅者方法后，下一步就是为每一个订阅者方法执行订阅流程了。

```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    // 01查找订阅者方法流程
    // 通过订阅者类找出该类中所有的订阅者方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        //遍历该订阅者类中所有订阅者方法，执行订阅操作
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            // 02订阅流程
            // 收集订阅者，事件类，分发粘性事件等
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

订阅流程大题分为三步

- 通过事件类找所有已经订阅过该事件的订阅者们，目的是为了进行优先级排序,全局缓存已注册订阅者等
- 通过订阅者类找所有已经注册过的 Event 们， 用于判断是否注册、解注册等
- 当前订阅者中有粘性事件，在 register 的时候根据当前订阅者方法的 event 直接执行分发

```java

private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    //事件类
    Class<?> eventType = subscriberMethod.eventType;
    //封装订阅者
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    //Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    //通过事件类找所有该事件的订阅者，
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    //还没缓存该事件就存且只能存一次
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        //不能有重复订阅者
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        //优先级排序，要么最小查到最后，要么之前的某个位置
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            //向事件对应的订阅者列表中添加当前订阅者，完成排序操作
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    //Map<Object, List<Class<?>>> typesBySubscriber;
    //通过订阅者类找所有Event
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    //将当前订阅者方法的 event 存到指定订阅者类下的列表里
    subscribedEvents.add(eventType);

    //粘性事件：先pst 后 订阅
    //当前订阅者中有粘性事件，在 register 的时候根据当前订阅者方法的 event 直接执行分发
    if (subscriberMethod.sticky) {
        //eventInheritance: 默认true
        // 默认情况下，EventBus 考虑事件类层次结构（将通知超类的订阅者）。 关闭此功能将改进事件的发布。
        // 对于直接扩展 Object 的简单事件类，我们测得事件发布速度提高了 20%。 对于更复杂的事件层次结构，加速应该大于 20%。
        //但是，请记住，事件发布通常只消耗应用程序内一小部分 CPU 时间，除非它以高速率发布，例如每秒数百/数千个事件

        if (eventInheritance) { //
            //必须考虑 eventType 的所有子类的现有粘性事件。注意：对于大量粘性事件，迭代所有事件可能效率低下，
            // 因此应更改数据结构以允许更有效的查找（例如，存储超类的子类的附加映射：Class -> List<Class>）。
            //stickyEvents = new ConcurrentHashMap<>();
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                //key: event.getClass(), value: event
                Class<?> candidateEventType = entry.getKey();
                //isAssignableFrom: 确定此Class对象表示的类或接口是否与指定的Class参数表示的类或接口相同，或者是其超类或超接口。 如果是，则返回true ； 否则返回false 。 如果此Class对象表示原始类型，则如果指定的Class参数正是此Class对象，则此方法返回true ； 否则返回false 。
                //具体来说，此方法测试是否可以通过标识转换或通过扩展引用转换将指定Class参数表示的类型转换为此Class对象表示的类型。 有关详细信息，请参阅Java 语言规范5.1.1 和 5.1.4 节
                if (eventType.isAssignableFrom(candidateEventType)) { //比较class
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            //通过粘性事件类查找所有粘性事件对象
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}

private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    if (stickyEvent != null) {
        // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
        // --> Strange corner case, which we don't take care of here.
        //如果订阅者试图中止事件，它将失败（事件在发布状态下不被跟踪）--> 奇怪的极端情况，我们在这里不处理。
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

##### 注册订阅者流程总结

注册订阅者分为查找和订阅两个过程

查找过程具体是通过当前订阅者类找出该类中声明的所有订阅者方法。查找逻辑封装在 SubscriberMethodFinder 类中，查找方式有两中，如果使用了注解处理器模块就可以通过生成的索引类查找，这种方式不需要通过反射就能收集到所有订阅者方法，效率较高。

还有一种就是运行时通过反射订阅者类的getMethods 或者getDeclaredMethods这两个方法收集订阅者方法，这种方式效率低，不推荐使用。

订阅过程具体是针对当前订阅者的每一个订阅者方法（查找流程得到）以事件类为范围进行全局排序，收集当前订阅者类的所有事件以及对粘性事件的分发。

亮点整理：

- CopyOnWriteArrayList 保证线程安全

```java
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
```

- ConcurrentHashMap 保证了效率和安全

```java
stickyEvents = new ConcurrentHashMap<>();
```

#### 10.2.3 事件发布

发布粘性事件，粘性事件特别之处在于发布前存到了一个map中，当注册时直接执行粘性事件的发布

```java
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        stickyEvents.put(event.getClass(), event);
    }
    //放置后应发布，以防订阅者想立即删除
    post(event);
}
```

发布普通事件，通过ThreadLocal保证发送时的同步问题

```java
public void post(Object event) {
    //currentPostingThreadState = new ThreadLocal<PostingThreadState>()
    //每个线程都有一份 postingState 实例，
    //封装 PostingThreadState 对于 ThreadLocal，设置（并获得多个值）要快得多。
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    if (!postingState.isPosting) { // 默认 false
        postingState.isMainThread = isMainThread(); //判断主线程还是子线程
        postingState.isPosting = true; //这里保证 cancelEventDelivery 是在同一个线程调用的
        if (postingState.canceled) { //cancelEventDelivery
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            //可能发送了多个事件
            while (!eventQueue.isEmpty()) {
                //发送队列依次取出第一个事件执行发布
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            //发送完事件后重置标志位
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

事件发布时考虑事件的继承关系

```java
//发送队列的第一个事件
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    //是否考虑订阅者的继承关系
    if (eventInheritance) {
        //事件类和事件类的父类们
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            //当前类和父类有一个没收到就算失败
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        //兜底方案，发送一个通知事件，告诉订阅者刚才的事件没发送成功
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

通过单事件，找到所有观察者，遍历所有观察该事件的观察者，执行发布操作

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        //通过单事件，找到所有观察者
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            //存入线程
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                //重置状态
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

发布时会根据订阅者的线程模型做出不同处理

```java
//发布到订阅
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    //根据线程模型不同处理
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING: //订阅和发布是同一个线程
            invokeSubscriber(subscription, event);
            break;
        case MAIN: //订阅在主线程
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                //通过Handler 发送到主线程
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED: //订阅在主线程排队
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                //临时：技术上不正确，因为海报没有与订阅者分离
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:////如果发帖线程非主线程则订阅者的处理会在工作线程中执行否则和发布者同一个线程处理。
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC://无论事件在哪个线程发布，订阅者都会在新建的工作线程中执行。
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

三个发布器 mainThreadPoster,backgroundPoster,asyncPoster  对应四种线程模型

反射调用观察者

```java
//同一个线程执行订阅者方法
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        //方法、类、参数
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

##### 事件发布流程总结

发布事件通过ThreadLocal保证发送时的同步问题，粘性事件就是发布前存到一个map中，当订阅者注册时会遍历该map执行粘性事件的发布，事件发布时默认会考虑事件的继承关系即订阅事件的父类的订阅者也会收到事件。

具体发布过程为：通过单事件，找到所有观察者，遍历所有观察该事件的观察者，执行发布操作，发布时会根据订阅者的线程模型做出不同处理，通过三个发布器 mainThreadPoster,backgroundPoster,asyncPoster  完成四种线程模型的切换，最后通过反射调用观察者。



#### 10.2.4 注销订阅者

```java
public override fun onStop() {
    super.onStop()
    EventBus.getDefault().unregister(this)
}
```

根据订阅者类找该类的所有事件

```java
public synchronized void unregister(Object subscriber) {
    //找到订阅者类对应的事件类列表
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        //根据每个事件类解除每个订阅者
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        //从内存map 移除
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

根据每个事件找对应的所有订阅者，如果订阅者中的订阅者类和当前类一样才执行解除订阅

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    //根据每个事件类找到所有该事件的订阅者
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            //每个订阅者：类+方法
            Subscription subscription = subscriptions.get(i);
            //确认是当前类的订阅者
            if (subscription.subscriber == subscriber) {
                //修改解除订阅标志位
                subscription.active = false;
                subscriptions.remove(i);
                i--; //防止越界
                size--;
            }
        }
    }
}
```

##### 注销订阅者流程总结

注销流程就是先通过订阅者类找所有事件，遍历每一个事件找所有订阅者，判断订阅者中的订阅者类是否和当前类一致，一致就将订阅者中的 active 标记为false 用来阻止继续发布事件，并从内存map中移除。



## 11. 参考

**[Github | EventBus](https://github.com/greenrobot/EventBus)**

[EventBus Documentation](https://greenrobot.org/eventbus/documentation/)

[极客时间| 设计模式之美](https://time.geekbang.org/column/intro/250?code=Grxvvkczx9tydhzn0RhJfNfwaF2RgJA9qeUWd8orIYo%3D)

[EventBus 如何使用及一些常见场景](https://cloud.tencent.com/developer/article/1383971)

[EventBus使用总结和使用场景](https://blog.csdn.net/f552126367/article/details/86571012)

