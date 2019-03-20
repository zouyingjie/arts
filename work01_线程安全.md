### 一. 何谓线程安全

随着硬件和互联网的发展，很多程序经常需要面对过百万级甚至更高的并发访问量，因此并发编程越来越成为一项必备的要求，并发编程可以带来一些好处：

- 对于异步性的操作，可以在阻塞时优先执行其他任务，提高了程序的整体执行效率
- 在多核时代，可以充分利用 CPU，如果是单线程程序，计算机有 100 核的话会损失 99% 的性能。

但是并发编程在带来收益的同时也引入了一些比较复杂的问题，那就是如何保证并发访问共享数据时的线程安全：

> 线程安全是计算机多线程编程中的概念。线程安全的代码确保所有线程在访问共享数据时能够正常运行，并且完全按照其设计的意图执行，不会出现任何意外的情况。当多个线程访问某个类时，这个类始终能够表现正确的行为，那么就称这个类是线程安全的。

***编写线程安全的代码核心在于要对状态访问操作进行管理，特别是对共享的和可变的状态的访问。***

### 二. 如何保证线程安全

线程安全问题的来源在于对共享、可变数据的访问可能存在数据失效等问题。引发问题的核心在于：

- 数据的共享
- 数据的变化

针对数据的变化和共享而来的并发编程，其核心问题为：

- 互斥：同一个时刻只有一个线程访问共享变量
- 同步：线程之间的同步、协作

因此只要确保上面提到的问题被解决，就可以保证并发编程时线程的安全性了。

既然线程安全问题来源于数据的共享和变化，那么避免这两种情况就可以保证线程安全了。

#### 1. 线程封闭 - 关闭共享访问

***局部变量***

局部变量位于执行线程的栈中，其他线程无法访问，因此保证了线程安全性。

对于基本类型，任何方法都无法获取基本类型的引用，因此 Java 中基本类型的局部变量始终是封闭在线程内的。

对于对象类型，需要保证引用唯一且封闭在线程中的。看下面代码：

```Java

public class Test{
    public List<Anmail> list;
    public void test(List<Animal> candidates) {
        int nums = 0;
        Animal animal = null;
        List<Animal> animals;

        animals = new ArrayList<>();
        animals.add(candidates)

        // animals 逸出，使得其他线程可以通过 this.list 访问到 animals
        // 从而破坏了封闭性
        this.list = animals;
    }
}

```

如果 animals 及其存储的元素没有被外部变量所引用，那么就不会破坏其线程封闭性，但如果对外发布了该对象，就会导致 animals 的逸出，从而破坏封闭性。

***Thread Local 数据存储***

在 Java 的线程 Thread 对象中，都封装了一个 ThreadLocalMap 对象，可以用来存放数据，存储在该对象中的数据仅能被其所在的线程访问，从而实现了线程封闭，保证数据的线程安全性。

#### 2. 不可变对象 - 关闭可变特性

> 不可变对象一定是安全的。

不可变对象没有复杂的状态变化，一旦创建赋值，就无法再次改变。因此无论有多少线程访问，其状态始终是唯一的，因此也就没有了线程安全问题。
Java 中通过 final 关键字定义不可变对象。

### 三. 互斥锁

上面提到的两种方式是通过改变数据特性的方式来保证线程安全，但是总有一些数据的可以被多线程访问、修改的，针对这些数据就得采用其他方式来保证线程安全性了，保证安全性的核心就在于保证：

> 同一时刻只有一个线程执行，线程之间对数据的修改是互斥的，从而保证数据修改的原子性，进而保证了线程安全。

实现互斥最常用的方案就是：锁。所有的数据都是存储的内存中的，我们可以把某个要访问的数据地址看做是一个房间，进入该房间访问数据的线程，为了保证在访问时不让其他线程进到房间，必须把房间门锁上，此时该线程就持有了房间的锁，其他线程没有锁也就进不了房间了。等线程执行完后，在把锁打开，告诉其他线程锁开了有想用的赶紧来吧，其他线程在竞争这把锁，竞争到的线程再次重复加锁-操作-解锁的步骤。

