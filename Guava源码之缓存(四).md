# Guava源码之缓存(四)

### CacheBuilder

#### 介绍

这是一个`LoadingCache`和`Cache`接口实例的构造器，它可以组合实现以下功能：

- 自动加载键值对到缓存中
- 超过一定时间访问最少的键值对将被丢弃
- 支持根据**自从写入后**或**最后一次读取后**一段时间来控制过期时间
- 键会自动包装在**弱引用**中；值会自动包装在**弱引用或软引用**中
- 丢弃(或以其他方式删除)键值对会有通知
- 访问缓存的统计信息

上述这些功能都是可选的；可以选择创建一个包含以上所有功能的Cache，也可以创建一个不包含以上任何功能的Cache。

**默认的情况下CacheBuilder会创建一个不涉及任何丢弃操作的Cache对象**，如下：

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
  .maximumSize(10000)
  .expireAfterWrite(10, TimeUnit.MINUTES)
  .removalListener(MY_LISTENER)
  .build(
			new CacheLoader<Key, Graph>(){
        public Graph load(Key key) throws AnyException {
          return createExpensiveGraph(key);
        }
      });
```

或者这样配置：

```java
String spec = "maximumSize=10000,expireAfterWrite=10m";//当然，通常在项目中这些是在配置文件中配置
LoadingCache<Key, Graph> graphs = CacheBuilder.from(spec)
  .removalListener(MY_LISTENER)
  .build(
			new CacheLoader<Key, Graph>(){
        public Graph load(Key key) throws AnyException {
          return createExpensiveGraph(key);
        }
      });
