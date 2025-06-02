---
title: Java核心技术I (Version.12)
tags: [基础知识, ]
category:
  - [技术总结]
mathjax: true
toc: true
date: 2023-10-07 21:52:08
description : 阅读《Java核心技术 卷I》后的一些总结
---
### IV. 对象与类

1. **记录**
   
   记录是一种特殊形式的类，其状态不可变，而且公共可读。如果需要完全由一组变量表示不可变数据，那么应该使用记录
   ```java
   //实例字段x、y称为组件
   record Point(double x, double y){
     //这是自定义构造器
     public Point() {this(0, 0)}   
     
     //【自动定义地】设置所有实例字段的构造器称为标准构造器
     //标准构造器也可以被提供
     public Point(double x, double y){
       ...
     }
     //还有简洁形式，但简洁形式的主体是标准构造器的“前奏”
     //它只是在为实例字段赋值前修改参数变量x、y
     /*
     *  public Point{
     *     if(x > y) y += x;
     *  }
     */
   }
   ```

<br/>

### V.继承

1. **完美equals编写**
   1. 将显式参数命名为otherObject， 稍后需要将它强转为other
   2. 检测 this 与 otherObject 是否相同（这句只是一个小优化）
   3. 检测 other0bject 是否为null, 如果为 null, 则返回 false
   4. 比较 this 与 otherObject 的类。如果 **equals的语义可以在子类中改变**, 就使用`getClass`检测；如果**所有的子类都有相同的相等性语义**，则可以使用`instanceof`检测
   5. 现在根据相等性概念的要求来比较字段。使用`==`比较基本类型字段，使用`Objects.equals()` 比较对象字段。如果所有的字段都匹配, 就返回 true。子类如果重新定义了`equals`，就要包含一个`super.equal(other)`调用
   ```java
   public boolean equals(Object otherObject){
       if(this == otherObject) return true;
       if(otherObject == null) return false;
       
       /* equals 子类语义有改变
       * if(getClass() != otherObject.getClass()) return false;
       * ClassName other = (ClassName) otherObject;
       */
       /* equals 子类语义均一致
       * if(!(otherObject instanceof ClassName other)) return false;
       */
       
       return filed1 == other.filed1
         && Object.equals(filed2, other.filed2)
         &&...
   }
   ```

2. `super`不是一个对象的引用，只是一个指示编译器调用超类方法的特殊关键字
3. 重写的方法，方法签名（指一个**方法的名称和参数列表**）必须一样，而返回类型只需要**更严格**就行（称为有**协变**的返回类型），方法可见性需要更宽松。
4. `protected`类型，同包和子类可访问。但是考虑下面这种情况
   ```java
   public class Fa{
     protected void fun(){}
   }
   
   public class Son1 extends Fa{}
   ```
   ```java
   //Son2在另一个包中
   class Son2 extends Fa{
     void f(){
       //ok
       fun()
       //error: The method fun() from the type Fa is not visible
       new Son1().fun();
     }
   }
   //原因就在于，Son1中的fun方法是从Fa中继承的，【其可见性是对于Son1而言的】
   //只对Son1同包或者Son1的子类可见，Son2中自然不可见
   ```

<br/>

### VI. 接口、lambda与内部类

1. 对于**匿名类**，以下测试会失败
   ```java
   //equals
   if(getClass() != other.getClass()) return false;
   
   //因为每个匿名类类型：外部类类名$1、外部类类名$2、...
   ```

2. 如何在静态方法中获取调用静态方法的类
   ```java
   static void test(){
     this.getClass();    //Error, 静态方法中没有this
     new Object(){}.getClass.getEnclosingClass()   //Success
   }
   ```

3.  Java9后，接口中的方法可以是`private`，但只能用做其他方法的辅助
4. Object中定义了`protected Object clone(){...}`，如果任何类想要实现克隆自己，那么应该在类中重写这个方法`public ClassName clone(){...}`（记得`Cloneable`）。如此一来，在其他类中便可以调用这个类的`clone`方法获取其克隆对象，否则调用`clone`是不可见的，详见 V.4

<br/>

### VII.异常、断言和日志

1. 一个方法必须声明所有可能抛出的**检查型异常**，而**非检查型异常**要么在你的控制之外(Error，jvm层面的错误)，要么从一开始就应该避免（RuntimeException）

2. 异常使用技巧：
   1. **早抛出，晚捕获**。例如，当栈为空时，`Stack.pop`是该返回`null`，还是抛出一个异常？著者认为是后者，因为这要好于以后出现一个`NullPointerException`异常
   2. **使用标准化方式报告异常**
      ```java
      Objects::
        requireNonNull
        checkIndex
        checkFromIndex
        ...
      ```

3. **断言失败**是致命的、不可恢复的错误；不要捕获断言异常，断言是用来测试而不是从中恢复，只适用于测试和开发环境；
   
   私认为，可以类比于用`printf`进行调式

