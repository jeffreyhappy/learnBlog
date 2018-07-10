---
title: LinkedHashMap学习
---
#LinkedHashMap
继承自HashMap实现了Map接口，使用上和HashMap一样，看名字就知道内部使用了链表


LinkedHashMap的元素先继承了HashMap.Node，添加了前后节点的zhi变成了双向的了，原来的HashMap.Node是单项的，用next来记录下一个
```
static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
        LinkedHashMapEntry<K,V> before, after;
        LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
}
```

LinkedHashMap就新加了三个成员变量
```
   //双向链表的头部
   transient LinkedHashMapEntry<K,V> head;

   //双向链表的尾部
   transient LinkedHashMapEntry<K,V> tail;

   /**
    * The iteration ordering method for this linked hash map: <tt>true</tt>
    * for access-order, <tt>false</tt> for insertion-order.
    *
    * @serial
    */
    //true ：通过访问排序 ，LRUCache就设置true，访问过一个节点后，就把该节点拿到最尾部
    //fale ：通过插入排序
   final boolean accessOrder;
```
LinkedHashMap是如果把HashMap.Node替换为LinkedHashMapEntry的呢？是通过newNode

```
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMapEntry<K,V> p =
            new LinkedHashMapEntry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
}
```

HashMap中put的时候，如果是新的key-value会创建新的Node，就是这里
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        .....
        tab[i] = newNode(hash, key, value, null);
        .....
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
    }
```
HashMap在插入新的node的时候会让老的链表尾部的next指向新节点，然后新节点变为链表尾部，这样就实现单项链表的绑定，LinkedHashMap是如何绑定的呢？
LinkedHashMap在创建新节点后linkNodeLast(p);新节点一定是尾部的
```
private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
        //拿到链表尾部
        LinkedHashMapEntry<K,V> last = tail;
        //新节点为新的尾部
        tail = p;
        //之前的链表尾部是Null的话说明这就是空的链表
        if (last == null)
            //空链表的话，有新的节点插入后，head = tail = p 。 相当初始化了
            head = p;
        else {
            //链表里面已经有节点了。
            //新节点的前一个是老尾部，老尾部的后一个是新节点
            //把前后节点的前后元素都绑定下
            p.before = last;
            last.after = p;
        }
}
```
之前的HashMap维护了个table数组，然后一个数组元素就是个单项链表(或者是红黑树)，第一个数组元素和第二个数组元素是毫无关系的
现在这个LinekedHashMap把Node扩展了下变成了双向链表，每次插入后都把新的节点绑定到之前的节点上。这样可以从head一直迭代到tail。为什么要这么干？
这样方便控制元素顺序啊
LRUCache(Least Recently Used)使用LinkedHashMap就是因为它的get。使用LinkedHashMap的get后，会把get的node放入到链表尾部。
1. 插入新的key-value。newNode函数后会成为tail。看LinkedHashMap的linkNodeLast
2. 插入key-value，value是新的，就更新下value，调用afterNodeAccess
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            .....
            //这里是更新value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
      ....
    }
```
然后LinkedHashMap重写了afterNodeAccess。
accessOrder为true是以访问顺序排序
```
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMapEntry<K,V> last;
        //以访问顺序排序，且尾元素不是e
        if (accessOrder && (last = tail) != e) {
            //p当前e元素，b：e的前一个元素，a:p的后一个元素
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            //先把p的后一个元素指向空    
            p.after = null;
            //1 这里是把e元素拿出来
            //2 放到尾部
            //我们是从更新key-value那里跟踪过来的。所以e的前后元素早已经绑定完成
            //b：e的前一个元素
            //已经绑定前后元素的e的前元素为空说明e是第一个元素
            //把后一个元素设置为头
            if (b == null)
                head = a;
            else
                //这里是正常情况，前一个元素的后一个是后一个元素
                b.after = a;
            //上面完成后正方向e已经消息了
            //开始从反方向拿出移除e    
            //下一个不为null,是正常元素    
            if (a != null)
                //下个元素的前一个元素是e的前袁术
                a.before = b;
            else
                //a为null说明e是最后一个，直接移除就行了
                last = b;

            //上边把e元素从双向链表里拿出来了
            //开始放到尾部    
            //last是null说明这是个空链表  
            if (last == null)
                //当前元素给head
                //head = tail = p
                head = p;
            else {
                //与last绑定
                p.before = last;
                last.after = p;
            }
            //绑定完了把tail移到p元素上
            tail = p;
            //修改过链表结构后，这个就要++
            ++modCount;
        }
}
```
在HashMap的forEach，modCount看起来是为了在遍历链表的时候保持链表的结构是没变化，如果变化了个modCount+1了，会抛出异常。所以LinkedHashMap和hashMap一样不适合并发
```
@Override
    public void forEach(BiConsumer<? super K, ? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            // Android-changed: Detect changes to modCount early.
            for (int i = 0; (i < tab.length && mc == modCount); ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key, e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
}
```
3. get一个元素就是调用的HashMap的getNode。之后调用了afterNodeAccess。get的元素也会被放到尾部
```
public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
}
```

总结下：LinkedHashMap在HashMap.Node之上添加了前后元素的指向，升级为了双向链表。保存了head，tail来提供插入删除操作。accessOrder为true是以访问顺序排序.访问或者新建元素都会放到尾部
