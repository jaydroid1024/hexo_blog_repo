---
title: Jetpack | Lifecycle 组件系详解第一篇：Lifecycle
date: 2021-05-21 14:16:55
cover: true
tags: 
    - Jetpack
    - Lifecycle
category: 
	- Jetpack
summary: Lifecycle 是什么，有什么，怎么用，应用场景，实现原理等





---

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/post_banner_jay.png)

# Jetpack | Lifecycle 组件系详解第一篇：Lifecycle

Lifecycle细组件主要包括：Lifecycle、LiveData、ViewModle、其它扩展组件(process 、service)等。

<img src="https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211122005956.png" alt="image-20211122005853174" style="zoom:50%;" />

## 1. Lifecycle问题汇总

- 什么是 Lifecycle
- 如何使用 Lifecycle 观察宿主状态
- Lifecycle 是如何分发宿状态的
- Fragment 是如何实现 Lifecycle 的
- Activity 是如何实现 Lifecycle 的
- Application 是如何实现 Lifecycle 的
- Service 是如何实现 Lifecycle 的
- View 是如何实现观察宿主 Lifecycle 的
- Lifecycle 涉及的依赖库是如何划分的
-  Lifecycle 实现观察宿主状态有几种方式
-  注解+反射/生成代码的方式为什么又被废弃了
- Activity 的生命周期分发为何通过 ReportFragment 实现
- Lifecycle Event 和 State 的关系
- 在 onResume 方法中注册观察者，是否能观察到其它生命周期的回调
- 分发宿状态过程中是如何同步 Event 和 State 的

## 2. Lifecycle是什么

Lifecycle 是一个对宿主生命周期的变化具有感知能力的组件 (Lifecycle-Aware Components) ，在 Android 中目前提供的可观察的宿主组件有 Activity / Fragment / Service / Application 等，除了jetpack 组件中内置的可观察的宿主我们也可以借助 LifeCycle 的能力自己实现可观察的宿主，例如 SDK 中的 Activity 是没有实现Lifecycle 的，此时我们就可以根据业务需求自定时实现了。

Lifecycle 的核心实现思想是观察者模式，Jetpack 组件中的 Activity / Fragment 组件中都已经接入了 Lifecycle 中的被观察者者部分的代码，我们只需要实现自己的观察者然后在 Activity / Fragment 组件中注册我们的观察者就可以监听到生命周期事件的变化了。

> 支持库 26.1.0 及更高版本中的 Fragment 和 Activity 已实现 [`LifecycleOwner`](https://developer.android.com/reference/androidx/lifecycle/LifecycleOwner) 接口。

一种常见的应用场景是有些逻辑需要依赖在 Activity 和 Fragment 的生命周期方法中实现，通过 Lifecycle  组件就可以将这部分代码从生命周期方法中提取到单独的类中，达到解耦被观察者和观察者的目的，从而帮助开发者写出简洁易维护代码。

> 观察者模式（Observer Design Pattern）也被称为发布订阅模式（Publish-Subscribe Design Pattern）
>
> **观察者模式定义了对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。一般情况下，被依赖的对象叫作被观察者（Observable），依赖的对象叫作观察者（Observer）。**
>
> 观察者模式属于行为型设计模式，行为型设计模式的目的是将不同的行为代码解耦，具体到观察者模式就是是将观察者和被观察者代码解耦。
>
> 观察者模式的实现一般分为两个角色：Observable、Observer，两个角色一般都提供抽象层。
>
> 被观察者部分的抽象层一般是抽象类，除了提供必要的抽象方法还提供添加、删除等复用的逻辑。
>
> 观察者部分的抽象层一般是一个SAM（Single Abstract Method）接口，观察者可以实现该方法做出更新操作。

![image-20211120233538055](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211120233721.png)

Lifecycle 是如何结合观察者模式实现 Activity 和 Fragment 组件的生命周期感知能力的

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211121181816.png)

Lifecycle 在实现被观察者时并没有采用传统的直接继承抽象类的方式，而是通过类似委托模式将被观察者的逻辑代码封装到了 LifecycleRegistry 类中，LifecycleRegistry 是真正的被观察者负责统一管理所有观察者的添加、删除、存储、分发等操作。当委托者（ Activity / Fragment）有生命周期事件产生时会通过受托者 LifecycleRegistry 执行具体的分发操作，从而实现委托者类的生命周期可感知能力。

这样做的优势是符合单一原则，有利于被观察者侧的代码复用，也不用破坏 Activity/Fragment 组件的继承结构。



## 3. Lifecycle 有什么

androidx.lifecycle 组下的组件,了解 lifecycle  有什么才能更好的运用。lifecycle 模块中除了自己实现观察者模式相关的代码

| lifecycle-common <br />lifecycle-common-java8 <br />lifecycle-compiler<br />lifecycle-runtime <br />lifecycle-runtime-ktx <br />lifecycle-runtime-ktx- lint <br />lifecycle-runtime-testing | lifecycle-livedata <br />lifecycle-livedata-core <br />lifecycle-livedata-core-ktx<br />lifecycle-livedata-core-ktx-lint <br />lifecycle-livedata-core-truth  <br />lifecycle-livedata-ktx<br />lifecycle-reactivestreams <br />lifecycle-reactivestreams-ktx | lifecycle-viewmodel <br />lifecycle-viewmodel-compose <br />lifecycle-viewmodel-ktx <br />lifecycle-viewmodel-savedstate | lifecycle-process <br />lifecycle-service <br />lifecycle-extensions |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| lifecycle相关                                                | livedata 相关                                                | viewmodel相关                                                | 其它扩展组件相关                                             |
| common-java8 已经废弃                                        | -                                                            | -                                                            | extensions 耦合重已经废弃                                    |

再来看一下其中的几个核心组件之间的依赖关系

![WechatIMG77](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211122011323.jpeg)

开发时按需添加 Lifecycle 的依赖项

