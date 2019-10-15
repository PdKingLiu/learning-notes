# 基本数据
JDK8的HashMap在内部实现上使用数组+链表+红黑树三种数据结构。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191015171734602.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
对于hash冲突，HashMap的解决是使用链地址法，将hash相同的记录放在同一个数组位置上，多个hash相同的记录被存储在一同链上。

链表节点

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; //哈希值，HashMap用这个值来确定记录的位置
        final K key; //记录key
        V value; //记录value
        Node<K,V> next;//链表下一个节点
		······
    }
```

Node数组

```java
transient Node<K,V>[] table;
```

```java

    //键值对数目
    transient int size;
    //作为扩容的阈值
    int threshold;    
     //负载因子
    final float loadFactor;   
    //默认容量
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16   
    //最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30; 
    //默认负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

HashMaptable大小默认是2的n次方，初始大小为16

计算最接近2的n次方：

```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

**确定记录的table位置**

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
首先计算出hash值，然后和table的长度取余来定位index。

取模的步骤是复杂的耗时的，但是将table的长度设置为2的n次方，在计算index的时候就可以通过下面算法进行：

```java
int index = hashCode & (length -1)
```

# put方法

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191015174647748.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

- 首先判空或长度为0，然后通过resize进行初始化。resize用在初始化和扩容。
- 计算hashcode及index，如果发现对应的值为null，则直接放入。
- 若不为空且记录和index的记录相等，则直接覆盖。
- 如果这个index上的节点是TreeNode，那么说明是一个红黑树，然后将其插入对应红黑树中。
- 如果这个节点类型不是TreeNode，说明冲突阈值还没有达到阈值。如果这个记录是新的会插入最尾部，负责将替换旧的记录。如果插入后达到阈值，则将其转化为一个红黑树。
- 检查哈希桶是否达到阈值，如果达到了，调用resize扩容。

## resize扩容

```java
 final Node<K, V>[] resize() {
     Node<K, V>[] oldTab = table;
     int oldCap = (oldTab == null) ? 0 : oldTab.length;
     int oldThr = threshold;
     int newCap, newThr = 0;
     if (oldCap > 0) {
         if (oldCap >= MAXIMUM_CAPACITY) { //如果旧的容量比最大容量要大，那就不再散列，直接返回旧的散列表
             threshold = Integer.MAX_VALUE;
             return oldTab;
         } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
             newThr = oldThr << 1; // double threshold 两倍的阈值，通过位运算向左移位得到
     } else if (oldThr > 0) // initial capacity was placed in threshold  如果旧的阈值 > 0，数组的新容量设置为就数组的阈值。
         newCap = oldThr;
     else {               // zero initial threshold signifies using defaults
         //第一次初始化散列表的操作
         newCap = DEFAULT_INITIAL_CAPACITY; 
         newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
     }
     if (newThr == 0) {
         float ft = (float) newCap * loadFactor;
         newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ?
                 (int) ft : Integer.MAX_VALUE);
     }
     threshold = newThr;
     @SuppressWarnings({"rawtypes", "unchecked"})
     Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
     table = newTab;
     //将旧的散列表复制到新的散列表。
     if (oldTab != null) {
         for (int j = 0; j < oldCap; ++j) {
             Node<K, V> e;
             if ((e = oldTab[j]) != null) {
                 oldTab[j] = null;
                 if (e.next == null)
                     newTab[e.hash & (newCap - 1)] = e;
                 else if (e instanceof TreeNode)  //如果是红黑树，这样操作
                     ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                 else { // preserve order
                     Node<K, V> loHead = null, loTail = null;  //如果是链表，这样操作
                     Node<K, V> hiHead = null, hiTail = null;
                     Node<K, V> next;
                     do {
                         next = e.next;
                         if ((e.hash & oldCap) == 0) {
                             if (loTail == null)
                                 loHead = e;
                             else
                                 loTail.next = e;
                             loTail = e;
                         } else {
                             if (hiTail == null)
                                 hiHead = e;
                             else
                                 hiTail.next = e;
                             hiTail = e;
                         }
                     } while ((e = next) != null);
                     if (loTail != null) {
                         loTail.next = null;
                         newTab[j] = loHead;
                     }
                     if (hiTail != null) {
                         hiTail.next = null;
                         newTab[j + oldCap] = hiHead;
                     }
                 }
             }
         }
     }
     return newTab;
 }
```
每次扩容会增大当前容量的两倍。如果Hash的容量已经达到最大值了那么就不会在进行扩容了。

在迁移内容时

table的长度确保是2的n次方，每次扩容容量变为原来的两倍，那么一个记录在新table中的位置要么就和原来一样，要么就需要迁移到(oldCap + index)的位置上。

# get方法

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

- 得到当前table对应该index的记录，如果为null，说明没有记录落在该位置上，直接返回null。
- 如果不为null，则说明至少有一个记录，如果使我们查找的记录，那么直接返回，否则判断该记录是链表还是一个红黑树来分别查找需要的记录。

## remove方法

```java
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    final Node<K, V> removeNode(int hash, Object key, Object value,
                                boolean matchValue, boolean movable) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
                (p = tab[index = (n - 1) & hash]) != null) { //判断桶是否为空，映射的哈希值是否存在
            Node<K, V> node = null, e;
            K k;
            V v;
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k)))) //如果是在首位找到就记录下来
                node = p;
            else if ((e = p.next) != null) { //不在首位则到红黑树或者链表中寻找
                if (p instanceof TreeNode)
                    node = ((TreeNode<K, V>) p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                                ((k = e.key) == key ||
                                        (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                    (value != null && value.equals(v)))) {  //如果找到了对应的结点，并且确定了value也对得上，就开始进行删除
                if (node instanceof TreeNode) //红黑树的删除
                    ((TreeNode<K, V>) node).removeTreeNode(this, tab, movable);
                else if (node == p) //链表的删除
                    tab[index] = node.next;
                else
                    p.next = node.next; //桶首位的删除
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

换汤不换药，其实 removeNode()方法，前面几步 get()方法没什么区别，主要在于后面执行删除的时候要把value做一次判断。

