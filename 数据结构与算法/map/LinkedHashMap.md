# 定义
- HashMap是无序的，当我们希望有顺序地去存储key-value时，就需要使用LinkedHashMap,Lrucache也是通过LinkeHasMap实现的。
# 实现
## 继承关系
<pre>
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{
}
</pre>
- 继承自hashmap，在hashmap的基础上加上双向链表来实现顺序存储。
## 构造方法
<pre>
  public LinkedHashMap() {
        // 调用HashMap的构造方法，其实就是初始化Entry[] table
        super();
        // 这里是指是否基于访问排序，默认为false
        accessOrder = false;
    }
</pre>
- 调用父类初始化方法，设置 accessOrder = false
- 在HashMap的构造函数中，调用了init方法，而在HashMap中init方法是空实现，但LinkedHashMap重写了该方法，所以在LinkedHashMap的构造方法里，调用了自身的init方法，init的重写实现如下：
<pre>
 /**
     * Called by superclass constructors and pseudoconstructors (clone,
     * readObject) before any entries are inserted into the map.  Initializes
     * the chain.
     */
    @Override
    void init() {
        // 创建了一个hash=-1，key、value、next都为null的Entry
        header = new Entry<>(-1, null, null, null);
        // 让创建的Entry的before和after都指向自身，注意after不是之前提到的next
        // 其实就是创建了一个只有头部节点的双向链表
        header.before = header.after = header;
    }
</pre>
## LinkedHashMap静态内部类Entry
<pre>
/**
     * LinkedHashMap entry.
     */
    private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }
		}
</pre>
## put方法
- LinkedHashMap没有重写put方法，所以还是调用HashMap得到put方法，如下：
<pre>
 public V put(K key, V value) {
        // 对key为null的处理
        if (key == null)
            return putForNullKey(value);
        // 计算hash
        int hash = hash(key);
        // 得到在table中的index
        int i = indexFor(hash, table.length);
        // 遍历table[index]，是否key已经存在，存在则替换，并返回旧值
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        
        modCount++;
        // 如果key之前在table中不存在，则调用addEntry，LinkedHashMap重写了该方法
        addEntry(hash, key, value, i);
        return null;
    }
</pre>
<pre>
  void addEntry(int hash, K key, V value, int bucketIndex) {
        // 调用父类的addEntry，增加一个Entry到HashMap中
        super.addEntry(hash, key, value, bucketIndex);

        // removeEldestEntry方法默认返回false，不用考虑
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
</pre>
<pre>
 void addEntry(int hash, K key, V value, int bucketIndex) {
        // 扩容相关
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        // LinkedHashMap进行了重写
        createEntry(hash, key, value, bucketIndex);
    }
</pre>
- LinkedHashMap重写了createEntry方法。
<pre>
 void createEntry(int hash, K key, V value, int bucketIndex) {
       HashMap.Entry<K,V> old = table[bucketIndex];
       // e就是新创建了Entry，会加入到table[bucketIndex]的表头
       Entry<K,V> e = new Entry<>(hash, key, value, old);
       table[bucketIndex] = e;
       // 把新创建的Entry，加入到双向链表中
       e.addBefore(header);
       size++;
   }
</pre>
- addBefore方法
<pre>
  private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }
</pre>
- 当put元素时，不但要把它加入到HashMap中去，还要加入到双向链表中，所以可以看出LinkedHashMap就是HashMap+双向链表，下面用图来表示逐步往LinkedHashMap中添加数据的过程，红色部分是双向链表，黑色部分是HashMap结构，header是一个Entry类型的双向链表表头，本身不存储数据。

## 扩容
- 在HashMap的put方法中，如果发现前元素个数超过了扩容阀值时，会调用resize方法，如下
<pre>
  void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        boolean oldAltHashing = useAltHashing;
        useAltHashing |= sun.misc.VM.isBooted() &&
                (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        boolean rehash = oldAltHashing ^ useAltHashing;
       // 把旧table的数据迁移到新table
        transfer(newTable, rehash);
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
</pre>
- LinkedHashMap重写了transfer方法，数据的迁移，它的实现如下
<pre>
 void transfer(HashMap.Entry[] newTable, boolean rehash) {
        // 扩容后的容量是之前的2倍
        int newCapacity = newTable.length;
        // 遍历双向链表，把所有双向链表中的Entry，重新就算hash，并加入到新的table中
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            if (rehash)
                e.hash = (e.key == null) ? 0 : hash(e.key);
            int index = indexFor(e.hash, newCapacity);
            e.next = newTable[index];
            newTable[index] = e;
        }
    }
