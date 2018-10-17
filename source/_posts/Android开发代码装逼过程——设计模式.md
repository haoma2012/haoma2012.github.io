---
title: Android开发代码装逼过程——设计模式
date: 2018-03-20 20:01:27
tags: 设计模式
auth：yangdehao
---

设计模式在设计者是一种流行的思考设计问题的方法，是一套被反复使用，多数人知晓的，经过分类编目的，代码设计经验的总结。

使用了设计模式，是为了使代码具有可重用性，让代码更容易被他人理解和保证代码的可靠性。

书籍推荐《设计模式 可复用面向对象软件的基础》

<!-- more -->

#### 设计模式分类与作用

总体来说设计模式分为三大类：

- 创建型模式，共五种：工厂方法模式、抽象工厂模式、单例模式、建造者模式、原型模式。
- 结构型模式，共七种：适配器模式、装饰器模式、代理模式、外观模式、桥接模式、组合模式、享元模式。
- 行为型模式，共十一种：策略模式、模板方法模式、观察者模式、迭代子模式、责任链模式、命令模式、备忘录模式、状态模式、访问者模式、中介者模式、解释器模式。

其实还有两类：并发型模式和线程池模式。

- (1) 设计模式来源众多专家的经验和智慧
- (2) 设计模式提供了一套通用的设计词汇和一种通用的形式来方便开发人员之间沟通和交流，使得设计方案更加通俗易懂。
- (3) 大部分设计模式都兼顾了系统的可重用性和可扩展性
- (4)在学习每一个设计模式时至少应该掌握如下几点：这个设计模式的意图是什么，它要解决一个什么问题，什么时候可以使用它；它是如何解决的，掌握它的结构图，记住它的关键代码；能够想到至少两个它的应用实例，一个生活中的，一个软件中的；这个模式的优缺点是什么，在使用时要注意什么。当你能够回答上述所有问题时，恭喜你，你了解一个设计模式了，至于掌握它，那就在开发中去使用吧，用多了你自然就掌握了。+

####  设计模式六大原则

[公共技术点之面向对象六大原则:](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis/blob/master/oop-principles/oop-principles.md/)

- 单一职责原则  *简单说就是一个类只做一件事*
- 里氏替换原则  *简单说就是父类出现的地方子类也可以出现，替换子类也是可以的*
- 依赖倒置原则 *简单说就是面向接口编程，或者说是面向抽象编程，这里的抽象指的是接口或者抽象类。面向接口编程是面向对象精髓之一。*
- 开闭原则  *开闭原则的定义是 : 一个软件实体如类、模块和函数应该对扩展开放，对修改关闭。*
- 接口隔离原则 *客户端不应该依赖它不需要的接口；一个类对另一个类的依赖应该建立在最小的接口上。根据接口隔离原则，当一个接口太大时，我们需要将它分割成一些更细小的接口，使用该接口的客户端仅需知道与之相关的方法即可。*
- 迪米特原则 *==一个对象应该对其他对象有最少的了解==。*



设计模式分类： 创建型模式、结构型模式、行为型模式

 **创建型模式：抽象实例化过程**
 
