# JUC系列的源码解读 - Unsafe原理简谈以及实际应用

## 核心原理

基于offset（偏移量）来时实现的。JVM加载class文件时，Field在此对象对应的内存段上的相对位置便可以确定且之后都不会变，即offset值是固定的，所以当为一个对象分配一段内存后，我们可以根据对象的首地址 + offset值，算出Field的内存地址，之后便可以基于内存地址对Field的值直接进行操作。

通俗理解：class文件相当于模具，模具上每个部位都是固定好的，即相对于模具上的某个点，各部位的相对位置是固定的，即当JVM加载class文件时，offset值就已经固定好了。后续创建对象，分配内存，相当于用模具创建成品，便可根据成品本身的起始点 + offset，算出各部位所在位置。

## 应用案例

**java.util.concurrent.atomic.AtomicLong**

```java
public class AtomicLong extends Number implements java.io.Serializable {
	...
    private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
    // offset值
    private static final long VALUE = U.objectFieldOffset(AtomicLong.class, "value");

    private volatile long value;  
    ...
    public final long getAndSet(long newValue) {
        return U.getAndSetLong(this, VALUE, newValue);
    }        
}

public final class Unsafe {
    @HotSpotIntrinsicCandidate
    public final long getAndSetLong(Object o, long offset, long newValue) {
        long v;
        do {
            v = getLongVolatile(o, offset);
        } while (!weakCompareAndSetLong(o, offset, v, newValue));
        return v;
    }
    
    @HotSpotIntrinsicCandidate
    public final boolean weakCompareAndSetLong(Object o, long offset,
                                               long expected,
                                               long x) {
        return compareAndSetLong(o, offset, expected, x);
    }

    @HotSpotIntrinsicCandidate
    public final native boolean compareAndSetLong(Object o, long offset,
                                                  long expected,
                                                  long x);
    
}
```



CAS的底层实现：unsafe.cpp

```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetLong(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jlong e, jlong x)) {
  oop p = JNIHandles::resolve(obj);
  if (p == NULL) {
    volatile jlong* addr = (volatile jlong*)index_oop_from_field_offset_long(p, offset);
    return RawAccess<>::atomic_cmpxchg(x, addr, e) == e;
  } else {
    assert_field_offset_sane(p, offset);
    return HeapAccess<>::atomic_cmpxchg_at(x, p, (ptrdiff_t)offset, e) == e;
  }
} UNSAFE_END
    
static inline void* index_oop_from_field_offset_long(oop p, jlong field_offset) {
  assert_field_offset_sane(p, field_offset);
  jlong byte_offset = field_offset_to_byte_offset(field_offset);

  if (p != NULL) {
    p = Access<>::resolve(p);
  }

  if (sizeof(char*) == sizeof(jint)) {   // (this constant folds!)
    return (address)p + (jint) byte_offset;
  } else {
    return (address)p +        byte_offset;
  }
}        
```

reactor.util.concurrent.MpscLinkedQueue

```java
final class MpscLinkedQueue<E> extends AbstractQueue<E> implements BiPredicate<E, E> {
	private volatile LinkedQueueNode<E> producerNode;

	private final static AtomicReferenceFieldUpdater<MpscLinkedQueue, LinkedQueueNode> PRODUCER_NODE_UPDATER
			= AtomicReferenceFieldUpdater.newUpdater(MpscLinkedQueue.class, LinkedQueueNode.class, "producerNode");

	private volatile LinkedQueueNode<E> consumerNode;
	private final static AtomicReferenceFieldUpdater<MpscLinkedQueue, LinkedQueueNode> CONSUMER_NODE_UPDATER
			= AtomicReferenceFieldUpdater.newUpdater(MpscLinkedQueue.class, LinkedQueueNode.class, "consumerNode");
    ...
}

public abstract class AtomicReferenceFieldUpdater<T,V> {
    @CallerSensitive
    public static <U,W> AtomicReferenceFieldUpdater<U,W> newUpdater(Class<U> tclass,
                                                                    Class<W> vclass,
                                                                    String fieldName){
        return new AtomicReferenceFieldUpdaterImpl<U,W>
            (tclass, vclass, fieldName, Reflection.getCallerClass());
    }

    private static final class AtomicReferenceFieldUpdaterImpl<T,V>
        extends AtomicReferenceFieldUpdater<T,V> {
        private static final Unsafe U = Unsafe.getUnsafe();
        private final long offset;
        /**
         * if field is protected, the subclass constructing updater, else
         * the same as tclass
         */
        private final Class<?> cclass;
        /** class holding the field */
        private final Class<T> tclass;
        /** field value type */
        private final Class<V> vclass;
        
        AtomicReferenceFieldUpdaterImpl(final Class<T> tclass,
                                        final Class<V> vclass,
                                        final String fieldName,
                                        final Class<?> caller) {
            final Field field;
            final Class<?> fieldClass;
            final int modifiers;
            try {
                field = AccessController.doPrivileged(
                    new PrivilegedExceptionAction<Field>() {
                        public Field run() throws NoSuchFieldException {
                            return tclass.getDeclaredField(fieldName);
                        }
                    });
                modifiers = field.getModifiers();
                sun.reflect.misc.ReflectUtil.ensureMemberAccess(
                    caller, tclass, null, modifiers);
                ClassLoader cl = tclass.getClassLoader();
                ClassLoader ccl = caller.getClassLoader();
                if ((ccl != null) && (ccl != cl) &&
                    ((cl == null) || !isAncestor(cl, ccl))) {
                    sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
                }
                fieldClass = field.getType();
            } catch (PrivilegedActionException pae) {
                throw new RuntimeException(pae.getException());
            } catch (Exception ex) {
                throw new RuntimeException(ex);
            }

            if (vclass != fieldClass)
                throw new ClassCastException();
            if (vclass.isPrimitive())
                throw new IllegalArgumentException("Must be reference type");

            if (!Modifier.isVolatile(modifiers))
                throw new IllegalArgumentException("Must be volatile type");

            // Access to protected field members is restricted to receivers only
            // of the accessing class, or one of its subclasses, and the
            // accessing class must in turn be a subclass (or package sibling)
            // of the protected member's defining class.
            // If the updater refers to a protected field of a declaring class
            // outside the current package, the receiver argument will be
            // narrowed to the type of the accessing class.
            this.cclass = (Modifier.isProtected(modifiers) &&
                           tclass.isAssignableFrom(caller) &&
                           !isSamePackage(tclass, caller))
                          ? caller : tclass;
            this.tclass = tclass;
            this.vclass = vclass;
            this.offset = U.objectFieldOffset(field);
        }        
    }
}
```



java.util.concurrent.locks.AbstractQueuedSynchronizer

JDK8采用Unsafe

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;

    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));

        } catch (Exception ex) { throw new Error(ex); }
    }
}
```



java.util.concurrent.locks.AbstractQueuedSynchronizer

JDK9以后，换了另一种方式：`VarHandle`

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    ...
    // VarHandle mechanics
    private static final VarHandle STATE;
    private static final VarHandle HEAD;
    private static final VarHandle TAIL;

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            STATE = l.findVarHandle(AbstractQueuedSynchronizer.class, "state", int.class);
            HEAD = l.findVarHandle(AbstractQueuedSynchronizer.class, "head", Node.class);
            TAIL = l.findVarHandle(AbstractQueuedSynchronizer.class, "tail", Node.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }

        // Reduce the risk of rare disastrous classloading in first call to
        // LockSupport.park: https://bugs.openjdk.java.net/browse/JDK-8074773
        Class<?> ensureLoaded = LockSupport.class;
    }

}
```