```groovy
def lifecycle_version = "2.4.0"
// ViewModel
implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$lifecycle_version"
// ViewModel utilities for Compose
implementation "androidx.lifecycle:lifecycle-viewmodel-compose:$lifecycle_version"
// LiveData
implementation "androidx.lifecycle:lifecycle-livedata-ktx:$lifecycle_version"
// Lifecycles only (without ViewModel or LiveData)
implementation "androidx.lifecycle:lifecycle-runtime-ktx:$lifecycle_version"
// Saved state module for ViewModel
implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:$lifecycle_version"
// Annotation processor
kapt "androidx.lifecycle:lifecycle-compiler:$lifecycle_version"
// alternately - if using Java8, use the following instead of lifecycle-compiler
implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"
// optional - helpers for implementing LifecycleOwner in a Service
implementation "androidx.lifecycle:lifecycle-service:$lifecycle_version"
// optional - ProcessLifecycleOwner provides a lifecycle for the whole application process
implementation "androidx.lifecycle:lifecycle-process:$lifecycle_version"
// optional - ReactiveStreams support for LiveData
implementation "androidx.lifecycle:lifecycle-reactivestreams-ktx:$lifecycle_version"
```

其它组件对lifecycle的依赖情况

```groovy
implementation 'androidx.core:core-ktx:1.7.0'
```

core-ktx api 了 core 

core api 了lifecycle-runtime

lifecycle-runtime api 了 lifecycle-common

```groovy
implementation 'androidx.appcompat:appcompat:1.3.0'
```

appcompat api 了core、activity、fragment

activity api 了 core、lifecycle-runtime、lifecycle-viewmodel、lifecycle-viewmodel-savedstate

fragment api 了core、activity、lifecycle-livedata-core、lifecycle-viewmodel、lifecycle-viewmodel-savedstate

lifecycle-runtime api 了 lifecycle-common

lifecycle-livedata-core api 了  lifecycle-livedata

所以一般情况下我们新建的Android 项目默认都会提供core-ktx、appcompat 这两个组件而他们又间接依赖了 lifecycle 系的组件

依赖 appcompat 间接依赖的 lifecycle 系组件

![image-20211121131315450](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211121134326.png)

依赖 core-ktx 间接依赖的 lifecycle 系组件

![image-20211121131403284](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211121134339.png)



## 4. Lifecycle 应用

生命周期感知型组件可以实现在各种情况下更轻松地管理生命周期。下面列举几个例子：

- 在粗粒度和细粒度位置更新之间切换。使用生命周期感知型组件可在位置应用可见时启用细粒度位置更新，并在应用位于后台时切换到粗粒度更新。
- 停止和开始视频缓冲。使用生命周期感知型组件可尽快开始视频缓冲，但会推迟播放，直到应用完全启动。此外，应用销毁后，还可以使用生命周期感知型组件终止缓冲。
- 开始和停止网络连接。借助生命周期感知型组件，可在应用位于前台时启用网络数据的实时更新（流式传输），并在应用进入后台时自动暂停。
- 暂停和恢复动画可绘制资源。借助生命周期感知型组件，可在应用位于后台时暂停动画可绘制资源，并在应用位于前台后恢复可绘制资源。
- Handler 的消息移除。
- Presenter 的 attach&detach View 。
- 为其他三方库加持生命周期感知的能力，例如：RxJava 、EventBus等。

## 5. Lifecycle 怎么用

### 5.1 观察者使用方式汇总

Lifecycle 的以下使用方式是以  Activity 或者 Fragment 为宿主举例。

- 方式一：运行时注解+反射
  - 自定义 LifecycleObserver 观察者，用 OnLifecycleEvent 注解配合 Lifecycle.Event 枚举标注生命周期方法；
  - 在宿主 Activity 或者 Fragment 中通过 getLifecycle().addObserver() 方法注册定义的观察者；
- 方式二：编译时注解+生成辅助类(XXX_LifecycleAdapter)
  - 添加注解处理器组件：lifecycle-compiler
  - 自定义 LifecycleObserver 观察者，用 OnLifecycleEvent 注解配合 Lifecycle.Event 枚举 标注生命周期方法；
  - 在宿主 Activity 或者 Fragment 中通过 getLifecycle().addObserver() 方法注册定义的观察者；
- 方式三：实现 FullLifecycleObserver (非公共方法，自己可以实现一个)
  - 自定义 FullLifecycleObserver 观察者，FullLifecycleObserver 是普通接口需要实现其中定义的所有生命周期方法；
  - 在宿主 Activity 或者 Fragment 中通过 getLifecycle().addObserver() 方法注册定义的观察者；
- 方式四：实现 LifecycleEventObserver(推荐方式)
  - 自定义 LifecycleEventObserver 观察者，通过实现 onStateChanged(LifecycleOwner ,Lifecycle.Event) 方法自行判断生命周期方法的回调；
  - 在宿主 Activity 或者 Fragment 中通过 getLifecycle().addObserver() 方法注册定义的观察者；
- **方式五：实现 DefaultLifecycleObserver (推荐方式)**
  - 自定义 DefaultLifecycleObserver 观察者，DefaultLifecycleObserver 中通过java default 关键字都实现了方法体，所以只需实现需要的声明后期方法即可；
  - 在宿主 Activity 或者 Fragment 中通过 getLifecycle().addObserver() 方法注册定义的观察者；

> DefaultLifecycleObserver 接口中的 default 关键字
>
> default 关键字修饰的方法能够向接口添加新功能方法，必须提供方法体，并确保兼容实现这个接口的之前的类不用在接口的子类中进行逐个实现该方法。可以按需实现
>
> default是在需要给接口新增方法时，但是子类数量过多，或者子类没必要实现的场景下使用。 比如java8中的List接口，新增了sort()方法
>
> ```java
> //@since 1.8
> public interface List<E> extends Collection<E> {
> ...
> default void sort(Comparator<? super E> c) {
>    Object[] a = this.toArray();
>    Arrays.sort(a, (Comparator) c);
>    ListIterator<E> i = this.listIterator();
>    for (Object e : a) {
>        i.next();
>        i.set((E) e);
>    }
> }
> 
> ```

### 5.2 观察 Activity Lifecycle