- 1.抽象工厂模式(Abstract Factory)
```
为创建一组相关或相互依赖的对象提供一个接口，而无需指定它们具体的类。
```
![image](http://img.blog.csdn.net/20140422145222703?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

```
AbstractProduct: 抽象产品类
ConcreteProductA : 产品的具体实现A
ConcreteProductB : 产品的具体实现B
AbstractFactory : 抽象工厂
ConcreteFactory : 具体工厂实现
```

优点 : 
- 1.抽象工厂模式隔离了具体类的生产，使得客户并不需要知道什么被创建；
- 2.容易改变产品的系列；
- 3.增加新的具体工厂和产品族很方便，无须修改已有系统，符合“开闭原则”。
- 4.将一个系列的产品族统一到一起创建，客户代码易于使用。

缺点 : 

抽象工厂模式的最大缺点就是产品族扩展非常困难，为什么这么说呢？我们以通用代码为例，如果要增加一个产品 C， 也就是说产品家族由原来的 2 个增加到 3 个，看看我们的程序有多大改动吧！抽象类 AbstractCreator 要增加一个方法 createProductC()， 然后两个实现类都要修改，想想看，这严重违反了开闭原则，而且我们一直说明抽象类和接口是一个契约。

 - 2.工厂方法模式(Factory Method)

```
定义一个用于创建对象的接口，让子类决定实例化哪个类。
Factory Method使一个类的实例化延迟到子类
```
![image](https://upload-images.jianshu.io/upload_images/6163786-0c1b187a6365928a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/497)

```
Product（抽象产品类）：要创建的复杂对象，定义对象的公共接口。
ConcreteProduct（具体产品类）：实现Product接口。
Factory（抽象工厂类）：该方法返回一个Product类型的对象。
ConcreteFactory（具体工厂类）：返回ConcreteProduct实例。
```
优点：
- 符合开放封闭原则。新增产品时，只需增加相应的具体产品类和相应的工厂子类即可。
- 符合单一职责原则。每个具体工厂类只负责创建对应的产品。
缺点

- 一个具体工厂只能创建一种具体产品。
- 增加新产品时，还需增加相应的工厂类，系统类的个数将成对增加，增加了系统的复杂度和性能开销。
- 引入的抽象类也会导致类结构的复杂化。

```
Android中的ThreadFactory就是使用了工厂方法模式来生成线程的，
线程就是ThreadFactory的产品。
```
- 3.简单工厂方法模式(simple factory method)：静态工厂方法模式

```
定义一个用于创建对象的接口，让子类决定实例化哪个类。
```
 ![image](https://upload-images.jianshu.io/upload_images/6163786-39720028bfa2aa18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/507)

```
Product（抽象产品类）：要创建的复杂对象，定义对象的公共接口。
ConcreteProduct（具体产品类）：实现Product接口。
Factory（工厂类）：返回ConcreteProduct实例。
```

- 4.建造者模式(Build)

```
将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-a2a91526c60dd2fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/687)

```
Product（产品类）：要创建的复杂对象。在本类图中，产品类是一个具体的类，
而非抽象类。实际编程中，产品类可以是由一个抽象类与它的不同实现组成，也
可以是由多个抽象类与他们的实现组成。
Builder（抽象建造者）：创建产品的抽象接口，一般至少有一个创建产品的抽
象方法和一个返回产品的抽象方法。引入抽象类，是为了更容易扩展。
ConcreteBuilder（实际的建造者）：继承Builder类，实现抽象类的所有抽象
方法。实现具体的建造过程和细节。
Director（指挥者类）：分配不同的建造者来创建产品，统一组装流程。
```
优点
- 封装性良好，隐藏内部构建细节。
- 易于解耦，将产品本身与产品创建过程进行解耦，可以使用相同的创建过程来得到不同的产品。也就说细节依赖抽象。
- 易于扩展，具体的建造者类之间相互独立，增加新的具体建造者无需修改原有类库的代码。
- 易于精确控制对象的创建，由于具体的建造者是独立的，因此可以对建造过程逐步细化，而不对其他的模块产生任何影响。

缺点
- 产生多余的Build对象以及Director类。
- 建造者模式所创建的产品一般具有较多的共同点，其组成部分相似；如果产品之间的差异性很大，则不适合使用建造者模式，因此其使用范围受到一定的限制。
- 如果产品的内部变化复杂，可能会导致需要定义很多具体建造者类来实现这种变化，导致系统变得很庞大。

```
Android中的AlertDialog.Builder就是使用了Builder模式来构建AlertDialog的。
```

- 5.原型模式(prototype)

```
用原型实例指定创建对象的种类，并通过拷贝这些原型创建新的对象。

一个已存在的对象（即原型），通过复制原型的方式来创建一个内部
属性跟原型都一样的新的对象，这就是原型模式。
原型模式的核心是clone方法，通过clone方法来实现对象的拷贝。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-0ac5da2038fb2ff1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

```
Prototype（抽象原型类）：抽象类或者接口，用来声明clone方法。
ConcretePrototype1、ConcretePrototype2（具体原型类）：即要被复制的对象。
Client（客户端类）：即要使用原型模式的地方。
```

浅拷贝：直接将源对象中的spec的引用值拷贝给新对象的spec字段，这种拷贝方式就叫浅拷贝。

深拷贝：原Card对象中的spec指向的对象创建一个新的相同的对象，将这个新对象的引用赋给新拷贝的Card对象的spec字段。

优点
- 可以解决复杂对象创建时消耗过多的问题，在某些场景下提升创建对象的效率。
- 保护性拷贝，可以防止外部调用者对对象的修改，保证这个对象是只读的。

缺点
- 拷贝对象时不会执行构造函数。
- 有时需要考虑深拷贝和浅拷贝的问题。

```
Android中的Intent就实现了Cloneable接口，但是clone()方法中却是
通过new来创建对象的。
public class Intent implements Parcelable, Cloneable {
        //其他代码略
        @Override
        public Object clone() {
            return new Intent(this);//这里没有调用super.clone()来实现拷贝，而是直接通过new来创建
        }

}
```

- 6.单例模式(Singleton)

```
确保某个类只有一个实例,并且自行实例化并向整个系统提供这个实例。

e饿汉式
懒汉式(线程不安全)
懒汉式（线程安全）
静态内部类：懒加载，线程安全，推荐使用
```

优点
- 内存中只存在一个对象，节省了系统资源。
- 避免对资源的多重占用，例如一个文件操作，由于只有一个实例存在内存中，避免对同一资源文件的同时操作。

缺点
- 获取对象时不能用new
- 单例对象如果持有Context，那么很容易引发内存泄露。
- 单例模式一般没有接口，扩展很困难，若要扩展，只能修改代码来实现。


**结构型模式 structure patterns**

```
结构型模式:涉及如何组合类和对象已获得更大的结构，采用继承机制来组合接口实现。
```

- 1.适配器模式(Adapter)

```
将一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口
不匹配而无法在一起工作的两个类能够在一起工作。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-a34ae66637917a60.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Adapter(适配器接口):即目标角色，定义把其他类转换为何种接口，也就是我们期望的接口。
- Adaptee(被适配角色):即源角色，一般是已存在的类，需要适配新的接口。
- ConcreteAdapter(具体适配器):实现适配器接口，把源角色接口转换为目标角色期望的接口。

优点

- 提高了类的复用性，适配器能让一个类有更广泛的用途。
- 提高了灵活性，更换适配器就能达到不同的效果。不用时也可以随时删掉适配器，对原系统没影响。
- 符合开放封闭原则，不用修改原有代码。没有任何关系的类通过增加适配器就能联系在一起。

缺点
- 过多的使用适配器，会让系统非常零乱，不易整体进行把握。明明调用A接口，却被适配成B接口。


```
源码使用的Adapter适配器模式：
ListView和RecyclerView就再熟悉不过了，ListView现在应该很少用了吧，这里就以RecyclerView来分析。
```

- 2.桥接模式(bridge)

```
将抽象部分与实现部分分离，使它们都可以独立的变化。

一条数据线，一头USB接口的可以连接电脑、充电宝等等，
另一头可以连接不同品牌的手机，通过这条数据线，两头
不同的东西就可以连接起来，这就是桥接模式。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-b646cbc3e5310ca7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Abstraction(抽象化角色)：一般是抽象类，定义该角色的行为，同时保存一个对实现化角色的引用。
- Implementor(实现化角色)：接口或者抽象类，定义角色必需的行为和属性。
- ConcreteImplementorA、ConcreteImplementorB(具体实现化角色):实现角色的具体行为。

优点
- 分离了抽象与实现。让抽象部分和实现部分独立开来，分别定义接口，这有助于对系统进行分层，从而产生更好的结构化的系统。
- 良好的扩展性。抽象部分和实现部分都可以分别独立扩展，不会相互影响。

 缺点
- 增加了系统的复杂性。
- 不容易设计，抽象与实现的分离要设计得好比较有难度。


```
桥接模式在Android中的源码应用还是非常广泛的。
比如AbsListView跟ListAdapter之间就是一个桥接模式。

这里AbsListView是抽象化角色，ListAdapter则是实现化角色。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-3efb79758a977f6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

- 3.组合模式(Composite)

```
将对象组合成树形结构以表示“部分-整体”的层次结构，使得
用户对单个对象和组合对象的使用具有一致性。

组合模式实际上就是个树形结构，一棵树的节点如果没有分支，
就是叶子节点;如果存在分支，则是树枝节点。最典型的组合结
构就是文件和文件夹了
```
![image](https://upload-images.jianshu.io/upload_images/6163786-47e11b31527d39ec.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Component（抽象组件角色）：定义参加组合对象的共有方法和属性，可以定义一些默认的函数或属性。
- Leaf（叶子节点）：叶子没有子节点，因此是组合树的最小构建单元。
- Composite（树枝节点）：定义所有枝干节点的行为，存储子节点，实现相关操作。


优点
- 高层模块（客户端）调用简单。局部和整体都是同样的结构，客户端无需关心是局部还是整体。
- 新增节点容易。无需对现有类库进行任何修改，符合开闭原则。

缺点
- 不同的组合模式实现有不同的缺点，具体看上面的分析。

```
Android源码中，ViewGroup 和 View就是典型的组合模式。
```

- 4.装饰者模式(Decorator)

```
动态地给一个对象添加一些额外的职责。就增加功能来说，装饰模式相比生成子类更为灵活。
一如一间房，放上厨具，它就是厨房;放上床，就是卧室。


```
![image](https://upload-images.jianshu.io/upload_images/6163786-780e75c9fb17ff1c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Component（抽象组件）：接口或者抽象类，被装饰的最原始的对象。具体组件与抽象装饰角色的父类。
- ConcreteComponent（具体组件）：实现抽象组件的接口。
- Decorator（抽象装饰角色）：一般是抽象类，抽象组件的子类，同时持有一个被装饰者的引用，用来调用被装饰者的方法;同时可以给被装饰者增加新的职责。
- ConcreteDecorator（具体装饰类）：抽象装饰角色的具体实现。

优点
- 采用组合的方式，可以动态的扩展功能，同时也可以在运行时选择不同的装饰器，来实现不同的功能。
- 有效避免了使用继承的方式扩展对象功能而带来的灵活性差，子类无限制扩张的问题。
- 被装饰者与装饰者解偶，被装饰者可以不知道装饰者的存在，同时新增功能时原有代码也无需改变，符合开放封闭原则。

缺点
- 装饰层过多的话，维护起来比较困难。
- 如果要修改抽象组件这个基类的话，后面的一些子类可能也需跟着修改，较容易出错。


```
Android 源码装饰模式：(Decorator)
我们都知道Activity、Service、Application等都是一个Context，
这里面实际上就是通过装饰者模式来实现的。

Context类在这里就充当了抽象组件的角色，ContextImpl类则是具体的组件
，而ContextWrapper就是具体的装饰角色，通过扩展ContextWrapper增加
不同的功能，就形成了Activity、Service等子类。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-57c4ef2d2818ad10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

- 5.外观模式(facade)

```
要求一个子系统的外部与其内部的通信必须通过一个统一的对象进行。外观模式提供一个高层次的接口，使得子系统更易于使用。

外观模式也叫门面模式。
```

![image](https://upload-images.jianshu.io/upload_images/6163786-1ab57bcd01dd1ffa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Facade(外观角色):对外的统一入口。
- Complex System(复杂系统)：一般由多个子系统构成，负责具体功能的实现。

优点
- 降低了客户端与子系统类的耦合度，实现了子系统与客户之间的松耦合关系。
- 外观类对子系统的接口封装，使得系统更易于使用。
- 提高灵活性，不管子系统如何变化，只要不影响门面对象，就可以自由修改。

缺点
- 增加新的子系统可能需要修改外观类的源代码，违背了“开闭原则”。
- 所有子系统的功能都通过一个接口来提供，这个接口可能会变得很复杂。

```
Android 源码使用外观模式(facade)
```
- 6.享元模式(flyweight)

```
使用共享技术可有效地支持大量的细粒度的对象.

享元模式是池技术的重要实现方式，它可以减少重复对象的
创建，使用缓存来共享对象，从而降低内存的使用。

使用场景：
系统存在大量相似或相同的对象。
外部状态相同类似情况下。
需要缓冲池时。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-a6e38b8cbc11b449.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Flyweight(抽象享元角色)：接口或抽象类，可以同时定义出对象的外部状态和内部状态的接口或实现。
- ConcreteFlyweight（具体享元角色）：实现抽象享元角色中定义的业务。
- UnsharedConcreteFlyweight（不可共享的享元角色）：并不是所有的抽象享元类的子类都需要被共享，不能被共享的子类可设计为非共享具体享元类；当需要一个非共享具体享元类的对象时可以直接通过实例化创建。该对象一般不会出现在享元工厂中。
- FlyweightFactory（享元工厂）：管理对象池和创建享元对象。

优点
- 大大减少了系统创建的对象，降低了程序内存的使用。

缺点
- 将对象分为内部状态和外部状态两部分，导致系统变复杂，逻辑也更复杂。
- 将享元对象的状态外部化，而读取外部状态使得运行时间稍微变长。


```
关于享元模式，我们接触到最多的还是Java中的String。如果字符
串常量池中有此字符则直接返回，否则先在字符串常量池中创建字
符串。
```
- 7.代理模式(proxy)

```
为其他对象提供一种代理以控制这个对象的访问。

以海外代购为例，在国内的人想买国外的东西只能去找国外的人去进行代购。


```
![image](https://upload-images.jianshu.io/upload_images/6163786-3faad0f148a5b5a9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Subject（抽象主题类）：接口或者抽象类，声明真实主题与代理的共同接口方法。
- RealSubject（真实主题类）：也叫做被代理类或被委托类，定义了代理所表示的真实对象，负责具体业务逻辑的执行，客户端可以通过代理类间接的调用真实主题类的方法。
- Proxy（代理类）：也叫委托类，持有对真实主题类的引用，在其所实现的接口方法中调用真实主题类中相应的接口方法执行。
- Client（客户端类）：使用代理模式的地方。

从代码的角度来分，代理可以分为两种：一种是静态代理，另一种是动态代理。

根据适用范围，代理模式可以分为以下几种：

- 远程代理：为一个对象在不同的地址空间提供局部代表，这样系统可以将Server部分的事项隐藏。
- 虚拟代理：如果要创建一个资源消耗较大的对象，可以先用一个代理对象表示，在真正需要的时候才真正创建。
- 保护代理：用代理对象控制对一个对象的访问，给不同的用户提供不同的访问权限。
- 智能引用：在引用原始对象的时候附加额外操作，并对指向原始对象的引用增加引用计数。

优点
- 代理作为调用者和真实主题的中间层,降低了模块间和系统的耦合性。
- 可以以一个小对象代理一个大对象,达到优化系统提高运行速度的目的。
- 代理对象能够控制调用者的访问权限，起到了保护真实主题的作用。

缺点
- 由于在调用者和真实主题之间增加了代理对象，因此可能会造成请求的处理速度变慢。
- 实现代理模式需要额外的工作（有些代理模式的实现非常复杂），从而增加了系统实现的复杂度。


```
Android的源码中多个地方都用到代理模式，比如ActivityManagerProxy这个代理类。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-be1f9186a0a790f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

**行为模式：涉及到算法和对象之间职责分配**

- 1.责任链模式(chain of responsibilily)

```
一个请求沿着一条“链”传递，直到该“链”上的某个处理者处理它为止。

多个对象处理同一请求时，但是具体由哪个对象去处理需要运行时做判断。
具体处理者不明确的情况下，向这组对象提交了一个请求。

```

![image](https://upload-images.jianshu.io/upload_images/6163786-104da00f64f61001.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Handler（抽象处理者）：抽象类或者接口,定义处理请求的方法以及持有下一个Handler的引用.
- ConcreteHandler1,ConcreteHandler2（具体处理者）：实现抽象处理类,对请求进行处理,如果不处理则转发给下一个处理者.
- Client (客户端):即要使用责任链模式的地方。

优点
- 代码的解耦，请求者与处理者的隔离分开。
- 易于扩展，新增处理者往链上加结点即可。

缺点
- 责任链过长的话，或者链上的结点判断处理时间太长的话会影响性能，特别是递归循环的时候。
- 请求有可能遍历完链都得不到处理。

```
Android 源码使用责任链模式：
Android中的事件分发机制就是类似于责任链模式;
OKhttp中对请求的处理也是用到了责任链模式
```

- 2.命令模式（Command）

```
将一个请求封装成一个对象，从而让你使用不同的请求把客户端参
数化，对请求排队或者记录日志，可以提供命令的撤销和恢复功能。

命令模式属于行为型模式。
最常见的命令模式就是关机操作了，我们只需点击一下关机按钮
就可以了，至于计算机是如何关机的，我们不需要关心其实现细节。

需要对行为进行记录，撤销，重做，事务处理时。
对于大多数请求——响应模式的功能，比较适合使用命令模式。
```

优点
- 调用者与接受者之间的解藕。
- 易于扩展，扩展命令只需新增具体命令类即可，符合开放封闭原则。

缺点
- 过多的命令会造成过多的类。


```
Android中的源码分析:
1.实际上Thread的使用就是一个简单的命令模式
2.另一个比较典型的常用到命令模式就是Handler了
```

- 3.解释器模式(interpreter)


```
给定一门语言，定义它的文法的一种表示，并定义一个解释器，
该解释器使用该表示来解释语言中的句子。
解释器模式属于行为型模式。
解释器模式实际开发中很少用到。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-adac34772e9ff338.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

```
Android 源码使用解释器模式：
对于AndroidManifest.xml这个文件，由PackageManagerService使
用了PackageParser这个类来解释的，这里面就用到了解释器模式。

```

- 4.迭代器模式(Iterator)

```
提供一种方法访问一个容器对象中各个元素，而又不需暴露该对象的内部细节。
又叫做游标（Cursor）模式;Java中的Map、List等等容器，都使用到了迭代器模式。

```
![image](https://upload-images.jianshu.io/upload_images/6163786-d4b190f34818430e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)
- Iterator（迭代器接口）：负责定义、访问和遍历元素的接口。
- ConcreteIterator（具体迭代器类）:实现迭代器接口。
- Aggregate（容器接口）：定义容器的基本功能以及提供创建迭代器的接口。
- ConcreteAggregate（具体容器类）：实现容器接口中的功能。
- Client（客户端类）：即要使用迭代器模式的地方。

```
除了Java中的Map、List等有用到迭代器模式之外，Android中
使用数据库查询时返回的Cursor游标对象，实际上就是使用了
迭代器模式来实现
```

- 5.中介者模式(mediator)


```
用一个中介者对象来封装一系列的对象交互。中介者使得各
对象不需要显式地相互引用，从而使其松散耦合，而且可以
独立地改变它们之间的交互。
中介者模式也称为调解者模式或者调停者模式。

在程序中，如果类的依赖关系过于复杂，呈现网状的结构，
可以使用中介者模式对其进行解耦。

Android中的锁屏功能就用到了中介者模式，KeyguardService（锁屏服务）
通过KeyguardViewMediator（锁屏中介者）来协调各种Manager的状态以
达到锁屏的功能。
```

- 6.备忘录模式(memento)

```
在不破坏封装性的前提下，捕获一个对象的内部状态，
并在该对象之外保存这个状态，这样以后就可以将该对象
恢复到先前保存的状态。

备忘录模式比较适合用于功能复杂，但是需要维护和纪录历史
的地方，或者是需要保存一个或者多个属性的地方；在未来某
个时刻需要时，将其还原到原来纪录的状态。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-14818e017bdc751f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Originator（发起人角色）：负责创建一个备忘录（Memoto），能够记录内部状态，以及恢复原来记录的状态。并且能够决定哪些状态是需要备忘的。
- Memoto（备忘录角色）：将发起人（Originator）对象的内部状态存储起来；并且可以防止发起人（Originator）之外的对象访问备忘录（Memoto）。
- Caretaker（负责人角色）：负责保存备忘录（Memoto），不能对备忘录（Memoto）的内容进行操作和访问，只能将备忘录传递给其他对象。


```
Android中的源码分析：onSaveInstanceState、onRestoreInstanceState
```
Activity实际上就是负责人角色（Caretaker），负责保存和恢复UI信息；Activity、View、ViewGroup、Fragment等都是发起人角色（Originator），他们各自负责需要保存的信息；而备忘录角色（Memoto）则是Bundle类了，相关状态信息都是保存在Bundle中。

- 7.观察者模式(observer)

```
定义对象间的一种一个对多的依赖关系，当一个对象的状态发送改变时，
所以依赖于它的对象都得到通知并被自动更新。
观察者模式又被称作发布/订阅模式。

使用场景：
当一个对象的改变需要通知其它对象改变时，而且它不知道具体有
多少个对象有待改变时。
当一个对象必须通知其它对象，而它又不能假定其它对象是谁
跨系统的消息交换场景，如消息队列、事件总线的处理机制。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-2d69c5996015cc3c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

优点
- 解除观察者与主题之间的耦合。让耦合的双方都依赖于抽象，而不是依赖具体。从而使得各自的变化都不会影响另一边的变化。
- 易于扩展，对同一主题新增观察者时无需修改原有代码。

缺点
- 依赖关系并未完全解除，抽象主题仍然依赖抽象观察者。
- 使用观察者模式时需要考虑一下开发效率和运行效率的问题，程序中包括一个被观察者、多个观察者，开发、调试等内容会比较复杂，而且在Java中消息的通知一般是顺序执行，那么一个观察者卡顿，会影响整体的执行效率，在这种情况下，一般会采用异步实现。
- 可能会引起多余的数据通知。


```
Android源码使用Observer 观察者模式：
1.控件中Listener监听方式
2.Adapter的notifyDataSetChanged()方法
3.BroadcastReceiver
4.RxJava、RxAndroid、EventBus、otto
```

- 9.状态模式(state)

```
当一个对象的内在状态改变时允许改变其行为，这个对象
看起来像是改变了其类。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-25ebafc72a8b93c6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- 10.策略模式(strategy)

```
定义一系列的算法，把每一个算法封装起来,并且使它们可相
互替换。策略模式模式使得算法可独立于使用它的客户而独立变化。

提供了一组算法给客户端调用，使得客户端能够根据不同的条件来
选择不同的策略来解决不同的问题。
如排序算法，可以使用冒泡排序、快速排序等等
以追妹纸为例，遇到不同的妹纸追求的套路（策略）是不一样的。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-019e977293e65c54.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)
- Strategy(抽象策略类)：抽象类或接口，提供具体策略类需要实现的接口。
- ConcreteStragetyA、ConcreteStragetyB（具体策略类）：具体的策略实现，封装了相关的算法实现。
- Context（环境类）：用来操作策略的上下文环境。

 优点
- 策略类可以互相替换
- 由于策略类都实现同一个接口，因此他们能够互相替换。
- 耦合度低，方便扩展
- 增加一个新的策略只需要添加一个具体的策略类即可，基本不需要改变原有的代码，符合开闭原则。
- 避免使用多重条件选择语句（if-else或者switch）。

缺点
- 策略的增多会导致子类的也会变多
- 客户端必须知道所有的策略类，并自行决定使用哪一个策略类。


```
1.动画中的插值器（ValueAnimator 的 setInterpolator 方法）也是有运用到策略模式
2.通过设置不同的Adapter（即不同的策略），我们就可以写出符合
我们需求的ListView布局。
```
- 11.模板方法(Template Method)

```
定义一个操作中的算法框架，而将一些步骤延迟到子类中，使得
子类可以不改变一个算法的结构即可重定义算法的某些特定步骤。

多个子类中拥有相同的行为时，可以将其抽取出来放在父类中，避免重复的代码。
一次性实现算法的执行顺序和固定不变部分，可变部分则交由子类来实现。
```

优点
- 提高代码复用性，去除子类中的重复代码。
- 提高扩展性，不同实现细节放到不同子类中，易于增加新行为。

缺点：
- 每个不同的实现都需要定义一个子类，这会导致类的个数的增加，设计更加抽象。


```
Android中View的draw方法就是使用了模板方法模式：
Activity的生命周期，AsyncTask等等也是用到了模板方法模式
```

- 11.访问者模式(Visitor)


```
封装某些作用于某种数据结构中各元素的操作，它可以在不改变数
据结构的前提下定义作用于这些元素的新的操作。
```
![image](https://upload-images.jianshu.io/upload_images/6163786-8ef98b2e13a0fa77.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/524)

- Visitor（抽象访问者）：接口或者抽象类，为每一个元素（Element）声明一个访问的方法。
- ConcreteVisitor（具体访问者）：实现抽象访问者中的方法，即对每一个元素都有其具体的访问行为。
- Element（抽象元素）：接口或者抽象类，定义一个accept方法，能够接受访问者（Visitor）的访问。
- ConcreteElementA、ConcreteElementB（具体元素）：实现抽象元素中的accept方法，通常是调用访问者提供的访问该元素的方法。
- Client（客户端类）：即要使用访问者模式的地方。

优点
- 各种角色各司其职，符合单一职责原则。
- 原有的类上新增操作只需实现一个具体访问者就可以，不 必修改整个类层次，符合开闭原则。
- 良好的扩展性，新增访问操作变得简单。
- 数据操作和数据结构的解耦。

缺点：
- 具体元素对访问者公布了实现细节，破坏了类的封装性，违反了迪米特原则。
- 违反了依赖倒置原则，为了达到区别对待依赖了具体而不是抽象。
- 具体元素修改的成本太大。
- 新增具体元素困难，需要在抽象访问者角色中增加一个新的抽象操作，违反了开闭原则。


 **参考**
 
- [Android的设计模式集合](https://www.jianshu.com/p/3e912410f21b)
- [设计模式——刘伟整理gitbook阅读](https://gof.quanke.name/)
- [史上最全设计模式资料汇总——刘伟整理](http://blog.csdn.net/lovelion/article/details/17517213)
- [Android源码设计模式解析与实战](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis)
- [谈谈23种设计模式在Android源码及项目中的应用](http://linbinghe.com/2017/1646c9f3.html)

**源码分析**

- 1.Android线程总结(工厂方法模式)
- 2.RecycleView 源码总结(适配器)
- 3.View/ViewGroup 自定义View
- 4.Context 源码解析
- 5.ActivityManagerProxy ActivityManager ActivityManagerService源码分析
- 6.Android 事件分发机制及Handler机制
- 7.Manifest.xml解析器工作原理
- 8.观察者模式
```
1.控件中Listener监听方式
2.Adapter的notifyDataSetChanged()方法
3.BroadcastReceiver
4.RxJava、RxAndroid、EventBus、otto
```
- 9.Android draw 过程
- 10.Activity生命周期/AsyncTask源码分析 模板方法


