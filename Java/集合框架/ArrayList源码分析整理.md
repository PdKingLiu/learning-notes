[toc]

# 概述
ArrayList继承于AbstractList，并实现了Serializable，Cloneable。Iterable，Collection，List，RandomAccess这些接口。

- Seriallizable ，序列化接口，支持序列化，可通过序列化传输
- Cloneable，能被克隆
- Iterable，可以被迭代器遍历
- Collection，拥有集合操作的所有方法
- 实现了List接口，拥有增删改
- 实现了RandomAccess，支持快速随机访问

**特点：**

- 底层是一个动态扩容的数组结构，初始容量为10，扩容时增加1.5倍容量（addAll可能有例外）
- 允许存放重复数据，存储顺序按照元素添加顺序，允许存在null
- 底层使用Arrays.copyOf函数进行扩容，扩容产生新的数组、对数组内容的深拷贝会耗费性能
- ArrayList不是一个线程安全的集合，若涉及线程安全，可使用CopyOnWriteArrayList或Collections.synchronizedList()

# 属性

```java
	//序列化ID
    private static final long serialVersionUID = 8683452581122892189L;

    //默认容量
    private static final int DEFAULT_CAPACITY = 10;

    //默认空数组
    private static final Object[] EMPTY_ELEMENTDATA = {};

    //无参调用时使用该数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    //存储ArrayList元素的数组
    transient Object[] elementData; // non-private to simplify nested class access

    //元素的个数
    private int size;
```

# 构造方法

```java
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
- 第一个，指定大小的构造方法，如果传入为0，直接使用EMPTY_ELEMENTDATA，否则new一个
- 第二个，空无参构造，将默认数组赋值给他
- 第三个，参数为集合类，首先直接利用Collection.toArray()方法得到一个对象数组，并赋值给elementData。

不管调用哪个方法都会初始化内部elementData 。

# add系列

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
执行`ensureCapacityInternal(size + 1)`确保内部容量

```java
private void ensureCapacityInternal(int minCapacity) {
    // 如果创建 ArrayList 时候，使用的无参的构造方法，那么就取默认容量 10 和最小需要的容量（当前 size + 1 ）中大的一个确定需要的容量。
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}
```
若使用的无参的构造方法，那么就取默认容量 10 和 size + 1 中大的一个确定需要的容量。

第一次执行 ensureCapacityInternal 的时候，要扩容的容量就是 DEFAULT_CAPACITY = 10。

```java
private void ensureExplicitCapacity(int minCapacity) {
    // 修改 +1 
    modCount++;
    // 如果 minCapacity 比当前容量大， 就执行grow 扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // 拿到当前的容量
    int oldCapacity = elementData.length;
    // oldCapacity >> 1 意思就是 oldCapacity/2，所以新容量就是增加 1/2.
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 如果新容量小于，需要最小扩容的容量，以需要最小容量为准扩容
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 如果新容量大于允许的最大容量，则以 Inerger 的最大值进行扩容
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 使用 Arrays.copyOf 函数进行扩容。
    elementData = Arrays.copyOf(elementData, newCapacity);
}

// 允许的最大容量
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}

```

- ensureExplicitCapacity 方法：如果minCapacity 比当前容量大则表明需要扩容。
- 通过oldCapacity 增加1.5倍 得到 newCapacity ，如果1.5倍后的newCapacity 还不比minCapacity 大，那么就指定newCapacity 为minCapacity。
- 如果新容量大于最大容量，则以Integer最大值进行扩容。
- 最后调用Arrays.copyOf进行扩容。

可以看到，也并不是每次都是1.5倍扩容的。

扩容完成后：

```java
elementData[size++] = e;
return true;
```

**指定位置添加元素**

```java
public void add(int index, E element) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
} 
```

- 首先判断越界
- 然后确认需不需要扩容
- 再进行深拷贝

**addAll方法**

```java
public boolean addAll(int index, Collection<? extends E> c) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);
    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

- 基本上和add方法差不多

# remove系列
删除指定下标的元素

```java
public E remove(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    modCount++;
    E oldValue = (E) elementData[index];
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
} 
```

- 先判断是否越界
- 把要删除的元素拿出来
- 计算要移动的长度，执行System.arraycopy()
- 最后一个元素置为null


**删除指定元素**

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

分为两种情况

- 元素为null，循环查找第一个为null的元素
- 元素不为null，使用o.equals(elementData[index])进行判等，然后进行移动。

# set系列

```java
public E set(int index, E element) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    E oldValue = (E) elementData[index];
    elementData[index] = element;
    return oldValue;
}
```

- 判断下标越界情况
- 修改值

# get系列

```java
public E get(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    return (E) elementData[index];
}
```
- 先判断越界，然后返回

# clear方法

```java
public void clear() {
    modCount++;
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    size = 0;
}
```

- 遍历所有元素赋值为null

# indexOf 方法

```java
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
} 
```
分两种情况

- null，返回第一个为null的元素
- 不为null，使用o.equals(elementData[i])判等并返回。

# lastIndexOf方法

```java
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
} 
```

- 和indexOf 类似，只不过先从后面查起

# fail-fast事件
>The iterators returned by this class's iterator() and
listIterator(int) methods are fail-fast
if the list is structurally modified at any time after the iterator is
created, in any way except through the iterator's own
ListIterator remove() or ListIterator add(Object) methods,
the iterator will throw a ConcurrentModificationException. Thus, in the face of
concurrent modification, the iterator fails quickly and cleanly, rather
than risking arbitrary, non-deterministic behavior at an undetermined
time in the future.

>ArrayList 迭代器中的方法都是均具有快速失败的特性，当遇到并发修改的情况时，迭代器会快速失败，以避免程序在将来不确定的时间里出现不确定的行为。


```java
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        // 并发修改检测，检测不通过则抛出异常
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
```

- 在使用迭代器遍历时会提前保存一份`expectedModCount = modCount`，在遍历时会进行并发修改检测。

```java
Iterator<String> it = a.iterator();
while (it.hasNext()) {
    String temp = it.next();
    System.out.println("temp: " + temp);
    if("1".equals(temp)){
        a.remove(temp);
    }
}
```
在开发中我们需要避免上面的操作，正确的做法是使用迭代器提供的删除方法，而不是直接删除。

# 总结
- ArrayList 底层是一个动态扩容的数组结构，初始容量为10，每次容量不够的时候，扩容增加1.5倍容量
- 增加删除会改变modCount
- 增加和删除都要移动元素比较低效，查找和修改时很高效
- ArrayList支持null的存储
- ArrayList是非线程安全的
- 可以使用CopyOnWriteArrayList 或者使Collections.synchronizedList() 函数返回一个线程安全的 ArrayList 