```kotlin
class MyLifecycleActivityObserver : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) {
        super.onStart(owner)
        Log.d("MyLifecycleActivity", "onStart")
    }

    override fun onStop(owner: LifecycleOwner) {
        super.onStop(owner)
        Log.d("MyLifecycleActivity", "onStop")
    }
}
```

```kotlin
class MyLifecycleActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        addLifecycleObserver()
    }
    private fun addLifecycleObserver() {
        lifecycle.addObserver(MyLifecycleActivityObserver())
    }
}
```

### 5.3 观察 SDK Activity Lifecycle

```kotlin
class MySdkActivity : Activity(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleRegistry = LifecycleRegistry(this)
        addLifecycleObserver()
        MySDKFragment.inject(this)
    }

    private fun addLifecycleObserver() {
        lifecycle.addObserver(MySDKActivityObserver())
    }

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }
}

class MySDKActivityObserver : DefaultLifecycleObserver {

    override fun onPause(owner: LifecycleOwner) {
        super.onPause(owner)
        Log.i("SDKActivity", "Observer onPause")
    }

    override fun onResume(owner: LifecycleOwner) {
        super.onResume(owner)
        Log.i("SDKActivity", "Observer onResume")

    }
}
```

### 5.4 观察 Fragment Lifecycle

```kotlin
class MyLifecycleFragmentObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun connectListener() {
        Log.i("MyLifecycleFragment", "onResume")
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun disconnectListener() {
        Log.i("MyLifecycleFragment", "onPause")

    }
}
```

```kotlin
class MyLifecycleFragment : Fragment() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        addLifecycleObserver()
    }

    private fun addLifecycleObserver() {
        lifecycle.addObserver(MyLifecycleFragmentObserver())
    }
  
}
```

### 5.5 观察 SDK Fragment Lifecycle

```kotlin
class MySDKFragment : Fragment(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleRegistry = LifecycleRegistry(this)
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
        addLifecycleObserver()
    }

    private fun addLifecycleObserver() {
        lifecycle.addObserver(MySDKFragmentObserver())
    }

    companion object {
        fun inject(activity: Activity) {
            val manager = activity.fragmentManager
            if (manager.findFragmentByTag("MyLifecycleFragment") == null) {
                manager.beginTransaction()
                    .add(MySDKFragment(), "MyLifecycleFragment")
                    .commit()
                manager.executePendingTransactions()
            }
        }
    }
    
    override fun onResume() {
        super.onResume()
        Log.i("SDKFragment", " onResume")
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME)

    }

    override fun onPause() {
        super.onPause()
        Log.i("SDKFragment", " onPause")
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    }

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }

}

class MySDKFragmentObserver : DefaultLifecycleObserver {

    override fun onPause(owner: LifecycleOwner) {
        super.onPause(owner)
        Log.i("SDKFragment", "Observer onPause")
    }

    override fun onResume(owner: LifecycleOwner) {
        super.onResume(owner)
        Log.i("SDKFragment", "Observer onResume")

    }
}
```

### 5.6 观察 Service Lifecycle

```kotlin
public class MyLifecycleServiceObserver implements LifecycleEventObserver {
    
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_START) {
            Log.d("MyLifecycleService", "onStart()");
        } else if (event == Lifecycle.Event.ON_DESTROY) {
            Log.d("MyLifecycleService", "onDestroy()");
        }
    }
}
```

```kotlin
public class MyLifecycleService extends LifecycleService {
    public MyLifecycleService() {
        getLifecycle().addObserver(new MyLifecycleServiceObserver());
    }
}
```

### 5.7 观察 Application Lifecycle

```kotlin
class MyLifecycleApplicationObserver(private val application: Application) :
    LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun appInResumeState() {
        Toast.makeText(application, "In Foreground", Toast.LENGTH_LONG).show()
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun appInPauseState() {
        Toast.makeText(application, "In Background", Toast.LENGTH_LONG).show()
    }
}
```

```kotlin
public class MyLifecycleApplication extends MultiDexApplication {

    @Override
    public void onCreate() {
        super.onCreate();
        //饿汉式单例获取 ProcessLifecycleOwner
        ProcessLifecycleOwner.get().getLifecycle().addObserver(new MyLifecycleApplicationObserver(this));
    }
}
```

### 5.8 View 观察 Lifecycle

```kotlin
class MyLifecycleView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyle: Int = 0
) : View(context, attrs, defStyle) {
    init {
        addOnAttachStateChangeListener(object : OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(v: View?) {
                Log.d("MyLifecycleView", "onViewAttachedToWindow")
                findViewTreeLifecycleOwner()?.lifecycle
                    ?.addObserver(object : LifecycleEventObserver {
                        override fun onStateChanged(
                            source: LifecycleOwner,
                            event: Lifecycle.Event
                        ) {
                            Log.d("MyLifecycleView", "onStateChanged：source:$source, event: $event")
                        }
                    })
            }

            override fun onViewDetachedFromWindow(v: View?) {
                Log.d("MyLifecycleView", "onViewDetachedFromWindow")

            }

        })

    }
}
```

### 5.9 各种方式的观察者的执行顺序

- DefaultLifecycleObserver 所有方法将在 [LifecycleOwner] 的生命周期回调方法被调用之前被调用，这里需要注意Fragment 生命周期的回调时机。

* LifecycleEventObserver onStateChanged 方法在当状态转换事件发生时调用。
*  如果一个类同时实现了DefaultLifecycleObserver 和LifecycleEventObserver ，则首先调用DefaultLifecycleObserver方法，然后调用LifecycleEventObserver.onStateChanged(LifecycleOwner, Lifecycle.Event) 方法。
*  如果一个类实现了这个接口并且同时使用了OnLifecycleEvent 注解，那么注解将被忽略。

