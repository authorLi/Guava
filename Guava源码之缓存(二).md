# Guava源码之缓存(二)

### 接口LoadingCache

#### 介绍

这是一个半持久化的键值对映射，不同的是，**它的值由缓存`自动的`加载到缓存里面知道键值对被删除。**

实现了这个接口的子类被希望是**线程安全的**，并可以在并发环境下安全的访问

#### 关于接口

```java
@GwtCompatible
public interface LoadingCache<K, V> extends Cache<K, V>, Function<K, V> {
```

此接口是`Cache`接口的实现，继承了Cache

#### 接口中的方法

##### get()

```java
V get(K key) throws ExecutionException;
```

此方法会返回传入的键对应的值，如果缓存中不存在对应的值将加载此值，在加载完成之前不会影响与该键值对关联的键的可见性。如果此时另一个`get()`或`getUnchecked()`调用当前正在加载的键来获取值，那么只需等待该线程完成加载并返回加载的值即可。

缓存的加载是通过`CacheLoader`的`load()`方法执行的，加载完毕以后将调用`Cache.asMap().putIfAbsent()`来将新值添加到缓存中，如果在加载完成之后查到已有值与此键相关联，那么会删除之，使用新加载的值代替之

如果缓存加载器已知不会抛出checked exception，那么最好使用`getUnchecked()`方法

##### getUnchecked()

```java
V getUnchecked(K key);
```

与get()方法类似，不同的是它应该仅当在缓存加载器不抛出checked exception的时候调用。

需要注意的是，此方法会将内部的unchecked exception转换成checked exception，它不应该和会抛出checked exception的缓存加载器一起使用，那种情况下应该使用`get()`方法

##### getAll()

```java
ImmutableMap<K, V> getAll(Iterable<? extends K> keys) throws ExecutionException;
```

此方法会返回一个所有键值对映射的map，它可能会直接查询返回或者先创建再返回，但是它永远不会返回空的键或者值。

由缓存加载器加载的缓存将向`CacheLoader.loadAll()`发送单个请求以请求缓存中尚不存在的所有键。`CacheLoader.loadAll()`返回的所有键值对都将被添加到缓存中，并覆盖以前的值(如果有)。

如果`CacheLoader.loadAll()`返回了空键或空的映射，或无法为每个请求的键返回键值对，那么此方法将引发异常

##### apply()

```java
@Deprecated
@Override
V apply(K key);
```

该方法已被舍弃，不建议使用

##### refresh()

```java
void refresh(K key);
```

此方法为指定的键加载一个新值，可能是异步的。当新值加载完成之前，如果调用`get()`方法依旧会返回老值，除非老值被丢弃了。如果新值加载成功它将代替老值存入到缓存中；如果加载新值的过程出现了异常，那么将依旧使用老值，并且会将此次的异常记录在Logger日志中

如果当前有与键映射的值，那么加载新值时使用的是`CacheLoader.reload()`方法；否则直接调用`CacheLoader.load()`方法来加载。仅当`CacheLoader.reload()`被异步重写时加载的过程才是异步的

如果在加载的过程中发现有其他线程在对该键进行refresh，那么本次加载什么也不会做。

如果与此缓存相关联的缓存加载器异步的执行加载，那么此方法也许会在加载完成之前返回

##### asMap()

```java
@Override
ConcurrentMap<K, V> asMap();
```

重写了父接口的方法，需要注意的是尽管视图是可以更改的，但是返回的map上的任何方法都不会导致自动加载键值对

### 抽象类AbstractLoadingCache

#### 介绍

此抽象类提供了Cache接口的基本实现。它的子类只需要继承它并实现`get(Object)`和`getIfPresent()`方法即可。

`getUnchecked()`、`get(Object, Callable)`和`getAll()`都是根据get()方法实现的；`getAllPresent()`是根据getIfPresent()实现的；`putAll()`是根据put()实现的；`invalidateAll(Iterator)`是根据invalidate()实现的；`cleanUp()`依旧没有做任何操作；其他的方法都会直接抛出`UnsupportedOperationException`异常

#### 关于抽象类

```java
@Beta
public abstract class AbstractLoadingCache<K, V>
    extends AbstractCache<K, V> implements LoadingCache<K, V> {
```

此抽象类继承了`AbstractCache`抽象类，实现了`LoadingCache`接口

#### 方法

##### getUnchecked()

```java
@Override
public V getUnchecked(K key) {
  try {
    return get(key);
  } catch (ExecutionException e) {
    throw new UncheckedExecutionException(e.getCause());
  }
}
```

内部直接调用`LoadingCache#get()`方法来实现并使用try-catch包裹，如果发生异常则统一抛出UncheckedExecutionException异常

##### getAll()

```java
@Override
public ImmutableMap<K, V> getAll(Iterable<? extends K> keys) throws ExecutionException {
  Map<K, V> result = Maps.newLinkedHashMap();
  for (K key : keys) {
    if (!result.containsKey(key)) {
      result.put(key, get(key));
    }
  }
  return ImmutableMap.copyOf(result);
}
```

简单的遍历传入的迭代器，然后根据keys调用`LoadingCache#get()`获取到键，最终存储到一个ImmutableMap中

##### apply()

```java
@Override
public final V apply(K key) {
  return getUnchecked(key);
}
```

实际上内部调用了本类的`getUnchecked()`方法

##### refresh()

```java
@Override
public void refresh(K key) {
  throw new UnsupportedOperationException();
}
```

被抽象类不支持此方法，直接抛出`UnsupportedOperationException`异常

### 总结

LoadingCache接口继承了Cache接口在其基础上，扩展了通过缓存加载器来加载映射的值。而AbstractLoadingCache的设计是在接口的实现基础上提供了成本更低的抽象类。