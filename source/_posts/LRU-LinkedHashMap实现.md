---
title: LRU-LinkedHashMap实现
date: 2016-07-21 14:25:50
tags:
---

## LRU
java中可以使用LinkedHashMap实现LRU,比如mybatis实现LruCahce 代码片段

```java

 	keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
      private static final long serialVersionUID = 4267176411845948333L;

      //重写改方法,实现移除策略,返回true表示移除,默认返回false
      @Override
      protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
        boolean tooBig = size() > size;
        if (tooBig) {
          eldestKey = eldest.getKey();
        }
        return tooBig;
      }
    };

```

LinkedHashMap继承HashMap,有着相同的访问方法,不同的是它还提供插入顺序迭代器,LRU(最近访问的元素放在链表尾部)  


## LinkedHashMap实现
LinkedHashMap使用了双向链表(Entry作为节点,继承HashMap.Node单向链表节点),也就是说HashMap中的元素之间通过双向  
链表连接了,元素之间多了一种关联

### put
LinkedHashMap添加元素,执行HashMap.put,假如没有Hash冲突,那么直接调用newNode,LinkedHashMap重写改方法,创建节点  
LinkedHashMap.Entry而不是HashMap.Node,将新建的节点放在双向链表的尾部,既保证了插入顺序,又保证了LRU  

在放入元素后会调用空白方法afterNodeInsertion,LinkedHashMap重写该方法,并根据removeEldestEntry()方法的返回值  
来决定是否移除双向联调的head节点,LRU就是通过移除head来实现,用户需要重写removeEldestEntry()来决定在什么条件下  
移除head, 返回true表示移除, false表示不移除  

### get
LinkedHashMap访问元素,执行重写的方法,在这里如果accessOrder为true, 那么需要调整双向链表,将访问的元素放到链表尾    
执行afterNodeAccess(...)代码片段如下
```java
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)//e为head, e.before为null, 边界
                head = a;//head指向的下一个元素为head
            else
                b.after = a;//e的上一个节点指向e的下一个节点
            if (a != null)//如果p不是tail
                a.before = b;//e的下一个节点指向e的上一个节点
            else
                last = b;//a == null , 这b为新的tail
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;//将新插入的e放到双向链表末尾
            }
            tail = p;
            ++modCount;
        }
    }

```
移动链表元素,要考虑边界条件比较麻烦

### remove
调用HashMap.remove, 移除双向链表中的一个节点


## 总结
LinkedHashMap采用双向链表实现LRU和插入顺序的迭代,在插入元素,访问元素维护双向链表,从而具备LRU功能  

## LRU实现
使用双向链表:  
1.插入节点,将新插入的节点放到链表尾部,如果超过链表最大长度,移除第一个节点  
2.访问元素,将被访问的节点放到链表尾部  

在使用LRU时,因为链表的查找速度是O(N),当N很大时效率低下,LinkedHashMap继承HashMap,令查找效率在没有hash冲突为O(1)  