```kotlin
D/Life_Owner: onCreate
D/Life_Observer: onCreate
D/Life_Observer: onStateChanged,event:ON_CREATE
D/Life_Owner: onStart
D/Life_Observer: onStart
D/Life_Observer: onStateChanged,event:ON_START
D/Life_Owner: onResume
D/Life_Observer: onResume
D/Life_Observer: onStateChanged,event:ON_RESUME
D/Life_Observer: onPause
D/Life_Observer: onStateChanged,event:ON_PAUSE
D/Life_Owner: onPause
D/Life_Observer: onStop
D/Life_Observer: onStateChanged,event:ON_STOP
D/Life_Owner: onStop
D/Life_Observer: onDestroy
D/Life_Observer: onStateChanged,event:ON_DESTROY
D/Life_Owner: onDestroy
```

在Activity 的onPause 方法注册观察者，当宿主执行onPause时 观察者也是会从 onCreate 开始直到对齐当前状态，Lifecycle 内部做了同步和对齐的处理。

```
D/Life_Owner: onCreate
D/Life_Owner: onStart
D/Life_Owner: onResume
D/Life_Observer: onCreate
D/Life_Observer: onStateChanged,event:ON_CREATE
D/Life_Observer: onStart
D/Life_Observer: onStateChanged,event:ON_START
D/Life_Owner: onPause
D/Life_Observer: onStop
D/Life_Observer: onStateChanged,event:ON_STOP
D/Life_Owner: onStop
D/Life_Observer: onDestroy
D/Life_Observer: onStateChanged,event:ON_DESTROY
D/Life_Owner: onDestroy
```

## 6. Lifecycle 最小原型设计

![观察者模式](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211121181816.png)

#### 6.1 代码实现最小原型

被观察者部分

```kotlin
//抽象接口层
abstract class Lifecycle {
    abstract fun addObserver(observer: LifecycleObserver)
    abstract fun removeObserver(observer: LifecycleObserver)
    enum class State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;
    }

    enum class Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY;
    }
}
//受托方
class LifecycleRegistry(private val lifecycleOwner: LifecycleOwner) : Lifecycle() {

    private var lifecycleObserver: LifecycleObserver? = null

    private val lifecycleObservers = arrayListOf<LifecycleObserver>()

    override fun addObserver(observer: LifecycleObserver) {
        lifecycleObservers.add(observer)
        lifecycleObserver = observer
    }

    override fun removeObserver(observer: LifecycleObserver) {
        lifecycleObservers.remove(observer)
    }

    fun handleLifecycleEvent(event: Event) {
        lifecycleObservers.forEach {
            if (it is LifecycleEventObserver) {
                it.onStateChanged(lifecycleOwner, event)
            }
        }
    }

}
```

观察者部分

```kotlin
interface LifecycleObserver {}
interface LifecycleEventObserver : LifecycleObserver {
    fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event)
}
interface DefaultLifecycleObserver : FullLifecycleObserver {

    override fun onCreate(owner: LifecycleOwner) {}

    override fun onStart(owner: LifecycleOwner) = Unit

    override fun onResume(owner: LifecycleOwner) {}

    override fun onPause(owner: LifecycleOwner) {}

    override fun onStop(owner: LifecycleOwner) {}

    override fun onDestroy(owner: LifecycleOwner) {}

}
```

测试被观察者部分

```kotlin
class Activity : LifecycleOwner {

    private val lifecycleRegistry: LifecycleRegistry = LifecycleRegistry(this)

    init {
        lifecycleRegistry.addObserver(ActivityObserver())
    }

    override fun getLifecycle(): Lifecycle = lifecycleRegistry

    fun onStart() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START)
    }

    fun onStop() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP)
    }

}
```

测试观察者部分

```kotlin
class ActivityObserver : LifecycleEventObserver {
    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        println("source: $source, event: $event ")
    }
}
//运行测试
fun main() {
    val app = Activity()
    app.onStart()
    app.onStop()
}
```



## 7. Lifecycle 实现原理

### 7.1 Fragment 的实现

jetpack 中的 Fragment 组件已经被观察部分的接口 LifecycleOwner

```java
//androidx.fragment.app.Fragment 中已经实现 LifecycleOwner 
public class Fragment implements 
  			ComponentCallbacks, 
				OnCreateContextMenuListener, 
				LifecycleOwner,
        ViewModelStoreOwner, 
				HasDefaultViewModelProviderFactory, 
				SavedStateRegistryOwner,
        ActivityResultCaller {...}
		//通过覆写 LifecycleOwner 的 getLifecycle 方法向外暴露宿主的生命周期管理类
    @Override
    @NonNull
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }

   LifecycleRegistry mLifecycleRegistry;
	//构造函数中进行了受托者 LifecycleRegistry 的初始化
   public Fragment() initLifecycle();}
   private void initLifecycle() {
        mLifecycleRegistry = new LifecycleRegistry(this);
    }

	//当发生生命周期事件时执通过委托类分发该事件到所有观察者中
 void performCreate(Bundle savedInstanceState) {
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
  }
```

Fragment 的生命周期感知实现很简单，就是委托给 mLifecycleRegistry 全权负责

### 7.2 Activity 实现

```java
//androidx.core.app.ComponentActivity，@hide标注，不对外使用，只做了 Lifecycle 和 KeyEvent 的封装
public class ComponentActivity extends Activity implements
        LifecycleOwner,
        KeyEventDispatcher.Component {...}
```

```java
//androidx.activity.ComponentActivity，以上特性 + 集成了 Jitpack 的其它组件，例如：Lifecycle，ViewModel等
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        ContextAware,
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner,
        ActivityResultRegistryOwner,
        ActivityResultCaller {
        //通过覆写 LifecycleOwner 的 getLifecycle 方法向外暴露宿主的生命周期管理类
        private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
			 	public Lifecycle getLifecycle() {
        		return mLifecycleRegistry;
    		}
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
          	// ReportFragment 是继承自 sdk 中的 Fragment, 所以这里可以兼容 SDK 中的 Activity 也能实现声明周期感知
            ReportFragment.injectIfNeededIn(this);
            if (mContentLayoutId != 0) {
                setContentView(mContentLayoutId);
            }
    }
}
```

```java
//androidx.fragment.app.FragmentActivity，以上特性 + 简化Fragment 的使用，例如：FragmentManager
public class FragmentActivity extends ComponentActivity implements
        ActivityCompat.OnRequestPermissionsResultCallback,
        ActivityCompat.RequestPermissionsRequestCodeValidator {...}
```

