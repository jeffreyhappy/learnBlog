---
title: LinkedBlockingQueque初识
---

LinkedBlockingQueque是个线程安全的队列，先进先出，插入的时候从尾部添加，获取的时候从头部获取，里面维护了一个单项链表

然而我看了下他的出队操作  
1 拿到头（head）  
2 拿到第二个节点（first）
3 设置头的下一节点是自己，帮助GC。（对GC不太熟，猜测是他不持有别的的对象，别的对象也不持有它的引用了。这样等到GC的时候就可以标记清除）  
4  第二个节点设置为头  
5  取出第二个节点的item（就是我们添加的对象），在后面返回
6  把第二个节点的item设置为空

说好的出队是头部出队呢，这明明是第二个出队。
```
/**
     * Removes a node from head of queue.
     *
     * @return the node
     */
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
```
然后带着疑问看看Node的前世今生。
Node的定义
```

/**
   * Linked list node class.
   */
  static class Node<E> {
      E item;

      /**
       * One of:
       * - the real successor Node
       * - this Node, meaning the successor is head.next
       * - null, meaning there is no successor (this is the last node)
       */
      Node<E> next;

      Node(E x) { item = x; }
  }

```
一个Node就是单向链表的一个节点，item是该节点的内容，next指向下一个节点，而队列是对尾部进行插入操作，对头部进行读操作，所以只需要记录头和尾就可以了。LinkedBlockingQueque里面：

```
/**
 * Head of linked list.
 * Invariant: head.item == null
 */
 //链表的头，不变的量：head.item 永远是null
transient Node<E> head;

/**
 * Tail of linked list.
 * Invariant: last.next == null
 */

//链表的尾巴，不变的量：last.next 永远是null
private transient Node<E> last;

```
这里有个transient关键字，transient的意思是LinkedBlockingQueque实现了Serilizable接口，但是Head这个属性不会序列化。  

有个疑问：这样的话序列化之后再反序列化 head的值就没了。数据咋读呢？

上面的代码注释直接就说了 head.item 永远是null
看下head和last的创建、插入操作  

创建：在构造函数中初始化了head last ，head就是last且都是空item
```
public LinkedBlockingQueue(int capacity) {
       if (capacity <= 0) throw new IllegalArgumentException();
       this.capacity = capacity;
       last = head = new Node<E>(null);
   }
```
插入：在LinkedBlockingQueue的put方法中调用了enqueue

```
public void put(E e) throws InterruptedException {
  .....
  Node<E> node = new Node<E>(e);
  .....
  enqueue(node);
  .....
}

private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
}
```
添加的时候就是把尾部的next指向最新的node，然后移动last到最后。全程没有head的事  
last = last.next = node 拆开来是  
```
last.next = node  
last = last.next  
```
但是，当链表是空的时候是last=head 插入第一个node时  
```
last.next = node  =>    head.next = node  （node这个节点就是第一个插入值）
last = last.next  =>    last = head.next  
```
第一次插入完成后head和last分道扬镳，从这里就可以看出，head其实是起了个标志的作用，head的next才是第一个值所在的节点，所以dequeue才会读取head的后一个节点

##阻塞
blocking是阻塞的意思，看take（从头部获取） 和 put（插入尾部）可以了解下大概
```
  /** Lock held by take, poll, etc */
  private final ReentrantLock takeLock = new ReentrantLock();

  /** Wait queue for waiting takes */
  private final Condition notEmpty = takeLock.newCondition();


  public E take() throws InterruptedException {
          E x;
          int c = -1;
          final AtomicInteger count = this.count;
          final ReentrantLock takeLock = this.takeLock;
          takeLock.lockInterruptibly();
          try {
              while (count.get() == 0) {
                //为0等待新的put的信号
                  notEmpty.await();
              }
              x = dequeue();
              c = count.getAndDecrement();
              if (c > 1)
                  notEmpty.signal();
          } finally {
              takeLock.unlock();
          }
          if (c == capacity)
              signalNotFull();
          return x;
      }



    public void put(E e) throws InterruptedException {
            if (e == null) throw new NullPointerException();
            // Note: convention in all put/take/etc is to preset local var
            // holding count negative to indicate failure unless set.
            int c = -1;
            Node<E> node = new Node<E>(e);
            final ReentrantLock putLock = this.putLock;
            final AtomicInteger count = this.count;
            putLock.lockInterruptibly();
            try {
                /*
                 * Note that count is used in wait guard even though it is
                 * not protected by lock. This works because count can
                 * only decrease at this point (all other puts are shut
                 * out by lock), and we (or some other waiting put) are
                 * signalled if it ever changes from capacity. Similarly
                 * for all other uses of count in other wait guards.
                 */
                while (count.get() == capacity) {
                    notFull.await();
                }
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    notFull.signal();
            } finally {
                putLock.unlock();
            }
            if (c == 0)
                signalNotEmpty();
        }    
```
通过takeLock这个重入锁来实现阻塞等待
AtomicInteger是一个提供原子操作的integer对象，count是存储对象的总数。   
现在我们假设目前存储的对象为空，最大的容量为Integer.MAX_VALUE  
get()的时候count是为0的，那么就等待notEmpty这个condition的信号，get所在的线程挂起等待  
再看put的简化版本就一目了然了，如果e插入成功，count就增加一位，那么就通知下take锁的notEmpty这个condition。然后get线程就可以等待结束，继续运行读取对象。这样就实现了读的阻塞等待。
```
    public void put(E e) throws InterruptedException {
        ....
        int c = -1;
        ....
            c = count.getAndIncrement();
        ....
        if (c == 0)
            signalNotEmpty();
    }  

    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
```

写的阻塞等待也是一样，有个叫putLock和他的一个叫notFull的condition来控制。满了就通过notFull挂起等待读取操作，有线程执行get后通知notFull继续运行
