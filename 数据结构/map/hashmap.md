# 概念与使用
- 提供了key-value的形式存取数据，使用put方法存数据，get方法取数据。
<pre>
    Map<String, String> hashMap = new HashMap<String, String>();
    hashMap.put("name", "josan");
    String name = hashMap.get("name");
</pre>
# 源码实现
## 定义
- HashMap继承了Map接口，实现了Serializable等接口。
<pre>
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    transient Entry<K,V>[] table;
}
</pre>
- HashMap的数据是存在table数组中的，它是一个Entry数组，Entry是HashMap的一个静态内部类。
## Entry定义
<pre>
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
	}
</pre>
- Entry其实就是封装了key和value，也就是我们put方法参数的key和value会被封装成Entry，然后放到table这个Entry数组中。但值得注意的是，它有一个类型为Entry的next，它是用于指向下一个Entry的引用，所以table中存储的是Entry的单向链表。
## 构造方法
<pre>
 /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }


    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        // 找到第一个大于等于initialCapacity的2的平方的数
        int capacity = 1; 
        while (capacity < initialCapacity)
            capacity <<= 1;

        this.loadFactor = loadFactor;
        // HashMap扩容的阀值，值为HashMap的当前容量 * 负载因子，默认为12 = 16 * 0.75
        threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        // 初始化table数组，这是HashMap真实的存储容器
        table = new Entry[capacity];
        useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        // 该方法为空实现，主要是给子类去实现
        init();
    }
</pre>
- initialCapacity是HashMap的初始化容量(即初始化table时用到)，默认为16。
loadFactor为负载因子，默认为0.75。
- threshold是HashMap进行扩容的阀值，当HashMap的存放的元素个数超过该值时，会进行扩容，它的值为HashMap的容量乘以负载因子。比如，HashMap的默认阀值为16*0.75，即12。
- HashMap提供了指定HashMap初始容量和负载因子的构造函数，这时候会首先找到第一个大于等于initialCapacity的2的平方数，用于作为初始化table。
- init是个空方法，主要给子类实现，比如LinkedHashMap在init初始化头部节点。
## put方法
<pre>
 public V put(K key, V value) {
        // 对key为null的处理
        if (key == null)
            return putForNullKey(value);
        // 根据key算出hash值
        int hash = hash(key);
        // 根据hash值和HashMap容量算出在table中应该存储的下标i
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            // 先判断hash值是否一样，如果一样，再判断key是否一样
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }

</pre>
- 如果key为null调用putForNullKey来处理。然后调用hash方法，根据key来算得hash值，得到hash值以后，调用indexFor方法，去算出当前值在table数组的下标，我们可以来看看indexFor方法：
<pre>
static int indexFor(int h, int length) {
        return h & (length-1);
    }
</pre>
-  h&（length-1）保证获取的index一定在数组范围内，举个例子，默认容量16，length-1=15，h=18,转换成二进制计算为
<pre>
        1  0  0  1  0
    &   0  1  1  1  1
    __________________
        0  0  0  1  0    = 2
</pre>
- 据key得到hash值，然后根据hash值算出在table的存储下标了，接着就是这段for代码了：
<pre>
  for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            // 先判断hash值是否一样，如果一样，再判断key是否一样
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
</pre>
- 首先取出table中下标为i的Entry，然后判断该Entry的hash值和key是否和要存储的hash值和key相同，如果相同，则表示要存储的key已经存在于HashMap，这时候只需要替换已存的Entry的value值即可。如果不相同，则取e.next继续判断，其实就是遍历table中下标为i的Entry单向链表，找是否有相同的key已经在HashMap中，如果有，就替换value为最新的值，所以HashMap中只能存储唯一的key。
关于需要同时比较hash值和key有以下两点需要注意

 - 为什么比较了hash值还需要比较key：因为不同对象的hash值可能一样
 - 为什么不只比较equal：因为equal可能被重写了，重写后的equal的效率要低于hash的直接比较
- 假设我们是第一次put，则整个for循环体都不会执行，我们继续往下看put方法。
<pre>
  modCount++;
        addEntry(hash, key, value, i);
        return null;
</pre>
- 主要看addEntry方法，它应该就是把key和value封装成Entry，然后加入到table中的实现。来看看它的方法体：
<pre>
  void addEntry(int hash, K key, V value, int bucketIndex) {
        // 当前HashMap存储元素的个数大于HashMap扩容的阀值，则进行扩容
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        // 使用key、value创建Entry并加入到table中
        createEntry(hash, key, value, bucketIndex);
    }
</pre>
<pre>
void createEntry(int hash, K key, V value, int bucketIndex) {
        // 取出table中下标为bucketIndex的Entry
        Entry<K,V> e = table[bucketIndex];
        // 利用key、value来构建新的Entry
        // 并且之前存放在table[bucketIndex]处的Entry作为新Entry的next
        // 把新创建的Entry放到table[bucketIndex]位置
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        // HashMap当前存储的元素个数size自增
        size++;
    }
</pre>
- 根据hash、key、value以及table中下标为bucketIndex的Entry去构建一个新的Entry，其中table中下标为bucketIndex的Entry作为新Entry的next，这也说明了，当hash冲突时，采用的拉链法来解决hash冲突的，并且是把新元素是插入到单边表的表头。
## 扩容
- 当前HashMap中存储的元素个数达到扩容的阀值，会进行扩容处理
<pre>
 if ((size >= threshold) && (null != table[bucketIndex])) {
            // 将table表的长度增加到之前的两倍
            resize(2 * table.length);
            // 重新计算哈希值
            hash = (null != key) ? hash(key) : 0;
            // 从新计算新增元素在扩容后的table中应该存放的index
            bucketIndex = indexFor(hash, table.length);
        }