```java
//androidx.appcompat.app.AppCompatActivity 以上特性 + 简化 Material 设计，例如主题、暗黑、导航条等
public class AppCompatActivity extends FragmentActivity implements 
  			AppCompatCallback,
        TaskStackBuilder.SupportParentable, 
				ActionBarDrawerToggle.DelegateProvider {...}
```

androidx.activity 组件下的 **ComponentActivity** 可以说是 androidx 系或者说是 Jetpack 开发套件中的最顶层 Activity 基类了，可以看到 ComponentActivity 类中已经实现了 LifecycleOwner，所以和 Fragment 一样将被观察者的逻辑委托给了LifecycleRegistry。

但是我们在ComponentActivity 生命周期的回调方法中并没有看到 LifecycleRegistry 执行的分发操作。在 onCreate 方法中我们看到ReportFragment.injectIfNeededIn(this); 这句代码，这里就是 Activity 声明周期可感知做的的兼容处理，ReportFragment 是继承自 sdk 中的 Fragment, 所以这里可以兼容 SDK 中的 Activity 也能实现声明周期感知。

#### ReportFragment

```java
		public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            // 在 API 29+ 上，可以直接注册 Activity 中的 registerActivityLifecycleCallbacks 回调方法获取Activity 的生命周期				回调。
            LifecycleCallbacks.registerIn(activity);
        }
        //在 API 29 之前和进程的宿主 ProcessLifecycleOwner 都是通过内嵌一个空的 Fragment 获间接取 Activity 的生命周期回调。
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
		}	
		//29以上分发的方式
		@RequiresApi(29)
    static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
        static void registerIn(Activity activity) {
            activity.registerActivityLifecycleCallbacks(new LifecycleCallbacks());
        }

        @Override
        public void onActivityPostCreated(@NonNull Activity activity,
                @Nullable Bundle savedInstanceState) {
            dispatch(activity, Lifecycle.Event.ON_CREATE);
        }

        @Override
        public void onActivityPostStarted(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_START);
        }
        @Override
        public void onActivityPreDestroyed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_DESTROY);
        }

        @Override
        public void onActivityDestroyed(@NonNull Activity activity) {
        }
    }



		//Fragment 声明周期方法触发是
    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);// 进程29以下需要需要的
        dispatch(Lifecycle.Event.ON_START);// 分发29以下 Avtivity 
    }

   private void dispatch(@NonNull Lifecycle.Event event) {
        if (Build.VERSION.SDK_INT < 29) {
            dispatch(getActivity(), event);
        }
    }
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event); // 29以下最终的分发方法
            }
        }
    }

```

总的来说 Activity 将生命周期方法通知给订阅者的方式有别于 Fragment 的直接在生命周期回调中委托给 mLifecycleRegistry 全权负责。Activity 的方式显现复杂，搞复杂的主要原因也是为了兼容低版本和方便移植不得不做的。

根据SDK版本可分为两种方式，大于等于29(Androi 10) 以上是通过  activity.registerActivityLifecycleCallbacks(new Application.ActivityLifecycleCallbacks) 方式注册生命周期回调方法的方式获取 Activity 的生命周期回调，然后再回调中在通过mLifecycleRegistry 通知订阅者。

29(Androi 10) 以下是通过内嵌一个空的 Fragment 获间接获取 Activity 的生命周期回调，然后再回调中在通过mLifecycleRegistry 通知订阅者。

### 7.3 LifecycleRegistry-真正的被观察者

LifecycleRegistry 通常被 Fragments 和 Activity 组件使用。 如果有自定义 LifecycleOwner的需求也可以直接使用它。

LifecycleRegistry  可以看做是Fragment 和 Activity 实现生命周期可感知的受托方。所有的被观察的逻辑都在这里实现。

#### 构造方法

```java
private LifecycleRegistry(@NonNull LifecycleOwner provider, boolean enforceMainThread) {
    mLifecycleOwner = new WeakReference<>(provider); // 宿主类通过弱引用包裹
    mState = INITIALIZED;
    mEnforceMainThread = enforceMainThread;
}
LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
if (lifecycleOwner == null) {
    throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
            + "garbage collected. It is too late to change lifecycle state.");
}
```

宿主类通过弱引用包裹，当方式GC时会回收宿主类避免内存泄漏的产生，每次获取宿主时都会先判空，如果被GC了是不会继续执行的。

#### 添加观察者

```java
//宿主中向LifecycleRegistry 中添加观察者
lifecycle.addObserver(MyLifecycleActivityObserver())
```

```java
//缓存观察者的数据容器
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();

//对分发事件的封装
static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
      	//根据事件获取状态
        State newState = event.getTargetState();
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event); //分发
        mState = newState; //前进一个状态
    }
}

@Override
public void addObserver(@NonNull LifecycleObserver observer) {
  	//初始值状态为：mState = INITIALIZED;
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
  	//包装观察者
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
		//之前缓存过了，直接返回
    if (previous != null) {
        return;
    }
  	//宿主被GC了
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }
		//
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);//计算出宿主当前的状态
    mAddingObserverCounter++;
  	//同步到宿主当前的状态，例如在 onResume 方法注册的观察者，之前的生命周期方法也会回调
    while ((statefulObserver.mState.compareTo(targetState) < 0 //比较状态，是否小于宿主的状态,枚举类根据 ordinal 序号进行比较，越靠后的序号越大
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
      	//向前移动一个生命周期方法，ON_CREATE-ON_START-ON_RESUME 知道对齐宿主
        final Event event = Event.upFrom(statefulObserver.mState);
        if (event == null) {
            throw new IllegalStateException("no event up from " + statefulObserver.mState);
        }
      	//每次向前移动一个生命周期方法就分发落后的生命周期方法
        statefulObserver.dispatchEvent(lifecycleOwner, event);
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}

@Nullable
public static Event upFrom(@NonNull State state) {
    switch (state) {
        case INITIALIZED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        default:
            return null;
    }
}

```

