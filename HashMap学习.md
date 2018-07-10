##HashMap
这个实现对get和put提供了固定时间的性能
迭代完全部集合的时间是与HashMap的容量+键值对的长度是正相关的。所以，如果对迭代性能要求很高的话不要吧初始容量设置过高或者load factor太低

一个HashMap的实例有两个影响其性能的参数：init capacity 和load factor。
capacity：在hash table中buckets(存储数据的地方)的数量
initial capacity :hash table创建时候的大小
load factor ：是一个度量，在容量自动增长之前，hash table允许达到的大小。
当hash table中的条目超过了 load factor 乘以 当前capacity，hash table将会重新hash（就是说，内部数据结构将会重建），这样的话 hash table的代销将会差不多翻倍
默认的loadFactor是 0.75f

######构造函数
```

/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 */

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}


public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
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
       this.loadFactor = loadFactor;
       this.threshold = tableSizeFor(initialCapacity);
   }
```





######构造函数中的阀值初始化
```
/**
    * Returns a power of two size for the given target capacity.
    */
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
这里>>>是无符号的右位移，>>>1就是有位移1位
10 >>> 1   分解下
10的二进制是1010
1010 >>> 1
=> 1010 >>> 1
=> 101 (翻译为十进制是 5 (2的2次方 + 2的0次方))   

10 >>> 2  分解下
10的二进制是1010
1010 >>> 2
=> 1010 >>> 2
=> 10 （十进制是2 ，2的一次方）


|是位或但是看完还是很难理解打印下就能理解了
```
testHash((int) (Math.pow(2,12) +1));

public static void testHash(int cap){
    System.out.println("start " + cap + " " + Integer.toBinaryString(cap));
    int n = cap - 1;
    print("testHash 0",n);
    n |= n >>> 1;
    print("testHash 1",n);
    n |= n >>> 2;
    print("testHash 2",n);
    n |= n >>> 4;
    print("testHash 4",n);
    n |= n >>> 8;
    print("testHash 8",n);
    n |= n >>> 16;
    print("testHash 16",n);
    System.out.println("start " + n + " " + Integer.toBinaryString(n));
}
打印出来的值
start 4097 1000000000001
testHash 0 : 1000000000000
testHash 1 : 1100000000000
testHash 2 : 1111000000000
testHash 4 : 1111111100000
testHash 8 : 1111111111111
testHash 16 : 1111111111111
start 8191 1111111111111
```
这里的输出就是 2^cap的二次方长度-1



put的时候会对key进行hash，key.hashCode就是对应的类型的hashCode，例如 Integer.hashCode就是自己，然后把hash值无符号右移16位，
原因：  
1. 一个Integer是32位，正好分成做左右两个区域，然后做异或，增加了低位的随机性，因为低位后面要用来算key在table数组的哪个位置
1. 是扰动函数减少碰撞的  
1. 参考 https://www.zhihu.com/question/20733617，
```
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```  



####放入key，value
在放入key，value时，走的是 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)  

看下定义的局部变量
Node<K,V>[] tab; Node<K,V> p; int n, i;
tab对应着全局的table
n是table的长度
i是将要插入的位置
p是即将要插入的节点
我来尝试下注释
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                  boolean evict) {
       Node<K,V>[] tab; Node<K,V> p; int n, i;
       //将全局table赋值给局部tab
       //n就是table的长度
       if ((tab = table) == null || (n = tab.length) == 0)
       //进入这行说明table没有初始化或者table的长度为0
       //那么就创建table数组，重新给n赋值
           n = (tab = resize()).length;  
       //  table的第i个元素没有的话，就直接新建个节点放进去      
       if ((p = tab[i = (n - 1) & hash]) == null)
           tab[i] = newNode(hash, key, value, null);
       else {
           //进入这里后，p = table的第i个node
           //如果有的话，就根据类型放入
           Node<K,V> e; K k;
           if (p.hash == hash &&
               ((k = p.key) == key || (key != null && key.equals(k))))
               //key值一样，就e=p
               e = p;
           else if (p instanceof TreeNode)
               //是个红黑树
               e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
           else {
               //binCount是这个node的链表的长度
               //不停的把链表中的下一个node（e）赋值给p
               for (int binCount = 0; ; ++binCount) {
                   //会一直找到链表的尾部
                   //如果到了尾部就建个新的node放入尾部的next，如果链表的长度超过阀值，转换为红黑树
                   if ((e = p.next) == null) {
                       p.next = newNode(hash, key, value, null);
                       if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                           treeifyBin(tab, hash);
                       break;
                   }
                   //如果找了就key相等的节点，就结束了
                   if (e.hash == hash &&
                       ((k = e.key) == key || (key != null && key.equals(k))))
                       break;
                   p = e;
               }
           }
           //这里就对应着上面的 if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))
           //如果是找到尾部都没找到的话 e = null
           //在原来的hashMap里找到了节点
           if (e != null) { // existing mapping for key
               V oldValue = e.value;
               //onlyIfAbsent为true的话是只有空值的时候才覆盖
               if (!onlyIfAbsent || oldValue == null)
                   //把新值覆盖旧值
                   e.value = value;
               afterNodeAccess(e);
               return oldValue;
           }
       }
       ++modCount;
       //总容量大于阀值，就重新hash
       //这里的size是放入hashMap的元素数量
       //从这里就看出threshold这个临界值是指的hashMap中所有元素数量的临界值
       if (++size > threshold)
           resize();
       afterNodeInsertion(evict);
       return null;
   }
```


