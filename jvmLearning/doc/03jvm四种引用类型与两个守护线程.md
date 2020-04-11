## 垃圾收集的算法与策略

在jvm中，线程私有的区域程序计数器，栈，本地方法栈都会随着线程自生自灭，所以我们不必担心这些区域的内存回收。  但是堆是共享区域，堆中创建的对象又不能自生自灭，所以必须需要GC手段去按一定的策略清理堆中无用的对象。对于 Java 堆和方法区，我们只有在程序运行期间才能知道会创建哪些对象，这部分内存的分配和回收都是动态的，垃圾收集器所关注的正是这部分内存。

### 一：判断一个对象是否存活

若一个对象不被任何对象或变量引用，那么它就是无效对象，需要被回收。

### 判断是否回收的策略：

#### (1)引用计数算法：

维护一个计数器，一个对象被引用一次，计数器就+1，如果该引用失效，计数器就-1。但是这种方法有个问题，万一堆中的两个甚至多个对象成环形引用呢，那么就无法被回收了。所以这种方式已经被摒弃了。

#### （2）可达式分析算法：

所有和 GC Roots 直接或间接关联的对象都是有效对象，和 GC Roots 没有关联的对象就是无效对象。  

GC Roots 是指：  

* Java 虚拟机栈（栈帧中的本地变量表）中引用的对象  （传入的参数引用）
* 本地方法栈中引用的对象  
* 方法区中常量引用的对象  （static final）
* 方法区中类静态属性引用的对象  （比如static 字段引用的对象）

GC Roots 并不包括堆中对象所引用的对象，这样就不会有循环引用的问题。说白了，就是与外部的结构断开联系，比如一个对象与栈中的一个引用指针构成一条链，当该引用指针不再指向该对象时，该对象就会通知被GC。 但是不仅仅是栈引用，类中的字段也可能会引用对象（比如static 字段引用的对象）。

## 二：java中的四种引用：

判定对象是否存活与“引用”有关。在 JDK 1.2 以前，Java 中的引用定义很传统，一个对象只有被引用或者没有被引用两种状态，我们希望能描述这一类对象：当内存空间还足够时，则保留在内存中；如果内存空间在进行垃圾手收集后还是非常紧张，则可以抛弃这些对象。很多系统的缓存功能都符合这样的应用场景。  

在 JDK 1.2 之后，Java 对引用的概念进行了扩充，将引用分为了以下四种。不同的引用类型，主要体现的是对象不同的可达性状态`reachable`和垃圾收集的影响。

### 强引用（Strong Reference）

类似 "Object obj = new Object()" 这类的引用，就是强引用，只要强引用存在，垃圾收集器永远不会回收被引用的对象。但是，如果我们**错误地保持了强引用**，比如：赋值给了 static 变量，那么对象在很长一段时间内不会被回收，会产生内存泄漏。  

可以总结为：

- 强引用可以直接访问目标对象。
- 强引用所指向的对象在任何时候都不会被系统回收。
- 强引用可能导致内存泄漏。

### 软引用（Soft Reference）

软引用是一种相对强引用弱化一些的引用，可以让对象豁免一些垃圾收集，只有当 JVM 认为内存不足时，才会去试图回收软引用指向的对象。JVM 会确保在抛出 OutOfMemoryError 之前，清理软引用指向的对象。软引用通常用来**实现内存敏感的缓存**，如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。  

```java
SoftReference<Bean> bean = new SoftReference<Bean>(new Bean("name", 10)); 
System.out.println(bean.get());// “name:10”
```

软引用有以下特征：

- 软引用使用 get() 方法取得对象的强引用从而访问目标对象。
- 软引用所指向的对象按照 JVM 的使用情况（Heap 内存是否临近阈值）来决定是否回收。
- 软引用可以避免 Heap 内存不足所导致的异常。
- 软引用适用于实现内存敏感的缓存；

当垃圾回收器决定对其回收时，会先清空它的 SoftReference，也就是说 SoftReference 的 get() 方法将会返回 null，然后再调用对象的 finalize() 方法，并在下一轮 GC 中对其真正进行回收。

