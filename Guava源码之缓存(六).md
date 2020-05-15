# Guava源码之缓存(六)

### 枚举类RemovalCause

#### 介绍

代表了一个缓存键值对被删除的原因

#### 方法

如果因为**被丢弃**而**自动删除**(原因既不是因为`EXPLICIT`也不是因为`REPLACED`)，则返回true

```java
abstract boolean wasEvicted();
```

#### 枚举项

##### EXPLICIT

用户已经手动将键值对删除。当用户调用`Cache#invalidate`、`Cache#invalidateAll(Iterator)`、`Cache#invalidateAll()`、`Map#remove()`、`ConcurrentMap#remove()`或者`Iterator#remove()`可能会返回此值

```java
EXPLICIT {
  @Override
  boolean wasEvicted() {
    return false;
  }
}
```

##### REPLACED

键值对并没有被删除，但是其值已经被用户替换成新值。当用户调用`Cache#put()`、`LoadingCache#refresh()`、`Map#put()`、`Map#putAll()`、`ConcurrentMap#replace(Object, Object)`或`ConcurrentMap#replace(Object, Object, Object)`可能会返回此值

```java
REPLACED {
  @Override
  boolean wasEvicted() {
    return false;
  }
}
```

##### COLLECTED

键值对由于其键或者值被垃圾收集器回收而被删除。当用户调用`CacheBuilder#weakKeys()`、`CacheBuilder#weakValues()`或者`CacheBuilder#softValues()`方法可能返回此值

```java
COLLECTED {
  @Override
  boolean wasEvicted() {
    return true;
  }
}
```

##### EXPIRED

标志着键值对到达过期时间。当调用`CacheBuilder#expireAfterWrite()`或者`CacheBuilder#expireAfterAccess()`方法可能返回此值

```java
EXPIRED {
  @Override
  boolean wasEvicted() {
    return true;
  }
}
```

##### SIZE

标志着键值对由于大小限制而被丢弃。当用户调用`CacheBuilder#maximumSize()`或者`CacheBuilder#maximumWeight()`方法可能返回此值

```java
SIZE {
  @Override
  boolean wasEvicted() {
    return true;
  }
}
```



### 接口RemovalListener

#### 介绍

这是一个当缓存中的键值对被删除时可以接收到删除通知的对象。当出现上述`EXPLICIT`、`REPLACED`、`COLLECTED`、`EXPIRED`或者`SIZE`这五种情况时，都可能发送一个通知

#### 方法

接口中仅定义了这一个方法。用于通知监听器删除的行为发生在过去某个时间点

```java
void onRemoval(RemovalNotification<K, V> notification);
```



### 类RemovalListeners

#### 介绍

定义了一组常用的删除监听器，此类使用final修饰，是一个最终类

#### 方法

方法也只有一个。这是一个异步的实现，内部定义了自定义的实现了`RemovalListener`接口的对象，并传入了一个listener对象和executor对象。实现了接口的`onRemoval()`方法，内部通过调用执行器executor来实现(其实执行器内部也还是通过listener对象来实现)，这个设计在上篇文章中也用到了

```java
public static <K, V> RemovalListener<K, V> asynchronous(
    final RemovalListener<K, V> listener, final Executor executor) {
  checkNotNull(listener);
  checkNotNull(executor);
  return new RemovalListener<K, V>() {
    @Override
    public void onRemoval(final RemovalNotification<K, V> notification) {
      executor.execute(new Runnable() {
        @Override
        public void run() {
          listener.onRemoval(notification);
        }
      });
    }
  };
}
```



### 类RemovalNotification

#### 介绍

这是一个最终类，代表了一条删除通知。当键值对被垃圾收集器回收后，可能返回的键和值为null。此类保存对键和值的强引用而不管缓存使用的键和值的引用强度如何。

#### 关于类

最终类。实现了Map的Entry接口，说明他其实是一个键值对。

```java
@Beta
@GwtCompatible
public final class RemovalNotification<K, V> implements Entry<K, V> {
```

#### 成员变量与构造器

不仅保存了删除的键和值，还保存了删除原因。(确实可以看做一条删除通知)

```java
@Nullable private final K key;
@Nullable private final V value;
private final RemovalCause cause;//删除原因
private static final long serialVersionUID = 0;

RemovalNotification(@Nullable K key, @Nullable V value, RemovalCause cause) {
  this.key = key;
  this.value = value;
  this.cause = checkNotNull(cause);
}
```

#### 内部方法

```java
//返回键值对被删除的原因
public RemovalCause getCause() {
  return cause;
}
//返回是否是被丢弃的
public boolean wasEvicted() {
  return cause.wasEvicted();
}

@Nullable @Override public K getKey() {
  return key;
}

@Nullable @Override public V getValue() {
  return value;
}
//提供了setValue()方法，但是会直接抛出异常，这有什么意义？
@Override public final V setValue(V value) {
  throw new UnsupportedOperationException();
}
//equals()方法内部比较的是键和值的内容
@Override public boolean equals(@Nullable Object object) {
  if (object instanceof Entry) {
    Entry<?, ?> that = (Entry<?, ?>) object;
    return Objects.equal(this.getKey(), that.getKey())
        && Objects.equal(this.getValue(), that.getValue());
  }
  return false;
}
//重写了hashCode()方法，是通过键和值的哈希值做异或的运算。在hashmap中也有类似运算，貌似约等于取余运算
@Override public int hashCode() {
  K k = getKey();
  V v = getValue();
  return ((k == null) ? 0 : k.hashCode()) ^ ((v == null) ? 0 : v.hashCode());
}
//重写了toString()方法，展示了键和值
@Override public String toString() {
  return getKey() + "=" + getValue();
}
```



### 总结

这几个类和接口看下来大概能够了解缓存的删除通知机制了，它定义了一个通知类并专门创建了一个枚举类用来展示删除原因，通知类还会返回键和值，可以说对于查看缓存的删除很有意义。