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

