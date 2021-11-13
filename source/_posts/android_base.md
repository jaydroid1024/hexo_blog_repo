---
title: 筑基系列-Android基础知识小抄版（更新中...）
date: 2021-07-31 14:16:55
cover: true
tags: 
    - Android
    - Java
category: 
	- Android
summary: Android/Java 基础知识小抄版提纲挈领，包括：Java（集合、并发、IO、语言特性）Android（四大组件原理、Handler、布局等）



---

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/post_banner_jay.png)



Android各版本对应JDK版本

| 平台版本      | 版本名称           | SDK版本 | 市场占有率 | JDK版本 |
| :------------ | :----------------- | :------ | :--------- | :------ |
| 2.3.3 - 2.3.7 | Gingerbread        | 10      | 0.4%       | 6       |
| 4.0.3 - 4.0.4 | Ice Cream Sandwich | 15      | 0.5%       | 6       |
| 4.1.x         | Jelly Bean         | 16      | 2.0%       | 6       |
| 4.2.x         | Jelly Bean         | 17      | 3.0%       | 6       |
| 4.3           | Jelly Bean         | 18      | 0.9%       | 6       |
| 4.4           | KitKat             | 19      | 13.4%      | 6       |
| 5.0           | Lollipop           | 21      | 6.1%       | 7       |
| 5.1           | Lollipop           | 22      | 20.2%      | 7       |
| 6.0           | Marshmallow        | 23      | 29.7%      | 7       |
| 7.0           | Nougat             | 24      | 19.3%      | 7       |
| 7.1           | Nougat             | 25      | 4.0%       | 7       |
| 8.0           | Oreo               | 26      | 0.5%       | 8       |



Android Gradle 插件版本所需的 Gradle 版本。为了获得最佳性能，您应使用 Gradle 和插件这两者的最新版本。

| 插件版本      | 所需的 Gradle 版本 |
| :------------ | :----------------- |
| 1.0.0 - 1.1.3 | 2.2.1 - 2.3        |
| 1.2.0 - 1.3.1 | 2.2.1 - 2.9        |
| 1.5.0         | 2.2.1 - 2.13       |
| 2.0.0 - 2.1.2 | 2.10 - 2.13        |
| 2.1.3 - 2.2.3 | 2.14.1 - 3.5       |
| 2.3.0+        | 3.3+               |
| 3.0.0+        | 4.1+               |
| 3.1.0+        | 4.4+               |
| 3.2.0 - 3.2.1 | 4.6+               |
| 3.3.0 - 3.3.3 | 4.10.1+            |
| 3.4.0 - 3.4.3 | 5.1.1+             |
| 3.5.0 - 3.5.4 | 5.4.1+             |
| 3.6.0 - 3.6.4 | 5.6.4+             |
| 4.0.0+        | 6.1.1+             |
| 4.1.0+        | 6.5+               |
| 4.2.0+        | 6.7.1+             |
| 7.0           | 7.0+               |



# 1.Java

## Java 基础

### JDK/JRE/JVM

JDK：Java Develpment Kit java 开发工具
JRE：Java Runtime Environment java运行时环境
JVM：java Virtual Machine java 虚拟机

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20210805234734.png)

### ==和equals比较

**==：**对比的是栈中的值，基本数据类型是**变量值**，引用类型是堆中内存**对象的地址**

**equals：**object中默认也是采用==比较，通常会重写，比如字符串类重写了equals比较的是每个char的值是否相等

```java
//Object.java
public boolean equals(Object obj) {
    return (this == obj);
}
```

