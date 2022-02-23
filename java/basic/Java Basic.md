# Java Basic

## 1. 面向对象

### 1.1 面向对象和面向过程

面向过程：面向过程比面向对象性能要好，因为类的调用和实例化需要开销，比较消耗资源，但是没有面向对象易维护，扩展性没有面向对象好。

面向对象：面向对象易维护，扩展性好，但是性能不好。

### 1.2 在Java中写一个不做事且无参的构造函数

 在Java中，调用子类的构造函数之前如果没有super()调用父类特定的构造方法，则会调用父类中“无参构造函数”，如果父类只定义了有参构造，那么调用不到无参构造就会发生编译错误，所以一般我们都给一个无参构造。

### 1.3 对象实例和对象引用

使用new运算符创建对象实例，对象实例存放在堆中，对象引用指向堆中的对象实例，对象引用存放在栈中。

### 1.4 对象相等和指向它们的引用相等

对象相等，是比较的内存中存放的内容是否相等。引用相等比较的是指向内存地址是否相等。

### 1.5 封装，继承，多态

封装就是将对象的状态信息隐藏在类的内部，不允许外部对象直接访问对象的内部信息，但是可以提供一些方法来操作属性。

继承就是在已有类的基础之上建立新的类，新的类可以增加新的数据或新的功能。1.子类拥有父类对象的所有属性和方法（包括私有），但是无法访问，只是拥有。2.子类可以对父类进行扩展。3.子类可以用自己的方式实现父类的方法。

多态就是多种形态，具体表现为父类的引用指向子类的指针。

### 1.6 接口和抽象类

1. 接口中方法默认是public，方法不能有实现（Java8之后可以有默认实现），抽象类中可以有非抽象方法。
2. 接口中除了static和final类型的变量，不能有其他变量，而抽象类中则不一定
3. 一个类可以实现多个接口，但只能实现一个抽象类。
4. 接口的默认修饰符是public，抽象方法可以有public，protected和default这些修饰符。

### 1.7 Object类的主要方法

```java
public final native Class<?> getClass()//native方法，用于返回当前运行时对象的Class对象，使用了final关键字修饰，故不允许子类重写。

public native int hashCode() //native方法，用于返回对象的哈希码，主要使用在哈希表中，比如JDK中的HashMap。
public boolean equals(Object obj)//用于比较2个对象的内存地址是否相等，String类对该方法进行了重写用户比较字符串的值是否相等。

protected native Object clone() throws CloneNotSupportedException//naitive方法，用于创建并返回当前对象的一份拷贝。一般情况下，对于任何对象 x，表达式 x.clone() != x 为true，x.clone().getClass() == x.getClass() 为true。Object本身没有实现Cloneable接口，所以不重写clone方法并且进行调用的话会发生CloneNotSupportedException异常。

public String toString()//返回类的名字@实例的哈希码的16进制的字符串。建议Object所有的子类都重写这个方法。

public final native void notify()//native方法，并且不能重写。唤醒一个在此对象监视器上等待的线程(监视器相当于就是锁的概念)。如果有多个线程在等待只会任意唤醒一个。

public final native void notifyAll()//native方法，并且不能重写。跟notify一样，唯一的区别就是会唤醒在此对象监视器上等待的所有线程，而不是一个线程。

public final native void wait(long timeout) throws InterruptedException//native方法，并且不能重写。暂停线程的执行。注意：sleep方法没有释放锁，而wait方法释放了锁 。timeout是等待时间。

public final void wait(long timeout, int nanos) throws InterruptedException//多了nanos参数，这个参数表示额外时间（以毫微秒为单位，范围是 0-999999）。 所以超时的时间还需要加上nanos毫秒。

public final void wait() throws InterruptedException//跟之前的2个wait方法一样，只不过该方法一直等待，没有超时时间这个概念

protected void finalize() throws Throwable { }//实例被垃圾回收器回收的时候触发的操作
```



