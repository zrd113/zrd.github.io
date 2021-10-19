---
layout: post
title: ThreadLocal源码分析
date: 2021-08-10
Author: zrd
tags: [Java]
toc: true
---

## 介绍

当我们在多线程环境下访问同一个共享变量，可能会出现线程安全的问题，为了保证线程安全，往往会在访问这个共享变量的时候加锁，以达到同步的效果，加锁虽然能够保证线程的安全，但是却增加了开发人员对锁的使用技能，如果锁使用不当，则会导致死锁的问题。而ThreadLocal能够做到在创建变量后，每个线程对变量访问时访问的是线程自己的本地变量。
如果我们创建了一个ThreadLocal变量，则访问这个变量的每个线程都会有这个变量的一个本地副本。如果多个线程同时对这个变量进行读写操作时，实际上操作的是线程自己本地内存中的变量，从而避免了线程安全的问题。

## 原理

首先我们看下 Thread 类的源码属性：
```
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
由Thread类的源码可以看出来，每个线程的本地变量不是存放在ThreadLocal实例里面的，而是存放在调用线程的threadLocals变量里面的，接下来，我们分析下ThreadLocal类的set()、get()和remove()方法的实现逻辑。

### set

```
public void set(T value) {
    //获取当前线程
    Thread t = Thread.currentThread();
    //以当前线程为Key，获取ThreadLocalMap对象
    ThreadLocalMap map = getMap(t);
    //获取的ThreadLocalMap对象不为空
    if (map != null)
        //设置value的值
        map.set(this, value);
    else
        //获取的ThreadLocalMap对象为空，创建Thread类中的threadLocals变量
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### get

```
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程的threadLocals成员变量
    ThreadLocalMap map = getMap(t);
    //获取的threadLocals变量不为空
    if (map != null) {
        //返回本地变量对应的值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //初始化threadLocals成员变量的值
    return setInitialValue();
}

private T setInitialValue() {
    //调用初始化Value的方法
    T value = initialValue();
    Thread t = Thread.currentThread();
    //根据当前线程获取threadLocals成员变量
    ThreadLocalMap map = getMap(t);
    if (map != null)
        //threadLocals不为空，则设置value值
        map.set(this, value);
    else
        //threadLocals为空,创建threadLocals变量
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```

### remove

```
public void remove() {
    //根据当前线程获取threadLocals成员变量
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        //threadLocals成员变量不为空，则移除value值
        m.remove(this);
}
```

## 不可传递性

使用ThreadLocal存储本地变量不具有传递性，也就是说，同一个ThreadLocal在父线程中设置值后，在子线程中是无法获取到这个值的，我们可以使用InheritableThreadLocal来解决这个问题。

我们看下InheritableThreadLocal源码
```
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    public InheritableThreadLocal() {}

    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

由源代码可知，当调用ThreadLocal的set()方法时，创建的是当前Thread线程的inheritableThreadLocals成员变量而不再是threadLocals成员变量。
这里，我们需要思考一个问题：InheritableThreadLocal类的childValue()方法是何时被调用的呢？这就需要我们来看下Thread类的构造方法了，其中有下面这段逻辑：

```
if (inheritThreadLocals && parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```
```
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}

private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (Entry e : parentTable) {
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                //调用重写的childValue方法
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

可以看到，如果父线程创建子线程，在Thread类的构造函数中会把父线程中的inheritableThreadLocals变量里面的本地变量复制一份保存到子线程的inheritableThreadLocals变量中。

