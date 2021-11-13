---
title: 实战系列-设计模式总则篇
date: 2021-05-31 14:16:55
cover: true
tags: 
    - 设计模式
category: 
	- 设计模式
summary: 设计模式概述与24种设计模式简要概述，六种创建型模式，五种结构型模式，11中行为型模式



---

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/post_banner_jay.png)

# 实战系列-设计模式总则篇

![image-20210812012737030](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/20210812012744.png)

## 1.设计模式概述

- 设计模式是什么
  - **设计模式**是软件设计中常见问题的典型解决方案。 它们就像能根据需求进行调整的预制蓝图， 可用于解决代码中反复出现的设计问题。

- 设计模式的主要目的
  - 设计模式是针对软件开发中经常遇到的一些设计问题，总结出来的一套解决方案或者设计思路。应用设计模式的主要目的是提高代码的可扩展性。

- 设计模式与设计原则
  - 从抽象程度上来讲，设计原则比设计模式更抽象。设计模式更加具体、更加可执行
- 设计模式与方法或库
  - 设计模式与方法或库的使用方式不同， 你很难直接在自己的程序中套用某个设计模式。 模式并不是一段特定的代码， 而是解决特定问题的一般性概念。 你可以根据模式来实现符合自己程序实际所需的解决方案。
- 设计模式与算法
  - 设计模式模式和算法两者在概念上都是已知特定问题的典型解决方案。 但算法总是明确定义达成特定目标所需的一系列步骤， 而模式则是对解决方案的更高层次描述。 同一模式在两个不同程序中的实现代码可能会不一样。
  - 算法更像是菜谱： 提供达成目标的明确步骤。 而模式更像是蓝图： 你可以看到最终的结果和模式的功能， 但需要自己确定实现步骤。
