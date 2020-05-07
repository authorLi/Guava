# Guava源码之缓存(一)

## 接口Cache

#### 介绍

它是一个半持久的键值对映射，可以使用`get(Object, Callable)`或者`put(Object, Object)`来添加键值对并把它们存储到缓存中直到我们手动的将其删除

实现了这个接口的子类被希望是**线程安全的**，并可以在高并发环境下安全的访问

#### 接口中定义的方法

##### getIfPresent()

```java
@Nullable
V getIfPresent(Object key);
```

当存在与键映射的值将返回该值，否则将返回null。

##### get()

```java
V get(K key, Callable<? extends V> valueLoader) throws ExecutionException;
```

在缓存中查找与键映射的值，如果不存在则使用valueLoader来创建与键映射的值，存入缓存中并返回。所以它是一定可以返回一个值的。并且在创建过程中不会影响该键对其他线程的可见性，如在创建过程中发生错误，则会抛出异常。

##### getAllPresent()

```java
ImmutableMap<K, V> getAllPresent(Iterable<?> keys);
```

根据传入的键获取对应的所有存在的映射的值。它会返回一个`ImmutableMap`，这个需要注意。

##### put()

```java
void put(K key, V value);
```

添加一个键值对到缓存中，如果缓存中存在与键映射的值，**那么旧的值将被覆盖**

##### putAll()

```java
void putAll(Map<? extends K,? extends V> m);
```

将传入的Map都添加到缓存中去,相当于对于每一个映射都调用一次`put()`方法

##### invalidate()

```java
void invalidate(Object key);
```

丢弃与键映射的值，使其无效

##### invalidateAll()

```java
void invalidateAll(Iterable<?> keys);
```

根据传入的键，丢弃与它们映射的值，使其无效

##### invalidateAll()

```java
void invalidateAll();
```

直接丢弃缓存中所有的键值对

##### size()

```java
void invalidateAll();
```

返回一个**近似的**键值对总数

##### stats()

```java
CacheStats stats();
```

返回一个缓存的累计统计的**快照**。所有的信息都是从零开始，在缓存的生命周期内单调递增

##### asMap()

```java
ConcurrentMap<K, V> asMap();
```

返回一个所有键值对的视图，并且此视图是线程安全的(ConcurrentMap)，修改此视图将直接影响缓存中的键值对。返回的视图的迭代器至少于缓存是弱一致的，它们可以安全的在并发情况下使用；但如果在返回了视图并生成了迭代器之后再修改缓存(包括丢弃)，那么这些更改不会影响到已生成的迭代器

##### cleanUp()

```java
void cleanUp();
```

执行缓存所需的任何需要挂起的维护操作，具体执行哪些操作需要取决于具体实现



## 抽象类AbstractCache

#### 介绍

此类为`Cache`接口的一种基本的实现，用来简化开发者实现Cache的成本，开发者只需要去实现`put()`方法和`getIfPresent()`方法即可。其中`getPresent()`使用`getIfPresent()`来实现；`putAll()`使用`put()`来实现；`invalidateAll(Iterator)`使用`invalidateAll()`来实现；`cleanUp()`并没有多余的操作；所有其他的方法都可能抛出一个`UnsupportedOperationException`异常

#### 构造器

```java
protected AbstractCache() {}
```

这是一个供子类使用的构造函数

#### 方法

##### getAllPresent()

```java
@Override
public ImmutableMap<K, V> getAllPresent(Iterable<?> keys) {
  Map<K, V> result = Maps.newLinkedHashMap();
  for (Object key : keys) {//遍历keys
    if (!result.containsKey(key)) {//当result中不包含当前key的信息才进行操作，过滤key
      @SuppressWarnings("unchecked")
      K castKey = (K) key;
      V value = getIfPresent(key);//调用getIfPresent()方法根据key获取value
      if (value != null) {
        result.put(castKey, value);//value不为空才添加到result
      }
    }
  }
  return ImmutableMap.copyOf(result);//最终将返回一个ImmutableMap
}
```

方法里面最值得一提的是判断当前遍历到的key不存在于result中才进行操作，这意味着如果后面有相同的key将没有放到result的可能；还有就是当根据key取到value时会判断value是否为空，不为空才会被添加到result中

##### putAll()

```java
@Override
public void putAll(Map<? extends K, ? extends V> m) {
  for (Map.Entry<? extends K, ? extends V> entry : m.entrySet()) {
    put(entry.getKey(), entry.getValue());
  }
}
```

方法内部就是通过遍历传入的Map来逐个的调用`put()`方法实现添加

##### invalidateAll()