向前同步时：先根据状态推倒事件，分发是根据事件推导出状态，再更新状态。

假设宿主中是在onResume 方法中注册的观察者，那么他的同步流程应该是怎样的呢？

观察者最终会受到onCreate-onStart-onRsume 三个回到方法。

####  State 和 Event

针对Lifecycle 中 State 和 Event 的对应关系我们通过官方提供的流转图分析一下。

Lifecycle 接口中提供两个枚举：State表示宿主状态，Event表示宿主生命周期事件。

```java
enum class State {
    DESTROYED,
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED;
}
enum class Event {
    ON_CREATE,
    ON_START,
    ON_RESUME,
    ON_PAUSE,
    ON_STOP,
    ON_DESTROY,
    ON_ANY;
}
```

两个过程：前进和后腿

前进：INITIALIZED-ON_CREATE-CREATED-ON_START-STARTED-ON_RESUME-RESUMED

倒退：RESUMED-ON_PAUSE-STARTED-ON_STOP-CREATED-ON_DESTROY-DESTROYED



![生命周期状态示意图](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211121224101.svg)

状态是图中的节点，事件可以看作这些节点之间的边。

通过事件获取状态

```java
public State getTargetState() {
    switch (this) {
        case ON_CREATE:
        case ON_STOP:
            return State.CREATED;
        case ON_START:
        case ON_PAUSE:
            return State.STARTED;
        case ON_RESUME:
            return State.RESUMED;
        case ON_DESTROY:
            return State.DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException(this + " has no target state");
}
```

如果上面的图看不太明白，看看下面这个就清除它们的对应关系了。

![img](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20211027015118.jpeg)



#### 通知观察者

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    enforceMainThreadIfNeeded("handleLifecycleEvent");
  	//根据事件推导状态，再执行移动
    moveToState(event.getTargetState());
}
//条件的判断
private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}

private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // mState 表示宿主的状态，比观察者的小说明是后腿流程 onPause-onStop
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Map.Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        // 前进流程
      	if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
//while 循环的条件，是否都同步完了
private boolean isSynced() {
    if (mObserverMap.size() == 0) {
        return true;
    }
    State eldestObserverState = mObserverMap.eldest().getValue().mState;
    State newestObserverState = mObserverMap.newest().getValue().mState;
    return eldestObserverState == newestObserverState && mState == newestObserverState;
}
```



前进和后腿

```java

private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Map.Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Map.Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = Event.downFrom(observer.mState); //循环倒退
            if (event == null) {
                throw new IllegalStateException("no event down from " + observer.mState);
            }
            pushParentState(event.getTargetState());
          	//分发生命周期方法
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
public static Event downFrom(@NonNull State state) {
    switch (state) {
        case CREATED:
            return ON_DESTROY;
        case STARTED:
            return ON_STOP;
        case RESUMED:
            return ON_PAUSE;
        default:
            return null;
    }
}


private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Map.Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Map.Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            final Event event = Event.upFrom(observer.mState);
            if (event == null) {
                throw new IllegalStateException("no event up from " + observer.mState);
            }
          	//分发生命周期方法
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}

public static Event upFrom(@NonNull State state) {
    switch (state) {
        case INITIALIZED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        default:
            return null;
    }
}
```

#### 区分观察者类型-Lifecycling

结论是无论哪种方式的观察者都通过适配器模式转换为LifecycleEventObserver类型，当分发事件时，只要执行  mLifecycleObserver.onStateChanged(owner, event); Lifecycling适配的多种类型都会得到相应的分发执行。

```java
ObserverWithState(LifecycleObserver observer, State initialState) {
    mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
    mState = initialState;
}
void dispatchEvent(LifecycleOwner owner, Event event) {
    State newState = event.getTargetState();
    mState = min(mState, newState);
    mLifecycleObserver.onStateChanged(owner, event);
    mState = newState;
}
```

适配器模式转换观察者包装成 LifecycleEventObserver

```java
static LifecycleEventObserver lifecycleEventObserver(Object object) {
    boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
    boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
  	//实现了这个连个接口
    if (isLifecycleEventObserver && isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                (LifecycleEventObserver) object);
    }
    if (isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
    }

    if (isLifecycleEventObserver) {
        return (LifecycleEventObserver) object;
    }
  	//实现的LifecycleObserver + 注解的方式，最新版本已经废弃，存在拖慢编译速度，反射效率低的问题
    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass); // 通过反射apt 生成的 adapter 类是否发生ClassNotFoundException异常来判断是否采用了apt
    if (type == GENERATED_CALLBACK) { // 采用的 apt 的方式
      //GeneratedAdapter 是生成类的接口
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    return new ReflectiveGenericLifecycleObserver(object); // 运行时反射的方式
}
```

适配器转换为 LifecycleEventObserver 类型

```java
class FullLifecycleObserverAdapter implements LifecycleEventObserver {

    private final FullLifecycleObserver mFullLifecycleObserver;
    private final LifecycleEventObserver mLifecycleEventObserver;

    FullLifecycleObserverAdapter(FullLifecycleObserver fullLifecycleObserver,
            LifecycleEventObserver lifecycleEventObserver) {
        mFullLifecycleObserver = fullLifecycleObserver;
        mLifecycleEventObserver = lifecycleEventObserver;
    }
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mFullLifecycleObserver.onCreate(source);
                break;
            case ON_START:
                mFullLifecycleObserver.onStart(source);
                break;
            case ON_RESUME:
                mFullLifecycleObserver.onResume(source);
                break;
            case ON_PAUSE:
                mFullLifecycleObserver.onPause(source);
                break;
            case ON_STOP:
                mFullLifecycleObserver.onStop(source);
                break;
            case ON_DESTROY:
                mFullLifecycleObserver.onDestroy(source);
                break;
            case ON_ANY:
                throw new IllegalArgumentException("ON_ANY must not been send by anybody");
        }
        if (mLifecycleEventObserver != null) {
            mLifecycleEventObserver.onStateChanged(source, event);
        }
    }
}
```

apt 生成的方式转换为 LifecycleEventObserver类型

```java
class SingleGeneratedAdapterObserver implements LifecycleEventObserver {

