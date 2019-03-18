### 一. 何谓线程安全

#### 1. 并发编程的好处

- 对于异步性的操作，可以在阻塞时优先执行其他任务，提高了程序的整体执行效率
- 在多核时代，可以充分利用 CPU，如果是单线程程序，计算机有 100 核的话会损失 99% 的性能。

#### 2. 线程安全定义

并发编程在带来优势的同时也引入了更为复杂的问题，那就是如何保证多线程并发访问共享数据时的线程安全：

> 线程安全是计算机多线程编程中的概念。线程安全的代码确保所有线程在访问共享数据时能够正常运行，并且完全按照其设计的意图执行，不会出现任何意外的情况。当多个线程访问某个类时，这个类始终能够表现正确的行为，那么就称这个类是线程安全的。

***编写线程安全的代码核心在于要对状态访问操作进行管理，特别是对共享的和可变的状态的访问。***

### 二. 如何保证线程安全

线程安全问题的来源在于对共享、可变访问可能存在数据失效等问题。引发问题的核心在于：

- 数据的共享
- 数据的变化

针对数据的变化和共享而来的并发编程，其核心问题为：

- 互斥：同一个时刻只有一个线程访问共享变量
- 同步：线程之间的同步、协作

因此只要确保上面提到的问题被解决，就可以保证并发编程时线程的安全性了。


既然线程安全问题来源于数据的共享和变化，那么避免这两种情况就可以保证线程安全了。

#### 1. 线程封闭 - 关闭共享特点

##### 【1】重入锁

##### 【2】Thread Local 变量

- 局部变量
- Thread Local
-
#### 2. 不可变对象 - 关闭可变特性


### 三. 锁

上面提到的两种方式是通过改变数据特性的方式来保证线程安全，但是总有一个数据的可以被多线程访问、修改的，针对这些数据就得采用其他方式来保证线程安全性了，最常用的方式就是：***锁***。

通过锁将某段代码加锁，表示该段代码在同一时间只能由一个线程访问，其他线程必须等待当前线程释放锁后才可以再次获取锁并执行代码。

### 四. 多线程与锁对性能的影响

***并发编程的目的是为了提高资源使用率，进而提升性能。但是并发编程中最重要的始终是安全性，其次才是性能。***

#### 1. 多线程中的性能损耗

***线程的创建销毁与资源占用***

锁是需要占用机器资源的，当锁过多时，创建锁和释放锁占用的资源过多的话留给应用的资源就响应的减少，从而降低应用的性能。

***线程上下文切换***

对于同一个进程内的不同线程，当可运行线程数量大于 CPU 数量时，会导致线程互相竞争 CPU 的执行权，导致 CPU 进行线程调度，从而引发上下文切换。和进程上下文切换相比，虽然线程的上下文切换只需要切换线程的寄存器、私有数据等私有信息，但是当次数过多时也会导致性能问题。

***锁竞争***

当某些热点数据被多个线程频繁访问时会因为锁的竞争而阻塞，降低程序的性能。


#### 2. 降低多线程性能损耗的方式

##### 【1】缩短锁的持有时间

该方式最重要的是识别出哪些才是真正的需要加锁的操作，从而将不需要加锁的操作移出同步代码块。

下面是《Java 并发编程实战》中的一个例子:

```Java

public class AttrStore{
    
    private Map<String, String> attrs = new HashMap();
    
    public synchronized boolean userLocationMatches(String name, String regexp) {
        
        String key = "user:" + name;
        // 仅有这句需要同步
        String location = arrts.get(key);
        
        if (location == null) {
            return false;
        }else {
            return Pattern.matches(regexp, location);
        }
    }
}
```

在上面的代码中，在方法上加了锁，从而所有使用该方法的线程都需要同步等待前一个执行完才行，如果方法执行时间为 2ms，那么在 1 s 内最多只能执行 500 次。

其实只有从 Map 查询数据时才需要同步，因为只有这一步有访问 attrs 这个共享数据的情况。因此代码可以优化如下：

```Java

public class AttrStore{
    
    private Map<String, String> attrs = new HashMap();
    
    public boolean userLocationMatches(String name, String regexp) {
        
        String key = "user:" + name;
        String location；
        // 仅有这句需要同步
        synchronized {
             location = arrts.get(key);   
        }
        
        if (location == null) {
            return false;
        }else {
            return Pattern.matches(regexp, location);
        }
    }
}
```

只在访问 map 时加锁，这样省去了正则表达式匹配和字符串拼接占用锁的时间。

##### 【2】降低锁的请求频率

如果程序中所有的共享变量都由一个锁控制，那么所有的线程都要请求同一个锁，这样会导致大量的竞争，程序可以认为变为了串行化执行，从而降低性能。此时可以通过锁分段、锁分解等技术来进行优化。