</pre>
- HashMap是先遍历旧table，再遍历旧table中每个元素的单向链表，取得Entry以后，重新计算hash值，然后存放到新table的对应位置。
- LinkedHashMap是遍历的双向链表，取得每一个Entry，然后重新计算hash值，然后存放到新table的对应位置。
- 从遍历的效率来说，遍历双向链表的效率要高于遍历table，因为遍历双向链表是N次（N为元素个数）；而遍历table是N+table的空余个数（N为元素个数）。
## 双向链表的重排序
- 当前LinkedHashMap中不存在当前key时，新增Entry的情况。当key如果已经存在时，则进行更新Entry的value。就是HashMap的put方法中的如下代码：
<pre>
 for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                // 重排序
                e.recordAccess(this);
                return oldValue;
            }
        }
</pre>
- 主要看e.recordAccess(this)，这个方法跟访问顺序有关，而HashMap是无序的，所以在HashMap.Entry的recordAccess方法是空实现，但是LinkedHashMap是有序的,LinkedHashMap.Entry对recordAccess方法进行了重写。
<pre>
 void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            // 如果LinkedHashMap的accessOrder为true，则进行重排序
            // 比如前面提到LruCache中使用到的LinkedHashMap的accessOrder属性就为true
            if (lm.accessOrder) {
                lm.modCount++;
                // 把更新的Entry从双向链表中移除
                remove();
                // 再把更新的Entry加入到双向链表的表尾
                addBefore(lm.header);
            }
        }
</pre>
- 在LinkedHashMap中，只有accessOrder为true，即是访问顺序模式，才会put时对更新的Entry进行重新排序，而如果是插入顺序模式时，不会重新排序，这里的排序跟在HashMap中存储没有关系，只是指在双向链表中的顺序。
## get方法
<pre>
 public V get(Object key) {
        // 调用genEntry得到Entry
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        // 如果LinkedHashMap是访问顺序的，则get时，也需要重新排序
        e.recordAccess(this);
        return e.value;
    }
</pre>
- 先是调用了getEntry方法，通过key得到Entry，而LinkedHashMap并没有重写getEntry方法，所以调用的是HashMap的getEntry方法，在上一篇文章中我们分析过HashMap的getEntry方法：首先通过key算出hash值，然后根据hash值算出在table中存储的index，然后遍历table[index]的单向链表去对比key，如果找到了就返回Entry。
- 后面调用了LinkedHashMap.Entry的recordAccess方法，上面分析过put过程中这个方法，其实就是在访问顺序的LinkedHashMap进行了get操作以后，重新排序，把get的Entry移动到双向链表的表尾。
## remove方法
- LinkedHashMap没有提供remove方法，所以调用的是HashMap的remove方法，实现如下：
<pre>
    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

    final Entry<K,V> removeEntryForKey(Object key) {
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                // LinkedHashMap.Entry重写了该方法
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }
</pre>
- 在上一篇HashMap中就分析了remove过程，其实就是断开其他对象对自己的引用。比如被删除Entry是在单向链表的表头，则让它的next放到表头，这样它就没有被引用了；如果不是在表头，它是被别的Entry的next引用着，这时候就让上一个Entry的next指向它自己的next，这样，它也就没被引用了。

- 在HashMap.Entry中recordRemoval方法是空实现，但是LinkedHashMap.Entry对其进行了重写，如下：
<pre>
  void recordRemoval(HashMap<K,V> m) {
            remove();
        }

        private void remove() {
            before.after = after;
            after.before = before;
        }
</pre>
- 把双向链表中的Entry删除，也就是要断开当前要被删除的Entry被其他对象通过after和before的方式引用。所以，LinkedHashMap的remove操作。首先把它从table中删除，即断开table或者其他对象通过next对其引用，然后也要把它从双向链表中删除，断开其他对应通过after和before对其引用。