---
layout: post
title:  "Java并发编程ConcurrentHashMap源码阅读笔记"
date:   2013-07-21 22:43:50
---
相对于线程不安全的HashMap来说,HashTable在存储table[]数组操作方法上的粗粒度synchronized则对性能损耗太多,先看看下面性能对比情况下(数值表示运行花费的毫秒数):
{% highlight ruby %}
     线程数           HashTable          ConcurrentHashMap
      
      1                  19                     20
      2                  32                     27
      10                 131                    110
      40                 68                     264
      100                124                    646
      200                1318                   247      
      500                3244                   673

{% endhighlight %}
测试代码的逻辑是一个map实例,每个并发的线程进行10,000次的随机put或者get的时间消耗

`HashTable`通过synchronized实现table[]的线程安全,方法上粗力度的加锁实现方式,限制了table[]数组的操作上同个时间段都将被一个线程独占,其他并发的线程只能等待或轮循,所以HashTable的伸缩性较差,即使在系统的资源充足的情况下也无法通过更多的cpu线程占用率来提高Hashtable的性能,在读多写少的特定场景下性能更是较HashTable有更大的差距.HashTable在Iterator并不保证table中数据的一致性,迭代的过程中并未锁住table[],采用了fast fail的方式验证mod的值是否和expectedModCount值相等,若不等抛出ConcurrentModificationException,在并发的环境下同时进行迭代和put或delete很容易抛出该异常

`ConcourrentHashMap`在锁的设计上进行了优化,设计了多组的分段锁,不同于HashTable的独占锁,ConcurrentHashMap采用了Segment数组的数据结构,每个Segment对应一个分段锁,其实在设计上每个Segment都相当于一个HashTable,简单的说就是一个ConcurrentHashMap的存储对应多个Segment数组(类似HashTable),每次必要的锁操作只对应到单个Segment,并不会锁住这个ConcurrentHashMap实例

{% highlight ruby %}
  
 ConcurrentHashMap() {        //默认的构造方法
        this(16, 0.75f, 16);
    }

 public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();

        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;   
            //MAX_SEGMENTS = 1 << 16  所以concurrencyLevel的值最大为65535

        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        while (ssize < concurrencyLevel) {
            ++sshift;       //偏移量
            ssize <<= 1;   
            //ssize恰好大于concurrencyLevel,且为2的整数倍,即Segment的数组大小
        }
        segmentShift = 32 - sshift;//32与sshift的差值,用于segment的定位,下文会提到
        segmentMask = ssize - 1;//比Segment数组长度小1的掩码,应该用于求余
        this.segments = Segment.newArray(ssize);//创建ssize大小的数组

        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
       //ConcurrentHashMap的每个Segment能够存储的最大值  MAXIMUM_CAPACITY = 1 << 30; 
        int c = initialCapacity / ssize;           
        if (c * ssize < initialCapacity)
            ++c;//调整c的值恰好为ssize*C >initialCapacity的条件下的最小值
        int cap = 1;
        while (cap < c)
            cap <<= 1; //cap为略大于c的2的倍数

        for (int i = 0; i < this.segments.length; ++i)
            this.segments[i] = new Segment<K,V>(cap, loadFactor);
            //初始化Segment数组,初始大小为cap
    }


    static final class Segment<K,V> extends ReentrantLock implements Serializable {


        transient volatile int count;

        transient int modCount;

        transient int threshold;

        transient volatile HashEntry<K,V>[] table;
        //和hashtable结构类似,存放Entry的table

        final float loadFactor;

        .....
  }
{% endhighlight %}
ConcurrentHashMap初始化即通过一系列参数调整设置Segment的大小,ConcurrentHashMap维护了power of 2的Segment的数组.

{% highlight ruby %}
   
   public V put(K key, V value) {
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key.hashCode());   //第一次hash
        return segmentFor(hash).put(key, hash, value, false);
    }
   
   final Segment<K,V> segmentFor(int hash) {
        return segments[(hash >>> segmentShift) & segmentMask];
    }
   	
   private static int hash(int h) { 
        h += (h <<  15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h <<   3);
        h ^= (h >>>  6);
        h += (h <<   2) + (h << 14);
        return h ^ (h >>> 16);
    }