<br/>

### VIII.泛型程序设计

1. **PESC原则：**（将泛型容器看作P或者C）
   1. 频繁往外读取内容的，适合用上界`Extends`
   2. 经常往里插入的，适合用下界`Super`
   
   为何会出现**通配符类型：**假定有`Fruit`类和`Apple`类，二者有继承关系，然后有一个简单的泛型容器`Plate<T>`，代表装...的盘子。问题来了：`Plate<Fruit>`和`Plate<Apple>`没有任何继承关系**（即泛型没有协变性）**， 所以下述语句虽然逻辑上有理，但Java编译器不通过：
   ```java
   Plate<Fruit> p= new Plate<Apple>(new Apple());
   ```
   
   为了让泛型容器之间产生联系，便有了上界通配符`<? extends T>`和下界通配符`<? super T>`，举个例子，`<? extends Fruit>`的子类型有`<Apple>、<Orange>…`， `<? super Apple>`的子类型有`<Fruit>、<Object>…`

2. **虚拟机中的泛型擦除**：
   1. 虚拟机中没有泛型，只有普通的类和方法
   2. 所有类型参数都会替换为它们的**限定类型**`(e.g. T -> Object, U extends Comparable&Runnable -> Comparable，必要时添加Runnable强转)`
   3. 会**合成桥方法**来保持多态
      
      在Java代码中，方法名称和参数类型指定一个方法；而在**虚拟机**中，**方法名称**、**参数类型**和**返回值**共同指定一个方法。故而桥方法可以合理存在；
      ```java
      class A<T>{
        public void fun(T args1){...}
      }
      
      class B extends A<LocalDate>{
        @Override
        public void fun(LocalDate args1){...}
      }
      
      /*上述代码目的是重写fun，但类型擦除后B中会有这俩fun
      * public void fun(Object args1)
      * public void fun(LoaclDate args1)
      * 两个方法显然不一样，但逻辑上应该一样，从而保证多态
      * 因此编译器在B中合成一个桥方法，这才是真正重写fun的方法
      * public void fun(Object args1){fun((LoaclDate)args1)}
      */
      ```
      ```java
      //桥方法不只用于泛型类型，还出现在方法重写
      public class Employee implements Cloneable{
        @Override
        public Employee clone() throws CloneNotSupportedException{...}
      }
      /*实际上，Employee有俩clone方法：
      * Employee clone()    自己定义的
      * Object clone()      合成的桥方法，覆盖了Object.clone
      */
      ```
   4. 为保持类型安全性，必要时会插入强制类型转换

<br/>

### IX.集合

1. `Iterator`迭代器，不要想着迭代器指向某个元素，而应该认为迭代器位于两个元素中间，当调用`next`时，迭代器就会**越过**下一个元素，并返回刚刚越过的那个元素
   
   <img src="/Blog/img/d5605913432ca6c83ef602e065f2ed64.jpg" alt="iterator.jpg" style="zoom:10%;" />
2. 在迭代过程中，如果集合被其他线程修改，迭代失效，抛出`ConcurrentModificationException`异常；但`CopyOnWriteArrayList`和`CopyOnWriteArraySet`除外，**写时拷贝数组**的迭代器在迭代过程中，即使数组被修改仍然可以正常访问（但可能过时）视图。
   
   <br/>

### XII.并发

1. java线程有6种状态：`New` `Runnable` `Blocked` `Waiting` `Timed waiting` `Terminated`
   
   <img src="/Blog/img/a2ee154e13f7cb642148cfc548521932.jpg" alt="thread.jpg" style="zoom:10%;" />
2. 除了已经废弃的`stop`方法，没有办法强制一个线程终止，但可以用`interrupted`来请求终止一个线程。
   ```java
   //阻塞和中断相遇，会抛出InterruptedException
   实例方法isInterrupted 调用后不会改变中断状态 
   静态方法interrupted 调用后会清除【当前执行线程】的中断状态
   ```
3. 同步操作：
   1. **Lock/Condition**
   2. **synchronized**
      ```java
      //等同于使用该对象的内部锁
      public synchronized void method(){
        ...
      }
      //而wait、notifyAll/notify等同于使用内部锁关联的条件对象
      ```
      ```java
      //使用一个对象的锁来实现额外的原子操作，称为客户端锁定
      //但这样完全依赖于：Vector类会对自己的所有更改器方法使用内部锁
      public void method(Vector<Integer> vInt, ...){
          //同步块
          synchronsized(vInt){
              vInt.set(...);
              vInt.set(...);
          }
      }
      ```
   3. **volatile字段**
      
      适用于共享变量只存在**赋值操作**，不会进行其他操作
   4. **线程安全的集合**
      
      阻塞队列、并发散列映射、并发集、写时拷贝数组、使用同步包装器
