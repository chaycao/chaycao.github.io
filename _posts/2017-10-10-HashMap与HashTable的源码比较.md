---
layout:     post
title:     HashMap与HashTable的源码比较
date:       2017-10-10
author:     ChayCao
header-img: img/post-bg-2015.jpg 
catalog: true
tags:  Java
---


## 一、前言

一直都知道HashMap是常考的，所以今天把HashMap的源码看了一遍，然后又想起了HashTable，便想做一个比较。

```java
HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

HashMap继承于AbstractMap，Hashtable继承于Dictionary。

```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable 
```

通过查阅jdk，有下面这句话：

```
NOTE: This class is obsolete. New implementations should implement the Map interface, rather than extending this class.
```

得知Dictionary类已经过时了，而推荐实现Map接口。

而Hashtable也是一个过时的集合类，从jdk1.0开始就存在了。在Java 4中被重写了，实现了Map接口，所以自此以后也成了java集合框架的一部分。

## 二、主要区别

### 1. 线程安全性

HashMap是线程不安全的，Hashtable是线程安全的。Hashtable的线程安全是用**synchronized**关键字实现的。

```java
public synchronized int size();
public synchronized boolean isEmpty();
public synchronized V get(Object key);
public synchronized V put(K key, V value);
```

以上方法是Hashtable源码里的，其实和HashMap几乎一样，只是多了synchronized关键字。则Hashtable是线程安全的，多个线程可以共享一个Hashtable。而如果没有正确同步的话，多个线程不能共享HashMap。Java 5 提供了ConcurrentHashMap，它是Hashtable的替代，比Hashtable的扩展更好

### 2. null的键和值

HashMap是可以接受null的键和值的，而Hashtable则不允许。

先从Hashtable的put()方法讲起：

```java
public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
```

在put()方法里，首先会对value进行检查，若为null，则抛出NullPointerException。对于key，则直接使用key.hashcode()，若key为null，则仍会抛出NullPointerException。

下面再看下HashMap里的put()实现：

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
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
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

put()会调用putVal()，而putVal()中则不会对value做null的检查，再看看key，是如果获得null值的key的hash值。这是用到了HashMap里的hash()。

```java
static final int hash(Object key) {
  int h;
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

很明显，若key为null，则hash值用0。这便是HashMap如何支持null值的key和value。

### 3. 速度

Hashtable是线程安全的，所以在单线程环境下它比HashMap要慢。如果不需要同步，只需要单一线程，那么使用HashMap性能要好过Hashtable。



## 三、让HashMap同步

HashMap可以通过下面的语句进行同步：

```java
Map m = Collections.synchronizeMap(hashMap);
```

实现方法仍然是给方法加上synchronized关键字。

```java
public int size() {
            synchronized (mutex) {return m.size();}
        }
        public boolean isEmpty() {
            synchronized (mutex) {return m.isEmpty();}
        }
        public boolean containsKey(Object key) {
            synchronized (mutex) {return m.containsKey(key);}
        }
        public boolean containsValue(Object value) {
            synchronized (mutex) {return m.containsValue(value);}
        }
        public V get(Object key) {
            synchronized (mutex) {return m.get(key);}
        }

        public V put(K key, V value) {
            synchronized (mutex) {return m.put(key, value);}
        }
        public V remove(Object key) {
            synchronized (mutex) {return m.remove(key);}
        }
        public void putAll(Map<? extends K, ? extends V> map) {
            synchronized (mutex) {m.putAll(map);}
        }
        public void clear() {
            synchronized (mutex) {m.clear();}
        }

        private transient Set<K> keySet;
        private transient Set<Map.Entry<K,V>> entrySet;
        private transient Collection<V> values;

        public Set<K> keySet() {
            synchronized (mutex) {
                if (keySet==null)
                    keySet = new SynchronizedSet<>(m.keySet(), mutex);
                return keySet;
            }
        }
```

## 四、疑惑

看别人的文章里说，HashMap和HashTable的迭代器是不同的，HashMap用的是iterator是fail-fast的，而HashTable用的是enumerator不是fail-fast的。但我看1.8jdk里的Hashtable的enumerator 如下：

```java
private class Enumerator<T> implements Enumeration<T>, Iterator<T>
```

是有实现iterator接口的，也就是Hashtable其实是iterator和Enumeration都有支持的。
后续再补充吧。对于fail-fast还是没透彻理解。