```java
@Override
public void invalidateAll(Iterable<?> keys) {
  for (Object key : keys) {
    invalidate(key);
  }
}
```

方法内部也是通过遍历传入的迭代器keys来逐个的调用无参的重载方法`invalidateAll()`来实现无效的

##### cleanUp()

```java
@Override
public void cleanUp() {}
```

方法内部没有做任何事

##### 其他方法

此抽象类也同样实现了父接口的其他所有方法，然而只是在方法内部抛出了一个`UnsupportedOperationException`异常

#### 子接口StatsCounter

##### 介绍

该子接口用来统计缓存是否命中，命中状态的记录

##### 用来记录的方法

```java
void recordHits(int count);//用来记录缓存命中，当缓存请求返回缓存值时应当调用该函数

void recordMisses(int count);//用来记录缓存未命中,当缓存请求返回缓存值找不到时,应该调用该函数;加载线程和加载时阻塞的线程都应该调用此函数(加载应该指的是get方法里callable对键的计算)

void recordLoadSuccess(long loadTime);//用来记录缓存加载成功,当缓存请求导致加载条目并加载成功时,应调用此函数;与recordMisses()方法相反，此方法只能由加载线程调用

void recordLoadException(long loadTime);//用来记录缓存加载出错，当缓存请求导致加载并在加载过程中出现异常时，应该调用此函数；与recordMisses()方法相反，此方法只能由加载线程调用

void recordEviction();//用来记录从缓存中被丢弃的项，只有在由于被缓存的驱逐策略而丢弃条目而不是由于手动丢弃时，才可调用此函数
```

##### snapshot()

```java
CacheStats snapshot();
```

此方法会返回一个计数器的快照，需要注意，这可能是一个不一致的视图，因为该方法可能与更新操作交织在一起

#### 内部实现类SimpleStatsCounter

##### 介绍

这是一个为Cache提供的**线程安全的**StatsCounter的实现

##### 成员变量

```java
private final LongAddable hitCount = LongAddables.create();//命中次数
private final LongAddable missCount = LongAddables.create();//未命中次数
private final LongAddable loadSuccessCount = LongAddables.create();//加载成功次数
private final LongAddable loadExceptionCount = LongAddables.create();//加载出现异常次数
private final LongAddable totalLoadTime = LongAddables.create();//总加载次数
private final LongAddable evictionCount = LongAddables.create();//驱逐策略丢弃次数
```

分别声明了多个**private final**的LongAddable对象，这个类可以实现在并发环境中操作long值

```java
@GwtCompatible
interface LongAddable {
  void increment();
  
  void add(long x);
  
  long sum();
}
```

该接口只定义了三个方法，分别对应”加一“、”增加指定值“和”统计和“的操作

```java
@GwtCompatible(emulated = true)
final class LongAddables {
  private static final Supplier<LongAddable> SUPPLIER;
  
  static {
    Supplier<LongAddable> supplier;
    try {
      new LongAdder();
      supplier = new Supplier<LongAddable>() {
        @Override
        public LongAddable get() {
          return new LongAdder();
        }
      };
    } catch (Throwable t) { // we really want to catch *everything*
      supplier = new Supplier<LongAddable>() {
        @Override
        public LongAddable get() {
          return new PureJavaLongAddable();
        }
      };
    }
    SUPPLIER = supplier;
  }
  
  public static LongAddable create() {
    return SUPPLIER.get();
  }
  
  private static final class PureJavaLongAddable extends AtomicLong implements LongAddable {
    @Override
    public void increment() {
      getAndIncrement();
    }

    @Override
    public void add(long x) {
      getAndAdd(x);
    }

    @Override
    public long sum() {
      return get();
    }
  }
}
```

这个类提供了具体的LongAddable对象的创建，算是它的工具类。可以看到`PureJavaLongAddable`使用原子类实现了上面的三个方法来保证其在并发环境下的安全

##### 内部方法

可以看到内部的方法都是通过成员变量的来操作的；在加载成功和加载出现异常的统计中还使加载次数(totalLoadTime)也加了一；`snapshot()`方法则是简单的创建了一个CacheStats对象并将所有成员变量作为参数传入，此类下次再讲。`incrementBy()`方法通过传入一个StatsCounter对象，将传入对象的对应参数都加到了本类的成员变量上实现了增加指定的值



## 总结

介绍了Guava Cache本地缓存的核心接口Cache，里面定义了一些基本的方法。接着介绍了其其中一个实现类AbstractCache，该类虽说实现了父接口的全部方法，但是只有三个方法写了实现，其他方法都还是直接抛出异常。此抽象类关键的还是定义了StatsCounter接口和其实现类SimpleStatsCounter为统计类提供了实现