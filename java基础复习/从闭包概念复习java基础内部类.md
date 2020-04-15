## 一：前言

昨天晚上玩手机时看到这么一条消息，“不会闭包就xxxxxxx”，好巧，我不会，尴尬，然后去百度了一下， 

> 闭包就是能够读取其他函数内部变量的函数。例如在javascript中，只有函数内部的子函数才能读取[局部变量](https://baike.baidu.com/item/%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F/9844788)，所以闭包可以理解成“定义在一个[函数](https://baike.baidu.com/item/%E5%87%BD%E6%95%B0/301912)内部的函数“。在本质上，闭包是将函数内部和函数外部连接起来的桥梁。

闭包这个概念源于离散数学，在计算机中早就被广泛应用了，它的特点是未绑定到特定对象。   我是这么理解的，闭包就是个函数，但是这个函数可以定义到函数体内部，并且可以访问外部函数的局部变量。 而且因为函数可以作为第一等公民，所以闭包也可以被返回（不理解就联想到对象，对象就是第一等公民，既然可以return对象，那么也就可以return函数），从而使这个闭包持有一个外部函数的引用，既然有外部引用了，根据java学的GC原理，那么这个外部函数也不会被GC。

然后就回到了java，很多人说java没有闭包，但是个人觉得其实是有的，java中的内部类设计就起到了闭包的作用，但仅仅是面向对象的闭包设计，java8的函数式（lambda表达式）满足了函数式编程的要求，但其本质是一个语法糖，相当于封装了内部类的结构。  

回想起内部类，发现自己对内部类的概念很陌生，还是基础很水，那么就复习一下，复习内容：java编程思想第十章。



## 二：内部类分析

内部类，顾名思义，定义在类内部的类，有普通，静态，匿名，局部内部类之分，特性也不相同。需要注意的是，把匿名内部类与类代码块的概念分清。

### 1：普通内部类

```java
/*
* 普通内部类
* 迭代器就是利用内部类实现的
* */
public class OrigOuter {
    private Object[] items;
    private int next=0;

    public OrigOuter(int size) {
        items=new Object[size];
    }
    public void add(Object object){
        if(next<items.length){
            items[next]=object;
            next++;
        }
    }
    public OriInner getOriInner(){
        return new OriInner();
    }

    public static void main(String[] args) {
        OrigOuter out=new OrigOuter(10);
        for (int i = 0; i <10 ; i++) {
            out.add(Integer.toString(i));
        }
        Selector selector=out.getOriInner();
        while (!selector.end()){
            System.out.print(selector.current()+"  ");
            selector.next();
        }
          //显示地创建内部类对象的格式
        OrigOuter.OriInner inner=out.new OriInner();
    }

    private class OriInner implements Selector{
        private int i=0;
         //在内部类中获取外部类的对象，利用this关键字就行
        public OrigOuter getOuter(){
            return OrigOuter.this;
        }
        @Override
        public boolean end() {
            return i==items.length;
        }
        @Override
        public Object current() {
            return items[i];
        }
        @Override
        public void next() {
            if(i<items.length){
                i++;
            }
        }
    }
}
interface Selector{
    boolean end();
    Object  current();
    void  next();
 }
```

上面的就是利用普通内部类实现的迭代器，我们可以发现，内部类OriInner可以获取到外部类的属性，在本例中获取到了外部类的item数组。 那么为什么内部类对象能够获取到外部类对象的字段呢？

> 当某个外部类对象创建一个内部类对象时，此内部类对象必定会秘密获得一个指向外部类对象的引用指针。 因为有个这个引用，所以内部类对象才能获取到外部类对象的字段。 这个引用是编译后自动产生的，如果编译器访问不到这个引用，会报错。



在内部类中，如果需要获取一个对外部类的引用，那么只需要返回外部类名.this就行。就如同上面的代码一样。  因为this关键字本就指的当前对象。



如果需要比如在main方法中创建该内部类对象，只需要 外部类对象.new 内部类()就行。代码上面也有。

> 注意：在拥有外部类对象之前是不可能有内部类对象的，因为内部类对象会偷偷地持有一个外部类对象的引用。当然这只是普通内部类的情况，如果是静态内部类，因为静态内部类中是不能有非静态资源的，所以它就不需要对外部类对象的引用。



#### 内部类与向上转型

private修饰的内部类，可以阻止任何依赖于类型操作的编码，隐藏内部的实现细节。

### 2：在方法和作用域中定义内部类

在类体里定义内部类比较容易理解，但是内部类的作用范围不仅仅于此。

例如可以在一个方法或者任意的作用域里定义内部类，理由如下：

- 实现了某类型的接口，可以创建并返回对其的引用。

- 要解决一个复杂的问题，需要内部类，但是又不想这个内部类公共可用。

代码如下：

```java
public class OuterDemo {
    //方法体内部定义内部类
    public Destination get(String val){
        class DestinationImpl implements Destination{
            @Override
            public String value() {
                return val;
            }
        }
        return new DestinationImpl();
    }
    //在作用域中内嵌一个内部类
    public void interTracking(boolean b){
        if(b){ //在if作用域中内嵌一个内部类
            class Tracking{
                private int id;
                public Tracking(int id) {
                    this.id = id;
                }
                public int getId() {
                    return id;
                }
            }
            Tracking t=new Tracking(22);
            int k=t.getId();
        }
    }
}
interface Destination{
    String value();
}
```

可以发现，完全就可以在方法体的内部定义一个类，它在编译时也是编译过的，但是只能在相应的作用域内起作用，否则就是不可用的。  而这，就是java中的闭包实现。

当然，为了简化操作，于是可以在方法体中不需要类名，也就是匿名内部类。

#### 匿名内部类

```java
//匿名内部类，基类是无参的，比如结构
 public Destination getNoName(String str){
        return new Destination() { //这就是匿名内部类对象的创建，使用默认的构造方法

            @Override
            public String value() {
                return str;
            }
        };
    }
```

匿名内部类的具体语法是：创建一个继承自xx 的匿名类对象，这里就是创建了实现Destination的匿名实现类对象。 而且这个匿名类对象也还可以访问外部方法中的参数。 

但是上面的例子仅仅是使用了默认的构造方法来创建一个匿名类对象。那需要一个有参的构造器该怎么写？ 如下：

```java
 //匿名内部类，如果基类是有参的构造函数
    public A getNoName1(String str){
        return new A(str){  //继承自A的匿名内部类对象， （）中传入的是构造的参数，在jdk8引入lambda后，传入的参数是符合@function 接口中的方法参数

            public String getVal(){
                return getStr();
            }
        };
    }

class A{
    private String str;
    public A(String str) {
        this.str = str;
    }

    public String getStr() {
        return str;
    }
}
```

以上就是我们经常用到的匿名内部类的构建模式， 比如lambda表达式,:  () -> {}

，前面的（）小括号中就是传入的参数，后面的{ }就是匿名内部类对象的结构，说白了就是对上面代码的简化，lambda就是一个语法糖。

在java中，匿名内部类中可以访问到外部方法体中的变量的，但是变量必须是final修饰的，也就是说java不提供闭包对外部变量的修改，只是提供了访问。这点需要清楚。

```java
  //匿名内部类,基类是无参的，比如接口，
    public Destination getNoName(String str){
        int x=12;
        return new Destination() { //这就是匿名内部类
            @Override
            public String value() {
                if(x==12){
                  //    x=1;引用的内部类以外的参数引用必须是final的
                    System.out.println("外部方法中的变量必须是final修饰的");
                }
                return str;
            }
        };
    }
```

> 在java中，匿名内部类调用外部方法区的参数引用，其必须是final的。即只能引用，不能更改外部变量。



比如线程的创建，就是经典的匿名内部类的应用。

```java
 public static void main(String[] args) {
        Thread t1=new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("线程的创建就是匿名内部类的典型");
            }
        });
        new Thread(()->{
            System.out.println("lambada表达式对线程创建的简化");
        }).start();
    }
    
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
//但是在java中，使用lambda表达式，必须定义@FunctionalInterface修饰的接口方法
```



### 3:嵌套类（static）：

将内部类声明为static，即为嵌套类（与C++的嵌套类概念类似）。使用static内部类的作用：

- 要创建嵌套类的对象，并不需要其外围对象的引用。

- 不能从嵌套类对象中访问非静态的外围类对象。

嵌套类中没有指向外围类的一个引用，它更相当于一个外部类的静态方法，所以常用来创建线程安全的单例模式。

```java
//静态内置类的方式实现线程安全的单例
public class Outer {
    private static class Inner{
        public static Outer outer=new Outer();
    }
    private Outer() {
    }
    public static Outer getOuter(){
        return Inner.outer;
    }
}

```



### 4：为什么需要内部类。

> 每个内部类都可以继承一个外部类，相当于实现了“多继承”，比起C++的多继承来说，java的内部类结构就简单多了。
> 
> 而且具有灵活性。当一个类中必须继承两个类实现功能时，这个时候用内部类就很方便了，不用再多层继承了。

还有以下特性：

- 内部类可以有多个实例，每个实例都有自己的状态信息，并且与外部类对象的信息相互独立。（普通内部类中通过内部类对象隐式持有一个指向外部对象的引用来联系。）

- 单个外围类中，可以用多个内部类实现同一个接口，或者继承同一个类，这样做的好处其实能立马相当，就是简化工厂方法的结构。

- 创建内部类对象的时刻并不依赖于外围类对象的创建。

- 内部类就是一个独立的实体。



> 内部类就是面向对象的闭包，因为它不仅包含外围对象（创建内部类作用域的）的信息，还自动拥有一个指向外围类对象的引用，在此作用域内，内部类有权操作所有的成员，包括private成员。



#### 回调

众所周知，java中没有实现指针，所以就不能通过指针实现回调函数，回调函数的实现是通过内部类的闭包功能实现的。 具体是在内部类中的相关方法通过.this获取外围对象，调用外部类对象的相关方法。  

```java
public interface Incrementable {
    void increment();
}

public class MyIncrement {
    public void increment(){
        System.out.println("Other Operation.");
    }
    public static void f(MyIncrement m){
        System.out.println("直接调用");
        m.increment();
    }
}

public class Callee2 extends MyIncrement {
    private int i=0;

    @Override
    public void increment() {
        super.increment();
        i++;
        System.out.println("Callee2:"+i);
    }
      //内部类实现了Incrementable，以提供返回Caller2的钩子，而且还是一个安全的钩子。
    private class Closure implements Incrementable{
        @Override
        public void increment() {
            Callee2.this.increment(); //使用内部类就可以轻松实现回调函数
        }
    }
    public Incrementable getCallBackRef(){ //获取回调引用，即获取钩子

        return new Closure();
    }
}

public class Caller {
    private Incrementable callbackReferance;

    public Caller(Incrementable cbh) {
        this.callbackReferance = cbh;
    }
    public void go(){
        callbackReferance.increment();
    }

    public static void main(String[] args) {
        Callee2 c2=new Callee2();
        MyIncrement.f(c2);
        //获取回调引用
        Caller c=new Caller(c2.getCallBackRef());
        c.go(); //调用回调函数，即钩子的调用

    }
}
```

> 利用内部类对象对外部类对象的这个隐式的指针引用，就可以实现回调函数。  



#### 控制框架

应用程序框架被设计用来解决某个特定问题的一个类或者一组类。 主要采用的是模板方法的设计模式，对于模板方法，就是先定义好主要主要步骤，然后需要自己实现模板方法中的需要重写的方法步骤，具体可参考并发包中的AQS类，这个类就是依靠模板方法模式设计的。

控制框架是一种特殊的应用程序框架，它用来解决响应事件的需求。主要用来响应事件的系统被称为响应驱动系统。

内部类在控制框架中的作用如下;

- 控制框架的完整实现是由单个的类创建的（比如Event类），从而使得实现的细节被封装了起来。 内部类用来表示解决问题的action（Event类被内部类继承重写action方法）。

- 内部类可以很容易地访问外围类的成员。



## 三：总结：

今天通过闭包的概念重写学习了内部类， 闭包是可以访问外部函数变量的内部函数，也可以将闭包函数返回使用。  但是java是面向对象的，所以只有类的概念，没有函数的语法，java中的除了不能将函数返回之外，其余的闭包功能都有相应的实现。

<br>

java的闭包就是内部类，具体是普通内部类，内部类对象在创建时会隐式持有一个指向外部类对象的引用，所以它就可以访问外部类的成员变量和调用外部类的方法，迭代器就是利用内部类设计的；  同时在内部类可以通过  外部类名.this 就可以获取到外部类的当前对象（还是通过那个隐式指针获取的），利用这个作用就可以将内部类当做一个钩子，调用外部类的相关方法，实现回调函数的功能。

<br>

至于匿名内部类，它就是定义在方法体和作用域内部的内部类的简写，匿名内部类可以访问方法体中的外部变量，但是这些外部变量必须是final修饰的，即不可修改，java8中的lambda表达式就是对匿名内部类封装，lambda相当于语法糖功能。

> 匿名内部类在表现上最符合回调函数的写法，比如js中的闭包就是内部函数可以访问外部函数的变量，且可以修改变量，函数能返回，并且持有外部函数的一个引用（类比于内部类）。    而匿名内部类仅仅是可以获取外部变量的值，并不能修改传入的外部变量， 能返回的也只是对象。

<br>

至于嵌套类，即static修饰的内部类，因为static与对象无关，所以嵌套类就不会对外部类对象持有引用，简单来说，嵌套类对外的表现形式相当于一个静态方法。




