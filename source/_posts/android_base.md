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



# 1.Java

## 1.1 集合

#### 1.1.1 ArrayList和LinkedList区别

- [ArrayList 源码](http://androidos.net.cn/android/9.0.0_r8/xref/libcore/ojluni/src/main/java/java/util/ArrayList.java)
- [LinkedList 源码](http://androidos.net.cn/android/9.0.0_r8/xref/libcore/ojluni/src/main/java/java/util/LinkedList.java)

- ⾸先，他们的底层数据结构不同，ArrayList底层是基于数组实现的，LinkedList底层是基于链表实现的

- 由于底层数据结构不同，他们所适⽤的场景也不同，ArrayList更适合随机查找，LinkedList更适合删除和添加，查询、添加、删除的时间复杂度不同

- 另外ArrayList和LinkedList都实现了List接⼝，但是LinkedList还额外实现了Deque接⼝，所以LinkedList还可以当做队列来使⽤

#### 1.1.2 HashMap的Put⽅法的⼤体流程

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



#### ThreadLocal 简述

- [ThreadLocal 源码](http://androidos.net.cn/android/9.0.0_r8/xref/libcore/ojluni/src/main/java/java/lang/ThreadLocal.java)

- ThreadLocal是Java中所提供的线程本地存储机制，可以利⽤该机制将数据缓存在某个线程内部，该线
  程可以在任意时刻、任意⽅法中获取缓存的数据

- ThreadLocal底层是通过ThreadLocalMap来实现的，每个Thread对象（注意不是ThreadLocal对象）中都存在⼀个ThreadLocalMap，Map的key为ThreadLocal对象，Map的value为需要缓
  存的值

- 如果在线程池中使⽤ThreadLocal会造成内存泄漏，因为当ThreadLocal对象使⽤完之后，应该要把设置的key，value，也就是Entry对象进⾏回收，但线程池中的线程不会回收，⽽线程对象是通过强引⽤指向ThreadLocalMap，ThreadLocalMap也是通过强引⽤指向Entry对象，线程不被回收，Entry对象也就不会被回收，从⽽出现内存泄漏，解决办法是，在使⽤了ThreadLocal对象之后，⼿动调⽤ThreadLocal的remove⽅法，⼿动清楚Entry对象

- ThreadLocal经典的应⽤场景就是连接管理（⼀个线程持有⼀个连接，该连接对象可以在不同的⽅法之间进⾏传递，线程之间不共享同⼀个连接）

