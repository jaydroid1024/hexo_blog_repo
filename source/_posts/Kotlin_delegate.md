---
title: Kotlin | 委托(Delegation﻿)详解
date: 2021-08-31 14:16:55
cover: true
tags: 
    - Kotlin 委托
    - 属性委托
    - ViewBinding
    - ViewModel
category: 
	- Kotlin
summary: 委托模式与代理模式、接口委托、属性委托、映射委托、延迟属性、非空属性、可观察属性、ViewBinding+属性委、ViewModel+属性委托、SP+属性委托




---

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/post_banner_jay.png)

# Kotlin | 委托(Delegation﻿)详解



本文要点概述

- 辨析委托模式与代理模式 

- 接口委托(Delegated interface)

- 属性委托(Delegated properties)

- 映射委托(Map delegation)

- 延迟属性(lazy properties)

- 非空属性(Delegates.notNull)

- 变量值更新后的监听(Delegates.observable)

- 变量值更新前的拦截(Delegates.vetoable)

- ViewBinding+属性委托

- ViewModel+属性委托

- SP +属性委托

  

## 1.委托模式 VS 代理模式 

委托模式和代理模式都属于结构型设计模式，结构型模式主要总结了一些类或对象组合在一起的经典结构，这些经典的结构可以解决特定应用场景的问题。

### 1.1 代理模式（Proxy Pattern）

在不改变原始类接口的条件下，为原始类定义一个代理类，主要目的是控制访问，而非加强功能，这是它跟装饰器模式最大的不同。在代理类中还可以提供额外的逻辑， 例如当真实对象上的操作是资源密集型任务时可以在代理中添加缓存操作，或者在调用真实对象操作之前做一些权限校验或检查行为等。一般情况下，我们让代理类和原始类实现同样的接口或者让代理类继承原始类来实现代理模式。

### 1.2 委托模式（Delegation pattern）

委托可以理解为是代理的一种变体，是一种使组合像继承一样实现代码复用的另一种设计模式。主要目的是组合委托调用和代码复用而不考虑控制访问等逻辑。在委托中处理请求涉及两个对象，委托方将操作交给受托方实现，类似于子类将请求推迟到父类。一般情况下，委托也可以通过接口约束或受托方继承委托方实现委托模式。

### 1.3 法律层面区分代理与委托

为了加深代理和委托的理解，我们看一下**法律层面的代理与委托的不同之处**

- 民事主体活动的名义不同。 代理是指被代理人在代理权限范围内，以被代理人的名义同第三人独立为民事法律行为，由此产生的法律效果直接归属于被代理人的一种法律制度，即代理人必须以被代理人的名义为代理行为。 委托则是委托人委托受托人处理一定事务，受托人接受委托的协议，受托人可以以委托人的名义活动，也可以以自己的名义活动。

- 适用范围不尽相同：代理只是代理人在代理权范围内以被代理人的名义同第三人的民事法律行为；而委托中的受托人办理委托人委托事务的行为可以是民事法律行为，还可以是有经济意义的行为（如整理账簿）和单纯的事实行为（如抄写文件）。

- 效力范围不同：代理涉及三方当事人，即被代理人、代理人、第三人；委托则属于双方当事人之间的关系，即委托人、受托人。

### 1.4 代码实现委托和代理

代理模式和委托模式在代码实现上并没有太多差别，他们的差异还是在使用场景上，为了能更好的理解代理模式和委托模式，我们再来看看如何通过代码来实现