{% endhighlight %}
ConcurrentHashMap的put操作,其中hash()函数的作用是根据不同的key计算散列码(数学问题)目的是使数据通过hash值&mask数组大小达到数据均匀分配到Entry数组上的目的,segments()通过hash的值定位数据到一个确定的Segment上,通过Segments可以确定ConcurrentHashMap的Segments数组大小在实例初始化后就已经确定的不允许扩充的,所以segment数组,segmentMask和segmentShift属性都是final的.

{% highlight ruby  %}

 V put(K key, int hash, V value, boolean onlyIfAbsent) {
            lock();  //获得当前Segment的锁对象
            try {
                int c = count;
                if (c++ > threshold) // ensure capacity
                    rehash();  //大小大于当前阀值,扩充Entry[]数组的大小
                HashEntry<K,V>[] tab = table;
                int index = hash & (tab.length - 1);  //hash值定位到一个slot
                HashEntry<K,V> first = tab[index];
                HashEntry<K,V> e = first;
                while (e != null && (e.hash != hash || !key.equals(e.key)))
                    e = e.next; //查找当前slot的链表中是否存在当前插入key

                V oldValue;
                if (e != null) {    //存在的条件下
                    oldValue = e.value; 
                    if (!onlyIfAbsent)    
                        e.value = value; //在onlyIfAbsent为true,即不存在key的情况下才插入
                }
                else {
                    oldValue = null;   //不存在的条件下
                    ++modCount;
                    tab[index] = new HashEntry<K,V>(key, hash, first, value);  
                    //采用头插法插入新key
                    count = c; // write-volatile
                }
                return oldValue;
            } finally {
                unlock(); //释放当前锁对象
            }
        }

{% endhighlight %}
Segment插入类似于HashTable的put获得锁的情况下进行操作,最大的不同是HashTable要获得当前实例的全局锁,阻塞其他线程对实例的syn操作,而ConcuurentHashMap采用了分段的锁机制,当前获得的锁只是阻塞了其对应的Segment,而相对其他的ssize-1的Segments并无影响,分段锁达到了细粒化锁的目的.多消耗的仅是segment()定位segment的操作

{% highlight ruby %}

     static final class HashEntry<K,V> { //Entry的实现类
        //一个Entry的实例变量的key是永久不变的,推之hash也是final
        final K key;
        final int hash;
        //对于多线程的更改操作即保证值更改的立即可见性
        volatile V value;
        //修饰符为final决定了next的不变性,所以链表中的删除和rehash的过程都要重新build之前的所有e节点,
        final HashEntry<K,V> next;
                                          
        HashEntry(K key, int hash, HashEntry<K,V> next, V value) {
            this.key = key;
            this.hash = hash;
            this.next = next;
            this.value = value;
        }

	@SuppressWarnings("unchecked")
	static final <K,V> HashEntry<K,V>[] newArray(int i) {
	    return new HashEntry[i];
	}
    }

     void rehash() {   
            HashEntry<K,V>[] oldTable = table;
            int oldCapacity = oldTable.length;
            if (oldCapacity >= MAXIMUM_CAPACITY)   //每个Segment的最大值为 1 >> 32
                return;

           //新开辟2倍的oldCapacity
            HashEntry<K,V>[] newTable = HashEntry.newArray(oldCapacity<<1);
            threshold = (int)(newTable.length * loadFactor);
            int sizeMask = newTable.length - 1;    //新数组长度的掩码
            for (int i = 0; i < oldCapacity ; i++) {
                // We need to guarantee that any existing reads of old Map can
                //  proceed. So we cannot yet null out each bin.
                HashEntry<K,V> e = oldTable[i];

                if (e != null) {
                    HashEntry<K,V> next = e.next;
                    int idx = e.hash & sizeMask;

                    //  Single node on list
                    if (next == null)
                        newTable[idx] = e;  
       //next节点为null保证了hash值到这个槽位的值仅此一家,别无分店,所有直接对应到newTable数组的idx的
      //槽位上,细想想,每个key的hash值是和掩码即数组长度在只有一家的情况下在扩容后同样也不会出现第二个                                           
                        //和它冲突的key
                    else {
                        // Reuse trailing consecutive sequence at same slot
                        HashEntry<K,V> lastRun = e;      
                        int lastIdx = idx;
                         //rehash的算法有点小技巧
                        for (HashEntry<K,V> last = next;        
                             last != null;
                             last = last.next) {
                            int k = last.hash & sizeMask;
                            if (k != lastIdx) {
                                lastIdx = k;//算出old链表下最后一个key在新sizemask下不同的Idx值
                                lastRun = last;
                            }
                        }
                        newTable[lastIdx] = lastRun;

                        // Clone all remaining nodes
                        for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {   
                           //只用重新new lastRun之前的节点,而在lastrun之后的节点因为会                                                                        
                           //定位到同一个Idx上所以直接由lastRun带过去就可以了
                            int k = p.hash & sizeMask;
                            HashEntry<K,V> n = newTable[k];   //采用头插入法
                            newTable[k] = new HashEntry<K,V>(p.key, p.hash,
                                                             n, p.value);
                        }
                    }
                }
            }
            table = newTable;
        }    