######tab[i = (n - 1) & hash]
tab[i = (n - 1) & hash]大有名堂。
例如：当n就是初始化DEFAULT_INITIAL_CAPACITY的长度的时候
1.  DEFAULT_INITIAL_CAPACITY = 1 << 4，二进制为 10000 十进制为 16  
1.  n = 10000  (十进制16)  
1.  n-1 = 1111 (十进制15)  
1.  &是位与操作，只有有0就必定是0，所以不管Hash多么长都是最后4位进行操作
1.  假设hash = 111101111011
1.  (h-1)&hash => 1111 & 111101111011 高位与完后全是0，低位算出来是1011，十进制为1+2+8=11。
1.  11 稳稳的在16以内，如果hash的最后4位全为1,1111那么最高也才15，全为0的话0000就是0
1.  所以任你hash是多大(n - 1) & hash完了后十进制的值永远是 0 ~ 2的n次方-1，永远在数组范围内。


######创建table数组
这里有个关键参数 threshold 在putVal中
```
if (++size > threshold)
    resize();
```    
这里的size是放入hashMap的元素数量
从这里就看出threshold这个临界值是指的hashMap中所有元素数量的临界值.
从resize的代码中就可以看出threshold是与table表的长度和loadFactor相关的

```
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //table已经初始化过，有长度了
        if (oldCap > 0) {
            //如果老的数组的长度大于最大值了，直接返回旧数组，不扩容了
            //  MAXIMUM_CAPACITY = 1 << 30  是 2的30次方
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //将oldCap*2再赋值给newCap，然后再判断新的容量小于最大值，旧的大于初始值
            //这种情况是老的容量不够了 需要扩容，扩容的话每次就是左移动一位（x2）
            //newCap翻倍了，对应的阀值newThr也要翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0)
            // initial capacity was placed in threshold
            // 进入这里的情况是用的HashMap(int initialCapacity, float loadFactor)这个构造函数，
            //这个构造函数只初始化了loadFactor和threshold,table还是空
            newCap = oldThr;
        else {               
            //这里是没有初始化过，设置newCap为初始化的大小，newThr为 0.75*16
            // DEFAULT_INITIAL_CAPACITY = 1 << 4 是2的4次方等于16
            // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //话说newThr不可能为0了啊 之前的代码都赋值过了。
        //还是可能的
        //从oldCap > 0进入，oldCap小于DEFAULT_INITIAL_CAPACITY不需要扩容，newThr就是0
        //这种情况下初始化下newThr的值
        if (newThr == 0) {
            //进入这里的话，已经做过newCap = oldCap << 1这个操作
            float ft = (float)newCap * loadFactor;
            //确保新的阀值是不会超过最大值的
            //这里的newThr
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //赋值给threshold
        //之前的tableSizeFor算是白算了
        //因为tableSizeFor是构造函数的时候算的，然而没有初始化table，会走newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        threshold = newThr;

        //根据新的长度来创建table
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //如果老的table不为空的话，需要把老table里面的值转移到新table里面去
        if (oldTab != null) {
            //遍历oldTab
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                //取出节点值
                if ((e = oldTab[j]) != null) {
                    //把oldTab中的值清空
                    oldTab[j] = null;
                    //根据数组中不同的数据类型来进行不同的复制操作
                    if (e.next == null)
                        //这个节点就这自己一个数据
                        //把数据放入的新的table里去
                        //这里有个疑问，万一数组位置冲突了，咋办
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //看了下TreeNode的定义 ，这难道就是传说中的红黑树，后面在看实现
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //这个就是链表
                        //这个是table中下标为0的链表记录
                        Node<K,V> loHead = null, loTail = null;
                        //这个是table中下标为不为0的链表记录
                        //问题：不知道为什么还要分开来记录
                        //答：看最后的代码，0的话还是放到新table的0中，其他的放到j+oldCap的位置中
                        Node<K,V> hiHead = null, hiTail = null;

                        Node<K,V> next;
                        //while ((e = next) != null)指的是table[j]中元素的下一个不为空
                        do {
                            //不停的往后取
                            next = e.next;
                            //e.hash & oldCap是取出e所在table数组的下标
                            if ((e.hash & oldCap) == 0) {

                                //如果loTail还没赋值，就把e赋值给loHead
                                //如果赋值了，就把e赋值给loTail,并把loTail往后移动
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                //单向列表的正常操作，head记录头部一直不动，tail一直往后移动，然后就能链接成链表    
                                loTail = e;
                            }
                            else {
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
                            //问题：放到j+oldCap中hash&newCap对的上么
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
#####通过key读取value
读取就没那么多套路了
```
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //全局table数组的长度要大于0
        //能够通过(n - 1) & hash计算出的下标，找到对应的节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //检查第一个  
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //往后一个找    
            if ((e = first.next) != null) {
                //如果当前的这个Node是个红黑树,就走红黑树的查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //如果是链表的话，循环下一个找    
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

小结：
1. HashMap是链表和数组的结合
1. 维护了一个全局table数组，数组的内容一开始是一个链表，随着node长度的增加会转变为红黑树
1. 放入key，value的时候，先对key进行hash运算。Hashmap的hash计算就是key对象的hash与自己右移16位位异或的结果，这样可以减少碰撞
1. 然后用hash值与table的长度-1进行与运算算出这次key-value应该放入table的下标
1. 找到对应的table[i]后先遍历下key和hash是否已经存在，存在就覆盖，不存在就添加到链表的尾部
1. 需要转换为红黑树就转换
1. 放完了如果大于阀值就扩容
1. 读取的话就很简单了，对key进行hash操作，先找到所属链表的位置，然后遍历链表找到对应的值
