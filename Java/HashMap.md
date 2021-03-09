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