### 弱引用（Weak Reference）

弱引用的**强度比软引用更弱**一些。当 JVM 进行垃圾回收时，**无论内存是否充足，都会回收**只被弱引用关联的对象。  所以是只能存活到下一次GC之前,剩下的特性与软引用基本相同。

- 弱引用适用于实现无法防止其键（或值）被回收的规范化映射

### 虚引用（Phantom Reference）

虚引用也称幽灵引用或者幻影引用，它是**最弱**的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响。它仅仅是提供了一种确保对象被 finalize 以后，做某些事情的机制，比如，通常用来做所谓的 Post-Mortem 清理机制。  虚引用指向的对象一般不会被自动回收，所以client需要像强引用一样对其进行处理回收。

虚引用有以下特征：

- 虚引用永远无法使用 get() 方法取得对象的强引用从而访问目标对象。
- 虚引用所指向的对象在被系统内存回收前，虚引用自身会被放入 ReferenceQueue 对象中从而跟踪对象垃圾回收。
- 虚引用不会根据内存情况自动回收目标对象。
- 

> 其实 SoftReference, WeakReference 以及 PhantomReference 的构造函数都可以接收一个 ReferenceQueue 对象。当 SoftReference 以及 WeakReference 被清空的同时，也就是 Java 垃圾回收器准备对它们所指向的对象进行回收时，调用对象的 finalize() 方法之前，它们自身会被加入到这个 `ReferenceQueue 对象`中，此时可以通过 ReferenceQueue 的 poll() 方法取到它们。而 PhantomReference 只有当 Java 垃圾回收器对其所指向的对象真正进行回收时，会将其加入到这个 `ReferenceQueue 对象`中，这样就可以追综对象的销毁情况。

### 三：Ref包的分析：