***锁分解***


> 如果一个锁需要保护多个相互独立的状态变量，那么可以将这个锁分解，每个锁只保护一个变量，从而提高可伸缩性，降低每个锁被请求的频率。

***锁分段***

锁分解可以降低锁的竞争，但在某些不能再继续进行分解的情况下，可能依然存在锁竞争激烈的情况。

> 在某些情况系，可以将锁分解技术进一步拓展为对一组独立对象上的锁进行分解，这种情况成为锁分段。

锁分段最典型的应用就是 ConcurrentHashMap，其使用一个包含 16 个锁的数组，每个锁保护容器所占空间(即所有的散列桶)的 1/16，第 N 个桶由第 N%16 个锁来保护。

如果 Map 中的数据分布是合理的，没有过于明显的热点数据域，那么 ConcurrentHashMap 可以将对于锁的请求减少到原来的 1/16。也就是说其可以支持多达 16 个的并发写入器。当然还可以自行调高，但一般来说已经足够满足需求了。


当然，锁分段技术因为要增加锁的数量，也会带来一定的锁的创建、上下文开销，同时也增加了代码复杂度，不过和带来的优化相比这些都是可以承受的。

##### 【3】避免热点数据

锁分段和锁分解通过减少锁的请求频率，使得不同的变量、数据访问互不干扰。

但是如果某个数据的访问次数明显高于其他变量，就会导致程序频繁就请求同一个锁，这样也会导致性能问题。


以 HashMap 为例，其通过一个全局的共享计数器来计算当前容器的 size。这样可以将 size() 方法的时间复杂度降低为 O(1)。但是
对 Map 的任何增删操作都要请求锁访问该共享变量，此时该共享计数器就是热点数据，对该数据的访问会导致锁的激烈竞争。

如果 ConcurrentHashMap 也采用该方式，那么锁分段就变的没有意义了，因为对 Map 的增删最终都要竞争同一个计数器锁，对此 ConcurrentHashMap 的解决方式是每个分段都维护一个计数器，ConcurrentHashMap有 16 个分段锁也就是有 16 个各自的计数器，计算 size 时就是将这 16 个计数器求和，从而保证了并发性。


##### 【4】使用更友好的方式管理共享数据

上面提到的方式都可以视为独占锁，即使是锁分段，也只是将锁控制的数据范围变小，其依然是独占锁。一种更加友好的、降低锁竞争的方式是放弃使用独占锁，转而使用其他的友好并发方式来管理，此时有如下几种选择:

- 读写锁：如果多个读取操作都不会修改共享数据，那么这么读取操作可以同时访问共享资源。但是在写入时必须以独占的方式来获取锁。将原来一个一个执行的读取操作变为批量操作从而提高了并发性能。非常适合读多写少的场景。
- 并发容器：像 ConcurrentHashMap、CopyOnWriteArrayList 等并发容器，采用锁分段、读写锁等方式优化了对容器的并发读写性能，在使用需要并发访问的容器时应该优先考虑使用这些，而不是基于基础容器的封装。
- 原子变量：原子变量将获取锁的粒度缩小到单个变量，这是我们可以获得的最小粒度的锁。

### 四. Java 中的线程安全类

下面看 Java 中一些线程安全类的实现，其思路都是基于前面提到的知识点的。

#### 1. Thread Local  - 实现线程封闭

通过 ThreadLocal 可以将数据封装在线程内部，仅对所属线程可见。这样就避免了数据被多线程访问带来的问题。

ThreadLocal 的部分代码如下:

```Java

public class ThreadLocal<T> {
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    
     public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    // 获取线程的 threadLocals 对象
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
}


# Thread 中提前声明了成员变量 threadLocals
# 即每个线程都有一个 ThreadLocalMap 实例。
public class Thread implements Runnable {

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

通过看源码我们知道，每个 Thread 中都封装了一个 ThreadLocal.ThreadLocalMap 对象。

在 ThreadLocal 的 set 和 get 方法中，每次都通过 ```Thread.currentThread();``` 方法获取当前线程，然后获取其 ThreadLocalMap 对象，也就是说所有对 ThreadLocal 的读写操作都是操作的当前线程的 ThreadLocalMap ，不会对其他线程有影响，从而保证了线程安全。


#### 2. Collections.synchronized - 封装基础容器类

ThreadLocal 的方式可以称之为线程封闭，即将数据封装在线程内部，从而解决线程安全问题。还有另外一种方式可以称之为实例封闭。

> 将数据封装在对象内部，将对数据的访问转为对对象的访问。这样我们只需要关系对象访问时的线程安全即可，降低了工作难度。

一个典型的例子是 ``` Collections.synchronized*``` 系列方法。我们知道 Java 中的基础容器类 List、Map 等是非线程安全的。通过 ``` Collections.synchronized``` 可将其转为线程安全类。

下面是  Collections.synchronizedList 的部分源码：

```Java