- 设计模式的历史
  - 模式的概念是由克里斯托佛·亚历山大在其著作 《[建筑模式语言](https://refactoringguru.cn/pattern-language-book)》 中首次提出的。此后，由埃里希·伽玛、 约翰·弗利赛德斯、 拉尔夫·约翰逊和理查德·赫尔姆这四位作者接受了模式的概念。 1994 年， 他们出版了 《[设计模式： 可复用面向对象软件的基础](https://refactoringguru.cn/gof-book)》 一书， 将设计模式的概念应用到程序开发领域中。 该书提供了 23 个模式来解决面向对象程序设计中的各种问题， 很快便成为了畅销书。 由于书名太长， 人们将其简称为 “四人组 （Gang of Four， GoF） 的书”， 并且很快进一步简化为 “GoF 的书”。

- 为什么要学习设计模式
  - 设计模式是针对软件设计中常见问题的工具箱， 其中的工具就是各种**经过实践验证的解决方案**。 即使你从未遇到过这些问题， 了解模式仍然非常有用， 因为它能指导你如何使用面向对象的设计原则来解决各种问题。
  - 设计模式定义了一种让你和团队成员能够更高效沟通的通用语言。 你只需说 “哦， 这里用单例就可以了”， 所有人都会理解这条建议背后的想法。 只要知晓模式及其名称， 你就无需解释什么是单例。
- 关于模式的争议
  - 低效的解决方案：模式试图将已经广泛使用的方式系统化。 许多人会将这样的统一化认为是某种教条， 他们会 “全心全意” 地实施这样的模式， 而不会根据项目的实际情况对其进行调整。
  - 不当使用：如果你只有一把铁锤， 那么任何东西看上去都像是钉子。这个问题常常会给初学模式的人们带来困扰： 在学习了某个模式后， 他们会在所有地方使用该模式， 即便是在较为简单的代码也能胜任的地方也是如此。
- 设计模式分类
  - 所有模式可以根据其意图或目的来分类。
  - **创建型模式：**提供创建对象的机制， 增加已有代码的灵活性和可复用性。
    - 常用的有：单例模式、简单工厂模式、工厂方法模式、抽象工厂模式、建造者模式。
    - 不常用的有：原型模式。
  - **结构型模式：**介绍如何将对象和类组装成较大的结构， 并同时保持结构的灵活和高效。
    - 常用的有：代理模式、桥接模式、装饰者模式、门面模式、适配器模式。
    - 不常用的有：组合模式、享元模式。
  - **行为型模式：**负责对象间的高效沟通和职责委派。
    - 常用的有：观察者模式、模板方法模式、策略模式、责任链模式、迭代器模式、状态模式。
    - 不常用的有：访问者模式、备忘录模式、命令模式、解释器模式、 中介者模式。
      



## 2.设计模式目录



| 名称                                | 实现要点                                                     | 实战                                             |
| ----------------------------------- | :----------------------------------------------------------- | ------------------------------------------------ |
| **创建型模式**                      | **创建型设计模式主要解决对象的创建问题，<br/>封装复杂的创建过程，解耦对象的创建和使用<br />创建型设计模式包括：单例模式、工厂模式、建造者模式、<br />原型模式。** |                                                  |
| 01. 单例\|Singleton                 | 保证一个类仅有一个实例，并提供一个<br/>访问它的全局访问点。单例有几种经典的实现方式，它们分别<br/>是：饿汉式、懒汉式、双重检测、静态内部类、枚举。 |                                                  |
| 02. 简单工厂\|Sample Factory        | 当每个对象的创建逻辑都比较简单的时候，<br />推荐使用简单工厂模式，将多个对象的创建逻<br/>辑放到一个工厂类中 |                                                  |
| 03. 工厂方法\|Factory Method        | 定义一个创建对象的接口，让其子类自<br/>己决定实例化哪一个工厂类，<br/>工厂模式使其创建过程延迟到子类进行。 |                                                  |
| 04. 抽象工厂\|Abstract Factory      | 提供一个创建一系列列相关或相互依赖对<br/>象的接口，而无需指定它们具体的类。 |                                                  |
| 05. 建造者\|Builder                 | 建造者模式用来创建复杂对象，<br />可以通过设置不同的可选参数，定制化地创建不同的对象。<br/>建造者模式的原理和实现比较简单，<br />重点是掌握应用场景，避免过度使用。 | Dialog                                           |
| 06. 原型\|Prototype                 | 如果对象的创建成本比较大，<br />而同一个类的不同对象之间差别不大(大部分字段都相同)，<br/>在这种情况下，我们可以利用对已有对象（原型）<br />进行复制（或者叫拷贝）的方式，来创建新对象，<br/>以达到节省创建时间的目的。<br />原型模式有两种实现方法，深拷贝和浅拷贝。 |                                                  |
|                                     |                                                              |                                                  |
| **结构型模式**                      | **结构型模式主要总结了一些类或对象组合在一起的经典结构，<br />这些经典的结构可以解决特定应用场景的问题，<br />结构型模式包括：代理模式、桥接模式、装饰器模式、适配器模式、<br />门面模式、组合模式、享元模式。** |                                                  |
| 01. 适配器\|Adapter                 | 适配器模式是一种事后的补救策略，用来补救设计上的缺陷。<br />适配器提供跟原始类不同的接口。<br />它将不兼容的接口转换为可兼容的接口让原本由于接口<br/>不兼容而不能一起工作的类可以一起工作。<br />适配器模式有两种实现方式：类适配器和对象适配器。<br />类适配器使用继承关系来实现，<br />对象适配器使用组合关系来实现。 | RecyclerView                                     |
| 02. 桥接\|Bridge                    | 桥接模式有两种理解方式。<br />第一种理解方式是“将抽象和实现解耦，让它们能独立开发”。<br />抽象”和“实现”独立开发，<br />通过对象之间的组合关系组装在一起。<br />另一种理解方式更加简单，等同于“组合优于继承”设计原则，<br />不管是哪种理解方式，它们的代码结构都是相同的，<br />都是一种类之间的组合关系。 |                                                  |
| 03. 组合\|Composite                 | 将对象组合成树形结构以表示"部分-整体"的层次结构。<br />组合模式使得用户对<br/>单个对象和组合对象的使用具有一致性。 |                                                  |
| 04. 装饰\|Decorator                 | 装饰器模式主要解决继承关系过于复杂的问题，<br />通过组合来替代继承，给原始类添加增强功能。<br />这也是判断是否该用装饰器模式的一个重要的依据。 | Java IO                                          |
| 05. 门面\|Facade                    | 通过封装细粒度的接口，<br />向外提供组合各个细粒度接口的高层次接口，<br />来提高接口的易用性。 |                                                  |
| 06. 享元\|Flyweight                 | 当一个系统中存在大量重复对象的时候，<br />我们就可以利用享元模式，将对象设计成享元，<br />在内存中只保留一份实例，<br />供多处代码引用，这样可以减少内存中对象的数量，<br />以起到节省内存的目的 |                                                  |
| 07. 代理\|Proxy                     | 代理模式在不改变原始类接口的条件下，为原始类定义一个代理类，<br />主要目的是控制访问，而非加强功能，<br />这是它跟装饰器模式最大的不同。<br />一般情况下，我们让代理类和原始类实现同样的接口，<br />或者让代理类继承原始类来实现代理模式。<br />代理模式分为静态代理和动态代理。 | Retrofit 动态代理                                |
|                                     |                                                              |                                                  |
| **行为模式**                        | **型设计模式主要解决的就是“类或对象之间的交互”问题。<br />行为型模式包括：观察者模式、模板模式、策略模式、职责链模式、<br/>迭代器模式、状态模式、访问者模式、备忘录模式、<br />命令模式、解释器模式、中介模式。** |                                                  |
| 01. 责任链\|Chain of Responsibility | 避免请求发送者与接收者耦合在一起，让多<br/>个对象都有可能接收请求，将这些对象连接<br/>成一条链，并且沿着这条链传递请求，直到<br/>有对象处理理它为止。<br />职责链模式常用在框架开发中，<br />用来实现过滤器、拦截器功能，<br />让框架的使用者在不需要修改框架源码的情况下，<br />添加新的过滤、拦截功能。<br />这也体现了之前讲到的对扩展开放、<br />对修改关闭的设计原则。 | OkHttp 拦截器                                    |
| 02. 命令\|Command                   | 命令模式用到最核心的实现手段，<br />就是将函数封装成对象命令模式的主要作用和应用场景，<br />是用来控制命令的执行，比如，异步、延迟、排队执行命令、<br />撤销重做命令、存储命令、给命令记录日志等 |                                                  |
| 03. 迭代器\|Iterator                | 迭代器模式提供一种方法顺序访问一个容器对象中各元素, <br />而又无须暴暴露该对象的内部表示。<br />迭代器模式主要作用是解耦容器代码和遍历代码。<br />大部分编程语言都提供了现成的迭代器可以使用<br />在通过迭代器来遍历集合元素的同时，<br />增加或者删除集合中的元素，<br />有可能会导致某个元素被重复遍历或遍历不到。<br />解决方案是增删元素之后让遍历报错(fail-fast) | List<br />Map                                    |
| 04. 中介者\|Mediator                | 中介模式的设计思想跟中间层很像，<br />通过引入中介这个中间层，将一组对象之间的交互关系<br/>（或者说依赖关系）从多对多（网状关系）<br />转换为一对多（星状关系）.<br />在中介模式的应用场景中，<br/>参与者之间的交互关系错综复杂，<br />既可以是消息的发送者、也可以同时是消息的接收者。 |                                                  |
| 05. 备忘录\|Memento                 | 备忘录模式也叫快照模式，具体来说，<br />就是在不违背封装原则的前提下，<br />捕获一个对象的内部状态，<br />并在该对象之外保存这个状态，<br />以便之后恢复对象为先前的状态。 |                                                  |
| 06. 观察者\|Observer                | 定义对象间的一种一对多的依赖关系，<br />当一个对象的状态发生改变时，<br />所有依赖于它的对象都得到通知并被自动更新。<br />同步阻塞是最经典的实现方式，主要是为了代码解耦；<br />异步非阻塞除了能实现代码解耦之外，<br />还能提高代码的执行效率；<br />进程间的观察者模式解耦更加彻底，<br />一般是基于消息队列来实现，<br />用来实现不同进程间的被观察者和观察者之间的交互。 | EventBus<br />RxJava                             |
| 07. 状态\|State                     | 允许对象在内部状态发生改变时改变它的行为<br />状态模式建议为对象的所有可能状态新建一个类， <br />然后将所有状态的对应行为抽取到这些类中。<br />所有状态类都必须遵循同样的接口，<br /> 而且上下文必须仅通过接口与这些对象进行交互。 | 音乐播放器                                       |
| 08. 策略\|Strategy                  | 策略模式用来解耦策略的定义、创建、使用。<br />策略类的定义比较简单，包含一个策略接口<br />和一组实现这个接口的策略类。<br />策略的创建由工厂类来完成，封装策略创建的细节。<br />策略模式包含一组策略可选，客户端代码选择使用哪个策略，<br />有两种确定方法：编译时静态确定和运行时动态确定。<br />其中，“运行时动态确定”才是策略模式最典型的应用场景。<br />最常见的应用场景是，利用它来避免冗长的<br/>if-else 或 switch 分支判断。 | 支付方式                                         |
| 09. 模板方法\|Template Method       | 模板方法模式在一个方法中定义一个算法骨架，<br />并将某些步骤推迟到子类中实现。<br />模板方法模式可以让子类在不改变算法整体结构的情况下，<br />重新定义算法中的某些步骤。<br />模板模式有两大作用：复用和扩展。<br />回调跟模板模式具有相同的作用<br />回调基于组合关系来实现，模板模式基于继承关系来实现。<br />回调比模板模式更加灵活。 | BaseActivity<br />BaseViewModel<br />BaseNetwork |
| 10. 访问者\|Visitor                 | 访问者模式允许一个或者多个操作应用到一组对象上，<br />设计意图是解耦操作和对象本身，<br />保持类职责单一、<br />满足开闭原则以及应对代码的复杂性 | ASM                                              |
| 11. 解释器\|Interpreter             | 用来实现根据语法规则解读“句子”的解释器。<br />核心思想将语法解析的工作拆分到各个小类中，<br />以此来避免大而全的解析类。一般的做法是，<br />将语法规则拆分一些小的独立的单元，<br />然后对每个单元进行解析，最终合并为对整个语法规则的解析。 |                                                  |















## 参考

- [极客时间| 设计模式之美](https://time.geekbang.org/column/intro/250?code=Grxvvkczx9tydhzn0RhJfNfwaF2RgJA9qeUWd8orIYo%3D)

- [慕课网 | java设计模式精讲 Debug 方式+内存分析](https://coding.imooc.com/class/270.html?mc_marking=6eab7b8c9bc28db4f23571353f1a9fe5&mc_channel=banner)

- [在线书籍 | 深入设计模式](https://refactoringguru.cn/design-patterns)

- [bugstack虫洞栈 | 实战设计模式](https://bugstack.cn/itstack/itstack-demo-design.html)

- [菜鸟教程 | 设计模式](https://www.runoob.com/design-pattern/design-pattern-tutorial.html)

  