```

它返回的Cache接口的实现像是一个哈希表，其特征与`ConcurrentHashMap`相同；asMap()方法所返回的视图(及其集合)具有**弱一致性**的迭代器，这可以保证在多线程中是线程安全的。但是如果其他线程在迭代器生成之后又改变了缓存，那么这些改变会不会影响到迭代器就不好说了。这些迭代器永远不会抛出`ConcurrentModificationException`异常。

通常情况下，返回的Cache对于键和值的相等性比较是通过`equals()`方法的；但是如果指定了弱引用的键和值，那么将会使用`==`这种比较地址的方式判断相等性

#### 关于类

```java
@GwtCompatible(emulated = true)
public final class CacheBuilder<K, V> {
```

是一个顶级类也是一个最终类，没有继承或实现任何类或接口

#### 构造方法

##### CacheBuilder()

```java
CacheBuilder() {}
```

默认的无参构造器

##### newBuilder()

```java
public static CacheBuilder<Object, Object> newBuilder() {
  return new CacheBuilder<Object, Object>();
}
```

这是一个会返回默认构造的CacheBuilder实例，包括强引用的键和值并且没有任何自动丢弃键值对的方法

##### from(CacheBuilderSpec)

```java
@Beta
@GwtIncompatible("To be supported")
public static CacheBuilder<Object, Object> from(CacheBuilderSpec spec) {
  return spec.toCacheBuilder()
      .lenientParsing();
}

@GwtIncompatible("To be supported")
CacheBuilder<K, V> lenientParsing() {
  strictParsing = false;
  return this;
}
```

使用spec构造的设置创建新的CacheBuilder实例

##### from(String)

```java
@Beta
@GwtIncompatible("To be supported")
public static CacheBuilder<Object, Object> from(String spec) {
  return from(CacheBuilderSpec.parse(spec));
}
```

使用指定的字符串配置创建新的CacheBuilder实例，这样的话对于命令号配置很友好

#### 成员变量

##### DEFAULT_INITIAL_CAPACITY

```java
private static final int DEFAULT_INITIAL_CAPACITY = 16;//默认初始容量

//类里面有一个相关的获取初始容量的方法，这又涉及到了变量initialCapacity
//如果initialCapacity没有被设置新值，那么将返回DEFAULT_INITIAL_CAPACITY，否则返回
int getInitialCapacity() {
  return (initialCapacity == UNSET_INT) ? DEFAULT_INITIAL_CAPACITY : initialCapacity;
}

int initialCapacity = UNSET_INT;//将initialCapacity初始化为初始int值

static final int UNSET_INT = -1;//类里面定义的当没有设置任何int初始值时，使用此值作为初始值
```

##### DEFAULT_CONCURRENCY_LEVEL

```java
private static final int DEFAULT_CONCURRENCY_LEVEL = 4;//默认并发级别；设计与DEFAULT_INITIAL_CAPACITY类似

int getConcurrencyLevel() {
  return (concurrencyLevel == UNSET_INT) ? DEFAULT_CONCURRENCY_LEVEL : concurrencyLevel;
}

int concurrencyLevel = UNSET_INT;
```

##### DEFAULT_EXPIRATION_NANOS

```java
private static final int DEFAULT_EXPIRATION_NANOS = 0;//默认过期时间；设计与DEFAULT_INITIAL_CAPACITY类似，只不过涉及到两个：写后多长时间过期和最后一次读取多长时间后过期

long getExpireAfterWriteNanos() {
  return (expireAfterWriteNanos == UNSET_INT) ? DEFAULT_EXPIRATION_NANOS : expireAfterWriteNanos;
}

long expireAfterWriteNanos = UNSET_INT;

long getExpireAfterAccessNanos() {
  return (expireAfterAccessNanos == UNSET_INT)
      ? DEFAULT_EXPIRATION_NANOS : expireAfterAccessNanos;
}

long expireAfterAccessNanos = UNSET_INT;
```

##### DEFAULT_REFRESH_NANOS

```java
private static final int DEFAULT_REFRESH_NANOS = 0;//默认刷新时间；设计与DEFAULT_INITIAL_CAPACITY类似

long getRefreshNanos() {
  return (refreshNanos == UNSET_INT) ? DEFAULT_REFRESH_NANOS : refreshNanos;
}

long refreshNanos = UNSET_INT;
```

##### NULL_STATS_COUNTER

```java
//这个是用来统计缓存的信息的，在AbstractCache中已经解释过这些方法的含义
static final Supplier<? extends StatsCounter> NULL_STATS_COUNTER = Suppliers.ofInstance(
    new StatsCounter() {
      @Override
      public void recordHits(int count) {}

      @Override
      public void recordMisses(int count) {}

      @Override
      public void recordLoadSuccess(long loadTime) {}

      @Override
      public void recordLoadException(long loadTime) {}

      @Override
      public void recordEviction() {}

      @Override
      public CacheStats snapshot() {
        return EMPTY_STATS;
      }
    });

Supplier<? extends StatsCounter> statsCounterSupplier = NULL_STATS_COUNTER;
```

##### EMPTY_STATS

```java
//空的统计信息，所有信息都初始化为零
static final CacheStats EMPTY_STATS = new CacheStats(0, 0, 0, 0, 0, 0);
```

##### CACHE_STATS_COUNTER

```java
static final Supplier<StatsCounter> CACHE_STATS_COUNTER =
    new Supplier<StatsCounter>() {
  @Override
  public StatsCounter get() {
    return new SimpleStatsCounter();//返回一个Abstract#SimpleStatsCounter
  }
};
```

##### NULL_TICKER

```java
static final Ticker NULL_TICKER = new Ticker() {
  @Override
  public long read() {//重写了read()方法，直接返回0
    return 0;
  }
};

Ticker getTicker(boolean recordsTime) {
  if (ticker != null) {
    return ticker;
  }
  return recordsTime ? Ticker.systemTicker() : NULL_TICKER;//返回系统纳秒数或者0
}
```

##### logger

```java
//类里面自带日志
private static final Logger logger = Logger.getLogger(CacheBuilder.class.getName());
```

##### strictParsing

```java
boolean strictParsing = true;//是否严格解析，默认为true
```

##### 其他成员变量

```java
int initialCapacity = UNSET_INT;//初始容量
int concurrencyLevel = UNSET_INT;//并发级别
long maximumSize = UNSET_INT;//最大数量
long maximumWeight = UNSET_INT;//最大权重
Weigher<? super K, ? super V> weigher;//权重

Strength keyStrength;//键的引用强度(强软弱)
Strength valueStrength;//值的引用强度

long expireAfterWriteNanos = UNSET_INT;//写后过期时间
long expireAfterAccessNanos = UNSET_INT;//最后一次读取后过期时间
long refreshNanos = UNSET_INT;//刷新时间

Equivalence<Object> keyEquivalence;//判断键的相等策略(equals或==)
Equivalence<Object> valueEquivalence;//判断值的相等策略

RemovalListener<? super K, ? super V> removalListener;//移除键值对监听器
Ticker ticker;//事件来源(系统或自定义{0})

Supplier<? extends StatsCounter> statsCounterSupplier = NULL_STATS_COUNTER;//缓存统计项
```

#### 方法

##### 关于键值对的相等性判断

默认使用`equals()`方法判断，如果指定了key或value的引用强度，那么使用`==`判断

```java
//判断键值相等性的get/set方法
@GwtIncompatible("To be supported")
CacheBuilder<K, V> keyEquivalence(Equivalence<Object> equivalence) {
  checkState(keyEquivalence == null, "key equivalence was already set to %s", keyEquivalence);
  keyEquivalence = checkNotNull(equivalence);
  return this;
}

Equivalence<Object> getKeyEquivalence() {
  return MoreObjects.firstNonNull(keyEquivalence, getKeyStrength().defaultEquivalence());
}

@GwtIncompatible("To be supported")
CacheBuilder<K, V> valueEquivalence(Equivalence<Object> equivalence) {
  checkState(valueEquivalence == null,
      "value equivalence was already set to %s", valueEquivalence);
  this.valueEquivalence = checkNotNull(equivalence);
  return this;
}

Equivalence<Object> getValueEquivalence() {
  return MoreObjects.firstNonNull(valueEquivalence, getValueStrength().defaultEquivalence());
}
```

##### 关于初始化容量

设置和获取内部哈希表的最小总大小。如果总容量为60，并发级别为8，那么会创建8个segment，每个segment都有一个容量为8的哈希表。

```java
public CacheBuilder<K, V> initialCapacity(int initialCapacity) {
  checkState(this.initialCapacity == UNSET_INT, "initial capacity was already set to %s",
      this.initialCapacity);
  checkArgument(initialCapacity >= 0);
  this.initialCapacity = initialCapacity;
  return this;
}

int getInitialCapacity() {
  return (initialCapacity == UNSET_INT) ? DEFAULT_INITIAL_CAPACITY : initialCapacity;
}
```

##### 关于并发级别

能够引导在更新操作中的并发操作。用做内部大小调整的提示。哈希表在内部进行分区(segment)来实现更新操作中的并发而不产生争用的情况。

当前实现使用并发级别来创建对应个数的segment，每个segment都由自己的写锁来控制。将来可能不会再继续使用segment而采用更好地并发控制方案。

```java
public CacheBuilder<K, V> concurrencyLevel(int concurrencyLevel) {
  checkState(this.concurrencyLevel == UNSET_INT, "concurrency level was already set to %s",
      this.concurrencyLevel);
  checkArgument(concurrencyLevel > 0);
  this.concurrencyLevel = concurrencyLevel;
  return this;
}

int getConcurrencyLevel() {
  return (concurrencyLevel == UNSET_INT) ? DEFAULT_CONCURRENCY_LEVEL : concurrencyLevel;
}
```

##### 关于最大值MaximumSize

指定一个缓存能够存储的键值对的最大数量。需要注意：当键值对的数量超过此设置的值，那么将会丢弃一个键值对。当键值对的数量接近最大值时，缓存会将使用的最少的(最近没有被访问到的)键值对丢弃。

> 当设置大小为零时，被加载到缓存的键值对会立即被缓存丢弃。这对于测试或者不修改代码来禁用缓存的程序来说是很有帮助的

**注意：**它不能与maximumWeight一起使用

```java
public CacheBuilder<K, V> maximumSize(long size) {
  checkState(this.maximumSize == UNSET_INT, "maximum size was already set to %s",
      this.maximumSize);
  checkState(this.maximumWeight == UNSET_INT, "maximum weight was already set to %s",
      this.maximumWeight);
  checkState(this.weigher == null, "maximum size can not be combined with weigher");
  checkArgument(size >= 0, "maximum size must not be negative");
  this.maximumSize = size;
  return this;
}
```

##### 关于最大权重MaximumWeight

指定一个缓存的键值对的最大权重。权重是使用Weigher来决定的，使用这种方法需要在调用`build()`之前调用相应的Weigher。

在超过权重之后，缓存可能会丢弃一个键值对。当权重接近最大权重时，缓存可能会丢弃一个最近不被使用的键值对。

> 当设置大小为零时，被加载到缓存中的键值对会立即被缓存丢弃。这对于测试或者不修改代码来禁用缓存的程序是很有帮助的

**注意：**权重仅仅决定了键值对是否超过了缓存的最大容量，它对于下一步会丢弃哪个键值对没有任何影响。它同样不能与MaximumSize一起使用

```java
@GwtIncompatible("To be supported")
public CacheBuilder<K, V> maximumWeight(long weight) {
  checkState(this.maximumWeight == UNSET_INT, "maximum weight was already set to %s",
      this.maximumWeight);
  checkState(this.maximumSize == UNSET_INT, "maximum size was already set to %s",
      this.maximumSize);
  this.maximumWeight = weight;
  checkArgument(weight >= 0, "maximum weight must not be negative");
  return this;
}

long getMaximumWeight() {
  if (expireAfterWriteNanos == 0 || expireAfterAccessNanos == 0) {
    return 0;
  }
  return (weigher == null) ? maximumSize : maximumWeight;
}
```

##### 关于Weigher

用于确定缓存中键值对的权重。当确定要丢弃哪些键值对时，`maximumWeight(long)`这个方法会考虑键值对的权重，并且如果想使用这个方法(maximumWeight(long))需要在调用`build()`之前调用它。当一个键值对被添加到缓存中的时候键值对的权重就被测量和记录下来了，所以在整个键值对的生命周期中，权重相当于一个静态的属性。

当权重为零时，将不会考虑根据数量丢弃键值对。

**注意：**它返回的是带泛型的CacheBuilder，这意味着它是安全的、有类型限制的，以防以后抛出`ClassCastException`

```java
@GwtIncompatible("To be supported")
public <K1 extends K, V1 extends V> CacheBuilder<K1, V1> weigher(
    Weigher<? super K1, ? super V1> weigher) {
  checkState(this.weigher == null);
  if (strictParsing) {
    checkState(this.maximumSize == UNSET_INT, "weigher can not be combined with maximum size",
        this.maximumSize);
  }

  // safely limiting the kinds of caches this can produce
  @SuppressWarnings("unchecked")
  CacheBuilder<K1, V1> me = (CacheBuilder<K1, V1>) this;
  me.weigher = checkNotNull(weigher);
  return me;
}

@SuppressWarnings("unchecked")
<K1 extends K, V1 extends V> Weigher<K1, V1> getWeigher() {
  return (Weigher<K1, V1>) MoreObjects.firstNonNull(weigher, OneWeigher.INSTANCE);
}
```

##### 关于键值的引用强度

```java
//键
@GwtIncompatible("java.lang.ref.WeakReference")
public CacheBuilder<K, V> weakKeys() {
  return setKeyStrength(Strength.WEAK);
}

CacheBuilder<K, V> setKeyStrength(Strength strength) {
  checkState(keyStrength == null, "Key strength was already set to %s", keyStrength);
  keyStrength = checkNotNull(strength);
  return this;
}

Strength getKeyStrength() {
  return MoreObjects.firstNonNull(keyStrength, Strength.STRONG);
}
//值
@GwtIncompatible("java.lang.ref.WeakReference")
public CacheBuilder<K, V> weakValues() {
  return setValueStrength(Strength.WEAK);
}
@GwtIncompatible("java.lang.ref.SoftReference")
public CacheBuilder<K, V> softValues() {
  return setValueStrength(Strength.SOFT);
}

CacheBuilder<K, V> setValueStrength(Strength strength) {
  checkState(valueStrength == null, "Value strength was already set to %s", valueStrength);
  valueStrength = checkNotNull(strength);
  return this;
}

Strength getValueStrength() {
  return MoreObjects.firstNonNull(valueStrength, Strength.STRONG);
}
```

##### 关于两种过期方式

```java
//写后过期
public CacheBuilder<K, V> expireAfterWrite(long duration, TimeUnit unit) {
  checkState(expireAfterWriteNanos == UNSET_INT, "expireAfterWrite was already set to %s ns", expireAfterWriteNanos);
  checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
  this.expireAfterWriteNanos = unit.toNanos(duration);
  return this;
}

long getExpireAfterWriteNanos() {
  return (expireAfterWriteNanos == UNSET_INT) ? DEFAULT_EXPIRATION_NANOS : expireAfterWriteNanos;
}
//最近访问后过期
public CacheBuilder<K, V> expireAfterAccess(long duration, TimeUnit unit) {
  checkState(expireAfterAccessNanos == UNSET_INT, "expireAfterAccess was already set to %s ns", expireAfterAccessNanos);
  checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
  this.expireAfterAccessNanos = unit.toNanos(duration);
  return this;
}

long getExpireAfterAccessNanos() {
  return (expireAfterAccessNanos == UNSET_INT)
      ? DEFAULT_EXPIRATION_NANOS : expireAfterAccessNanos;
}
```

##### 关于刷新时间

在键值对创建之后或者对其值的最新替换之后的一段时间之后自动刷新。刷新的语义由`LoadingCache#refresh()`指定，并且通过调用`CacheLoader#load()`方法执行刷新。

当请求一个已经过期的键值对那么就会触发刷新。当此请求触发了刷新时将会阻塞对`CacheLoader#load()`方法的调用，如果要返回的值已经被加载出来那么会立即返回新值，否则将返回旧值。

需要注意的是，在刷新期间所有的异常都会被记录到日志中，但是这些异常会被"catch"住而不会停止程序

```java
@Beta
@GwtIncompatible("To be supported (synchronously).")
public CacheBuilder<K, V> refreshAfterWrite(long duration, TimeUnit unit) {
  checkNotNull(unit);
  checkState(refreshNanos == UNSET_INT, "refresh was already set to %s ns", refreshNanos);
  checkArgument(duration > 0, "duration must be positive: %s %s", duration, unit);
  this.refreshNanos = unit.toNanos(duration);
  return this;
}

long getRefreshNanos() {
  return (refreshNanos == UNSET_INT) ? DEFAULT_REFRESH_NANOS : refreshNanos;
}
```

##### 关于纳秒精度时间源

指定纳秒精度的时间源，它用于确定键值对应该在何时过期。默认情况下使用系统的纳秒时间。制定此变量的目的是方便测试设置了过期的缓存。

```java
public CacheBuilder<K, V> ticker(Ticker ticker) {
  checkState(this.ticker == null);
  this.ticker = checkNotNull(ticker);
  return this;
}

Ticker getTicker(boolean recordsTime) {
  if (ticker != null) {
    return ticker;
  }
  return recordsTime ? Ticker.systemTicker() : NULL_TICKER;
}
```

##### 关于移除监听器

指定了一个监听器用来监听丢弃键值对的操作。

```java
@CheckReturnValue
public <K1 extends K, V1 extends V> CacheBuilder<K1, V1> removalListener(
    RemovalListener<? super K1, ? super V1> listener) {
  checkState(this.removalListener == null);

  // safely limiting the kinds of caches this can produce
  @SuppressWarnings("unchecked")
  CacheBuilder<K1, V1> me = (CacheBuilder<K1, V1>) this;
  me.removalListener = checkNotNull(listener);
  return me;
}

// Make a safe contravariant cast now so we don't have to do it over and over.
@SuppressWarnings("unchecked")
<K1 extends K, V1 extends V> RemovalListener<K1, V1> getRemovalListener() {
  return (RemovalListener<K1, V1>)
      MoreObjects.firstNonNull(removalListener, NullListener.INSTANCE);
}
```

##### 关于统计记录

用来记录对缓存的各种操作，对缓存的性能可能会有一定影响

```java
public CacheBuilder<K, V> recordStats() {
  statsCounterSupplier = CACHE_STATS_COUNTER;
  return this;
}

boolean isRecordingStats() {
  return statsCounterSupplier == CACHE_STATS_COUNTER;
}

Supplier<? extends StatsCounter> getStatsCounterSupplier() {
  return statsCounterSupplier;
}
```

##### 关于构建缓存

用来构建缓存，既可以指定一个key根据key来获取已有的值也可以根据提供的CacheLoader来检索或计算值。如果另一个线程正在加载该key的值，那么只需等待该线程计算完毕后直接返回新计算的值即可。

> 多个线程可以同时加载不同键的值

此方法不会改变CacheBuilder实例的状态，因此可以再次调用它来创建独立的缓存

```java
public <K1 extends K, V1 extends V> LoadingCache<K1, V1> build(
    CacheLoader<? super K1, V1> loader) {
  checkWeightWithWeigher();
  return new LocalCache.LocalLoadingCache<K1, V1>(this, loader);
}
```

构建一个不会再key被请求时加载值的缓存

```java
public <K1 extends K, V1 extends V> Cache<K1, V1> build() {
  checkWeightWithWeigher();
  checkNonLoadingCache();
  return new LocalCache.LocalManualCache<K1, V1>(this);
}

private void checkNonLoadingCache() {
  checkState(refreshNanos == UNSET_INT, "refreshAfterWrite requires a LoadingCache");
}

private void checkWeightWithWeigher() {
  if (weigher == null) {
    checkState(maximumWeight == UNSET_INT, "maximumWeight requires weigher");
  } else {
    if (strictParsing) {
      checkState(maximumWeight != UNSET_INT, "weigher requires maximumWeight");
    } else {
      if (maximumWeight == UNSET_INT) {
        logger.log(Level.WARNING, "ignoring weigher specified without maximumWeight");
      }
    }
  }
}
```

##### 关于输出

输出此CacheBuilder实例的信息

```java
@Override
public String toString() {
  MoreObjects.ToStringHelper s = MoreObjects.toStringHelper(this);
  if (initialCapacity != UNSET_INT) {
    s.add("initialCapacity", initialCapacity);
  }
  if (concurrencyLevel != UNSET_INT) {
    s.add("concurrencyLevel", concurrencyLevel);
  }
  if (maximumSize != UNSET_INT) {
    s.add("maximumSize", maximumSize);
  }
  if (maximumWeight != UNSET_INT) {
    s.add("maximumWeight", maximumWeight);
  }
  if (expireAfterWriteNanos != UNSET_INT) {
    s.add("expireAfterWrite", expireAfterWriteNanos + "ns");
  }
  if (expireAfterAccessNanos != UNSET_INT) {
    s.add("expireAfterAccess", expireAfterAccessNanos + "ns");
  }
  if (keyStrength != null) {
    s.add("keyStrength", Ascii.toLowerCase(keyStrength.toString()));
  }
  if (valueStrength != null) {
    s.add("valueStrength", Ascii.toLowerCase(valueStrength.toString()));
  }
  if (keyEquivalence != null) {
    s.addValue("keyEquivalence");
  }
  if (valueEquivalence != null) {
    s.addValue("valueEquivalence");
  }
  if (removalListener != null) {
    s.addValue("removalListener");
  }
  return s.toString();
}
```

#### 内部枚举类

实现了RemovalListener和Weigher的两个枚举类，各提供了一个实例并重写了一个方法

```java
enum NullListener implements RemovalListener<Object, Object> {
  INSTANCE;

  @Override
  public void onRemoval(RemovalNotification<Object, Object> notification) {}
}

enum OneWeigher implements Weigher<Object, Object> {
  INSTANCE;

  @Override
  public int weigh(Object key, Object value) {
    return 1;
  }
}
```



### 总结

此类整体看下来其实并不复杂，但是这个代码量很劝退！看下来之后才发现这些方法都是针对那一大堆成员变量写的类似”get/set“方法。所以只要理解了那些成员变量的含义即可。看完之后可以对Guava Cache的整体思路有一定理解。