下面是极客时间专栏《Java 并发编程实战》中的一张示例图，很直观的展示了加锁-执行-释放锁的过程。

![](https://static001.geekbang.org/resource/image/3d/a2/3df991e7de14a788b220468836cd48a2.png)

看下面一段示例代码：

```Java
public class Test{
    private int value;

    public synchronized int get() {
        return value;
    }

    public synchronized void addOne() {
        value += 1;
    }
}
```

synchronized 是 Java 内置的加锁关键字，可以对方法、代码块执行加锁、释放锁的操作。通过加锁：

- value += 1 变为了原子操作，操作过程中不会被中断
- 基于 Happens-before 原则，前一个线程的解锁操作对后一个线程的加锁操作是可见的。也就是说当 addOne 执行完成后，另一个线程访问 get 方法的话获取的是 addOne 操作后的最新值

关于锁的使用注意几点：

- 尽可能使用细粒度的锁
- 加锁之后一定要释放锁
- 对于同一资源的保护，锁必须是唯一的，否则会无效

### 四. 多线程与锁对性能的影响

***并发编程的目的是为了提高资源使用率，进而提升性能。但是并发编程中最重要的始终是安全性，其次才是性能。***

#### 1. 多线程中的性能损耗

##### 【1】线程的创建与销毁

线程是需要占用机器资源的，当线程过多时，创建线程和销毁线程占用的资源过多的话留给应用的资源就会相应的减少，从而降低应用的性能。

##### 【2】线程的上下文切换

对于同一个进程内的不同线程，当可运行线程数量大于 CPU 数量时，会导致线程互相竞争 CPU 的执行权，导致 CPU 进行线程调度，从而引发线程上下文切换。和进程上下文切换相比，虽然线程的上下文切换只需要切换线程的寄存器、私有数据等私有信息，但是当次数过多时也会导致性能问题。

##### 【3】锁竞争导致的性能问题

当锁的粒度过大时，如果多个资源都被同一个锁锁住，那么所有对资源的访问只能竞争同一个锁，这会严重影响性能。比如对于一个 Person 对象:

```Java
class Persion {
    String name;
    int age;
    int gender;
}
```

其有 姓名、年龄、性别三个字段，如果使用一个锁的话，那么当有线程在修改姓名时，age 字段也是不能访问的。这是不合理的。因此在使用锁时要注意使用细粒度的锁，只在需要的地方加锁，并且尽可能使用不同的锁保护不同资源。

另外就是如果锁定的代码如果有耗时较长的代码，会导致锁长时间得不到释放，引发其他线程的阻塞，也会影响性能。

##### 【4】锁释放

synchronized 默认执行了加锁和释放锁操作，但是在对于某些使用显式锁的地方，我们一定要手动释放锁，不然会导致锁得不到释放，其他线程会一直阻塞，导致程序性能严重降低。

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

如果 Map 中的数据分布是合理的，没有过于明显的热点数据域，那么 ConcurrentHashMap 可以将对于锁的请求减少到原来的 1/16。也就是说其可以支持多达 16 个并发请求。当然还可以自行调高，但一般来说已经足够满足需求了。

当然，锁分段技术因为要增加锁的数量，也会带来一定的锁的创建、上下文切换开销，同时也增加了代码复杂度，不过和带来的优化相比这些都是可以承受的。

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

> 将数据封装在对象内部，将对数据的访问转为对对象的访问。这样我们只需要关心对象访问时的线程安全即可，降低了工作难度。

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
Segment 中封装了一个 HashEntry<K,V>[] table 对象，HashEntry 就是 Map 中存储键值对的对象，Java1.7 通过在 Segment 中封装 table 数组，从而将对整个 Map 的读写转为对 Segment 内部 table 数组的读写，因为 Segment 本身就是自带锁的，因此是线程安全的。那么由此可以想到，ConcurrentHashMap 的读写过程应该是：

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
    //对 hash 值再次 hash，主要是为了避免 hash 冲突
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

### 五. 无锁机制简记

无锁机制，主要是基于 CAS (Compare And Swap) 操作来实现的，其借用了 CPU 的原子指令实现数据修改。
CAS 直译过来就是比较与交换，在设置某个值之前，先判断当前值是否有效，有效的话则执行交换操作。

类似的操作还有：

```C
// 引用皓叔《无锁队列设计》中的代码：
// 返回旧的值
int compare_and_swap (int* reg, int oldval, int newval)
{
  int old_reg_val = *reg;
  if (old_reg_val == oldval)
     *reg = newval;
  return old_reg_val;
}

// 返回是否交换成功，可以用来判断是否设置成功
bool compare_and_swap (int *accum, int *dest, int newval)
{
  if ( *accum == *dest ) {
      *dest = newval;
      return true;
  }
  return false;
}
```

下面是 CAS 应用的一些简单的例子：

#### 1. 原子变量

Java 中的原子变量就是基于 CAS 实现安全更新操作的。
```Java
AtomicInteger num = new AtomicInteger();
// 如果 num 值为 10，则设置为 20。底层是基于 CAS 原子操作实现，因此不会出现并发问题。
num.compareAndSet(10, 20);

```

#### 2. 利用 CAS 实现计数器

这是[Go并发编程之美-CAS操作](http://ifeve.com/go%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E7%BE%8E-cas%E6%93%8D%E4%BD%9C/) 中的一个例子，通过 CAS 实现计数器，
不对计数器变量加锁，而是通过 CAS + 无限循环的方式实现原子交换和重试。

```
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

var (
	counter int32 //计数器
	wg sync.WaitGroup //信号量

)

func main() {

	threadNum := 10000

	wg.Add(threadNum)

    // 开启协程，
	for i := 0; i < threadNum; i++ {
		go incCounter(i)
	}

    // 等待结束
	wg.Wait()
	fmt.Println(counter)

}

func incCounter(index int) {

	defer wg.Done()
	spinNum := 0
    // 无限循环，保证重试
    // 当执行 CAS 前协程被调度了，会导致 CAS 失败，因此需要重试操作，保证最终执行一定成功
	for {
        // 原子操作
		old := counter
		ok := atomic.CompareAndSwapInt32(&counter, old, old + 1)

		if ok {
			break
		} else {
			spinNum++
		}
	}
	fmt.Printf("thread,%d,spinnum,%d\n", index, spinNum)

}
```

#### 3. 无锁容器

引用自 [设计不使用互斥锁的并发数据结构](https://www.ibm.com/developerworks/cn/aix/library/au-multithreaded_structures2/index.html) 的例子：

无锁堆栈的入栈操作：
```C++
void Stack<T>::push(const T& data) 
{ 
    Node *n = new Node(data); 
    while (1) { 
        n->next = top;
        // 通过 CAS 完成栈顶的交换操作
        if (__sync_bool_compare_and_swap(&top, n->next, n)) { // CAS
            break;
        }
    }
}
```

上面是无锁堆栈的入栈操作，这里没有使用所而是 CAS 操作，while 循环保证了 CAS 执行前线程被抢占时重试的操作，因此入栈是线程安全的。

无锁堆栈的弹栈：

```C++
T Stack<T>::pop( ) 
{ 
    while (1) { 
        Node* result = top;
        if (result == NULL) 
           throw std::string(“Cannot pop from empty stack”);      
        if (top && __sync_bool_compare_and_swap(&top, result, result->next)) { // CAS
            return top->data;
        }
    }
}
```
依旧用 while 循环保证 CAS 执行前线程被抢走执行权的情况，保证弹栈操作最终执行完成。

传统的加锁机制是一种悲观锁，一旦有资源需要修改，立即上锁，不允许别的线程再去占用该资源。而 CAS 是一种乐观锁机制，不给资源上锁，其他线程可以占用资源，但是自己在执行前每次都要判断一次，看是否有其他线程改动过资源。因此CAS 的使用一般都伴随着重试、循环，本质上是通过抢占 CPU 资源从而节省了锁带来的性能损耗，比如线程的上下文切换等。

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
- [并发编程网](http://ifeve.com/)