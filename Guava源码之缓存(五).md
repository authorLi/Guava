# Guava源码之缓存(五)

### CacheLoader

#### 介绍

主要作用就是根据传入的键，检索或计算值以用于填充一个`LoadingCache`。大多数情况下只需要去重写`load()`方法即可，其他方法可以视情况来重写。

关于用法可以参考：

```java
CacheLoader<Key, Graph> loader = new CacheLoader<Key, Graph>(){
  public Graph load(Key key) throws AnyException{
    return createExpensive(key);
  }
}；
  
LoadingCache<Key, Graph> cache = CacheBuilder.newBuilder().build(loader);
```

#### 关于类

```java
@GwtCompatible(emulated = true)
public abstract class CacheLoader<K, V> {
```

它是个抽象类，前面也说了，只需实现`load()`方法就好，其他的按需求实现

#### 重要方法

##### load()

根据相应的key**计算**或**检索**与其对应的值

```java
public abstract V load(K key) throws Exception;
```

##### reload()

根据缓存中已存在的一个键计算或者检索一个新值。当通过`CacheBuilder#refreshAfterWrite()`来刷新现有缓存的键值对或调用`LoadingCache#refresh()`方法刷新缓存时就会调用到此方法

```java
@GwtIncompatible("Futures")
public ListenableFuture<V> reload(K key, V oldValue) throws Exception {
  checkNotNull(key);
  checkNotNull(oldValue);
  return Futures.immediateFuture(load(key));//这里构建了一个Futures工具类的内部类ImmediateSuccessfulFuture对象
}
```

##### loadAll()

计算或者检索键对应的值。这个方法会被`LoadingCache#getAll()`方法调用。

如果此方法返回的Map中没有包括所有请求中包含的键,那么只会将返回中请求包括的那些键的键值对添加到缓存中,然后**此方法会抛出一个异常**；如果返回的Map中包含了请求中没有的键，那么**返回的所有的这些键值对都将被添加到缓存中**，但是**调用它的`getAll()`(前面说了getAll()方法会调用本方法)方法仅仅会返回请求的键中包含的这些键值对**

> 一句话：却或者多的情况都会添加到缓存，但缺的会报错，多的也不会多返回

```java
public Map<K, V> loadAll(Iterable<? extends K> keys) throws Exception {
  // This will be caught by getAll(), causing it to fall back to multiple calls to
  // LoadingCache.get
  throw new UnsupportedLoadingOperationException();
}
```

##### from(Function<K, V>)

此方法会返回一个基于现存函数实例的缓存加载程序。

```java
@Beta
public static <K, V> CacheLoader<K, V> from(Function<K, V> function) {
  return new FunctionToCacheLoader<K, V>(function);
}
```

##### 内部类 FunctionToCacheLoader

可以看到此类继承了其外部类`CacheLoader`并实现了序列化接口。内部声明了一个不可变对象computingFunction并通过构造函数进行赋值。另外重写了父类的`load()`方法，返回了经由函数处理后的key的结果。

```java
private static final class FunctionToCacheLoader<K, V>
    extends CacheLoader<K, V> implements Serializable {
  private final Function<K, V> computingFunction;

  public FunctionToCacheLoader(Function<K, V> computingFunction) {
    this.computingFunction = checkNotNull(computingFunction);
  }

  @Override
  public V load(K key) {
    return computingFunction.apply(checkNotNull(key));
  }

  private static final long serialVersionUID = 0;
}
```

##### from(Supplier<**V**>)

此方法会返回一个基于现存供应者实例实现的缓存加载程序。

```java
@Beta
public static <V> CacheLoader<Object, V> from(Supplier<V> supplier) {
  return new SupplierToCacheLoader<V>(supplier);
}
```

##### 内部类SupplierToCacheLoader

整体设计与`FunctionToCacheLoader`相同，不再赘述

```java
private static final class SupplierToCacheLoader<V>
    extends CacheLoader<Object, V> implements Serializable {
  private final Supplier<V> computingSupplier;

  public SupplierToCacheLoader(Supplier<V> computingSupplier) {
    this.computingSupplier = checkNotNull(computingSupplier);
  }

  @Override
  public V load(Object key) {
    checkNotNull(key);
    return computingSupplier.get();
  }

  private static final long serialVersionUID = 0;
}
```

##### asyncReloading()

此方法返回了一个包装了`Loader`的`CacheLoader`，并使用传入的`executor`调用了`CacheLoader#reload()`方法。

就是返回了一个自定义的CacheLoader对象，重写了三个load方法，都是通过传入的loader对象来实现。其中`reload()`方法使用了Guava工具类中的`ListenableFutureTask`来实现异步任务，本质的加载还是通过传入的loader对象来执行，但是此异步任务是通过传入的executor来执行

```java
@Beta
@GwtIncompatible("Executor + Futures")
public static <K, V> CacheLoader<K, V> asyncReloading(final CacheLoader<K, V> loader,
    final Executor executor) {
  checkNotNull(loader);
  checkNotNull(executor);
  return new CacheLoader<K, V>() {
    @Override
    public V load(K key) throws Exception {
      return loader.load(key);
    }

    @Override
    public ListenableFuture<V> reload(final K key, final V oldValue) throws Exception {
      ListenableFutureTask<V> task = ListenableFutureTask.create(new Callable<V>() {
        @Override
        public V call() throws Exception {
          return loader.reload(key, oldValue).get();
        }
      });
      executor.execute(task);
      return task;
    }

    @Override
    public Map<K, V> loadAll(Iterable<? extends K> keys) throws Exception {
      return loader.loadAll(keys);
    }
  };
}
```

#### 两个自定义异常

第一个`UnsupportedLoadingOperationException`异常是一个最终类，直接集成了`UnsupportedOperationException`

第二个`InvalidCacheLoadException`异常也是一个最终类，但是他继承的是`RuntimeException`即运行时异常，内部还是直接调用了父类的构造器，没有其他变化

```java
static final class UnsupportedLoadingOperationException extends UnsupportedOperationException {}

public static final class InvalidCacheLoadException extends RuntimeException {
  public InvalidCacheLoadException(String message) {
    super(message);
  }
}
```



### 总结

CacheLoader是一个抽象类，它内部定义的三个load方法尤为重要。通过它内部定义的`asyncReloading()`可以看到它的一种实现，那就是可以通过异步创建任务的方式来重新加载。这样就给我们提供了一个模板，我们就可以照葫芦画瓢，写出更多更适合自己场景的CacheLoader实现类。它内部还定义了两个加载方式——通过函数计算或者提供者来确定加载值的方式，这都是可以参考的