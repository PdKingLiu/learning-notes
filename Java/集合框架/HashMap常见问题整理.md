[toc]
## 一、HashMap实现原理
### 你看过HashMap源码嘛，知道原理嘛?

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191015201229736.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
HashMap采用Entry数组来存储key-value对，每一个键值对组成了一个Entry实体，Entry类实际上是一个单向的链表结构，它具有Next指针，可以连接下一个Entry实体。
只是在JDK1.8中，链表长度大于8的时候，链表会转成红黑树！
### 为什么用数组+链表？
数组是用来确定桶的位置，利用元素的key的hash值对数组长度取模得到.
链表是用来解决hash冲突问题，当出现hash值一样的情形，就在数组上的对应位置形成一条链表
### hash冲突你还知道哪些解决办法？
比较出名的有四种：开放定址法、链地址法、再哈希法、公共溢出区域法

- 开放地址法
	从发生冲突的那个单元起，按照一定的次序，从哈希表中找到一个空闲的单元。然后把发生冲突的元素存入到该单元的一种方法。开放定址法需要的表长度要大于等于所需要存放的元素。
- 再散列
	当发生冲突时，使用第二个、第三个、哈希函数计算地址，直到无冲突时。缺点：计算时间增加。

- 链接地址法的思路是将哈希值相同的元素构成一个同义词的单链表，并将单链表的头指针存放在哈希表的第i个单元中，查找、插入和删除主要在同义词链表中进行。在java中，链接地址法也是HashMap解决哈希冲突的方法之一。
- 公共溢出区域法
	将哈希表分为公共表和溢出表，当溢出发生时，将所有溢出数据统一放到溢出区。

### 我用LinkedList代替数组结构可以么?
答案很明显，必须是可以的。
既然是可以的,为什么HashMap不用LinkedList,而选用数组?
因为用数组效率最高！
在HashMap中，定位桶的位置是利用元素的key的哈希值对数组长度取模得到。此时，我们已得到桶的位置。显然数组的查找效率比LinkedList大。
### 既然是可以的,为什么HashMap不用LinkedList，而选用数组?
因为采用基本数组结构，扩容机制可以自己定义，HashMap中数组扩容刚好是2的次幂，在做取模运算的效率高。
而ArrayList的扩容机制是1.5倍扩容，那ArrayList为什么是1.5倍扩容这就不在本文说明了。

## 二、HashMap在什么条件下扩容
### HashMap在什么条件下扩容
如果bucket满了(超过load factor*current capacity)，就要resize。
load factor为0.75，为了最大程度避免哈希冲突
current capacity为当前数组大小。

### 为什么扩容是2的次幂
HashMap为了存取高效，要尽量较少碰撞，就是要尽量把数据分配均匀，每个链表长度大致相同，这个实现就在把数据存到哪个链表中的算法；这个算法实际就是取模，hash%length。
但是，大家都知道这种运算不如位移运算快。
因此，源码中做了优化hash&(length-1)。
也就是说hash%length==hash&(length-1)
位运算比较高效，当b为2的n次方时，有如下替换公式：
a % b = a & (b-1)(b=2n)
即：a % 2n = a & (2n-1)

### 为什么为什么要先高16位异或低16位再取模运算
hashmap这么做，只是为了降低hash冲突的几率。

比如，当我们的length为16的时候，哈希码(字符串“abcabcabcabcabc”的key对应的哈希码)对(16-1)与操作，对于多个key生成的hashCode，只要哈希码的后4位为0，不论不论高位怎么变化，最终的结果均为0。
如下图所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191015203155696.png)
而加上高16位异或低16位的“扰动函数”后，结果如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191015203219969.png)
## 三、hashmap的get/put的过程
### 知道hashmap中put元素的过程是什么样吗
对key的hashCode()做hash运算，计算index;
如果没碰撞直接放到bucket里；
如果碰撞了，以链表的形式存在buckets后；
如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD)，就把链表转换成红黑树(JDK1.8中的改动)；
如果节点已经存在就替换old value(保证key的唯一性)
如果bucket满了(超过load factor*current capacity)，就要resize。

