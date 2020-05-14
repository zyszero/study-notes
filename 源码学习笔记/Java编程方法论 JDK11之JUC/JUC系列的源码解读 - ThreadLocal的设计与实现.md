# JUC系列的源码解读 - ThreadLocal的设计与实现

## ThreadLocal 是什么？



## 父子线程如何传递信息？

如源码所示：

```java
public class Thread implements Runnable {
    ...
    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;    
        
    /**
     * Initializes a Thread.
     *
     * @param g the Thread group
     * @param target the object whose run() method gets called
     * @param name the name of the new Thread
     * @param stackSize the desired stack size for the new thread, or
     *        zero to indicate that this parameter is to be ignored.
     * @param acc the AccessControlContext to inherit, or
     *            AccessController.getContext() if null
     * @param inheritThreadLocals if {@code true}, inherit initial values for
     *            inheritable thread-locals from the constructing thread
     */
    private Thread(ThreadGroup g, Runnable target, String name,
                   long stackSize, AccessControlContext acc,
                   boolean inheritThreadLocals) {
        ...
        Thread parent = currentThread();
        ...
        // inheritThreadLocals 默认是传true的
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        ...
    }	
}

public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    ...
    /**
     * Create the map associated with a ThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }    
}
```



## Thread .ThreadLocalMap 如何与Thread 关联在一起？

我们可以直接从源代码中一窥究竟：java.lang.ThreadLocal

通过查看源码，我们可以发现，ThreadLocal的构造函数什么都没做，单纯只是分配了一段内存。

Thread 与 ThreadLocal 之间的关联是发生在 get()、set(T value)方法里的，为什么呢？

因为并不是每个Thread都需要ThreadLocal，所以为了避免内存的浪费，关联发生在get()、set(T value)方法里。

查看核心方法：`void createMap(Thread t, T firstValue) `

`t.threadLocals = new ThreadLocalMap(this, firstValue);`

```java
public class ThreadLocal<T> {
    /**
     * Creates a thread local variable.
     * @see #withInitial(java.util.function.Supplier)
     */
    public ThreadLocal() {
    }
    
    public T get() {
        ...
        return setInitialValue();
    }
    
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value); // 核心
        }
		...
        return value;
    }
    
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);// 核心
        }
    }
    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }    
}
```



## 在同一个Thread中，不同的ThreadLocal如何获取值/设置值？

设计非常巧妙，从上一节的源码，我们可以看出，不同的ThreadLocal的值是通过ThreadLocal#set 方法，将各自的值通过  Thread.threadLocals 绑定在了一起。同一个Thread内获取不同ThreadLocal的值与同一个Thread内对不同的ThreadLocal进行设值，思路是一样的：

```java
public class ThreadLocal<T> {
    // 第一个ThreadLocal的 threadLocalHashCode=0，之后线性增长。
    // 原因是nextHashCode()和nextHashCode是静态的
    private final int threadLocalHashCode = nextHashCode();    
    
    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger(); // 【3】
    
    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */    
    private static final int HASH_INCREMENT = 0x61c88647;
 
    /**
     * Returns the next hash code.
     */    
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t); // 【1】
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this); // 【2】
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 初次获取值
        return setInitialValue();
    }
    
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value); // 核心
        }
		...
        return value;
    }    
    
    // 【1】
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 【4】
            map.set(this, value);
        } else {
            // 初次设值
            createMap(t, value);
        }
    }    
    
    static class ThreadLocalMap {    
        // 【2】
        private Entry getEntry(ThreadLocal<?> key) {
            // 【3】不同的ThreadLocal的threadLocalHashCode 是不同的值，且是线性增长的
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        } 
        
        /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }        
    }
}
```



## 为什么要慎用ThreadLocal？

```java
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }    
    }
```

由源码可知，如果`v`在外部由其他对象持有，就会存在对象逃逸，就有可能导致Entry不再是弱引用，无法被GC掉。