```java
//String.java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

```java
//test equals and ==
String str1 = "Hello";
String str2 = new String("Hello");
String str3 = str2; 
System.out.println(str1 == str2); // false
System.out.println(str1 == str3); // false
System.out.println(str2 == str3); // true
System.out.println(str1.equals(str2)); // true
System.out.println(str1.equals(str3)); // true
System.out.println(str2.equals(str3)); // true
```

### hashCode与equals

hashCode 是通过将对象的内部地址转换为整数来实现的，此内存地址不能通过 java sdk 获得，必须作为native方法实现。native 源码如下：

```java
//object.java
public native int hashCode();
```

```c
//view src/share/native/java/lang/Object.c
static JNINativeMethod methods[] = {
    {"hashCode",    "()I",                    (void *)&JVM_IHashCode},
    {"wait",        "(J)V",                   (void *)&JVM_MonitorWait},
    {"notify",      "()V",                    (void *)&JVM_MonitorNotify},
    {"notifyAll",   "()V",                    (void *)&JVM_MonitorNotifyAll},
    {"clone",       "()Ljava/lang/Object;",   (void *)&JVM_Clone},
};
JNIEXPORT void JNICALL Java_java_lang_Object_registerNatives(JNIEnv *env, jclass cls){
    (*env)->RegisterNatives(env, cls, methods, sizeof(methods)/sizeof(methods[0]));
}
JNIEXPORT jclass JNICALL Java_java_lang_Object_getClass(JNIEnv *env, jobject this){
    if (this == NULL) {
        JNU_ThrowNullPointerException(env, NULL);
        return 0;
    } else {
        return (*env)->GetObjectClass(env, this);
    }
}
```

此源包含 getClass() 方法的实现。 hashCode 被定义为一个函数指针 JVM_IHashCode

```c
//view src/share/vm/prims/jvm.cpp
// java.lang.Object ///////////////////////////////////////////////
JVM_ENTRY(jint, JVM_IHashCode(JNIEnv* env, jobject handle))
  JVMWrapper("JVM_IHashCode");
  // as implemented in the classic virtual machine; return 0 if object is NULL
  return handle == NULL ? 0 : ObjectSynchronizer::FastHashCode (THREAD, JNIHandles::resolve_non_null(handle)) ;