# SynchronizedList 父类
static class SynchronizedCollection<E> implements Collection<E>, Serializable {
        private static final long serialVersionUID = 3053995032091335093L;
        
        # 封装基础容器类
        final Collection<E> c;  // Backing Collection
        # 互斥锁
        final Object mutex;     // Object on which to synchronize

        SynchronizedCollection(Collection<E> c) {
            this.c = Objects.requireNonNull(c);
            mutex = this;
        }

        SynchronizedCollection(Collection<E> c, Object mutex) {
            this.c = Objects.requireNonNull(c);
            this.mutex = Objects.requireNonNull(mutex);
        }
}

 static class SynchronizedList<E>
        extends SynchronizedCollection<E>
        implements List<E> {
        private static final long serialVersionUID = -7754090372962971524L;

        final List<E> list;

        SynchronizedList(List<E> list) {
            super(list);
            this.list = list;
        }
        SynchronizedList(List<E> list, Object mutex) {
            super(list, mutex);
            this.list = list;
        }

        public boolean equals(Object o) {
            if (this == o)
                return true;
            synchronized (mutex) {return list.equals(o);}
        }
        public int hashCode() {
            synchronized (mutex) {return list.hashCode();}
        }
        
        # 通过内置的锁实现同步
        public E get(int index) {
            synchronized (mutex) {return list.get(index);}
        }
        public E set(int index, E element) {
            synchronized (mutex) {return list.set(index, element);}
        }
        public void add(int index, E element) {
            synchronized (mutex) {list.add(index, element);}
        }
        public E remove(int index) {
            synchronized (mutex) {return list.remove(index);}
        }
}
```

可以看到，SynchronizedList 内部有两个关键实例:

- list: 封装线程不安全的容器类
- mutex: 互斥锁，默认用自身对象作为锁。用来控制并发访问。

其通过互斥锁来控制并发访问，并将相应的操作转发到基本容器类上，是一种装饰器模式。只要程序保证装饰器对象拥有对原始容器的唯一引用，程序中所有对基础容器类的访问都是通过装饰器对象进行的，那么就可以保证容器的线程安全。

#### 3. ConCurrentHashMap - 替代 Map

上面已经提到了 ConCurrentHashMap 是使用了锁分段技术，这是 Java 在 1.7 版本的实现。其内部实现了一个 Segment 类，该类是 ReentrantLock 的子类，代码如下：

```Java
    static final class Segment<K,V> extends ReentrantLock implements Serializable {

        private static final long serialVersionUID = 2249069246763182397L;

        static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
            
        // 存储键值对的容器
        transient volatile HashEntry<K,V>[] table;

        
        transient int count;

        transient int modCount;

        transient int threshold;
        final float loadFactor;
        
}
```
Segment 中封装了一个 HashEntry<K,V>[] table 对象，HashEntry 就是 Map 中存储键值对的对象，Java1.7 通过在 Segment 中封装 table 数组，从而将对整个 Map 的读写转为对 Segment 内部 table 数组的读写，因为 Segment 本身就是自带锁的，因此是线程安全的。那么由此可以想到，ConcurrentHashMap 的读写就是：

- 1. 初始化 ConcurrentHashMap 的 Segment 分段
- 2. 找到对应的 Segment 段
- 3. 对段中的 table 进行读写操作
-
我们通过源码来验证一下，下面是 ConcurrentHashMap 中的初始化和 put 操作：

```Java
// ConcurrentHashMap 初始化分段

static final int DEFAULT_CONCURRENCY_LEVEL = 16;

public ConcurrentHashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}

 public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        
        // 计算分段数
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        // 初始化分段数组
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }



// ConcurrentHashMap 中的 put 方法
public V put(K key, V value) {
    Segment<K,V> s;
        
    // 求 key 的 hash 值
    int hash = hash(key);
    //对 hash 值再次 hash，主要是为了避免 hash 中途
    int j = (hash >>> segmentShift) & segmentMask;

    // 找到 对应的 Segment
    if ((s = (Segment<K,V>)UNSAFE.getObject          
            (segments, (j << SSHIFT) + SBASE)) == null) 
            s = ensureSegment(j);
    // 调用 Segment 中的 put 进行数据
    return s.put(key, hash, value, false);
}
    
// Segment 中的 put 方法