    private final GeneratedAdapter mGeneratedAdapter;

    SingleGeneratedAdapterObserver(GeneratedAdapter generatedAdapter) {
        mGeneratedAdapter = generatedAdapter;
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        mGeneratedAdapter.callMethods(source, event, false, null);
        mGeneratedAdapter.callMethods(source, event, true, null);
    }
}

class CompositeGeneratedAdaptersObserver implements LifecycleEventObserver {

    private final GeneratedAdapter[] mGeneratedAdapters;

    CompositeGeneratedAdaptersObserver(GeneratedAdapter[] generatedAdapters) {
        mGeneratedAdapters = generatedAdapters;
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        MethodCallsLogger logger = new MethodCallsLogger();
        for (GeneratedAdapter mGenerated: mGeneratedAdapters) {
            mGenerated.callMethods(source, event, false, logger);
        }
        for (GeneratedAdapter mGenerated: mGeneratedAdapters) {
            mGenerated.callMethods(source, event, true, logger);
        }
    }
}
```

### 7.4 Application 实现

注册

```java
class MyLifecycleApplication : MultiDexApplication() {
    override fun onCreate() {
        super.onCreate()
        //饿汉式单例获取 ProcessLifecycleOwner
        ProcessLifecycleOwner.get().lifecycle.addObserver(MyLifecycleApplicationObserver(this))
    }
}
```

通过 startup 初始化ProcessLifecycleInitializer

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge" >
    <meta-data
        android:name="androidx.lifecycle.ProcessLifecycleInitializer"
        android:value="androidx.startup" />
</provider>
```

ProcessLifecycleInitializer

```java
public final class ProcessLifecycleInitializer implements Initializer<LifecycleOwner> {

    @NonNull
    @Override
    public LifecycleOwner create(@NonNull Context context) {
        LifecycleDispatcher.init(context);
        ProcessLifecycleOwner.init(context);
        return ProcessLifecycleOwner.get();
    }

    @NonNull
    @Override
    public List<Class<? extends Initializer<?>>> dependencies() {
        return Collections.emptyList();
    }
}
```

ProcessLifecycleOwner 用于今天Application 的生命周期的变化

```java

public class ProcessLifecycleOwner implements LifecycleOwner {

    @VisibleForTesting
    static final long TIMEOUT_MS = 700; //mls

    // ground truth counters
    private int mStartedCounter = 0;
    private int mResumedCounter = 0;

    private boolean mPauseSent = true;
    private boolean mStopSent = true;

    private Handler mHandler;
    private final LifecycleRegistry mRegistry = new LifecycleRegistry(this);

    private Runnable mDelayedPauseRunnable = new Runnable() {
        @Override
        public void run() {
            dispatchPauseIfNeeded();
            dispatchStopIfNeeded();
        }
    };

    ActivityInitializationListener mInitializationListener =
            new ActivityInitializationListener() {
                @Override
                public void onCreate() {
                }

                @Override
                public void onStart() {
                    activityStarted();
                }

                @Override
                public void onResume() {
                    activityResumed();
                }
            };

    private static final ProcessLifecycleOwner sInstance = new ProcessLifecycleOwner();

    /**
     * The LifecycleOwner for the whole application process. Note that if your application
     * has multiple processes, this provider does not know about other processes.
     *
     * @return {@link LifecycleOwner} for the whole application.
     */
    @NonNull
    public static LifecycleOwner get() {
        return sInstance;
    }

    static void init(Context context) {
        sInstance.attach(context);
    }

    void activityStarted() {
        mStartedCounter++;
        if (mStartedCounter == 1 && mStopSent) {
            mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
            mStopSent = false;
        }
    }

    void activityResumed() {
        mResumedCounter++;
        if (mResumedCounter == 1) {
            if (mPauseSent) {
                mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
                mPauseSent = false;
            } else {
                mHandler.removeCallbacks(mDelayedPauseRunnable);
            }
        }
    }

    void activityPaused() {
        mResumedCounter--;
        if (mResumedCounter == 0) {
            mHandler.postDelayed(mDelayedPauseRunnable, TIMEOUT_MS);
        }
    }

    void activityStopped() {
        mStartedCounter--;
        dispatchStopIfNeeded();
    }

    void dispatchPauseIfNeeded() {
        if (mResumedCounter == 0) {
            mPauseSent = true;
            mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);
        }
    }

    void dispatchStopIfNeeded() {
        if (mStartedCounter == 0 && mPauseSent) {
            mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP);
            mStopSent = true;
        }
    }

    private ProcessLifecycleOwner() {
    }

    @SuppressWarnings("deprecation")
    void attach(Context context) {
        mHandler = new Handler();
        mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
        Application app = (Application) context.getApplicationContext();
        app.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
            @RequiresApi(29)
            @Override
            public void onActivityPreCreated(@NonNull Activity activity,
                    @Nullable Bundle savedInstanceState) {
                // We need the ProcessLifecycleOwner to get ON_START and ON_RESUME precisely
                // before the first activity gets its LifecycleOwner started/resumed.
                // The activity's LifecycleOwner gets started/resumed via an activity registered
                // callback added in onCreate(). By adding our own activity registered callback in
                // onActivityPreCreated(), we get our callbacks first while still having the
                // right relative order compared to the Activity's onStart()/onResume() callbacks.
                activity.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
                    @Override
                    public void onActivityPostStarted(@NonNull Activity activity) {
                        activityStarted();
                    }

                    @Override
                    public void onActivityPostResumed(@NonNull Activity activity) {
                        activityResumed();
                    }
                });
            }

            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
                // Only use ReportFragment pre API 29 - after that, we can use the
                // onActivityPostStarted and onActivityPostResumed callbacks registered in
                // onActivityPreCreated()
                if (Build.VERSION.SDK_INT < 29) {
                    ReportFragment.get(activity).setProcessListener(mInitializationListener);
                }
            }

            @Override
            public void onActivityPaused(Activity activity) {
                activityPaused();
            }

            @Override
            public void onActivityStopped(Activity activity) {
                activityStopped();
            }
        });
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mRegistry;
    }
}
```

