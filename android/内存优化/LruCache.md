## 定义
- LruCache 顾名思义就是使用LRU缓存策略的缓存，那么LRU是什么呢? 最近最少使用到的（least recently used），就是当超出缓存容量的时候，就优先淘汰链表中最近最少使用的那个数据。
- LruCache的源码本身很少，而实现了LRU的功能都是靠LinkedHashMap。所以我们先来一起学习一下LinkedHashMap。
## 实现
<pre>
public class LruCache<K, V> {
    private final LinkedHashMap<K, V> map;
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        // 注意第三个参数，是accessOrder，这里为true，后面会讲到
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
</pre>
- accessOrder为true，表示LinkedHashMap为访问顺序，当对已存在LinkedHashMap中的Entry进行get和put操作时，会把Entry移动到双向链表的表尾（其实是先删除，再插入）。
## put 方法
<pre>
 public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        // 对map进行操作之前，先进行同步操作
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }
        // 整理内存，看是否需要移除LinkedHashMap中的元素
        trimToSize(maxSize);
        return previous;
    }
</pre>
- HashMap是线程不安全的，LinkedHashMap同样是线程不安全的。所以在对调用LinkedHashMap的put方法时，先使用synchronized 进行了同步操作。
- 其中maxSize为我们给LruCache设置的最大缓存大小。
<pre>
 public void trimToSize(int maxSize) {
        // while死循环，直到满足当前缓存大小小于或等于最大可缓存大小
        while (true) {
            K key;
            V value;
            // 线程不安全，需要同步
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }
                // 如果当前缓存的大小，已经小于等于最大可缓存大小，则直接返回
                // 不需要再移除LinkedHashMap中的数据
                if (size <= maxSize || map.isEmpty()) {
                    break;
                }
                // 得到的就是双向链表表头header的下一个Entry
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                // 移除当前取出的Entry
                map.remove(key);
                // 从新计算当前的缓存大小
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }

</pre>
- 从注释上就可以看出，该方法就是不断移除LinkedHashMap中双向链表表头的元素，直到当前缓存大小小于或等于最大可缓存的大小。

- 由前面的重排序我们知道，对LinkedHashMap的put和get操作，都会让被操作的Entry移动到双向链表的表尾，而移除是从map.entrySet().iterator().next()开始的，也就是双向链表的表头的header的after开始的，这也就符合了LRU算法的需求。