JVM_END
```

```c
//view src/share/vm/runtime/synchronizer.cpp
intptr_t ObjectSynchronizer::FastHashCode (Thread * Self, oop obj) {
  ....
    if (mark->is_neutral()) {
    hash = mark->hash();              // this is a normal header
    if (hash) {                       // if it has hash, just return it
      return hash;
    }
    hash = get_next_hash(Self, obj);  // allocate a new hash code
    temp = mark->copy_set_hash(hash); // merge the hash code into header
    // use (machine word version) atomic operation to install the hash
    test = (markOop) Atomic::cmpxchg_ptr(temp, obj->mark_addr(), mark);
    if (test == mark) {
      return hash;
    }
    // If atomic operation failed, we must inflate the header
    // into heavy weight monitor. We could add more code here
    // for fast path, but it does not worth the complexity.
  }
  
 ...
   
  monitor = ObjectSynchronizer::inflate(Self, obj);
  // Load displaced header and check it has hash code
  mark = monitor->header();
  assert (mark->is_neutral(), "invariant") ;
  hash = mark->hash();
  if (hash == 0) {
    hash = get_next_hash(Self, obj);
    temp = mark->copy_set_hash(hash); // merge hash code into header
    assert (temp->is_neutral(), "invariant") ;
    test = (markOop) Atomic::cmpxchg_ptr(temp, monitor, mark);
    if (test != mark) {
      // The only update to the header in the monitor (outside GC)
      // is install the hash code. If someone add new usage of
      // displaced header, please update this code
      hash = test->hash();
      assert (test->is_neutral(), "invariant") ;
      assert (hash != 0, "Trivial unexpected object/monitor header usage.");
    }
  }
  // We finally get the hash
  return hash;
}
  
  static inline intptr_t get_next_hash(Thread * Self, oop obj) {
  intptr_t value = 0 ;
  if (hashCode == 0) {
     // This form uses an unguarded global Park-Miller RNG,
     // so it's possible for two threads to race and generate the same RNG.
     // On MP system we'll have lots of RW access to a global, so the
     // mechanism induces lots of coherency traffic.
     value = os::random() ;
  } else
  if (hashCode == 1) {
     // This variation has the property of being stable (idempotent)
     // between STW operations.  This can be useful in some of the 1-0
     // synchronization schemes.
     intptr_t addrBits = intptr_t(obj) >> 3 ;
     value = addrBits ^ (addrBits >> 5) ^ GVars.stwRandom ;
  } else
  if (hashCode == 2) {
     value = 1 ;            // for sensitivity testing
  } else
  if (hashCode == 3) {
     value = ++GVars.hcSequence ;
  } else
  if (hashCode == 4) {
     value = intptr_t(obj) ;
  } else {
     // Marsaglia's xor-shift scheme with thread-specific state
     // This is probably the best overall implementation -- we'll
     // likely make this the default in future releases.
     unsigned t = Self->_hashStateX ;
     t ^= (t << 11) ;
     Self->_hashStateX = Self->_hashStateY ;
     Self->_hashStateY = Self->_hashStateZ ;
     Self->_hashStateZ = Self->_hashStateW ;
     unsigned v = Self->_hashStateW ;
     v = (v ^ (v >> 19)) ^ (t ^ (t >> 8)) ;
     Self->_hashStateW = v ;
     value = v ;
  }
```

JVM_IHashCode 在 jvm.cpp 中定义。请参阅从第 504 行开始的代码。这又调用了在synchronizer.cpp 中定义的 ObjectSynchronizer::FastHashCode。请参阅第 576 行的 FastHashCode 和第 530 行的 get_next_hash 的实现。

**HashCode 的作用是：**

- hashCode 的作用是返回对象的哈希码值。支持此方法是为了有利于散列表，这个哈希码的作用是确定该对象在哈希表中的索引位

置。

- hashCode的默认行为是对堆上的对象产生独特值。如果没有重写hashCode()，则该class的两个对象无论如何都不会相等（即使这两个对象指向相同的数据）

 **HashCode 的一般约定是：**

- 在 Java 应用程序的执行过程中，只要在同一个对象上多次调用它,hashCode方法必须始终如一地返回相同的整数，前提是未修改对象的  equals比较中使用的信息。该整数不需要从应用程序的一次执行到同一应用程序的另一次执行保持一致。

- 如果两个对象根据 equals()方法相等，则对这两个对象中的每一个调用  hashCode}方法必须产生相同的整数结果。
- 如果两个对象根据 equals()方法不相等，则调用 hashCode}方法这两个对象中的每一个都必须产生不同的整数结果。
- 两个对象有相同的hashcode值，它们也不一定是相等的
- 因此，equals方法被覆盖过，则hashCode方法也必须被覆盖
- 为不相等的对象生成不同的整数结果可能会提高哈希表的性能。
- 在合理可行的情况下，类 Object定义的 hashCode 方法确实为不同的对象返回不同的整数。



### final 关键字

- 修饰类：表示类不可被继承
- 修饰方法：表示方法不可被子类覆盖，但是可以重载
- 修饰变量：表示变量一旦被赋值就不可以更改它的值
  - 如果final修饰的是类变量，只能在静态初始化块中指定初始值或者声明该类变量时指定初始值。
  - 如果final修饰的是成员变量，可以在非静态初始化块、声明该变量或者构造器中执行初始值。
  - 如果final修饰的是局部变量,系统不会为局部变量进行初始化，局部变量必须由程序员显示初始化。因此使用final修饰局部变量时，即可以在定义时指定默认值也可以在后面的代码中对final变量赋初值。
  - 如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；
  - 如果是引用类型的变量，则在对其初始化之后便不能再让其指向另一个对象。但是引用的值是可变的

```java
final Person p = new Person(25);
p.setAge(24);//合法
p=null;//非法
```

**为什么局部内部类和匿名内部类只能访问局部final变量？**

首先需要知道的一点是: 内部类和外部类是处于同一个级别的，内部类不会因为定义在方法中就会随着方法的执行完毕就被销毁。
这里就会产生问题：当外部类的方法结束时，局部变量就会被销毁了，但是内部类对象可能还存在(只有没有人再引用它时，才会死亡)。这里就出现了一个矛盾：内部类对象访问了一个不存在的变量。为了解决这个问题，就将局部变量复制了一份作为内部类的成员变量，这样当局部变量死亡后，内部类仍可以访问它，实际访问的是局部变量的"copy"。这样就好像延长了局部变量的生命周期将局部变量复制为内部类的成员变量时，必须保证这两个变量是一样的，也就是如果我们在内部类中修改了成员变量，方法中的局部变量也得跟着改变，怎么解决问题呢？就将局部变量设置为final，对它初始化后，我就不让你再去修改这个变量，就保证了内部类的成员变量和方法的局部变量的一致性。这实际上也是一种妥协。使得局部变量与内部类内建立的拷贝保持一致。



### String、StringBuffer、StringBuilder

- String是final修饰的，不可变，每次操作都会产生新的String对象
- StringBuffer和StringBuilder都是在原对象上操作
- StringBuffer是线程安全的，StringBuilder线程不安全的
- StringBuffer方法都是synchronized修饰的
- 性能：StringBuilder > StringBuffer > String
- 场景：经常需要改变字符串内容时使用后面两个
- 优先使用StringBuilder，多线程使用共享变量时使用StringBuffer

### 重载和重写的区别

- 重载： 发生在同一个类中，方法名必须相同，参数类型不同、个数不同、顺序不同，方法返回值和访问修饰符可以不同，发生在编译时。
- 重写： 发生在父子类中，方法名、参数列表必须相同，返回值范围小于等于父类，抛出的异常范围小于等于父类，访问修饰符范围大于等于父类；如果父类方法访问修饰符为private则子类就不能重写该方法。



### 接口和抽象类的区别

- 抽象类可以存在普通成员函数，而接口中只能存在public abstract 方法。
- 抽象类中的成员变量可以是各种类型的，而接口中的成员变量只能是public static final类型的。
- 抽象类只能继承一个，接口可以实现多个。
- 接口的设计目的，是对类的行为进行约束（更准确的说是一种“有”约束，因为接口不能规定类不可以有什么行为），也就是提供一种机制，可以强制要求不同的类具有相同的行为。它只约束了行为的有无，但不对如何实现行为进行限制。
- 而抽象类的设计目的，是代码复用。当不同的类具有某些相同的行为(记为行为集合A)，且其中一部分行为的实现方式一致时（A的非真子集，记为B），可以让这些类都派生于一个抽象类。在这个抽象类中实现了B，避免让所有的子类来实现B，这就达到了代码复用的目的。而A减B的部分，留给各个子类自己实现。正是因为A-B在这里没有实现，所以抽象类不允许实例化出来（否则当调用到A-B时，无法执行）。
- 抽象类是对类本质的抽象，表达的是 is a 的关系，比如： BMW is a Car 。抽象类包含并实现子类的通用特性，将子类存在差异化的特性进行抽象，交由子类去实现。
- 而接口是对行为的抽象，表达的是 like a 的关系。比如： Bird like a Aircraft （像飞行器一样可以飞），但其本质上 is a Bird 。接口的核心是定义行为，即实现类可以做什么，至于实现类主体是谁、是如何实现的，接口并不关心。
- 使用场景：当你关注一个事物的本质的时候，用抽象类；当你关注一个操作的时候，用接口。
- 抽象类的功能要远超过接口，但是，定义抽象类的代价高。因为高级语言来说（从实际设计上来说也是）每个类只能继承一个类。在这个类中，你必须继承或编写出其所有子类的所有共性。虽然接口在功能上会弱化许多，但是它只是针对一个动作的描述。而且你可以在一个类中同时实现多个接口。在设计阶段会降低难度



## Java 集合

### List和Set的区别

- List：有序，按对象进入的顺序保存对象，可重复，允许多个Null元素对象，可以使用Iterator取出所有元素，在逐一遍历，还可以使用get(int index)获取指定下标的元素
- Set：无序，不可重复，最多允许有一个Null元素对象，取元素时只能用Iterator接口取得所有元素，在逐一遍历各个元素



#### ArrayList和LinkedList区别

- [ArrayList 源码](http://androidos.net.cn/android/9.0.0_r8/xref/libcore/ojluni/src/main/java/java/util/ArrayList.java)
- [LinkedList 源码](http://androidos.net.cn/android/9.0.0_r8/xref/libcore/ojluni/src/main/java/java/util/LinkedList.java)

- ⾸先，他们的底层数据结构不同，ArrayList底层是基于数组实现的，LinkedList底层是基于链表实现的

- 由于底层数据结构不同，他们所适⽤的场景也不同，ArrayList更适合随机查找，LinkedList更适合删除和添加，查询、添加、删除的时间复杂度不同

- 另外ArrayList和LinkedList都实现了List接⼝，但是LinkedList还额外实现了Deque接⼝，所以LinkedList还可以当做队列来使⽤

- ArrayList：基于动态数组，连续内存存储，适合下标访问（随机访问），扩容机制：因为数组长度固定，超出长度存数据时需要新建数组，然后将老数组的数据拷贝到新数组，如果不是尾部插入数据还会涉及到元素的移动（往后复制一份，插入新元素），使用尾插法并指定初始容量可以极大提升性能、甚至超过linkedList（需要创建大量的node对象）
- LinkedList：基于链表，可以存储在分散的内存中，适合做数据插入及删除操作，不适合查询：需要逐一遍历LinkedList必须使用iterator不能使用for循环，因为每次for循环体内通过get(i)取得某一元素时都需要对list重新进行遍历，性能消耗极大。另外不要试图使用indexOf等返回元素索引，并利用其进行遍历，使用indexlOf对list进行了遍历，当结果为空时会遍历整个列表。

### HashMap和HashTable有什么区别？其底层实现是什么？



- HashMap方法没有synchronized修饰，线程非安全，HashTable线程安全；
- HashMap允许key和value为null，而HashTable不允许
- jdk8开始链表高度到8、数组长度超过64，链表转变为红黑树，元素以内部类Node节点存在
- 计算key的hash值，二次hash然后对数组长度取模，对应到数组下标，
- 如果没有产生hash冲突(下标位置没有元素)，则直接创建Node存入数组，
- 如果产生hash冲突，先进行equal比较，相同则取代该元素，不同，则判断链表高度插入链表，链
- 表高度达到8，并且数组长度到64则转变为红黑树，长度低于6则将红黑树转回链表
  

####  HashMap的Put⽅法的⼤体流程

- [HashMap 源码1.8](http://androidos.net.cn/android/9.0.0_r8/xref/libcore/ojluni/src/main/java/java/util/HashMap.java)
- [HashMap 源码1.7](http://androidos.net.cn/android/7.1.1_r28/xref/libcore/ojluni/src/main/java/java/util/HashMap.java)
- 根据Key通过哈希算法与与运算得出数组下标
- 如果数组下标位置元素为空，则将key和value封装为Entry对象（JDK1.7中是Entry对象，JDK1.8中是Node对象）并放⼊该位置
- 如果数组下标位置元素不为空，则要分情况讨论
  - 如果是JDK1.7，则先判断是否需要扩容，如果要扩容就进⾏扩容，如果不⽤扩容就⽣成Entry对象，并使⽤头插法添加到当前位置的链表中
  -  如果是JDK1.8，则会先判断当前位置上的Node的类型，看是红⿊树Node，还是链表Node
    - 如果是红⿊树Node，则将key和value封装为⼀个红⿊树节点并添加到红⿊树中去，在这个过程中会判断红⿊树中是否存在当前key，如果存在则更新value
    - 如果此位置上的Node对象是链表节点，则将key和value封装为⼀个链表Node并通过尾插法插⼊到链表的最后位置去，因为是尾插法，所以需要遍历链表，在遍历链表的过程中会判断是否存在当前key，如果存在则更新value，当遍历完链表后，将新链表Node插⼊到链表中，插⼊到链表后，会看当前链表的节点个数，如果⼤于等于8，那么则会将该链表转成红⿊树
    - 将key和value封装为Node插⼊到链表或红⿊树中后，再判断是否需要进⾏扩容，如果需要就
      扩容，如果不需要就结束PUT⽅法

```java

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```





## Java 多线程

#### ThreadLocal 简述

- [ThreadLocal 源码](http://androidos.net.cn/android/9.0.0_r8/xref/libcore/ojluni/src/main/java/java/lang/ThreadLocal.java)

- ThreadLocal是Java中所提供的线程本地存储机制，可以利⽤该机制将数据缓存在某个线程内部，该线
  程可以在任意时刻、任意⽅法中获取缓存的数据

- ThreadLocal底层是通过ThreadLocalMap来实现的，每个Thread对象（注意不是ThreadLocal对象）中都存在⼀个ThreadLocalMap，Map的key为ThreadLocal对象，Map的value为需要缓
  存的值

- 如果在线程池中使⽤ThreadLocal会造成内存泄漏，因为当ThreadLocal对象使⽤完之后，应该要把设置的key，value，也就是Entry对象进⾏回收，但线程池中的线程不会回收，⽽线程对象是通过强引⽤指向ThreadLocalMap，ThreadLocalMap也是通过强引⽤指向Entry对象，线程不被回收，Entry对象也就不会被回收，从⽽出现内存泄漏，解决办法是，在使⽤了ThreadLocal对象之后，⼿动调⽤ThreadLocal的remove⽅法，⼿动清楚Entry对象

- ThreadLocal经典的应⽤场景就是连接管理（⼀个线程持有⼀个连接，该连接对象可以在不同的⽅法之间进⾏传递，线程之间不共享同⼀个连接）