参考网址：[深入探讨java.lang.ref]([https://www.ibm.com/developerworks/cn/java/j-lo-langref/](https://www.ibm.com/developerworks/cn/java/j-lo-langref/)

百度百科：[java.lang.ref]([https://baike.baidu.com/item/java.lang.ref/5179901?fr=aladdin](https://baike.baidu.com/item/java.lang.ref/5179901?fr=aladdin))

可以详细看看百度百科，原理解释的已经比较详细，下面对照源码来理解一下：

#### 一：ref包的作用：

这个包是一个特殊的包，它提供了与java垃圾回收器密切相关的类，这些类就是上面提到的软、弱、虚引用，这些引用类对象可以指向其他的对象，但是它们的存在并不妨碍垃圾收集器对它们所指向的对象进行回收。 程序可以使用一个引用对象来维持对另外某一对象的引用，所采用的方式是使后者仍然可以被回收器[回收](https://baike.baidu.com/item/%E5%9B%9E%E6%94%B6/6369931)。程序还可以安排在回收器确定某一给定对象的可到达性已经更改之后的某个时间得到通知。  所以这个包在用来实现缓存时特别有用。

    下图为该包中的类的UML图：

![images\java_lang_ref](images\java_lang_ref.jpg)      

可以发现Reference 是一个抽象类，而 SoftReference，WeakReference，PhantomReference 以及 FinalReference 都是继承它的具体类。

接下来我们来分别介绍和分析强引用以及 java.lang.ref 包下各种虚引用的特性及用法。

#### 二：Reference类：

```java
public abstract class Reference<T> {
    /*
    引用的实例有四种可能的状态:
    活跃状态：服从于垃圾回收的特别处理,有时在收集器检测到可达到的引用对象已经改变成为适当的状态，收集器改变实例的状态为挂起或者不活跃，
    挂起状态: 一个挂起引用列表中等待被引用处理线程排队的元素。没有注册的实例不可能有这个状态。
     排队状态：一个队列里面的在创建时就被注册的实例元素。当一个实例被从他自己的引用队列移除，他就变成不活跃状态了。没有注册的实例不可能有这个状态。
     不活跃状态：无所事事。一个实例变成不活跃状态就不可能再改变状态了。
     状态像下面这样被编码到queue和next字段：
     *
     * 活跃：实例注册时queue = ReferenceQueue或者如果实例没有被注册queue=ReferenceQueue.NULL;next=null.
     *
     * 挂起：实例注册时queue = ReferenceQueue;next=queue里面的下一个实例，如果实例是队列最后一个元素，next=this
     *
     * 排队：queue = ReferenceQueue.ENQUEUED;next=queue里面的下一个实例，如果实例是队列最后一个元素，next=this
     *
     * 不活跃：queue = ReferenceQueue.NULL; next = this.
     *
     * 有了这些约束，收集器为了确定一个引用对象是否需要特别对待只需要检查next字段：如果next字段是null这个实例就是活跃的；如果不为null，这收集器就应该正常对待这个实例了。 

    */
      private T referent;         /* Treated specially by GC */

    //这个引用队列就是一个单向链表，用来添加引用对象，然后配合一个守护线程进行引用对象的回收

    volatile ReferenceQueue<? super T> queue; //维护一个引用队列
      @SuppressWarnings("rawtypes")
      Reference next;

    //返回引用的对象，不同的子类重写了get
      public T get() {
        return this.referent;
    }
    //清空这个对象（构造方法传入的对象）的引用referent，那么，
    //这个对象在没有外部引用的情况下，就会被GC回收。只要调用了这个方法，则代表着不用将该对象加入到引用队列中就可以快速删除
     public void clear() {
        this.referent = null;
    }
    /*构造方法*/
    Reference(T referent) {
        this(referent, null);
    }
    //通过构造方法可以发现，传入的对象引用给了referent属性

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
}
```

可以发现在构造方法中，通过referent属性就可以指向传入的对象（堆中创建的对象），然后该类提供了相应的get，clean方法进行获取该对象或者回收该对象。

> 传入的对象：  比如 new String(“aaa”)；   例如软引用就是给这个对象加一个软引用的外部链接，那么这个对象在没有其他外部引用的情况自然而然就是一个软引用对象。

#### Reference子类

##### （1）SoftReference

```java
public class SoftReference<T> extends Reference<T> {
    //时间戳的时钟，更新于垃圾收集时
    static private long clock;
    //时间戳
     private long timestamp;

  public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }

 public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }
    //获取堆中实际的对象，并更新最近一次垃圾收集的时间戳
     public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }
}
```

发现软引用类就仅仅只添加了一个最新gc时间的时间戳属性，其余皆是父类定义好的操作。 猜想这个时间戳是用来判断最近一次gc后内存的状态，如果将要OOM，就会自动清空软引用的对象。      所以可以作为缓存

Example:

```java
public static void main(String[] args) { 
SoftReference<Bean> bean = new SoftReference<Bean>(new Bean("name", 10)); 
System.gc(); 
System.runFinalization(); 

System.out.println(bean.get());// “name:10”
}
//上面展示内存充足时，软引用对象不会被回收 ，来看这部分：
public static void main(String[] args) { 
 Reference<Bean>[] referent = new SoftReference[100000]; 
 for (int i=0;i<referent.length;i++){ 
 referent[i] = new SoftReference<Bean>(new Bean("mybean:" + i,100)); 
 } 

 System.out.println(referent[100].get());// “null”
 }
//发现软引用对象在内存即将OOM时就会全部被回收， 这就是作为缓存的依据，如果这里是是强引用创建，那么早就OOM了。
```

##### （2）WeakReference

```java
public class WeakReference<T> extends Reference<T> {
    public WeakReference(T referent) {
        super(referent);
    }
      public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

弱引用与软引用基本相似，区别是，虚引用的对象在下次gc时就自动回收。

Example:

```java
public static void main(String[] args) {
 WeakReference<Bean> bean = new WeakReference<Bean>(new Bean("name", 10));   //可以不用传入引用队列就可以自动在下一次GC时回收
 System.gc();
 System.runFinalization();
 System.out.println(bean.get());// “null” ，返回了null，证明已经被回收
 }
```

##### (3) PhantomReference

```java
public class PhantomReference<T> extends Reference<T> {
     public T get() {
        return null;
    }
       public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

虚引用的区别还挺大的，发现它并不允许获取它的引用的实际对象，而且它必须配合引用队列生成虚引用的对象，  也就证明了虚引用对象必须加入到引用队列中才能被回收。    虚引用类似于强引用，需要人为地对实际的对象进行回收操作（例如调用GC），不能像软、弱引用那样被jvm自动调用回收。

Example:

```java
public static void main(String[] args) { 
ReferenceQueue<String> refQueue = new ReferenceQueue<String>(); 
//只有传入了引用队列才能回收
PhantomReference<String> referent = new PhantomReference<String>(
    new String("T"), refQueue);
     
    //get返回null，表示虚引用不能获取到对象实例
System.out.println(referent.get());// null 

//通知gc，显示调用gc才能完成对虚引用对象的回收
System.gc(); 
System.runFinalization(); 

System.out.println(refQueue.poll() == referent); //true 
}
```

可以发现，需要显示通知gc才能让虚引用对象在引用队列中进行回收。



### 自动清除引用（百度百科）

在将软引用和弱引用添加到向其注册的队列（如果有）之前，回收器将自动清除这些引用。所以，软引用和弱引用不需要向队列注册即可使用，而虚引用则需要这样做。通过虚引用可到达的对象将仍然保持原状，直到清除所有这类引用或者它们本身变得不可到达。



好了，通过以上的三个类的分析，对于软、弱、虚引用对象应该有个一个清晰的认识，对于软引用和弱引用来说，可以不用配合引用队列，jvm会在gc时做出判断，对其自动回收， 而虚引用必须加入到一个引用队列中，需要显示调用gc，才能使得虚引用对象在引用队列中被回收。  那么，问题来了，这个回收过程的逻辑到底在哪呢？

我们回过头，再来看一下Reference类：

```java
  transient private Reference<T> discovered;  /* used by VM */
 static private class Lock { };   //垃圾收集器的锁

    private static Lock lock = new Lock();
     //等待排队的引用列表。当引用处理器线程移除引用，收集器就他们加到这个列表。这个列表被上面的锁对象保护。
      private static Reference<Object> pending = null;
      
       private static class ReferenceHandler extends Thread {

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            for (;;) {
                Reference<Object> r;
                synchronized (lock) {
                    if (pending != null) {//回收引用对象的操作

                        r = pending;
                        pending = r.discovered;
                        r.discovered = null;
                    } else {
                        try {
                            try {
                                lock.wait(); //如果引用队列为空，就wait，让这段同步块代码等待

                            } catch (OutOfMemoryError x) { }
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }
                //当然也可以快速回收，通过调用clean方法，回收不可达的引用对象，这里的Cleaner机制就是利用了虚引用。

                // Fast path for cleaners
                if (r instanceof Cleaner) {
                    ((Cleaner)r).clean();
                    continue;
                }
            
                ReferenceQueue<Object> q = r.queue;
                //将引用对象加入到引用队列中，然后内部调用lock.notifyAll，唤醒等待的同步块代码

                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }
    
    //利用静态代码块，在类加载时启动一个优先级最高级别的守护线程，用来对引用对象的回收
  static {
         //利用最上层的线程组，  
         //主线程有一个main线程组[Thread[main,5,main], null, null, null]
        //上面还有一个system线程组[Thread[Reference Handler,10,system], Thread[Finalizer,8,system], Thread[Signal Dispatcher,9,system], Thread[Attach Listener,5,system]]。
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
```

发现，Reference类在类加载时就已经利用静态代码块通过jvm中的线程组，加入了一个守护线程---ReferenceHandler引用处理线程，这个守护线程的作用就是对引用对象的处理与回收，对于软或者弱引用对象，可以直接对其进行回收，不用通过引用队列的就可以完成。  而对于虚引用对象，必须将虚引用的对象将入到引用队列中，然后加锁（垃圾收集器的锁）进行回收， 注意：对于引用队列的操作都是加了同步的。



> 还可以从中理解到调用main方法后会创建几个线程，创建的最上层的线中的线程，初始5个，分为main主线程， ReferenceHandler：引用处理线程，Finalizer：与垃圾回收相关的线程， Signal Dispatcher:用于通知调度的线程;
> 
> Attach Listener:监听jvm运行中信息的线程



现在，我们可以说把各种引用的特性与回收原理大致弄清了，对于引用的理解，不再是书上那么简短的几句内容了，但是呢，还是有个问题，对于软引用和虚引用对象能干什么我们已经了解的比较明白了。但是虚引用呢，使用它又不能获取到对应的对象实例，也没提供其他操作，不过通过源码我们可以发现，这个虚引用对象可以全程跟踪实例对象的状态，但对实例对象并没有什么影响，在该实例对象被回收，虚引用对象配合引用队列进行回收。实例对象在，虚引用对象在，实例对象gc，虚引用随后就会被回收，那么，这个虚引用对象就可以作为在实例对象被垃圾回收成功时的一个通知。 那这个通知有什么作用呢？接下来就来分析ref包中的Finalizer类。



### 三：Finalizer类分析：

```java
//FinalReference也继承自Reference类
final class Finalizer extends FinalReference<Object> { 

      private static ReferenceQueue<Object> queue = new ReferenceQueue<>();
    private static Finalizer unfinalized = null;
    private static final Object lock = new Object();
    
    //该类维护了一个双向的链表

    private Finalizer
        next = null,
        prev = null;
        
private Finalizer(Object finalizee) { //将该对象加入到引用队列中

        super(finalizee, queue);
        add();
    }
    
 /* Invoked by VM */ //将一个强引用对象注册进Finalizer类中

    static void register(Object finalizee) {
        new Finalizer(finalizee);
    }
}                                      
```

FinalReference 代表的正是 **Java 中的强引用**，如这样的代码 :

Bean bean = new Bean();

在虚拟机的实现过程中，实际采用了 **FinalReference 类对其进行引用**。而 Finalizer，除了作为一个实现类外，更是在虚拟机中实现一个 FinalizerThread，以使虚拟机能够在所有的引用被解除后实现内存清理。--------五个基本线程其中的一个。



继续看该类的源码：

```java
 private static class FinalizerThread extends Thread {
        private volatile boolean running;
        FinalizerThread(ThreadGroup g) {
            super(g, "Finalizer");
        }
        public void run() {
            if (running)
                return;

            // Finalizer thread starts before System.initializeSystemClass
            // is called.  Wait until JavaLangAccess is available
            while (!VM.isBooted()) {
                // delay until VM completes initialization
                try {
                    VM.awaitBooted();
                } catch (InterruptedException x) {
                    // ignore and continue
                }
            }
            final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
            running = true;
            for (;;) {
                try {
                    Finalizer f = (Finalizer)queue.remove();
                    f.runFinalizer(jla); //执行对象的finalize方法

                } catch (InterruptedException x) {
                    // ignore and continue
                }
            }
        }
    }
    
 private void runFinalizer(JavaLangAccess jla) {
        synchronized (this) {
            if (hasBeenFinalized()) return;
            remove();
        }
        try {
            Object finalizee = this.get();
            if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
                jla.invokeFinalize(finalizee); //执行对象的finalize方法
                /* 注意，这里需要清空栈中包含该变量的的 slot, 
                   从而来减少因为一个保守的 GC 实现所造成的变量未被回收的假象 */                finalizee = null;
            }
        } catch (Throwable x) { }
        super.clear();
    }

    static {  //同样，设置了一个守护线程（优先级低）----用于回收强引用对象

        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread finalizer = new FinalizerThread(tg);
        finalizer.setPriority(Thread.MAX_PRIORITY - 2);
        finalizer.setDaemon(true);
        finalizer.start();
    }
```

注意，标记处所调用的 invokeFinalizeMethod 为 native 方法，由于 finalize 方法在 Object 类中被声明为 protected，这里必须采用 native 方法才能调用。随后通过将本地强引用设置为空，以便使垃圾回收器清理内存。



可以看到，通过这样的方法，Java 将四种引用对象类型：软引用 (SoftReference)，弱引用 (WeakReference)，强引用 (FinalReference)，虚引用 (PhantomReference) 平等地对待，并在垃圾回收器中进行统一调度和管理。

##### tips:

在 GC 的过程中，当一个引用被释放，并不会立即被回收，而是由**系统垃圾收集器标记后的对象**（对应的GC算法，如标记清理算法，这里是先标记过程，如果一时不明白，想想数组的删除操作，是不是可以先标记几个元素，然后再一起删除，这样效率肯定会提高），然后会被加入到 Finalizer 线程中的 ReferenceQueue 中去，并调用 Finalizer.runFinalizer() 来执行对象的 finalize 方法。  这个过程通过上述源码就可以了解。



> 顺便提一下，finalize方法是Object类中的方法，  对象可以利用重写这个方法可以在回收过程中自救（在一个引用链上添加该对象），因为从FinalizerThread中的run方法可以看出，在回收对象之前会执行对象的finalize方法。   但是，任何一个对象的finalize方法只会被系统自动调用一次。  我们可以利用finalize方法来进行一些资源的回收（关闭外部资源，比如释放sql连接），但是这个方法运行代价高，不确定性大，因为在finalizer线程中执行的，所以是并发的，不能保证执行顺序。  它能做的，try--catch块 都能做。



在finalizer线程gc执行的过程中，会进行第二次标记，如果在这里对该对象进行重新引用，那么第二次标记时会将该对象从引用队列中移除，证明了该对象自救成功。 所以在对象真正的死亡之前，至少需要经过两次标记。 第一次标记是将不可达的对象加入到回收线程中去,第二次是在对象执行完finalize方法后判断是不是能回收并标记。



> 不可达的对象， 我是这么理解的：对于一个对象，对象外部没有四种引用中的任意一类指向它，那么可以认为这个对象就是不可达的，就可以回收。当然虚引用比较特殊，它对一个对象的回收并不造成任何影响，如果一个对象外部仅仅只有虚引用指向它，那么这个对象就可以被判定为可gc的。  



到这里，虚引用的作用还是没说明，其实通过上述分析，虚引用的作用也可以猜到了，比如通常用来做所谓的 Post-Mortem 清理机制，比如Java 平台自身 Cleaner 机制等，也有人利用幻象引用监控对象创建和销毁。



## 四：总结：

对于java中的引用，ref包中的四类引用，到这就学习完了，个人觉得还是从线程的切入点来看， ref包中有两个线程，首先从ReferenceHandler引用处理线程分析，这个守护线程的作用是回收引用类的对象（优先级最高），比如WeakReference bean = new WeakReference(new Bean("name", 10));  对于这个软引用bean实例的回收，就是引用处理线程的工作。我们需要记住的是， 软引用对象在jvm内存不足时回收，弱引用在下一次gc时被回收。虚引用对对象实例状态不影响，仅仅是作为对象的状态跟踪，当对象被回收后（即调用完finalize方法后），虚引用会配合引用队列完成回收，当然这个对象的回收需要显示调用gc。 还有就是软或者虚引用对象的创建可以不用传入引用队列 ，jvm会自动将其回收。  虚引用必须配合引用队列完成回收。



第二个线程就是finalizer线程，即管理对象垃圾回收的线程（优先级最底），这个线程是基于强引用（FinalReference）设计的，在虚拟机的实现过程中，实际采用了 **FinalReference 类对其进行引用**。 该线程的作用就是在对象不可达时回收该对象，原理和ReferenceHandler类似，都是配合使用引用队列完成对象回收，对象中的finalize方法就是在该线程中被调用的，所以对象的自我拯救就是在该线程中完成的。

那两个线程的联系点在哪呢， 我觉得ReferenceHandler做的就是按不同类型策略断开引用对象的引用链，让引用对象变为不可达状态。  而finalizer线程就是具体的回收不可达对象的线程，虽然它是基于强引用设计的，但是它对所有引用类型的对象一视同仁，因为都是不可达的嘛，所以才会被回收，并且可以调用对象相应的finalize方法执行。 （当然虚引用对象无法进行自我拯救，因为get方法为null）。



> 还有一点就是main启动时创建多少个线程，5个，要记住，面试题常考内容，并且熟悉这五个线程的作用。


