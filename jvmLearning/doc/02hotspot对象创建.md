## 一. hotspot中对象的创建过程

>  一个普通的对象是怎样创建的呢？

  jvm在遇到一个new指令时，首先会去检查这个指令的参数能不能在常量池中定位到一个符号的参数引用，并检查这个符号引用代表的类是否被加载，解析和初始化过，如果没有，就执行类加载过程。



  接下来jvm就将为新生对象分配内存。至于如何在堆中给对象分配内存的，有两种策略：

- 利用一个指针作为已使用内存空间与空闲内存空间的中界点，分配的方式称为“指针碰撞”。

- 维护一个堆内存的空闲列表，然后分配可用空间给对象。 这是常用的方式。
  
  

但是在这个分配内存的过程中，如果是并发环境下，操作也不是安全的，解决的方式有两种：

- 使用CAS操作，失败重试；

- 在每个线程的调用时，都会在堆中先分配一小块区域，称为本地线程缓冲TLAB。

接下来就是设置对象头（Class类的信息，锁的状态，hashcode，GC分代年龄等），对象体（实际数据）等相关信息。



## 二.对象的内存布局

### 在 HotSpot 虚拟机中，对象的内存布局分为以下 3 块区域：

* 对象头（Header）  
* 实例数据（Instance Data）  
* 对齐填充（Padding）

![images\object-memory-layout](images\object-memory-layout.png)



如上图所示，接下来就具体分析这三部分内容：



#### 对象头

hotspot的对象头包括两部分信息：

第一部分用来存储对象自身的运行时数据，如hashcode、GC分代年龄、与锁相关的信息（Mark Word，比如偏向锁信息，指向轻量级锁的指针、指向重量级锁的指针信息等）。

第二部分是类型指针，即对象指向它的类元数据的指针，JVM通过这个指针来确定这个对象数据哪个类。



#### 实例数据

对象真正存储的有效信息，也就是在代码中定义的各种字段的内容，无论是从父类继承下来的，还是在子类定义的，都需要记录。



#### 对齐填充

仅起着占位符的作用，因为hotspot中对象的起始地址必须是8字节的整数倍。



## 三：对象的访问方式

有两种方式，如下：

### （1）：使用句柄：

堆会划分一块区域给句柄池，栈中的引用存储的就是对象的句柄地址，而句柄包含了对象的实例信息和元数据区域的该类对应的地址信息。

使用句柄的优势是，当更改对象的实例时，只会更改句柄池中存储对象实例的地址，栈中的引用可以不用更改。  

![images\handle-access](images\handle-access.jpg)



### ( 2 ) : 使用直接引用（指针）：

就是将栈中的引用指向堆中具体的对象实例的地址，我们平时用的就是这个，可以简单认为,栈中的引用变量保存了对象的地址拷贝，但是这个引用一定要明白，因为经常会遇到String值不可变的问题，考的就是引用。  

 好处是访问速度快，坏处是对象地址更改时，引用指针也要重新引用。

![images\direct-pointer](images\direct-pointer.jpg)





## 四：jvm结构中的各种OOM情况

### (1)栈中的OOM

分为两种情况：

- 比如使用不规范的递归，导致栈空间请求深度溢出，导致OOM。

- 栈空间的数据量过大，导致OOM。



### （2）堆中的OOM

看看是不是内存溢出（比如ThreadLocal导致的OOM），或者是内存泄漏的情况，都会导致OOM异常。



### （3）方法区中的OOM（现元数据区）

主要是因为反射机制或者cglib动态生成新的类的容量超过了对应的内存，导致的OOM。



## 五：String.intern方法

这是一个本地的方法，为了具体探究一下它，就直接分析一下String类的源码吧。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
    
     public String() {
        this.value = new char[0];
    } 
     public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
    

}
```



从上面的源码中可以发现，String内部就是维护着一个final修饰的char数组，final这个关键字比较熟悉，final修饰的内容都是最终的，修饰的类无法被继承，修饰的方法无法被重写，修饰的字段只能赋值一次，而且除了直接复制外，只能在构造方法中赋值，因为final的指令的内存屏障设计就是禁止把final域的写重排序到构造方法之外。

扯远了，详细可以看看并发编程中的重排序。  总之，final修饰的值只能被赋值一次，不能被修改，但是我们知道，引用指向数组对象时，实际上指向的是对象的首地址，即数组的地址并不能被改变，数组中的内容还是可以改变的，比如利用反射机制就可以在运行时更改char数组中的元素。



```java
/**
     * Returns a canonical representation for the string object.
     * <p>
     * A pool of strings, initially empty, is maintained privately by the
     * class {@code String}.
     * <p>
     * When the intern method is invoked, if the pool already contains a
     * string equal to this {@code String} object as determined by
     * the {@link #equals(Object)} method, then the string from the pool is
     * returned. Otherwise, this {@code String} object is added to the
     * pool and a reference to this {@code String} object is returned.
     * <p>
     * It follows that for any two strings {@code s} and {@code t},
     * {@code s.intern() == t.intern()} is {@code true}
     * if and only if {@code s.equals(t)} is {@code true}.
     * <p>
     * All literal strings and string-valued constant expressions are
     * interned. String literals are defined in section 3.10.5 of the
     * <cite>The Java&trade; Language Specification</cite>.
     *
     * @return  a string that has the same contents as this string, but is
     *          guaranteed to be from a pool of unique strings.
     */
    public native String intern();
    
```

通过百度翻译，可以发现，当调用intern方法时，如果字符串常量池中已经包含了这个对象，则返回字符串常量池中的对象，否则，就将这个对象添加到常量池中，并返回此对象的引用。  但是在jdk1.7之后，为了减小内存开销，**如果字符串常量池中没有这个对象，那么就会将这个对象的引用拷贝到字符串常量池中，并返回该引用**，这一点非常重要，在面试时遇到字符串的判断时，需要记得这一点。

>  忘了还有一点，String a="aaa";   直接创建到字符串常量池中
> 
> String a=new String("aaa")； 对象分配在堆内存中。
> 
> 所以说这两个的地址是不同的。 



下面有个例子：

```java
public class StringTest {
    public static void main(String[] args) {
        String b="aaa";
        String a=new String("aaa");
        //false
        System.out.println(a==b);

        //true 
        System.out.println(a.intern()==b);
    }
}
```