{% endhighlight %}
rehash是在拿到lock的情况下进行的,对于旧的oldTable不存在更改的情况,对于无锁的get的操作来说,只是happen before的关系,不影响读取,注意Concurrent不存在hashmap中并发rehash导致的死锁问题

{% highlight ruby %}
 	
    V get(Oemove; match on key only if value null, else match both.
         */
        V remove(Object key, int hash, Object value) {
            lock();
            try {
                int c = count - 1;
                HashEntry<K,V>[] tab = table;
                int index = hash & (tab.length - 1);
                HashEntry<K,V> first = tab[index];
                HashEntry<K,V> e = first;
                while (e != null && (e.hash != hash || !key.equals(e.key)))
                    e = e.next;

                V oldValue = null;
                if (e != null) {
                    V v = e.value;
                    if (value == null || value.equals(v)) {
                        oldValue = v;
                        // All entries following removed node can stay
                        // in list, but all preceding ones need to be
                        // cloned.
                        ++modCount;
                        HashEntry<K,V> newFirst = e.next;
                        for (HashEntry<K,V> p = first; p != e; p = p.next)
                            newFirst = new HashEntry<K,V>(p.key, p.hash,
                                                          newFirst, p.value);
                        tab[index] = newFirst;
                        count = c; // write-volatile
                    }
                }
                return oldValue;
            } finally {
                unlock();
            }
        }

{% endhighlight %}
remove的操作其实和rehash有些相似的地方,对于next的修饰符是final,所以对于remove之前的所有节点都需要重新build.

{% highlight ruby %}

         V get(Object key, int hash) {
            if (count != 0) { // read-volatile
                HashEntry<K,V> e = getFirst(hash);  //getFirst利用hash值返回key所落在的槽位
                while (e != null) {
                    if (e.hash == hash && key.equals(e.key)) {
                        V v = e.value;
                        if (v != null)
                            return v;
                        return readValueUnderLock(e); // recheck
                    }
                    e = e.next;
                }
            }
            return null;
      }

    HashEntry<K,V> getFirst(int hash) {
            HashEntry<K,V>[] tab = table;
            return tab[hash & (tab.length - 1)];
        }

{% endhighlight %}
get并不需要加锁,why?首先value的属性通过voliate修饰,轻量级的锁,若任何线程对v的修改,则get的操作会立即从main memory看到此改变.其次,`v != null`的判断,ConcurrentHashMap的put入口是不允许有null的value的值,那在segment的get内部实现中,为什么还要多此一举的判断?这是因为线程B进行put(key)的操作和线程A读取(key)的操作不存在happen before的关系,所以读到的可能为工作线程memory里的null值,当然这重情况出现的概率很小很小很小,所以防止出现这种情况,利用加锁强制刷新读取e的值.

ConcurrentHashMap不是绝对意义上的put和get等操作的线程安全,但它在并发的性能和数据缓存一致性方面是个很好的实现方式,数据绝对的一致性还是要用collections.synChronizedMap方法实现.它的实质是在hashMap上封装了一层synchronized的代理.效率和HashTable是差不多的.

相关文章
 
 *  [聊聊并发（四）——深入分析ConcurrentHashMap][1]
 *  [疫苗：Java HashMap的死循环][2]
 *  [用happen-before规则重新审视DCL][3]
 *  [The "Double-Checked Locking is Broken" Declaration][4]
[1]: http://www.infoq.com/cn/articles/ConcurrentHashMap
[2]: http://coolshell.cn/articles/9606.html
[3]: http://www.iteye.com/topic/260515/
[4]: http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html