final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 1. 加锁
    HasEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
    V oldValue;
    // 2. 元素的写入
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);

        for (HashEntry<K,V> e = first;;) {
            // 代码省略 ...
        }
    } finally {
        // 释放锁
        unlock();
    }
    return oldValue;
}
```

这里忽略具体细节，可以看到:

- 1. ConcurrentHashMap 定义了 DEFAULT_CONCURRENCY_LEVEL = 16 来确定分段数，默认为 16，即 ConcurrentHashMap 默认支持 16 个线程的并发访问
- 2. 对于 ConcurrentHashMap 的读写本质上是先找到 Segment，然后在对 Segment 中的 table 进行读写
- 3. Segment 中的 put 也是通过 try-finaly 进行加锁、执行操作、解锁的过程。

上面就是 ConcurrentHashMap 通过锁分段技术实现并发访问的基本流程。不过在 Java1.8 版本中 ConcurrentHashMap 的实现有了较大的变化:

> 从JDK1.7版本的 ReentrantLock+Segment+HashEntry 实现变为了 synchronized+CAS+HashEntry+红黑树  的实现。

具体细节这里先不做深究了，等后续有时间专门拿出时间研究 Java1.8 的 ConcurrentHashMap 细节。

#### 4. CopyOnWriteArrayList - 替代 List

上面提到了读写锁：

> 读写锁可以允许一个资源被多个读操作访问，或者被一个写操作访问，但是两者不能同时进行。

也就是说读是无需等待的，但是当有写入操作时，将会阻塞其他的读、写操作。读写锁是对标准互斥锁的的一种优化，在 Java 中 CopyOnWriteArrayList 就是采用的读写锁，提高了并发读取 List 的性能。部分源码如下：

```Java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    // 内置的的显式锁
    final transient ReentrantLock lock = new ReentrantLock();

    private transient volatile Object[] array;

    final Object[] getArray() {
        return array;
    }

    final void setArray(Object[] a) {
        array = a;
    }
    
    @SuppressWarnings("unchecked")
    private E get(Object[] a, int index) {
        return (E) a[index];
    }

    // 读不加锁
    public E get(int index) {
        return get(getArray(), index);
    }

    public E set(int index, E element) {
        final ReentrantLock lock = this.lock;
        // 显式加锁
        lock.lock();
        try {
            // 省略细节代码
            return oldValue;
        } finally {
            // 释放锁
            lock.unlock();
        }
    }

    
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        // 显式加锁
        lock.lock();
        try {
            // 省略细节代码
            return true;
        } finally {
            // 释放锁
            lock.unlock();
        }
    }
    
    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 省略细节代码
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
    
    public void clear() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            setArray(new Object[0]);
        } finally {
            lock.unlock();
        }
    }
    
   public void sort(Comparator<? super E> c) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            Object[] newElements = Arrays.copyOf(elements, elements.length);
            @SuppressWarnings("unchecked") E[] es = (E[])newElements;
            Arrays.sort(es, c);
            setArray(newElements);
        } finally {
            lock.unlock();
        }
    }
}
```

可以看到 CopyOnWriteArrayList 内置了一个 ReentrantLock，其读取操作是不加锁的，而对于 add、set、remove、clear、sort 等写操作，均通过try-finally 块的形式进行了加锁与锁的释放。另外内置数组是 volatile 类型，保证了写入操作后对于其他读操作的可见性。这样通过只对写加锁，避免了标准互斥锁会引发的 "读-读" 冲突，当 List 读多写少时可以极大提高其性能，

### 五. 无锁机制

#### 1. Java 中的 volatile 变量与原子变量

##### 【1】volatile 变量

***优点***

- volatile 变量用来确保将变量的更新操作通知到其他线程。
- 可以简单的理解为 volatile 的读写操作都是原子的，但是不会执行加锁操作。

***应用场景***

- 变量的写入不依赖当前值
- 在访问变量时不需要加锁
- 该变量不会与其他变量一起纳入不变形条件中

***局限性***

- 其语义不足以确保递增操作的原子性，除非保证只有一个线程操作

##### 【2】 原子变量


- https://coolshell.cn/articles/8239.html
- http://www.drdobbs.com/lock-free-data-structures/184401865



### 五. Go 语言的并发编程


***参考资料***

- [《Java 并发编程实战》](https://book.douban.com/subject/10484692/)
- 极客时间专栏 [Java 并发编程实战](https://time.geekbang.org/column/intro/159)
- [维基百科: Thread safety ](https://en.wikipedia.org/wiki/Thread_safety)
- [CoolShell: 无锁队列的实现](https://coolshell.cn/articles/8239.html)
- [设计不使用互斥锁的并发数据结构](https://www.ibm.com/developerworks/cn/aix/library/au-multithreaded_structures2/index.html)
- [ConcurrentHashMap原理分析（1.7与1.8）](https://www.cnblogs.com/study-everyday/p/6430462.html)
- JDK 1.7、1.8 源码
- [《Go 语言学习笔记》](https://book.douban.com/subject/26832468/)
- [Effective Go](https://golang.org/doc/effective_go.html#concurrency)