代码参考自：**[ java-design-patterns](https://github.com/iluwatar/java-design-patterns)**

[delegation](https://github.com/iluwatar/java-design-patterns/tree/master/delegation) ：打印机控制器将打印任务委托给不同的打印机

```java
// 委托二人组：打印机控制器、打印机（惠普打印机、佳能打印机、爱普生打印机）
// 电脑上有三种打印机设备的驱动，分别将打印任务委托给对应的打印机执行具体的打印操作
PrinterController hpPrinterController = new PrinterController(new HpPrinter());
PrinterController canonPrinterController = new PrinterController(new CanonPrinter());
PrinterController epsonPrinterController = new PrinterController(new EpsonPrinter());
hpPrinterController.print(MESSAGE_TO_PRINT);
canonPrinterController.print(MESSAGE_TO_PRINT);
epsonPrinterController.print(MESSAGE_TO_PRINT);

//打印机控制器
public class PrinterController implements Printer {
  private final Printer printer;
  public PrinterController(Printer printer) {
    this.printer = printer;
  }
  /** 
   * 此方法是从 {@link Printer} 实现的，在提供实现时它会调用通过构造函数传递的委托方的 print 方法。 
   * 这意味着委托关系一旦确定后，调用者不关心实现类，只关心拥有的打印机控制器就行。
   */
  @Override
  public void print(String message) {
    printer.print(message);
  }
}

//三个打印机也实现了 Printer 接口 并提供了具体的打印功能，代码可以参考上面的链接
```

[proxy](https://github.com/iluwatar/java-design-patterns/tree/master/proxy) ：巫师要进入塔内修炼法术，象牙塔只能通过代理访问并确保只有前三个巫师可以进入。

```java
// 代理三人组：巫师塔代理、象牙塔、巫师
WizardTowerProxy proxy = new WizardTowerProxy(new IvoryTower());
proxy.enter(new Wizard("Red wizard"));
proxy.enter(new Wizard("White wizard"));
proxy.enter(new Wizard("Black wizard"));
proxy.enter(new Wizard("Green wizard"));
proxy.enter(new Wizard("Brown wizard"));

//代理方：塔代理
public class WizardTowerProxy implements WizardTower {
  private static final int NUM_WIZARDS_ALLOWED = 3;
  private final WizardTower tower;
  private int numWizards;
  //被代理方通过构造方法传入
  public WizardTowerProxy(WizardTower tower) {
    this.tower = tower;
  }
  //对象牙塔的访问的代理。第三方通过 enter 方法进入
  @Override
  public void enter(Wizard wizard) {
    //超过三个就不让进了
    if (numWizards < NUM_WIZARDS_ALLOWED) {
      //代理方授权执行操作
      tower.enter(wizard);
      numWizards++;
    } else {
      System.out.println("{} is not allowed to enter!" + wizard);
    }
  }
}
//被代理方：塔
public class IvoryTower implements WizardTower {
  @Override
  public void enter(Wizard wizard) {
    System.out.println("{} enters the tower." + wizard);
  }
}
//第三方：巫师
public class Wizard {
  private String name;
  public Wizard(String wizard) { this.name = wizard; }
  @Override
  public String toString() { return name; }
}

```



## 2. 接口委托(Delegated interface)

我们看看Kotlin 是怎么内置接口委托的，先看一下实现的委托模式的传统方式

```kotlin
interface Api {
    fun a()
    fun b()
    fun c()
}
//委托方
class ApiImpl : Api {
    override fun a() { println("ApiImpl-a") }
    override fun b() { println("ApiImpl-b") }
    override fun c() { println("ApiImpl-c") }
}
//受托方
class ApiWrapper(private val api: Api) : Api {
    override fun a() {
        println("ApiWrapper-a")
        api.a()
    }
    override fun b() {
        println("ApiWrapper-b")
        api.b()
    }
    override fun c() {
        println("ApiWrapper-c")
        api.b()
    }
}
```

再看看Kotlin 通过 `by` 关键字实现的简便方式

```kotlin
//变量 api代替 ApiWrapperWithDelegate 实现了 Api 接口，ApiWrapperWithDelegate 就可以灵活的复写需要的函数
class ApiWrapperWithDelegate(private val api: Api) : Api by api {
    override fun a() {
        println("ApiWrapperWithDelegate-a")
    }
}


//反编译 Java 后的代码和 ApiWrapper 是一样的
public final class ApiWrapperWithDelegate implements Api {
   private final Api api;
   public void a() {
      String var1 = "ApiWrapperWithDelegate-a";
      boolean var2 = false;
      System.out.println(var1);
   }
   public ApiWrapperWithDelegate(@NotNull Api api) {
      Intrinsics.checkNotNullParameter(api, "api");
      super();
      this.api = api;
   }
   public void b() {
      this.api.b();
   }
   public void c() {
      this.api.c();
   }
}
```

可以看到 Kotlin 内置的接口委托是编译器帮我们生成了相关代码

再看一个实践的例子：利用接口代理实现一个集成了 map 和 list 的超级集合

```kotlin
// 对象 list 和 map 代理 SupperArrayWithDelegate 实现 MutableList，MutableMap
class SupperArrayWithDelegate<E>(
    private val list: MutableList<E?> = mutableListOf(),
    private val map: MutableMap<String, E> = mutableMapOf()
) : MutableList<E?> by list, MutableMap<String, E> by map {
    // 两个接口中都有，编译器不知道执行哪个，所以这些方法必须得重写
    override fun clear() {
        list.clear()
        map.clear()
    }
    override fun isEmpty(): Boolean {
        return list.isEmpty() && map.isEmpty()
    }
    override val size: Int get() = list.size + map.size
    override fun set(index: Int, element: E?): E? {
        if (index <= list.size) {
            repeat(index - list.size - 1) {
                list.add(null)
            }
        }
        return list.set(index, element)
    }
    override fun toString(): String {
        return "list:$list,map:$map"
    }
}
```



## 3. 属性委托(Delegated properties)



### 3.1 属性(property)

我们先通过对比 Java field 和 kotlin property  来探究一下 kt 中 property 的内部实现方式

```java
//PersonKotlin 尽量写的像Java好对比 Kotlin 属性背后做的事情
class PersonKotlin {
    constructor(age: Int, name: String) {
        this.age = age
        this.name = name
    }
    private var age: Int? = null
        //Redundant getter 属性的 get/set 方法由编译器自动生成
        get() {
            return field //这里的 field = backing field
        }
        set(value) {
            field = value
        }

    private var name: String? = null
        get() {
            return field
        }
        set(value) {
            field = value
        }
}

//反编译 Java 后的代码
public final class PersonKotlin {
   private Integer age; //field
   private String name; //field
   private final Integer getAge() {
      return this.age;
   }
   private final void setAge(Integer value) {
      this.age = value;
   }
   private final String getName() {
      return this.name;
   }
   private final void setName(String value) {
      this.name = value;
   }
   public PersonKotlin(int age, @NotNull String name) {
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.setAge(age);
      this.setName(name);
   }
}
```

#### property(kotlin)=field(java)+getField()+setField()

```java
// age 属性背后包含了三个角色，backing field、get、set
private var age: Int? = null
//等价于下面的代码
@Nullable
private Integer age;
@Nullable
public final Integer getAge() {
   return this.age;
}
public final void setAge(@Nullable Integer var1) {
   this.age = var1;
}
```

#### 属性引用(Property Reference)

通过属性引用我们可以更清楚的了解 property 背后的 get 和 set 以及代理信息等

```kotlin
val ageRef: KMutableProperty1<PersonKotlin, Int?> = PersonKotlin::age
//PersonKotlin::age 类名获取的属性引用不包含 receiver，操作时需要传递一个
val personKotlin = PersonKotlin(18, "Jay")
ageRef.set(personKotlin, 22)
println(ageRef.get(personKotlin))
//22
//public actual fun set(receiver: T, value: V)
//public actual fun get(receiver: T): V
//receiver - 用于获取属性值的接收器。 例如，如果这是该类的成员属性，则它应该是类实例，如果这是顶级扩展属性，则它应该是扩展接收器

//测试自定义属性委托后获取属性的委托信息
val nameRef: KProperty0<String?> = personKotlin::name
println("personKotlin: " + personKotlin.hashCode())
println("nameRef: " + nameRef.hashCode())
println("personKotlin.name: " + personKotlin.name.hashCode())

//取： is KMutableProperty -> javaField?.isAccessible ?: true && javaGetter?.isAccessible ?: true &&javaSetter?.isAccessible ?: true
//存： is KMutableProperty -> { javaField?.isAccessible = value javaGetter?.isAccessible = value javaSetter?.isAccessible = value }
//设置是否访问，只有设置为 true 才可以拿到 属性引用中的委托信息（如果被委托了）这个属性需要单独引入 kotlin-reflect 库
nameRef.isAccessible = true
//如果这是一个委托属性，则返回委托的值，如果此属性未委托，则返回null
val nameDelegate = nameRef.getDelegate()
println("nameDelegate： $nameDelegate") //返回委托信息
println(nameRef.getter.invoke()) //相当于调用 get 方法
//personKotlin: 1751075886
//nameRef: -954731934
//thisRef:1751075886
//property:-954731934
//personKotlin.name: 88377
//可以看到属性引用类和它的 receiver 在委托类和这里的 hashCode 相同

//com.jay.lib_kotlin.delegate.MyDelegate@5a63f509
//YYY

//测试lazy 属性代理方式
val sexRef: KProperty0<String?> = personKotlin::sex
personKotlin.sex
sexRef.isAccessible = true
println(sexRef.getDelegate())
//获取的代理信息就是lazy代码块中的值：sex is male


//测试 属性引用的类型
val kMutableProperty0: KMutableProperty0<Int> = ::sex //sex 是顶级属性
val s = kMutableProperty0 as CallableReference
println(s.owner) //file class com.jay.lib_kotlin.property.PersonKotlinKt
//属性引用的类型是 CallableReference
```

**receiver**：属性值的接收器。 例如，如果这是该类的成员属性，则它应该是类实例，如果这是顶级扩展属性，则它应该是扩展接收器

**CallableReference**：是 Kotlin 编译器为可调用引用类生成的所有类的超类



### 3.2 属性委托实现原理

```kotlin
public open class FooBy {
    //只要在by关键字后面带有一个委托对象，这个对象不一定要实现特定的接口，只要包含了getValue/setValue方法、那它就能作为一个代理属性来使用。
    val y by MyDelegate()
    var w: String by MyDelegate()
}

class MyDelegate {
    var value: String = "YYY"
    //todo 委托类里面必须提供 getValue 方法，或者扩展这个方法也可
    operator fun getValue(thisRef: Any, property: KProperty<*>): String {
        return value
    }
    operator fun setValue(thisRef: Any, property: KProperty<*>, s: String) {
        value = s
    }
}
```

反编译Java后的代码

```java
public class FooBy {
   // $FF: synthetic field
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.property1(new PropertyReference1Impl(FooBy.class, "y", "getY()Ljava/lang/String;", 0)), (KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(FooBy.class, "w", "getW()Ljava/lang/String;", 0))};
   @NotNull
   private final MyDelegate y$delegate = new MyDelegate();
   @NotNull
   private final MyDelegate w$delegate = new MyDelegate();
   @NotNull
   public final String getY() {
      return this.y$delegate.getValue(this, $$delegatedProperties[0]);
   }
   @NotNull
   public final String getW() {
      return this.w$delegate.getValue(this, $$delegatedProperties[1]);
   }

   public final void setW(@NotNull String var1) {
      Intrinsics.checkNotNullParameter(var1, "<set-?>");
      this.w$delegate.setValue(this, $$delegatedProperties[1], var1);
   }
}
```

当调用下面的代码时，就会调用到 FooBy.y 的 get 方法

```
val foo = FooBy()
println(foo.y)
```

看一下反编译后的 getY 方法，

```java
@NotNull
public final String getY() {
   return this.y$delegate.getValue(this, $$delegatedProperties[0]);
}
```

看一下 `y$delegate` 是什么 ,其实就是我们的代理类并在 FooBy 类构建的时候已经初始化

```java
@NotNull
private final MyDelegate y$delegate = new MyDelegate();
```

再看看MyDelegate中的的 getValue 方法, 就是我们在代理类中必须提供的方法

```java
//Java code
@NotNull
public final String getValue(@NotNull Object thisRef, @NotNull KProperty property) {
   Intrinsics.checkNotNullParameter(thisRef, "thisRef");
   Intrinsics.checkNotNullParameter(property, "property");
   return this.value;
}
```

到这里我们就可以看清了整个委托流程了

- 当类中有属性被委托时，Kotlin 会在当前类中添加委托类的实例并在实例化当前类时实例化委托类( y$delegate)，同时$$delegatedProperties 数组也是在类初始化时创建完成，里面方式所有属性的反射类信息
- 当要获取委托属性时，会调用到它的 get 方法，而 get 方法返回的是代理类的 getValue 方法
- getValue 方法是我们自己实现的，最终代理属性就会通过 getValue  方法赋上值了
- setValue 时还会把 属性 的backing field 传过去

### 3.3 PropertyReferenceImpl 

委托流程搞清楚了，我们再来看看 getValue 方法中 `thisRef: Any`， `property: KProperty<*>` 这两个参数是怎么来的，干什么用的

thisRef 这个参数是业务类本身可以看到就是在调用 getValue 方法时传递的 this

property 是委托属性的描述类 `KProperty` ,它是从这个数组里取的 `$$delegatedProperties[0]`，这个数组也是构建业务类时由Kotlin自动生成的，存放的是描述类属性的 KProperty 类型

`Reflection.property1`  是一个工厂函数，将传入的参数返回

PropertyReference1Impl 的父类也间接实现了 KProperty 接口，所以这里可以强转

```java
//$$delegatedProperties
static final KProperty[] $$delegatedProperties =
  new KProperty[]{(KProperty)Reflection.property1(
  new PropertyReference1Impl(FooBy.class, "y", "getY()Ljava/lang/String;", 0)), (KProperty)Reflection.mutableProperty1(
  new MutablePropertyReference1Impl(FooBy.class, "w", "getW()Ljava/lang/String;", 0))};

```

再看一下属性引用实现类 PropertyReference1Impl  的构造参数 

PropertyReference1Impl 类的构造器最终会调用到它的父类  CallableReference

**CallableReference**：是 Kotlin 编译器为可调用引用类生成的所有类的超类。

```kotlin
@SinceKotlin(version = "1.4")
public PropertyReference1Impl(Class owner, String name, String signature, int flags) {
    super(NO_RECEIVER, owner, name, signature, flags);
}
//NO_RECEIVER 如果属性没有 receiver 构造时会缺省添加一个 NO_RECEIVER
@SinceKotlin(version = "1.1")
public static final Object NO_RECEIVER = NoReceiver.INSTANCE;
@SinceKotlin(version = "1.2")
private static class NoReceiver implements Serializable {
    private static final NoReceiver INSTANCE = new NoReceiver();

    private Object readResolve() throws ObjectStreamException {
        return INSTANCE;
    }
}

//CallableReference
@SinceKotlin(version = "1.4")
protected CallableReference(Object receiver, Class owner, String name, String signature, boolean isTopLevel) {
    this.receiver = receiver; //可调用对象的属性值的接收器。 例如：类实例
    this.owner = owner; //可调用对象所在的类或包
    this.name = name; //可调用对象的 Kotlin 名称，即在源代码中声明的名
    this.signature = signature; //可调用对象的 JVM 签名。如果这是一个属性引用，则返回其 getter 的 JVM 签名，例如“getFoo(LjavalangString;)I”。
    this.isTopLevel = isTopLevel; //是否高等类型(文件中还是类中)，0 false; 1 true
}
```

利用Java的实现方式简单总结以下Kotlin 属性委托的背后原理

```java
class Person {
  static final Field[] delegatedProperties = Person.class.getFields();
  private final NameDelegate nameDelegate = new NameDelegate();
  public final String getName() {
    return this.nameDelegate.getValue(this, delegatedProperties[0]);
  }
}
class NameDelegate {
  String getValue(Person thisRef, Field property) {
    return "Jay";
  }
}
```



### 3.4 简化属性委托的内置接口们

Kotlin 内置的属性委托功能是**属性委托类**，不能像普通的委托模式一样通过接口或集成的方式来约束交互的方法和类型，做不了两方约束，但是可以通过泛型+接口约束一下委托类，也能达到一部分约束的效果。

Kotlin 标准库中提供了三个接口来简化委托类的实现

```kotlin
//val 属性
public fun interface ReadOnlyProperty<in T, out V>
//var 属性
public interface ReadWriteProperty<in T, V> : ReadOnlyProperty<T, V>
//创建委托类的工厂接口
public fun interface PropertyDelegateProvider<in T, out D>
//T：拥有委托属性的对象类型。 
//V：属性值的类型。
//D：委托类的类型
```

看一下三个接口的接口和方法签名

```kotlin
public fun interface ReadOnlyProperty<in T, out V> {
    public operator fun getValue(thisRef: T, property: KProperty<*>): V
}
public interface ReadWriteProperty<in T, V> : ReadOnlyProperty<T, V> {
    public override operator fun getValue(thisRef: T, property: KProperty<*>): V
    public operator fun setValue(thisRef: T, property: KProperty<*>, value: V)
}
@SinceKotlin("1.4")
public fun interface PropertyDelegateProvider<in T, out D> {
    public operator fun provideDelegate(thisRef: T, property: KProperty<*>): D
}
//前两个直接用就行，看一个 PropertyDelegateProvider 的使用场景
private val provider = PropertyDelegateProvider<FooBy, MyDelegate> { thisRef, property ->
    if (thisRef.y == "YYY") {
        MyDelegate()
    } else {
        MyDelegate2() //MyDelegate2:MyDelegate
    }
}
```

看一下他们几个综合使用的情况，同时也可以看到kt语音的强大，同样的功能,代码可以从十几行到三行再到一行。yyds!!!

```kotlin
val provider1 = object : PropertyDelegateProvider<FooReadWrite, ReadWriteProperty<FooReadWrite, Int>> {
        override fun provideDelegate(
            thisRef: FooReadWrite,
            property: KProperty<*>
        ): ReadWriteProperty<FooReadWrite, Int> {
            return object : ReadWriteProperty<FooReadWrite, Int> {
                var result=1024
                override fun getValue(thisRef: FooReadWrite, property: KProperty<*>): Int {
                    return result
                }
                override fun setValue(thisRef: FooReadWrite, property: KProperty<*>, value: Int) {
                    result=value
                }
            }
        }
    }

//lambda 简化版本
val provider2: PropertyDelegateProvider<FooReadWrite, ReadOnlyProperty<FooReadWrite, Int>> =
    PropertyDelegateProvider<FooReadWrite, ReadOnlyProperty<FooReadWrite, Int>> { pThisRef: Any?, pProperty: KProperty<*> ->ReadOnlyProperty<Any?, Int> { thisRef, property -> 1025 }
    
//智能类型推导再简化版本
private val provider3 =PropertyDelegateProvider { _: Any?, _ -> ReadOnlyProperty<Any?, Int> { _, _ -> 1026 } }

val delegate1: Int by provider1
val delegate2: Int by provider2
val delegate3: Int by provider3
```



## 4. 属性委托在 Kotlin Api 中的运用

Kotlin 标准库中提供了几种委托

- 映射委托(Map delegation)
- 延迟属性（lazy properties）: 其值只在首次访问时计算；
- 可观察属性（observable properties）: 监听器会收到有关此属性变更的通知；
- 非空属性(Delegates.notNull)

### 4.1 映射委托(Map delegation)

看一个map 作为属性委托方的示例

```kotlin
class User {
    //委托 val
    val map: Map<String, Any?> = mapOf("name2" to "Jay", "age" to 18)

    //可变 map 可以委托 val和var
    val map2: MutableMap<String, Any?> = mutableMapOf("name2" to "Jay", "age" to 18)
    val name: String by map
    var age: Int by map2

    //更新 age 的值，MutableMap 也会同步更新
    fun setValues(age: Int) {
        this.age = age
    }
}
```

map 中的 key 必须包含属性名，否则会报下面这个错

```kotlin
Exception in thread "main" java.util.NoSuchElementException: Key name2 is missing in the map.
```

所以在使用这个特性时除非我们完全确定支持映射的结构，否则应该避免基于映射的属性委托，要不然委托的类可能会失败并抛出异常

还有一种情况当value 为 null 时，只有第四种情况会发生：NullPointerException ，这个问题想了解的可以官方的 [bug report](https://youtrack.jetbrains.com/issue/KT-27672)

```kotlin
val map: HashMap<String, Any?> = hashMapOf("name" to null, "age" to null)
val name: String by map
val name: String？ by map
var age: Int? by map
var age: Int by map
```

再来窥探一下 Map delegation 的委托原理

```java
// $FF: synthetic field
static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.property1(new PropertyReference1Impl(User.class, "name", "getName()Ljava/lang/String;", 0)), (KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(User.class, "age", "getAge()Ljava/lang/Integer;", 0))};
@Nullable
private final Map age$delegate;
@Nullable
public final Integer getAge() {
   Map var1 = (Map)this.age$delegate;
   KProperty var3 = $$delegatedProperties[1];
   boolean var4 = false;
   return (Integer)MapsKt.getOrImplicitDefaultNullable(var1, var3.getName());
}
public final void setAge(@Nullable Integer var1) {
   Map var2 = (Map)this.age$delegate;
   KProperty var4 = $$delegatedProperties[1];
   boolean var5 = false;
   var2.put(var4.getName(), var1);
}
```

可以看到，Kotlin 编译器 也生成了KProperty[] 类型的 $$delegatedProperties 和 Map 类型 age$delegate，并在构造时实例化age$delegate 

Map相关的委托必要方法在**[MapAccessors](https://github.com/JetBrains/kotlin/blob/34e57a45f2/libraries/stdlib/src/kotlin/collections/MapAccessors.kt)** 这个类里面

```kotlin
//Map
@kotlin.internal.InlineOnly
public inline operator fun <V, V1 : V> Map<in String, @Exact V>.getValue(thisRef: Any?, property: KProperty<*>): V1 =@Suppress("UNCHECKED_CAST") (getOrImplicitDefault(property.name) as V1)
//MutableMap
@kotlin.jvm.JvmName("getVar")
@kotlin.internal.InlineOnly
public inline operator fun <V, V1 : V> MutableMap<in String, out @Exact V>.getValue(thisRef: Any?, property: KProperty<*>): V1 = @Suppress("UNCHECKED_CAST") (getOrImplicitDefault(property.name) as V1)
//MutableMap
@kotlin.internal.InlineOnly
public inline operator fun <V> MutableMap<in String, in V>.setValue(thisRef: Any?, property: KProperty<*>, value: V) {
    this.put(property.name, value)
}
```

在访问 age 的 get 时会调用委托 Map 的    `(Integer)MapsKt.getOrImplicitDefaultNullable(var1, var3.getName());`

在访问 age 的 set 时直接调用委托 Map 的put方法

下面是 **getOrImplicitDefaultNullable** 函数

```kotlin
//JvmName 这个注解是指定从此元素生成的 Java 类或方法的名称。
//扩展方法编译后会将方法的 reciver 作为第一个参数传入
@kotlin.jvm.JvmName("getOrImplicitDefaultNullable")
@PublishedApi
internal fun <K, V> Map<K, V>.getOrImplicitDefault(key: K): V {
    if (this is MapWithDefault)return this.getOrImplicitDefault(key)
    return getOrElseNullable(key, { throw NoSuchElementException("Key $key is missing in the map.") })
}
```

关于 map 的 put 和 get 操作是如何与委托的 getValue 和 setValue 如何联系在一起的 以及map 的 `getValue(thisRef: Any?, property: KProperty<*>) `方法为什么用 inline 修饰了，这里涉及到Kotlin 1.4 对委托属性的一个优化，稍后再解析 lazy 原理时会详细解释。

Map delegation 的一个实践，将推送消息封装并通知APP

```kotlin
override fun onNotificationReceivedInApp(
    context: Context,
    title: String,
    summary: String,
    extraMap: Map<String, String>,
) {
    val data = extraMap.withDefault { "" }
    val params = NotificationParams(data)
    EventBus.getDefault().post(params)
}

class NotificationParams(val map: Map<String, String>) {
    val title: String by map
    val content: String by map
}
```

### 4.2 延迟属性(lazy properties)

看一下lazy的简单使用

```kotlin
val x: String by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    println("xxx——lazy")
    "xxx——${index++}"
}
val y: String by lazy(LazyThreadSafetyMode.PUBLICATION) {
    println("yyy——lazy")
    "yyy——${index++}"
}
val z: String by lazy(LazyThreadSafetyMode.NONE) {
    println("zzz——lazy")
    "zzz——${index++}"
}
```

```kotlin
val fooLazy = FooLazy()
for (i in 1..100) {
    val thread = thread(true) {
        Thread.sleep(100)
        println(Thread.currentThread().name)
        println("=" + fooLazy.x)
        println("====" + fooLazy.y)
        println("=========" + fooLazy.z)
    }
}

//xxx——lazy 和 yyy——lazy 都只执行一次，x=0, y=1
//zzz——lazy 和 z 的值不能确定会执行几次
```

LazyThreadSafetyMode 有三种模式作用是指定 [Lazy] 实例如何在多个线程之间同步初始化。

- SYNCHRONIZED: 锁用于确保只有一个线程可以初始化[Lazy]实例。
- PUBLICATION: 并发访问未初始化的[Lazy]实例值时，可以多次调用Initializer函数，但是只有第一个返回的值将用作[Lazy]实例的值。
- NONE: 不使用锁来同步对 [Lazy] 实例值的访问；如果从多个线程访问该实例，可能会发生线程安全问题。除非保证 [Lazy] 实例永远不会从多个线程初始化，否则不应使用此模式。

#### lazy 原理解析

受托对象是Lazy

```kotlin
@NotNull
private final Lazy x$delegate;
```

受托对象在委托者构造方法中实例化

```kotlin
public FooLazy() {
   this.x$delegate = LazyKt.lazy(LazyThreadSafetyMode.SYNCHRONIZED, (Function0)(new Function0() {
      // $FF: synthetic method
      // $FF: bridge method
      public Object invoke() {
         return this.invoke();
      }
      @NotNull
      public final String invoke() {
         String var1 = "xxx——lazy";
         boolean var2 = false;
         System.out.println(var1);
         StringBuilder var10000 = (new StringBuilder()).append("xxx——");
         FooLazy var10001 = FooLazy.this;
         int var3;
         var10001.index = (var3 = var10001.index) + 1;
         return var10000.append(var3).toString();
      }
   }));
}
```

可以看到 `x$delegate`  是通过 LazyKt.lazy() 方法实例化的，两个参数分别是线程安全模式类型和一个接口回调

当调用x 的 get 方法时 反回了受托者的 getValue 方法 并没有调用 lazy 的扩展方法：LazyKt.getValue(thisRef: Any?, property: KProperty<*>) 

```java
public final String getX() {
   Lazy var1 = this.x$delegate;
   return (String)var1.getValue();
}
```

再看一下 lazy 是如何定义委托方法 getValue 的

```kotlin
@kotlin.internal.InlineOnly
public inline operator fun <T> Lazy<T>.getValue(thisRef: Any?, property: KProperty<*>): T = value
```

这里有没有注意到 lazy 利用属性委托的方式是不同的

- 没有自动生成属性数组  `KProperty[] $$delegatedProperties` 
- getX 时最终返回时调用的也不是 `getValue(thisRef: Any?, property: KProperty<*>)`
- lazy 的  `getValue(thisRef: Any?, property: KProperty<*>)` 方法是用 **inline** 修饰的 并且添加了`@kotlin.internal.InlineOnly` 注解，map 委托 也是这样的操作

其实这里是Kotlin 1.4 做的优化，当某些委托属性不会使用 KProperty。对于他们来说，在 `$$delegatedProperties` 中生成一个KProperty对象是多余的。Kotlin 1.4 版本将优化此类情况。如果委托的属性运算符是内联的，并且没有使用 KProperty 参数，则不会生成相应的反射对象。如果委托属性中有没有采用 inline 修饰的 ， 最终生成的`$$delegatedProperties`  数组中也之会单独生成它自己的反射对象，详细说明可以看官方的这篇博客

[What to Expect in Kotlin 1.4 and Beyond | Optimized delegated properties](https://blog.jetbrains.com/kotlin/2019/12/what-to-expect-in-kotlin-1-4-and-beyond/)

> 内联实际上是如何工作的？
>
> 粗略地说，内联采用被内联的函数的字节码，并将其插入到调用处，因此内联函数声明不必在调用处可见。
>
> `@kotlin.internal.InlineOnly` 注解的作用？
>
> `InlineOnly` 意味着与此 Kotlin 函数对应的 Java 方法被标记为私有，因此 Java 代码无法访问它（这是调用内联函数而不实际内联它的唯一方法）。这个注释还没有得到很好的验证，官方目前只在内部使用，很有可能稍后将其公之于众。

所以 lazy 和 map 的属性委托在 Kotlin 4.1 都是做了优化的，lazy 属性在调用 getter 时实际上是调用的的是 Lazy<T> 中 value 的 getter，map 属性在调用 getter/setter 时 实际上最终调用的也是 map 的 get/put 方法。

看一下 lazy 函数签名

```kotlin
public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }
```



#### SynchronizedLazyImpl

SynchronizedLazyImpl 采用 DCL 方式确保线程安全

```kotlin
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile // 用内存可见性来检查是否在其他线程初始化过，同时也会禁止指令重排序防止_value拿到不完整的实例
    private var _value: Any? = UNINITIALIZED_VALUE
    //实例使用自身进行同步
    private val lock = lock ?: this
    //Lazy 接口的 value 属性用于获取当前 Lazy 实例的延迟初始化值。一旦初始化后，它不得在此 Lazy 实例的剩余生命周期内更改。
    val value: T
        // 重写 get 来保证懒加载，只在使用的时候才执行函数
        get() {
            //局部变量可以将性能提高25%以上
            val _v1 = _value
            //检查单例实例是否已初始化。如果它被初始化就返回实例。
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }
            //到这里还没有初始化，但我们不能确定，因为可能有多个其他线程可能同时初始化了它。
            //所以为了以防万一，这里需要添加一把互斥锁来保证只有一个线程去实例化实例对象。
            return synchronized(lock) {
                //再次将实例分配给局部变量以检查它是否被其他线程初始化，而当前线程被阻止进入锁定区域。
                val _v2 = _value
                //如果它已经被其它线程初始化了，当前线程也能感知他的存在了，直接返回实例
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST")
                    (_v2 as T)
                } else {
                    //到这里该实例仍未初始化，因此我们可以安全地（没有其他线程可以进入该区域）创建一个实例了。
                    val typedValue = initializer!!() //执行 Function 对象的 invoke 并将函数的返回值缓存起来
                    _value = typedValue //_value赋值通知其它线程别进来了，拿走用吧
                    initializer = null //initializer在当前类实例已经没用了
                    typedValue // 返回最终的结果给 value
                }
            }
        }
}
```

看不惯这种DCL也可以恢复成传统方式看一下

```kotlin
val value: T
    get() {
        //局部变量将性能提高了 25% Joshua Bloch “Effective Java, Second Edition”，第 3 页。 283-284
        var _v1 = _value
        if (_v1 == UNINITIALIZED_VALUE) {
            synchronized(lock) {
                // 再次将实例分配给局部变量以检查它是否被其他线程初始化，而当前线程被阻止进入锁定区域。
                // 如果它被初始化，当前线程也能感知他的存在了。
                _v1 = _value
                if (_v1 == UNINITIALIZED_VALUE) {
                    // 该实例仍未初始化，因此我们可以安全地（没有其他线程可以进入该区域）创建一个实例并将其赋值给我们的单例引用。
                    _v1 = initializer!!()
                    initializer = null
                }
            }
        }
        @Suppress("UNCHECKED_CAST")
        return _v1 as T
    }
```



#### SafePublicationLazyImpl

**AtomicReferenceFieldUpdater** ：原子更新器是基于反射的工具类，用来对某个类中，被volatile修饰的字段进行原子更新。

通过调用AtomicReferenceFieldUpdater的静态方法`newUpdater`就能创建它的实例，该方法要接收三个参数：包含该字段所在的类、将被更新的对象的类型、将被更新的字段的名称

`compareAndSet` 如果期望值和字段当前值相等，说明目前是最新的值可以进行更新返回 true 同时原子地将字段设置为给定的更新值。

`getAndSet`原子地将此更新程序管理的给定对象的字段设置为给定值并返回旧值。

原子更新器的使用存在比较苛刻的条件如下

- 操作的字段不能是static类型。
- 操作的字段不能是final类型的，因为final根本没法修改。
- 字段必须是volatile修饰的，也就是数据本身是读一致的。
- 属性必须对当前的Updater所在的区域是可见的，如果不是当前类内部进行原子更新器操作不能使用private，protected子类操作父类时修饰符必须是protect权限及以上，如果在同一个package下则必须是default权限及以上，也就是说无论何时都应该保证操作类与被操作类间的可见性。

> CAS，Compare and Swap即比较并交换，设计并发算法时常用到的一种技术，java.util.concurrent包全完建立在CAS之上，没有CAS也就没有此包，可见CAS的重要性。当前的处理器基本都支持CAS，只不过不同的厂家的实现不一样罢了。**CAS有三个操作数：内存值V、旧的预期值A、要修改的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做并返回false**。
>
> Unsafe，JDK中的一个类，它提供了硬件级别的**原子操作**。

compareAndSet 方法调用流程

```java

private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();

public final boolean compareAndSet(T obj, V expect, V update) {
    accessCheck(obj);
    valueCheck(update);
    return U.compareAndSwapObject(obj, offset, expect, update);
}

public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

```

看一个例子了解 `AtomicReferenceFieldUpdater` 的使用方式

```java
public class AtomicReferenceFieldUpdaterTest {
  public static void main(String[] args) throws Exception {
    // T:持有可更新字段的对象的类型
    // V:字段的类型
    AtomicReferenceFieldUpdater<Dog, String> updater =
        // 包含该字段所在的类、将被更新的对象的类、将被更新的字段的名称
        AtomicReferenceFieldUpdater.newUpdater(Dog.class, String.class, "name");
    Dog dog = new Dog();

    // 如果期望值和字段当前值相等，说明目前是最新的值可以进行更新，则原子地将字段设置为给定的更新值。
    // 参数：
    // obj: 字段所在对象
    // expect - 期望值
    // update - 新值
    // 返回：如果成功则为true
    System.out.println(dog.name); // dog1 默认值
    boolean result = updater.compareAndSet(dog, "dog1", "dog2");
    System.out.println(result); // true 修改成功
    System.out.println(dog.name); // dog2 修改后的的值
    boolean result2 = updater.compareAndSet(dog, "dog1", "dog3");
    System.out.println(result2); // false 修改失败
    System.out.println(dog.name); // dog2 还是原来的值

    // 原子地将此更新程序管理的给定对象的字段设置为给定值并返回旧值。
    // 参数：
    // obj – 更新字段的对象
    // newValue – 新值
    // 返回：之前的的值
    String result3 = updater.getAndSet(dog, "dog4");
    System.out.println(result3); // dog2  原来的值
    System.out.println(dog.name); // dog4 修改后的值
  }
}

class Dog {
  volatile String name = "dog1";
}
```

SafePublicationLazyImpl 使用 AtomicReferenceFieldUpdater 来保证 _value 属性的原子操作。支持同时多个线程调用，并且可以在全部或部分线程上同时进行初始化。如果某个值已由另一个线程初始化，则将返回该值而不执行初始化。

```kotlin
private class SafePublicationLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    @Volatile private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    private val final: Any = UNINITIALIZED_VALUE
    override val value: T
        get() {
            val value = _value
            if (value !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return value as T
            }
            val initializerValue = initializer
            //如果在这里看到初始值已经为 null，则表示该值已被另一个线程设置过了，直接返回 _value ，否则就初始化
            if (initializerValue != null) {
                val newValue = initializerValue() //执行 Function 对象的 invoke 并将函数返回值原子化赋值给 _value
                //如果_value的值是UNINITIALIZED_VALUE说明还没有线程初始化它，此时可以将newValue设置给_value
              	if (valueUpdater.compareAndSet(this, UNINITIALIZED_VALUE, newValue)) {
                    initializer = null
                    return newValue //只有唯一的线程会从这里返回，其它都走下面的返回了
                }
              //如果_value的值不是UNINITIALIZED_VALUE，说明其它线程已经初始化完了，当前线程直接返回_value就行了
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }

		//如果一个序列化类中含有Object writeReplace()方法，那么实际序列化的对象将是作为writeReplace方法返回值的对象，
    private fun writeReplace(): Any = InitializedLazyImpl(value)
    companion object {
      	//初始化一个原子更新器：保证原子操作的字段是 _value
        private val valueUpdater = java.util.concurrent.atomic.AtomicReferenceFieldUpdater.newUpdater(
            SafePublicationLazyImpl::class.java,
            Any::class.java,
            "_value"
        )
    }
}
```

#### UnsafeLazyImpl

```kotlin
internal class UnsafeLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE
    override val value: T
        get() {
          	//普通的懒加载，只初始化一次，但是在多线程环境下不能保证只执行一次
            if (_value === UNINITIALIZED_VALUE) {
                _value = initializer!!() //多线程并发情况下可能出现空指针异常
                initializer = null
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }
  	//如果一个序列化类中含有Object writeReplace()方法，那么实际序列化的对象将是作为writeReplace方法返回值的对象，
    private fun writeReplace(): Any = InitializedLazyImpl(value)
}
```



### 4.3 NotNullVar

notNull 可以返回一个经过非空校验的属性值但是该属性值并没有初始化需要人为稍后setValue
在分配初始值之前尝试读取属性会导致异常，这也是返回非空属性的原理所在

> 非空属性应用场景分析
> 通常，声明为非空类型的属性必须在构造函数中初始化。然而，这通常并不方便。 例如，可以通过依赖注入或在单元测试的 setup 方法中初始化属性。在这种情况下，您不能在构造函数中提供非 null 初始值设定项，但您仍然希望在引用类体内的属性时避免空检查。
>
> notNull VS lateinit
> lateinit 不支持原始类型、只能用在可变属性var
> notNull 会为每个属性创建委托类 NotNullVar

notNull  的使用与原理

```kotlin
    var name2: String by Delegates.notNull()
    val age2: Int by Delegates.notNull() // notNull 会为每个属性创建委托类 NotNullVar
//    lateinit var age3: Int //lateinit 不支持原始类型
    lateinit var name3: String //lateinit 只能用在 var


public fun <T : Any> notNull(): ReadWriteProperty<Any?, T> = NotNullVar()

private class NotNullVar<T : Any>() : ReadWriteProperty<Any?, T> {
    private var value: T? = null

    public override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return value ?: throw IllegalStateException("Property ${property.name} should be initialized before get.")
    }
    public override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        this.value = value
    }
}
```

### 4.4 ObservableProperty

```kotlin
public abstract class ObservableProperty<V>(initialValue: V) : ReadWriteProperty<Any?, V> {
    private var value = initialValue
    protected open fun beforeChange(property: KProperty<*>, oldValue: V, newValue: V): Boolean = true
    protected open fun afterChange(property: KProperty<*>, oldValue: V, newValue: V): Unit {}
    public override fun getValue(thisRef: Any?, property: KProperty<*>): V {
        return value
    }
    public override fun setValue(thisRef: Any?, property: KProperty<*>, value: V) {
        val oldValue = this.value
    //beforeChange: 在尝试更改属性值之前调用的回调。 调用此回调时，该属性的值尚未更改。 如果回调返回true ，则属性的值被设置为新值，如果回调返回false ，则丢弃新值，属性保持其旧值
        if (!beforeChange(property, oldValue, value)) {
            return
        }
        this.value = value
    //afterChange: 进行属性更改后调用的回调。 调用此回调时，该属性的值已更改。
        afterChange(property, oldValue, value)
    }
}
```

#### observable 变量值更新后的监听

```kotlin
public inline fun <T> observable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Unit): ReadWriteProperty<Any?, T> =
    object : ObservableProperty<T>(initialValue) {
        override fun afterChange(property: KProperty<*>, oldValue: T, newValue: T) = onChange(property, oldValue, newValue)
    }
```

#### vetoable变量值更新前的拦截

```kotlin
public inline fun <T> vetoable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Boolean): ReadWriteProperty<Any?, T> =
    object : ObservableProperty<T>(initialValue) {
        override fun beforeChange(property: KProperty<*>, oldValue: T, newValue: T): Boolean = onChange(property, oldValue, newValue)
    }
```

测试代码

```kotlin
var items: List<String> by Delegates.observable(mutableListOf()) { property, oldValue, newValue ->
    println("${property.name} : $oldValue -> $newValue")
}

var nameAfter: String by Delegates.observable("no") { prop, old, new ->
    println("$old -> $new")
}
var nameBefore: String by Delegates.vetoable("no") { prop, old, new ->
    println("$old -> $new")
    true //返回true 表示 setValue 成功，否则不能覆盖原值
}

private fun <T> onChange(property: KProperty<*>, oldValue: T, newValue: T) {
    println("${property.name} : $oldValue -> $newValue")
}

var age: Int by Delegates.observable(18, ::onChange)

//运行结果
no -> first
first -> second
no -> 11111
11111 -> 2222
age : 18 -> 33
age : 33 -> 55
items : [] -> [new val]
items : [new val] -> [new val, new 111]
```



## 5. 属性委托在 Android 上的应用

### 5.1 ViewBinding

```kotlin
//1. 借助 lazy 属性委托  + 反射 VB 的 inflate 方法
private val binding: ActivityMainBinding by vb()
//2. 借助 lazy 属性委托  + 传递 inflate 方法引用
private val binding: ActivityMainBinding by vb(ActivityMainBinding::inflate)
```

**[VBHelper](https://github.com/jaydroid1024/VBHelper)**

```kotlin
@MainThread
inline fun <reified T : ViewBinding> ComponentActivity.vb(noinline inflateMethodRef: ((LayoutInflater) -> T)? = null): Lazy<T> =
    ActivityVBLazy(this, T::class, inflateMethodRef)


class ActivityVBLazy<T : ViewBinding>(
    private val activity: ComponentActivity,
    private val kClass: KClass<*>,
    private val inflateMethodRef: ((LayoutInflater) -> T)?
) : Lazy<T> {
    private var cachedBinding: T? = null
    override val value: T
        get() {
            var viewBinding = cachedBinding
            if (viewBinding == null) {
                viewBinding = if (inflateMethodRef != null) {
                    //借助 lazy 属性委托 + 传递 inflate 方法引用
                    inflateMethodRef.invoke(activity.layoutInflater)
                } else {
                    //借助 lazy 属性委托  + 反射绑定类的 inflate 方法
                    @Suppress("UNCHECKED_CAST")
                    kClass.java.getMethod(METHOD_INFLATE, LayoutInflater::class.java)
                        .invoke(null, activity.layoutInflater) as T
                }
                activity.setContentView(viewBinding.root)
                cachedBinding = viewBinding
            }
            return viewBinding
        }

    override fun isInitialized() = cachedBinding != null
}
```

### 5.2 ViewModel

```kotlin
//借助 lazy 属性委托  + ViewModelProvider
val model: MyViewModel by viewModels()
```

**[ActivityViewModelLazy](https://android.googlesource.com/platform/frameworks/support/+/0699f8f5b5aa7d79ba48d57a3710989ae2f50ee3/activity/ktx/src/main/java/androidx/activity/ActivityViewModelLazy.kt)**

```kotlin
@MainThread
inline fun <reified VM : ViewModel> ComponentActivity.viewModels(
    factory: ViewModelProvider.Factory? = null
): Lazy<VM> = ActivityViewModelLazy(this, VM::class, factory)

/**
 * An implementation of [Lazy] used by [ComponentActivity.viewModels] tied to the given [activity],
 * [viewModelClass], [factory]
 */
class ActivityViewModelLazy<VM : ViewModel>(
    private val activity: ComponentActivity,
    private val viewModelClass: KClass<VM>,
    private val factory: ViewModelProvider.Factory?
) : Lazy<VM> {
    private var cached: VM? = null
    override val value: VM
        get() {
            var viewModel = cached
            if (viewModel == null) {
                val application = activity.application
                    ?: throw IllegalArgumentException(
                        "ViewModel can be accessed " +
                                "only when Activity is attached"
                    )
                val resolvedFactory = factory ?: AndroidViewModelFactory.getInstance(application)
                viewModel = ViewModelProvider(activity, resolvedFactory).get(viewModelClass.java)
                cached = viewModel
            }
            return viewModel
        }

    override fun isInitialized() = cached != null
}
```

**[FragmentViewModelLazy](https://android.googlesource.com/platform/frameworks/support/+/0699f8f5b5aa7d79ba48d57a3710989ae2f50ee3/fragment/ktx/src/main/java/androidx/fragment/app/FragmentViewModelLazy.kt)**

```kotlin
@MainThread
inline fun <reified VM : ViewModel> Fragment.viewModels(factory: Factory? = null): Lazy<VM> =
    FragmentViewModelLazy(this, VM::class, factory)

/**
 * An implementation of [Lazy] used by [Fragment.viewModels] tied to the given [fragment],
 * [viewModelClass], [factory]
 */
class FragmentViewModelLazy<VM : ViewModel>(
    private val fragment: Fragment,
    private val viewModelClass: KClass<VM>,
    private val factory: Factory?
) : Lazy<VM> {
    private var cached: VM? = null
    override val value: VM
        get() {
            var viewModel = cached
            if (viewModel == null) {
                val application = fragment.activity?.application
                    ?: throw IllegalArgumentException(
                        "ViewModel can be accessed " +
                                "only when Fragment is attached"
                    )
                val resolvedFactory = factory ?: AndroidViewModelFactory.getInstance(application)
                viewModel = ViewModelProvider(fragment, resolvedFactory).get(viewModelClass.java)
                cached = viewModel
            }
            return viewModel
        }

    override fun isInitialized() = cached != null
}
```

### 5.3 SP delegates

```kotlin
fun SharedPreferences.int(def: Int = 0, key: String? = null) =
    delegate(def, key, SharedPreferences::getInt, SharedPreferences.Editor::putInt)

fun SharedPreferences.long(def: Long = 0, key: String? = null) =
    delegate(def, key, SharedPreferences::getLong, SharedPreferences.Editor::putLong)

fun SharedPreferences.string(def: String = "", key: String? = null) =
    delegate(def, key, SharedPreferences::getString, SharedPreferences.Editor::putString)


private inline fun <T> SharedPreferences.delegate(
    defaultValue: T,
    key: String?,
    crossinline getter: SharedPreferences.(String, T) -> T,
    crossinline setter: SharedPreferences.Editor.(String, T) -> SharedPreferences.Editor
) = object : ReadWriteProperty<Any, T> {
    override fun getValue(thisRef: Any, property: KProperty<*>) =
        getter(key ?: property.name, defaultValue)

    @SuppressLint("CommitPrefEdits")
    override fun setValue(thisRef: Any, property: KProperty<*>, value: T) =
        edit().setter(key ?: property.name, value).apply()
}
```

测试代码

```kotlin
class TokenHolder(prefs: SharedPreferences) {
    var token: String by prefs.string()
        private set
    var count by prefs.int()
        private set
    fun saveToken(newToken: String) {
        token = newToken
        count++
    }
    override fun toString(): String {
        return "TokenHolder(token='$token', count=$count)"
    }
}

class UserHolder(prefs: SharedPreferences) {
    var name: String by prefs.string()
        private set
    var pwd: String by prefs.string()
        private set
    fun saveUserAccount(name: String, pwd: String) {
        this.name = name
        this.pwd = pwd
    }
    override fun toString(): String {
        return "UserHolder(name='$name', pwd='$pwd')"
    }
}

val prefs = getSharedPreferences("sp_app_jay", Context.MODE_PRIVATE)

//缓存Token的场景
val tokenHolder = TokenHolder(prefs)
Log.d("Jay", "tokenHolder:$tokenHolder")
tokenHolder.saveToken("token_one")
tokenHolder.saveToken("token_second")

//缓存登录信息的场景
val userHolder = UserHolder(prefs)
Log.d("Jay", "userHolder:$userHolder")
userHolder.saveUserAccount("jay", "123456")
```

## 6. 总结

本篇文章围绕 Kotlin 的内置委托(Delegation﻿) 特性并结合代码实践分别阐述了 Kotlin 委托的原理(包括属性委托和接口委托)，尤其是属性委托从属性到委托详细阐述了其实现原理，

然后是实践部分，首先是Kotlin 标准库中利用属性委托为我们封装了很多简洁的API，比如：map、lazy、notNull、Observable 等；然后是Kotlin 属性委托在 Android 上的一些实践，包括 VB、VM、SP 等利用属性委托基本上都能完成一行代码实现set/get。Kotlin 委托显然在消除样板代码方面能发挥出强大的作用。但是这每个属性的背后却对应这一个委托类，所以在大量使用时也需要兼顾性能。



## 7. 参考

[官方文档 | 委托](https://www.kotlincn.net/docs/reference/delegation.html)

[官方文档 | 属性委托](https://www.kotlincn.net/docs/reference/delegated-properties.html)

[慕课网 | 新版 Kotlin 从入门到精通](https://coding.imooc.com/class/398.html)

[一文彻底搞懂Kotlin中的委托](https://juejin.cn/post/6844904038589267982)

[Wikipedia | Delegation pattern](https://en.wikipedia.org/wiki/Delegation_pattern)

[Wikipedia | Proxy pattern](https://en.wikipedia.org/wiki/Proxy_pattern)

[Medium | Kotlin Delegates in Android](https://proandroiddev.com/kotlin-delegates-in-android-1ab0a715762d)

