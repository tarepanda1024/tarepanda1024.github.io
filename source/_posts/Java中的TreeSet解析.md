---
title: Java中的TreeSet使用过程中的踩坑记录
tags: Java
categories: Java
date: 2018-10-28 16:06:03
---

> 最近在使用TreeSet的过程中遇到一个问题，发现对元素覆写compareTo方法后部分元素被吞掉了。于是查了相关资料并看了一下源码，在此记录一下使用过程中的注意事项

原始需求是对一系列规则根据优先级进行排序，从而构成规则链，规则的信息如下：
```java

/**
 * @author 刘晓东
 */
public class RuleInfo implements Comparable {
    private long id;
    private int priority;//规则优先级
    ......
     public boolean equals(Object obj) {
        if (obj instanceof RuleInfo) {
            RuleInfo ruleInfo = (RuleInfo) obj;
            return this.id == ruleInfo.id;
        }
        return false;
    }

    @Override
    public int compareTo(Object o) {
        if (o instanceof RuleInfo) {
            RuleInfo ruleInfo = (RuleInfo) o;
            return this.getPriority() - ruleInfo.getPriority();
        }
        return 0;
    }


}

```
在插入几条具有相同优先级的规则后，发现treeset中只会保留一条规则，其他相同优先级的规则都被吞掉了。然后看了一下TreeSet的源码，发现注释中有这么几句话。
```
 * <p>Note that the ordering maintained by a set (whether or not an explicit
 * comparator is provided) must be <i>consistent with equals</i> if it is to
 * correctly implement the {@code Set} interface.  (See {@code Comparable}
 * or {@code Comparator} for a precise definition of <i>consistent with
 * equals</i>.)  This is so because the {@code Set} interface is defined in
 * terms of the {@code equals} operation, but a {@code TreeSet} instance
 * performs all element comparisons using its {@code compareTo} (or
 * {@code compare}) method, so two elements that are deemed equal by this method
 * are, from the standpoint of the set, equal.  The behavior of a set
 * <i>is</i> well-defined even if its ordering is inconsistent with equals; it
 * just fails to obey the general contract of the {@code Set} interface.
```
翻译过来也就是：Set中各个元素是以equal方法来判断两个元素是否一样，而在TreeSet中是以compareTo方法来进行排序并判断两个元素是否一样的，覆写equal方法在TreeSet中不起作用。因此上面的compareTo方法可以改写为:
```java
 @Override
    public int compareTo(Object o) {
        if (o instanceof RuleInfo) {
            RuleInfo ruleInfo = (RuleInfo) o;
            if (this.getId() == ruleInfo.getId()) {
                return 0;
            }
            int ret = this.getPriority() - ruleInfo.getPriority();
            if (ret == 0) {
                return (int) (this.id - ruleInfo.id);
            } else {
                return ret;
            }
        }
        return 0;
    }
```
接下来看一下TreeSet的add方法是如何实现的。  
TreeSet是用TreeMap实现的，其中的add方法直接使用的TreeMap的put方法
```java
  public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }
```
TreeMap的put方法为:
```java
    public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }
```
可以看到如果元素自定义了比较器，那么当返回比较值为0时，直接将值覆盖了，从而相同优先级的元素只会保留一个。所以使用TreeSet自定义比较器时一定要注意这一点。
