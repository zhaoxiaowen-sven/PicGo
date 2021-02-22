# LinkedHashMap

LinkedHashMap有5个概念，头节点head，尾节点tail，当前节点通常定义为 p，p的前驱节点为before，p的后继节点after。

- 空列表时head == tail == null；
- 注意head和tail只是标记首尾节点；
- head和tail的前驱节点不涉及任何业务逻辑；
- 只有一个元素时 head == tail ==p，p.before == p.after == null

```
/**
 * The head (eldest) of the doubly linked list.
 */
transient LinkedHashMapEntry<K,V> head;

/**
 * The tail (youngest) of the doubly linked list.
 */
transient LinkedHashMapEntry<K,V> tail;


/**
 * HashMap.Node subclass for normal LinkedHashMap entries.
 */
static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    LinkedHashMapEntry<K,V> before, after;
    LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

# 二、CRUD

## 1.put

`LinkedHashMap`并没有重写任何put方法。但是其重写了构建新节点的`newNode()`方法.
`newNode()`会在`HashMap`的`putVal()`方法里被调用，`putVal()`方法会在批量插入数据`putMapEntries(Map m, boolean evict)`或者插入单个数据`public V put(K key, V value)`时被调用。

`LinkedHashMap`重写了`newNode()`,在每次**构建新节点**时，通过`linkNodeLast(p);`将**新节点链接在内部双向链表的尾部**。

    //在构建新节点时，构建的是`LinkedHashMap.Entry` 不再是`Node`.
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
    //将新增的节点，连接在链表的尾部
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        //集合之前是空的
        if (last == null)
            head = p;
        else {//将新节点连接在链表的尾部
            p.before = last;
            last.after = p;
        }
    }