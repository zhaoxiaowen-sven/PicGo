# HashMap

# 一、数据结构

散列表，数组 + 链表，链表长度大于8时，升级为红黑树

# 二、原理介绍

## 2.1、数据结构

## 2.2、Hash算法

## 2.3、扩容规则

# 四、源码分析

## 4.1、常量

```java
// 默认的table大小
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
// table最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
// 负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 树化阈值，链表长度
static final int TREEIFY_THRESHOLD = 8;
// 树降级为链表阈值
static final int UNTREEIFY_THRESHOLD = 6;
// 树化的另一个约束阈值，hash表的所有元素个数超过64，2个参数要同时满足
static final int MIN_TREEIFY_CAPACITY = 64;
```

## 4.2、变量

```java
// hash表
transient Node<K,V>[] table;
// hash表中元素个数
transient int size;
// 结构修改次数（插入或删除元素时 + 1）
transient int modCount;
// 扩容阈值    
int threshold;
// 用于计算扩容阈值，threshold = capacity * loadFactor
final float loadFactor;
```





返回一个大于当前CAPACITY的值且这个数字是2的次方数

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



hash

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```



put

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1.table不存在时创建
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
   // 2.table的hash不存在时，头元素不存在，创建新Node
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 3.头元素相同,进行替换
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 4.相同hash的节点不存在，且当前桶位是链表情况
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //5.相同hash的table是链表结构
            for (int binCount = 0; ; ++binCount) {
                //5.1 遍历到最后也没有相同节点则插入
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 循环从0开始，满足这个条件时，已经循环了TREEIFY_THRESHOLD次
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                         // 链表长度大于TREEIFY_THRESHOLD时，进行树化
                        treeifyBin(tab, hash);
                    break;
                }
                // 5.3 找到相同值
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                // 赋值替换
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 别修改的次数，替换不算
    ++modCount;
    // 插入元素个数+1 > 
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```





resetSize