LifecycleDispatcher 用于为所有 Activity 注入ReportFragment，这样之后对于SDK 中的 Activity 只要实现 LifecycleOwner 就能实现生命周期可观察的能力。

```java
class LifecycleDispatcher {

    private static AtomicBoolean sInitialized = new AtomicBoolean(false);

    static void init(Context context) {
        if (sInitialized.getAndSet(true)) {
            return;
        }
        ((Application) context.getApplicationContext())
                .registerActivityLifecycleCallbacks(new DispatcherActivityCallback());
    }

    @SuppressWarnings("WeakerAccess")
    @VisibleForTesting
    static class DispatcherActivityCallback extends EmptyActivityLifecycleCallbacks {

        @Override
        public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            ReportFragment.injectIfNeededIn(activity);
        }

        @Override
        public void onActivityStopped(Activity activity) {
        }

        @Override
        public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }
    }

    private LifecycleDispatcher() {
    }
}
```

### 7.5 Service 实现

```java
class MyLifecycleService : LifecycleService() {
    init {
        lifecycle.addObserver(MyLifecycleServiceObserver())
    }
}
```

LifecycleService 

```java
public class LifecycleService extends Service implements LifecycleOwner {

    private final ServiceLifecycleDispatcher mDispatcher = new ServiceLifecycleDispatcher(this);

    @CallSuper
    @Override
    public void onCreate() {
        mDispatcher.onServicePreSuperOnCreate();
        super.onCreate();
    }

    @CallSuper
    @Nullable
    @Override
    public IBinder onBind(@NonNull Intent intent) {
        mDispatcher.onServicePreSuperOnBind();
        return null;
    }

    @SuppressWarnings("deprecation")
    @CallSuper
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        mDispatcher.onServicePreSuperOnStart();
        super.onStart(intent, startId);
    }

    // this method is added only to annotate it with @CallSuper.
    // In usual service super.onStartCommand is no-op, but in LifecycleService
    // it results in mDispatcher.onServicePreSuperOnStart() call, because
    // super.onStartCommand calls onStart().
    @CallSuper
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        return super.onStartCommand(intent, flags, startId);
    }

    @CallSuper
    @Override
    public void onDestroy() {
        mDispatcher.onServicePreSuperOnDestroy();
        super.onDestroy();
    }

    @Override
    @NonNull
    public Lifecycle getLifecycle() {
        return mDispatcher.getLifecycle();
    }
}
```

ServiceLifecycleDispatcher

```java
public class ServiceLifecycleDispatcher {
    private final LifecycleRegistry mRegistry;
    private final Handler mHandler;
    private DispatchRunnable mLastDispatchRunnable;

    /**
     * @param provider {@link LifecycleOwner} for a service, usually it is a service itself
     */
    @SuppressWarnings("deprecation")
    public ServiceLifecycleDispatcher(@NonNull LifecycleOwner provider) {
        mRegistry = new LifecycleRegistry(provider);
        mHandler = new Handler();
    }

    private void postDispatchRunnable(Lifecycle.Event event) {
        if (mLastDispatchRunnable != null) {
            mLastDispatchRunnable.run();
        }
        mLastDispatchRunnable = new DispatchRunnable(mRegistry, event);
        mHandler.postAtFrontOfQueue(mLastDispatchRunnable);
    }

    /**
     * Must be a first call in {@link Service#onCreate()} method, even before super.onCreate call.
     */
    public void onServicePreSuperOnCreate() {
        postDispatchRunnable(Lifecycle.Event.ON_CREATE);
    }

    /**
     * Must be a first call in {@link Service#onBind(Intent)} method, even before super.onBind
     * call.
     */
    public void onServicePreSuperOnBind() {
        postDispatchRunnable(Lifecycle.Event.ON_START);
    }

    /**
     * Must be a first call in {@link Service#onStart(Intent, int)} or
     * {@link Service#onStartCommand(Intent, int, int)} methods, even before
     * a corresponding super call.
     */
    public void onServicePreSuperOnStart() {
        postDispatchRunnable(Lifecycle.Event.ON_START);
    }

    /**
     * Must be a first call in {@link Service#onDestroy()} method, even before super.OnDestroy
     * call.
     */
    public void onServicePreSuperOnDestroy() {
        postDispatchRunnable(Lifecycle.Event.ON_STOP);
        postDispatchRunnable(Lifecycle.Event.ON_DESTROY);
    }

    /**
     * @return {@link Lifecycle} for the given {@link LifecycleOwner}
     */
    @NonNull
    public Lifecycle getLifecycle() {
        return mRegistry;
    }

    static class DispatchRunnable implements Runnable {
        private final LifecycleRegistry mRegistry;
        final Lifecycle.Event mEvent;
        private boolean mWasExecuted = false;

        DispatchRunnable(@NonNull LifecycleRegistry registry, Lifecycle.Event event) {
            mRegistry = registry;
            mEvent = event;
        }

        @Override
        public void run() {
            if (!mWasExecuted) {
                mRegistry.handleLifecycleEvent(mEvent);
                mWasExecuted = true;
            }
        }
    }
}
```



## 8. 链接

- [Github | Lifecycle](https://github.com/androidx/androidx/tree/androidx-main/lifecycle)

- [Android 开发者 | Lifecycle](https://developer.android.com/jetpack/androidx/releases/lifecycle)

- [用户指南](https://developer.android.com/topic/libraries/architecture/lifecycle) 

- [代码示例](https://github.com/android/architecture-components-samples) 

- [Codelab](https://codelabs.developers.google.com/codelabs/android-lifecycles/index.html?index=#0)
- [Lifecycle 版本说明](https://developer.android.com/jetpack/androidx/releases/lifecycle#declaring_dependencies)
- [googlecodelabs | android-lifecycles](https://github.com/googlecodelabs/android-lifecycles)
- [What is lifecycle observer and how to use it correctly?](https://stackoverflow.com/questions/52369540/what-is-lifecycle-observer-and-how-to-use-it-correctly)