### 知道hashmap中get元素的过程是什么样吗
对key的hashCode()做hash运算，计算index;
如果在bucket里的第一个节点里直接命中，则直接返回；
如果有冲突，则通过key.equals(k)去查找对应的Entry;
若为树，则在树中通过key.equals(k)查找，O(logn)；
若为链表，则在链表中通过key.equals(k)查找，O(n)。
### 你还知道哪些hash算法？
先说一下hash算法干嘛的，Hash函数是指把一个大范围映射到一个小范围。把大范围映射到一个小范围的目的往往是为了节省空间，使得数据容易保存。
比较出名的有MurmurHash、MD4、MD5等等

### 说说String中hashcode的实现?(此题频率很高)

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```
String类中的hashCode计算方法还是比较简单的，就是以31为权，每一位为字符的ASCII值进行运算，用自然溢出来等效取模。

主要是因为31是一个奇质数，所以31*i=32*i-i=(i<<5)-i，这种位移与减法结合的计算相比一般的运算快很多。

## 四、为什么hashmap的在链表元素数量超过8时改为红黑树
### 知道jdk1.8中hashmap改了啥吗
- 由数组+链表的结构改为数组+链表+红黑树。
- 优化了高位运算的hash算法：h^(h>>>16)
 - 扩容后，元素要么是在原位置，要么是在原位置再移动2次幂的位置，且链表顺序不变。
 
 最后一条是重点，因为最后一条的变动，hashmap在1.8中，不会在出现死循环问题。
### 为什么在解决hash冲突的时候，不直接用红黑树?而选择先用链表，再转红黑树
因为红黑树需要进行左旋，右旋，变色这些操作来保持平衡，而单链表不需要。
当元素小于8个当时候，此时做查询操作，链表结构已经能保证查询性能。当元素大于8个的时候，此时需要红黑树来加快查询速度，但是新增节点的效率变慢了。
因此，如果一开始就用红黑树结构，元素太少，新增效率又比较慢，无疑这是浪费性能的。

### 我不用红黑树，用二叉查找树可以么
可以。但是二叉查找树在特殊情况下会变成一条线性结构（这就跟原来使用链表结构一样了，造成很深的问题），遍历查找会非常慢。

### 当链表转为红黑树后，什么时候退化为链表
为6的时候退转为链表。中间有个差值7可以防止链表和树之间频繁的转换。假设一下，如果设计成链表个数超过8则链表转换成树结构，链表个数小于8则树结构转换成链表，如果一个HashMap不停的插入、删除元素，链表个数在8左右徘徊，就会频繁的发生树转链表、链表转树，效率会很低。

## 五、HashMap的并发问题
### HashMap在并发编程环境下有什么问题
- 多线程扩容，引起的死循环问题
- 多线程put的时候可能导致元素丢失
- put非null元素后get出来的却是null

### 在jdk1.8中还有这些问题么
在jdk1.8中，死循环问题已经解决。其他两个问题还是存在。
[https://coolshell.cn/articles/9606.html](https://coolshell.cn/articles/9606.html)

## 六、你一般用什么作为HashMap的key
### 健可以为Null值么
可以，key为null的时候，hash算法最后的值以0来计算，也就是放在数组的第一个位置。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191015204345343.png)
### 你一般用什么作为HashMap的key
一般用Integer、String这种不可变类当HashMap当key，而且String最为常用。

- 因为字符串是不可变的，所以在它创建的时候hashcode就被缓存了，不需要重新计算。这就使得字符串很适合作为Map中的键，字符串的处理速度要快过其它的键对象。这就是HashMap中的键往往都使用字符串。
- 因为获取对象的时候要用到equals()和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的,这些类已经很规范的覆写了hashCode()以及equals()方法。

### 我用可变类当HashMap的key有什么问题
hashcode可能发生改变，导致put进去的值，无法get出。
### 如果让你实现一个自定义的class作为HashMap的key该如何实现
针对问题一，记住下面四个原则
- 两个对象相等，hashcode一定相等
- 两个对象不等，hashcode不一定不等
- hashcode相等，两个对象不一定相等
- hashcode不等，两个对象一定不等

### 如何写一个不可变类
- 类添加final修饰符，保证类不被继承
- 保证所有成员变量必须私有，并且加上final修饰
- 不提供改变成员变量的方法，包括setter
- 通过构造器初始化所有成员，进行深拷贝(deep copy)
	```java
	public final class MyImmutableDemo {  
	    private final int[] myArray;  
	    public MyImmutableDemo(int[] array) {  
	        this.myArray = array.clone();   
	    }   
	}
	```
- 在getter方法中，不要直接返回对象本身，而是克隆对象。