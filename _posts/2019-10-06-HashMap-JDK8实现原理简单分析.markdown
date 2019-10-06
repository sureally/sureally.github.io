---
layout: article
title: HashMap-JDK8实现原理简单分析
date: 2019/10/6 22:00
categories: Java
---

# 扰动函数
```java
    static final int hash(Object key) {
        int h;
        // 因为hash表多是取低位取余，为了降低hash碰撞概率，这里将低16位与高16位异或后放在后16位
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
# 初始化
```java
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
        // 一开始认为应该是 tableSizeFor(initialCapacity) * loadFactor
        // 但是其实table的初始化是放在了put中完成的，懒加载的意味
        this.threshold = tableSizeFor(initialCapacity);
    }
    
    ...
    // 在方法resize里面，初始化threshold
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    ...
```

## tableSizeFor
```java
    static final int tableSizeFor(int cap) {
        // 先减1防止已经是2的次方了的情况再次扩容。
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
# 扩容操作
- 先做put操作后，再判断是否扩容。
- 元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置
- 单节点/红黑树/链表三种情况分别扩容
```java
    // 完成table初始化或者double扩容操作
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 极端情况，无法扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            // 手动提供的参数
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 默认参数
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        // 创建新的Node数组
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 原数组指向新的数组
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        // 单个节点的情况，重新计算位置（仅仅是取余）
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 红黑树的情况
                        // TreeNode 是 static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 链表的情况
                        // 现将链表中的每个节点再次分配的未知缓存起来，然后分配，避免乱序
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
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
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

红黑树结构扩容细节
```java
        final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                // 元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
```

# 线程安全性
线程不安全的。  

- JDK7主要的不安全
    下面这种情况出现与JDK7中，JDK8中新老数据不存在引用关系所以不会出现
    >多线程同时put时，如果同时触发了rehash操作，会导致HashMap中的链表中出现循环节点，进而使得后面get的时候，会死循环
- JDK8中的主要线程不安全
    >键值对丢失。多线程情况下，可能会有多个线程进入 resize 方法，假设第一个线程进入了 resize 方法，在处理链表时会先记录一下，然后直接将对应的旧哈希表数组中的链表置 null，此时第二个线程进来了，因为上一个线程已经把链表置 null 了，线程 2 判定当前桶位置上没有键值对，如果线程 2 返回的哈希表数组覆盖了线程 1 的哈希表数组，就会丢失一部分因线程 1 置 null 的键值对。
    
    ```java
    ...
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
            
            ...
        
    ```

# 参考文档
- https://www.jianshu.com/p/8823ef4a76f4
- https://blog.csdn.net/login_sonata/article/details/76598675
- https://blog.csdn.net/codejas/article/details/85056606