</pre>
<pre>
 void resize(int newCapacity) {
        // 保存老的table和老table的长度
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        // 创建一个新的table，长度为之前的两倍
        Entry[] newTable = new Entry[newCapacity];
        // hash有关
        boolean oldAltHashing = useAltHashing;
        useAltHashing |= sun.misc.VM.isBooted() &&
                (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        // 这里进行异或运算，一般为true
        boolean rehash = oldAltHashing ^ useAltHashing;
        // 将老table的原有数据，从新存储到新table中
        transfer(newTable, rehash);
        // 使用新table
        table = newTable;
        // 扩容后的HashMap的扩容阀门值
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
</pre>
- transfer方法用来转移数据
<pre>
void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        // 遍历老的table数组
        for (Entry<K,V> e : table) {
            // 遍历老table数组中存储每条单项链表
            while(null != e) {
                // 取出老table中每个Entry
                Entry<K,V> next = e.next;
                if (rehash) {
                    //重新计算hash
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                // 根据hash值，算出老table中的Entry应该在新table中存储的index
                int i = indexFor(e.hash, newCapacity);
                // 让老table转移的Entry的next指向新table中它应该存储的位置
                // 即插入到了新table中index处单链表的表头
                e.next = newTable[i];
                // 将老table取出的entry，放入到新table中
                newTable[i] = e;
                // 继续取老talbe的下一个Entry
                e = next;
            }
        }
    }
</pre>
- 扩容就是先创建一个长度为原来2倍的新table，然后通过遍历的方式，将老table的数据，重新计算hash并存储到新table的适当位置，最后使用新的table，并重新计算HashMap的扩容阀值。
## get方法
<pre>
 public V get(Object key) {
        // 当key为null, 这里不讨论，后面统一讲
        if (key == null)
            return getForNullKey();
        // 根据key得到key对应的Entry
        Entry<K,V> entry = getEntry(key);
        // 
        return null == entry ? null : entry.getValue();
    }
</pre>
<pre>
 final Entry<K,V> getEntry(Object key) {
        // 根据key算出hash
        int hash = (key == null) ? 0 : hash(key);
        // 先算出hash在table中存储的index，然后遍历table中下标为index的单向链表
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            // 如果hash和key都相同，则把Entry返回
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
</pre>
- key的hash值算出key对应的Entry所在链表在在table的下标。这样，我们只要遍历单向链表就可以了，时间复杂度降低到O(n)。
## 对key为null的处理
### put
<pre>
  public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
       //其他不相关代码
       .......
    }
</pre>
<pre>
   private V putForNullKey(V value) {
        // 遍历table[0]上的单向链表
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            // 如果有key为null的Entry，则替换该Entry中的value
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        // 如果没有key为null的Entry，则构造一个hash为0、key为null、value为真实值的Entry
        // 插入到table[0]上单向链表的头部
        addEntry(0, null, value, 0);
        return null;
    }
</pre>
- 其实key为null的put过程，跟普通key值的put过程很类似，区别在于key为null的hash为0，存放在table[0]的单向链表上而已
### get
<pre>
  public V get(Object key) {
        if (key == null)
            return getForNullKey();
            //不相关的代码
            ......
    }
</pre>
<pre>
 private V getForNullKey() {
        //  遍历table[0]上的单向链表
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            // 如果key为null，则返回该Entry的value值
            if (e.key == null)
                return e.value;
        }
        return null;
    }
</pre>
- key为null的取值，跟普通key的取值也很类似，只是不需要去算hash和确定存储在table上的index而已，而是直接遍历talbe[0]。

- 所以，在HashMap中，不允许key重复，而key为null的情况，只允许一个key为null的Entry，并且存储在table[0]的单向链表上。

## remove方法
<pre>
public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }
</pre>
<pre>
 final Entry<K,V> removeEntryForKey(Object key) {
        // 算出hash
        int hash = (key == null) ? 0 : hash(key);
        // 得到在table中的index
        int i = indexFor(hash, table.length);
        // 当前结点的上一个结点，初始为table[index]上单向链表的头结点
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            // 得到下一个结点
            Entry<K,V> next = e.next;
            Object k;
            // 如果找到了删除的结点
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                // 如果是table上的单向链表的头结点，则直接让把该结点的next结点放到头结点
                if (prev == e)
                    table[i] = next;
                else
                    // 如果不是单向链表的头结点，则把上一个结点的next指向本结点的next
                    prev.next = next;  
                // 空实现
                e.recordRemoval(this);
                return e;
            }
            // 没有找到删除的结点，继续往下找
            prev = e;
            e = next;
        }

        return e;
    }
</pre>
# 线程安全
- 由前面HashMap的put和get方法分析可得，put和get方法真实操作的都是Entry[] table这个数组，而所有操作都没有进行同步处理，所以HashMap是线程不安全的。如果想要实现线程安全，推荐使用ConcurrentHashMap。
# 总结
- HashMap是基于哈希表实现的，用Entry[]来存储数据，而Entry中封装了key、value、hash以及Entry类型的next
- HashMap存储数据是无序的
- hash冲突是通过拉链法解决的
- HashMap的容量永远为2的幂次方，有利于哈希表的散列
- HashMap不支持存储多个相同的key，且只保存一个key为null的值，多个会覆盖
- put过程，是先通过key算出hash，然后用hash算出应该存储在table中的index，然后遍历table[index]，看是否有相同的key存在，存在，则更新value；不存在则插入到table[index]单向链表的表头，时间复杂度为O(n)
- get过程，通过key算出hash，然后用hash算出应该存储在table中的index，然后遍历table[index]，然后比对key，找到相同的key，则取出其value，时间复杂度为O(n)
- HashMap是线程不安全的，如果有线程安全需求，推荐使用ConcurrentHashMap。


