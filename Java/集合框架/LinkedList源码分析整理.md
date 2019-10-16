[toc]

# 概述
LinkedList是List的另一种实现，他的底层是基于双向链表实现的，因此它具有插入删除快，查询慢的特点，此外，对双向链表操作还可以实现队列和栈的功能。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191016145938663.png)
# 基本数据

结点类结构

```java
//结点内部类
private static class Node<E> {
   E item;          //元素
   Node<E> next;    //下一个节点
   Node<E> prev;    //上一个节点

   Node(Node<E> prev, E element, Node<E> next) {
       this.item = element;
       this.next = next;
       this.prev = prev;
   }
}
```

成员变量和构造方法

```java
//集合元素个数
transient int size = 0;

//头结点引用
transient Node<E> first;

//尾节点引用
transient Node<E> last;

//无参构造器
public LinkedList() {}

//传入外部集合的构造器
public LinkedList(Collection<? extends E> c) {
   this();
   addAll(c);
}
```

LinkedList持有头结点和尾结点的引用，他有两个构造器，一个是无参，一个是传入外部集合的构造器。

# 增删改

```java
    //增(添加)
    public boolean add(E e) {
       //在链表尾部添加
       linkLast(e);
       return true;
    }
    
    //增(插入)
    public void add(int index, E element) {
       checkPositionIndex(index);
       if (index == size) {
           //在链表尾部添加
           linkLast(element);
       } else {
           //在链表中部插入
           linkBefore(element, node(index));
       }
    }
    
    //删(给定下标)
    public E remove(int index) {
       //检查下标是否合法
       checkElementIndex(index);
       return unlink(node(index));
    }
    
    //删(给定元素)
    public boolean remove(Object o) {
       if (o == null) {
           for (Node<E> x = first; x != null; x = x.next) {
               if (x.item == null) {
                   unlink(x);
                   return true;
               }
           }
       } else {
           //遍历链表
           for (Node<E> x = first; x != null; x = x.next) {
               if (o.equals(x.item)) {
                   //找到了就删除
                   unlink(x);
                   return true;
               }
           }
       }
       return false;
    }
    
    //改
    public E set(int index, E element) {
       //检查下标是否合法
       checkElementIndex(index);
       //获取指定下标的结点引用
       Node<E> x = node(index);
       //获取指定下标结点的值
       E oldVal = x.item;
       //将结点元素设置为新的值
       x.item = element;
       //返回之前的值
       return oldVal;
    }
    
    //查
    public E get(int index) {
       //检查下标是否合法
       checkElementIndex(index);
       //返回指定下标的结点的值
       return node(index).item;
    }
```

-  添加元素的方法只要调用linkLast和linkBefore两个方法，linkLast方法是在链表后面链接一个元素，linkBefore方法是在链表中间插入一个元素。
- 删除方法是通过unlink方法将某个元素移除。

**看一看删除的核心方法**

```java
//链接到指定结点之前
void linkBefore(E e, Node<E> succ) {
   //获取给定结点的上一个结点引用
   final Node<E> pred = succ.prev;
   //创建新结点, 新结点的上一个结点引用指向给定结点的上一个结点
   //新结点的下一个结点的引用指向给定的结点
   final Node<E> newNode = new Node<>(pred, e, succ);
   //将给定结点的上一个结点引用指向新结点
   succ.prev = newNode;
   //如果给定结点的上一个结点为空, 表明给定结点为头结点
   if (pred == null) {
       //将头结点引用指向新结点
       first = newNode;
   } else {
       //否则, 将给定结点的上一个结点的下一个结点引用指向新结点
       pred.next = newNode;
   }
   //集合元素个数加一
   size++;
   //修改次数加一
   modCount++;
}

//卸载指定结点
E unlink(Node<E> x) {
   //获取给定结点的元素
   final E element = x.item;
   //获取给定结点的下一个结点的引用
   final Node<E> next = x.next;
   //获取给定结点的上一个结点的引用
   final Node<E> prev = x.prev;

   //如果给定结点的上一个结点为空, 说明给定结点为头结点
   if (prev == null) {
       //将头结点引用指向给定结点的下一个结点
       first = next;
   } else {
       //将上一个结点的后继结点引用指向给定结点的后继结点
       prev.next = next;
       //将给定结点的上一个结点置空
       x.prev = null;
   }

   //如果给定结点的下一个结点为空, 说明给定结点为尾结点
   if (next == null) {
       //将尾结点引用指向给定结点的上一个结点
       last = prev;
   } else {
       //将下一个结点的前继结点引用指向给定结点的前继结点
       next.prev = prev;
       x.next = null;
   }

   //将给定结点的元素置空
   x.item = null;
   //集合元素个数减一
   size--;
   //修改次数加一
   modCount++;
   return element;
}
```
- linkedBefore是中间插入
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191016153905302.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- unlink 删除指定节点

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191016153939885.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
插入删除的复杂度都是O（1），而对于链表的查找和修改操作都需要遍历整个链表进行。

```java
//根据指定位置获取结点
Node<E> node(int index) {
   //如果下标在链表前半部分, 就从头开始查起
   if (index < (size >> 1)) {
       Node<E> x = first;
       for (int i = 0; i < index; i++) {
           x = x.next;
       }
       return x;
   } else {
       //如果下标在链表后半部分, 就从尾开始查起
       Node<E> x = last;
       for (int i = size - 1; i > index; i--) {
           x = x.prev;
       }
       return x;
   }
}
```

- 对index先做了一个判断，然后从左边/右边开始查找。时间复杂度是O(n/2)。

# 单向队列、双向队列、栈
**单向队列**

```java
//获取头结点
public E peek() {
   final Node<E> f = first;
   return (f == null) ? null : f.item;
}

//获取头结点
public E element() {
   return getFirst();
}

//弹出头结点
public E poll() {
   final Node<E> f = first;
   return (f == null) ? null : unlinkFirst(f);
}

//移除头结点
public E remove() {
   return removeFirst();
}

//在队列尾部添加结点
public boolean offer(E e) {
   return add(e);
}
```

**双向队列**

```java
//在头部添加
public boolean offerFirst(E e) {
   addFirst(e);
   return true;
}

//在尾部添加
public boolean offerLast(E e) {
   addLast(e);
   return true;
}

//获取头结点
public E peekFirst() {
   final Node<E> f = first;
   return (f == null) ? null : f.item;
}

//获取尾结点
public E peekLast() {
   final Node<E> l = last;
   return (l == null) ? null : l.item;
}
```

**栈**

```java
//入栈
public void push(E e) {
   addFirst(e);
}

//出栈
public E pop() {
   return removeFirst();
}
```

- 不管是队列还是栈，其都是对链表的头尾结点进行操作。

# 总结

- LinkedList是基于双向链表实现的，增删改查、队列和栈，都可通过操作结点实现
- LinkedList因为基于链表操作，集合的容量随元素加入自动增加，删除而自动缩小
- LinkedList对方法没有进行同步，因此他不是线程安全的