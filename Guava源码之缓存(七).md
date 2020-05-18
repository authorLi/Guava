# Guava源码之缓存(七)

### 类CacheBuilderSpec

#### 介绍

这是一个CacheBuilder的配置规范。特别的是：CacheBuilderSpec支持从字符串中获取配置，这就使得命令行这种配置变得尤其的好用。其中字符串语法是一系列用逗号分隔的键或者键值对，每个对应了CacheBuilder的一个方法。例如：

- concurrencyLevel=[integer]：设置了CacheBuilder的**concurrencyLevel**
- initialCapacity=[integer]：设置了CacheBuilder的**initialCapacity**
- maximumSize=[long]：设置了CacheBuilder的**maximumSize**
- maximumWeight=[long]：设置了CacheBuilder的**maximumWeight**
- expireAfterAccess=[duration]：设置了CacheBuilder的**expireAfterAccess**
- expireAfterWrite=[duration]：设置了CacheBuilder的**expireAfterWrite**
- refreshAfterWrite=[duration]：设置了CacheBuilder的**refreshAfterWrite**
- weakKeys：设置了CacheBuilder的**weakKeys**
- softValues：设置了CacheBuilder的**softValues**
- weakValues：设置了CacheBuilder的**weakValues**
- recordStats：设置了CacheBuilder的**recordStats**

1. 对于缓存所支持的键将随着缓存的发展而越来越多，但是已经存在的键并不会被删除。
2. 对于duration，可以设置为一个int值后面带有如d、h、m、s等后缀代表天、小时、分钟和秒。
3. 在配置字符串中，等号和逗号前后的空格将被忽略。键是不可重复的；在单个值中使用以下键也是非法的：
   - maximumSize和maximumWeight
   - weakValues和softValues
4. CacheBuilderSpec不支持使用非值参数来配置CacheBuilder的方法，如果要这样应该在代码中配置。
5. 可以使用`CacheBuilder#from(String)`或者`CacheBuilder#from(CacheBuilderSpec)`从CacheBuilderSpec中实例化一个CacheBuilder对象

#### 关于类

```java
@Beta
public final class CacheBuilderSpec {
```

仅仅标明它是一个最终类。

#### 构造器

##### 私有构造器

私有构造器，通过传入一个配置的字符串来给specification变量赋值并创建实例

```java
private CacheBuilderSpec(String specification) {
  this.specification = specification;
}

/** 接收一个变量用于解析字符串 */
private final String specification;
```

##### 静态构造器

然而由于上面的构造器是私有的，所以不能被外界直接调用，所以便有了下面这个构造函数。首先它是静态的意味着它可以直接由类调用，其次里面在调用了私有构造器之后又对配置字符串做了解析，用到了`KEYS_SPLITTER`来分割每个键值对的配置，得到了键值对之后再使用`KEY_VALUE_SPLITTER`拿到每个键值对的键和值，最后调用专用的`valueParser`来做解析。

```java
public static CacheBuilderSpec parse(String cacheBuilderSpecification) {
  CacheBuilderSpec spec = new CacheBuilderSpec(cacheBuilderSpecification);
  if (!cacheBuilderSpecification.isEmpty()) {
    for (String keyValuePair : KEYS_SPLITTER.split(cacheBuilderSpecification)) {
      List<String> keyAndValue = ImmutableList.copyOf(KEY_VALUE_SPLITTER.split(keyValuePair));
      checkArgument(!keyAndValue.isEmpty(), "blank key-value pair");
      checkArgument(keyAndValue.size() <= 2,
          "key-value pair %s with more than one equals sign", keyValuePair);

      // Find the ValueParser for the current key.
      String key = keyAndValue.get(0);
      ValueParser valueParser = VALUE_PARSERS.get(key);
      checkArgument(valueParser != null, "unknown key %s", key);

      String value = keyAndValue.size() == 1 ? null : keyAndValue.get(1);
      valueParser.parse(spec, key, value);
    }
  }

  return spec;
}
```

此方法里面用到了三个成员变量：

```java
/** 用于分割配置字符串的每个键值对 */
private static final Splitter KEYS_SPLITTER = Splitter.on(',').trimResults();

/** 用于分割键值对 */
private static final Splitter KEY_VALUE_SPLITTER = Splitter.on('=').trimResults();

/** 储存了所有对应变量对应的解析器，不可变map */
private static final ImmutableMap<String, ValueParser> VALUE_PARSERS =
    ImmutableMap.<String, ValueParser>builder()
        .put("initialCapacity", new InitialCapacityParser())
        .put("maximumSize", new MaximumSizeParser())
        .put("maximumWeight", new MaximumWeightParser())
        .put("concurrencyLevel", new ConcurrencyLevelParser())
        .put("weakKeys", new KeyStrengthParser(Strength.WEAK))
        .put("softValues", new ValueStrengthParser(Strength.SOFT))
        .put("weakValues", new ValueStrengthParser(Strength.WEAK))
        .put("recordStats", new RecordStatsParser())
        .put("expireAfterAccess", new AccessDurationParser())
        .put("expireAfterWrite", new WriteDurationParser())
        .put("refreshAfterWrite", new RefreshDurationParser())
        .put("refreshInterval", new RefreshDurationParser())
        .build();
```

##### 特殊构造器

直接调用了静态构造器，那内部就还是调用了私有构造器。只不过此构造器特殊在它是不支持缓存的。它传入了一个配置字符串为`maximumSize=0`，前面说过此配置为0那么当键值对被加入到缓存中时会被立即丢弃，所以相当于禁用。

```java
public static CacheBuilderSpec disableCaching() {
  // Maximum size of zero is one way to block caching
  return CacheBuilderSpec.parse("maximumSize=0");
}
```

##### 可设置配置的构造器

首先要注意它并不是静态的，必须通过实例才能调用，那也就是说它是获取一个你已经设置好配置的CacheBuilder对象。

```java
CacheBuilder<Object, Object> toCacheBuilder() {
  CacheBuilder<Object, Object> builder = CacheBuilder.newBuilder();
  if (initialCapacity != null) {
    builder.initialCapacity(initialCapacity);
  }
  if (maximumSize != null) {
    builder.maximumSize(maximumSize);
  }
  if (maximumWeight != null) {
    builder.maximumWeight(maximumWeight);
  }
  if (concurrencyLevel != null) {
    builder.concurrencyLevel(concurrencyLevel);
  }
  if (keyStrength != null) {
    switch (keyStrength) {
      case WEAK:
        builder.weakKeys();
        break;
      default:
        throw new AssertionError();
    }
  }
  if (valueStrength != null) {
    switch (valueStrength) {
      case SOFT:
        builder.softValues();
        break;
      case WEAK:
        builder.weakValues();
        break;
      default:
        throw new AssertionError();
    }
  }
  if (recordStats != null && recordStats) {
    builder.recordStats();
  }
  if (writeExpirationTimeUnit != null) {
    builder.expireAfterWrite(writeExpirationDuration, writeExpirationTimeUnit);
  }
  if (accessExpirationTimeUnit != null) {
    builder.expireAfterAccess(accessExpirationDuration, accessExpirationTimeUnit);
  }
  if (refreshTimeUnit != null) {
    builder.refreshAfterWrite(refreshDuration, refreshTimeUnit);
  }

  return builder;
}
```

CacheBuilder类的变量也在此类中做了声明，所以可以直接通过设置它们来获取CacheBuilder对象

```java
@VisibleForTesting Integer initialCapacity;
@VisibleForTesting Long maximumSize;
@VisibleForTesting Long maximumWeight;
@VisibleForTesting Integer concurrencyLevel;
@VisibleForTesting Strength keyStrength;
@VisibleForTesting Strength valueStrength;
@VisibleForTesting Boolean recordStats;
@VisibleForTesting long writeExpirationDuration;
@VisibleForTesting TimeUnit writeExpirationTimeUnit;
@VisibleForTesting long accessExpirationDuration;
@VisibleForTesting TimeUnit accessExpirationTimeUnit;
@VisibleForTesting long refreshDuration;
@VisibleForTesting TimeUnit refreshTimeUnit;
```

#### 方法

##### toParsableString()

返回当前创建实例参考的配置字符串

```java
public String toParsableString() {
  return specification;
}
```

##### toString()

返回了ToStringHelper的字符串格式，这个类挺有意思的，以后看工具类的时候会细看

```java
@Override
public String toString() {
  return MoreObjects.toStringHelper(this).addValue(toParsableString()).toString();
}
```

##### hashCode()

调用了Objects工具类的hashCode()方法，哈希值是根据这些CacheBuilder的配置算出来的

```java
@Override
public int hashCode() {
  return Objects.hashCode(
      initialCapacity,
      maximumSize,
      maximumWeight,
      concurrencyLevel,
      keyStrength,
      valueStrength,
      recordStats,
      durationInNanos(writeExpirationDuration, writeExpirationTimeUnit),
      durationInNanos(accessExpirationDuration, accessExpirationTimeUnit),
      durationInNanos(refreshDuration, refreshTimeUnit));
}
```

##### equals()

equals回去对比每个参数的值，都相等才返回true

```java
@Override
public boolean equals(@Nullable Object obj) {
  if (this == obj) {
    return true;
  }
  if (!(obj instanceof CacheBuilderSpec)) {
    return false;
  }
  CacheBuilderSpec that = (CacheBuilderSpec) obj;
  return Objects.equal(initialCapacity, that.initialCapacity)
      && Objects.equal(maximumSize, that.maximumSize)
      && Objects.equal(maximumWeight, that.maximumWeight)
      && Objects.equal(concurrencyLevel, that.concurrencyLevel)
      && Objects.equal(keyStrength, that.keyStrength)
      && Objects.equal(valueStrength, that.valueStrength)
      && Objects.equal(recordStats, that.recordStats)
      && Objects.equal(durationInNanos(writeExpirationDuration, writeExpirationTimeUnit),
          durationInNanos(that.writeExpirationDuration, that.writeExpirationTimeUnit))
      && Objects.equal(durationInNanos(accessExpirationDuration, accessExpirationTimeUnit),
          durationInNanos(that.accessExpirationDuration, that.accessExpirationTimeUnit))
      && Objects.equal(durationInNanos(refreshDuration, refreshTimeUnit),
          durationInNanos(that.refreshDuration, that.refreshTimeUnit));
}
```

##### durationInNanos()

该方法将传入的过期时间转换为一个Long值

```java
@Nullable private static Long durationInNanos(long duration, @Nullable TimeUnit unit) {
  return (unit == null) ? null : unit.toNanos(duration);
}
```

#### 内部类和接口

##### 接口ValueParser

里面只定义了一个方法，即解析值

```java
private interface ValueParser {
  void parse(CacheBuilderSpec spec, String key, @Nullable String value);
}
```

##### IntegerParser

内部定义了一个抽象方法用于解析integer值

```java
/** 用于解析integer类型值的解析器 */
abstract static class IntegerParser implements ValueParser {
  protected abstract void parseInteger(CacheBuilderSpec spec, int value);

  @Override
  public void parse(CacheBuilderSpec spec, String key, String value) {
    checkArgument(value != null && !value.isEmpty(), "value of key %s omitted", key);
    try {
      parseInteger(spec, Integer.parseInt(value));
    } catch (NumberFormatException e) {
      throw new IllegalArgumentException(
          String.format("key %s value set to %s, must be integer", key, value), e);
    }
  }
}
```

##### LongParser

内部定义了一个抽象方法用于解析long值

```java
/** 用于解析long类型值的解析器 */
abstract static class LongParser implements ValueParser {
  protected abstract void parseLong(CacheBuilderSpec spec, long value);

  @Override
  public void parse(CacheBuilderSpec spec, String key, String value) {
    checkArgument(value != null && !value.isEmpty(), "value of key %s omitted", key);
    try {
      parseLong(spec, Long.parseLong(value));
    } catch (NumberFormatException e) {
      throw new IllegalArgumentException(
          String.format("key %s value set to %s, must be integer", key, value), e);
    }
  }
}
```

##### InitialCapacity

继承了IntegerParser，用于解析initialCapacity

```java
/** 解析初始容量 */
static class InitialCapacityParser extends IntegerParser {
  @Override
  protected void parseInteger(CacheBuilderSpec spec, int value) {
    checkArgument(spec.initialCapacity == null,
        "initial capacity was already set to ", spec.initialCapacity);
    spec.initialCapacity = value;
  }
}
```

##### MaximumSizeParser

继承了LongParser，用于解析maximumSize

```java
/** 解析最大值 */
static class MaximumSizeParser extends LongParser {
  @Override
  protected void parseLong(CacheBuilderSpec spec, long value) {
    checkArgument(spec.maximumSize == null,
        "maximum size was already set to ", spec.maximumSize);
    checkArgument(spec.maximumWeight == null,
        "maximum weight was already set to ", spec.maximumWeight);
    spec.maximumSize = value;
  }
}
```

##### MaximumWeightParser

继承了LongParser，用于解析maximumWeightParser

```java
/** 解析最大权重 */
static class MaximumWeightParser extends LongParser {
  @Override
  protected void parseLong(CacheBuilderSpec spec, long value) {
    checkArgument(spec.maximumWeight == null,
        "maximum weight was already set to ", spec.maximumWeight);
    checkArgument(spec.maximumSize == null,
        "maximum size was already set to ", spec.maximumSize);
    spec.maximumWeight = value;
  }
}
```

##### ConcurrencyLevelParser

继承了IntegerParser，用于解析currencyLevel

```java
/** 解析并发级别 */
static class ConcurrencyLevelParser extends IntegerParser {
  @Override
  protected void parseInteger(CacheBuilderSpec spec, int value) {
    checkArgument(spec.concurrencyLevel == null,
        "concurrency level was already set to ", spec.concurrencyLevel);
    spec.concurrencyLevel = value;
  }
}
```

##### KeyStrengthParser

实现了ValueParser接口,用于解析weakKeys

```java
/** 解析弱键 */
static class KeyStrengthParser implements ValueParser {
  private final Strength strength;

  public KeyStrengthParser(Strength strength) {
    this.strength = strength;
  }

  @Override
  public void parse(CacheBuilderSpec spec, String key, @Nullable String value) {
    checkArgument(value == null, "key %s does not take values", key);
    checkArgument(spec.keyStrength == null, "%s was already set to %s", key, spec.keyStrength);
    spec.keyStrength = strength;
  }
}
```

##### ValueStrengthParser

实现了ValueParser接口,用于解析softValues和weakValues

```java
/** 解析弱值和软值 */
static class ValueStrengthParser implements ValueParser {
  private final Strength strength;

  public ValueStrengthParser(Strength strength) {
    this.strength = strength;
  }

  @Override
  public void parse(CacheBuilderSpec spec, String key, @Nullable String value) {
    checkArgument(value == null, "key %s does not take values", key);
    checkArgument(spec.valueStrength == null,
      "%s was already set to %s", key, spec.valueStrength);

    spec.valueStrength = strength;
  }
}
```

##### RecordStatsParser

实现了ValueParser接口，用于解析recordStatsParser

```java
/** 解析缓存统计数据 */
static class RecordStatsParser implements ValueParser {

  @Override
  public void parse(CacheBuilderSpec spec, String key, @Nullable String value) {
    checkArgument(value == null, "recordStats does not take values");
    checkArgument(spec.recordStats == null, "recordStats already set");
    spec.recordStats = true;
  }
}
```

##### DurationParser

实现了ValueParser接口，用于解析各种duration数据。

内部定义了一个专用于转换duration的抽象方法。另一个方法parse则是判断时间单位并执行转换

```java
/** 用于解析duration数据 */
abstract static class DurationParser implements ValueParser {
  protected abstract void parseDuration(
      CacheBuilderSpec spec,
      long duration,
      TimeUnit unit);

  @Override
  public void parse(CacheBuilderSpec spec, String key, String value) {
    checkArgument(value != null && !value.isEmpty(), "value of key %s omitted", key);
    try {
      char lastChar = value.charAt(value.length() - 1);
      TimeUnit timeUnit;
      switch (lastChar) {
        case 'd':
          timeUnit = TimeUnit.DAYS;
          break;
        case 'h':
          timeUnit = TimeUnit.HOURS;
          break;
        case 'm':
          timeUnit = TimeUnit.MINUTES;
          break;
        case 's':
          timeUnit = TimeUnit.SECONDS;
          break;
        default:
          throw new IllegalArgumentException(
              String.format("key %s invalid format.  was %s, must end with one of [dDhHmMsS]",
                  key, value));
      }

      long duration = Long.parseLong(value.substring(0, value.length() - 1));
      parseDuration(spec, duration, timeUnit);
    } catch (NumberFormatException e) {
      throw new IllegalArgumentException(
          String.format("key %s value set to %s, must be integer", key, value));
    }
  }
}
```

##### AccessDurationParser

继承了DurationParser，用于解析expireAfterAccess

```java
/** 解析”最后一次访问多久后过期“参数 */
static class AccessDurationParser extends DurationParser {
  @Override protected void parseDuration(CacheBuilderSpec spec, long duration, TimeUnit unit) {
    checkArgument(spec.accessExpirationTimeUnit == null, "expireAfterAccess already set");
    spec.accessExpirationDuration = duration;
    spec.accessExpirationTimeUnit = unit;
  }
}
```

##### WriteDurationParser

继承了DurationParser，用于解析expireAfterWrite

```java
/** 解析”写入缓存后多久过期“参数 */
static class WriteDurationParser extends DurationParser {
  @Override protected void parseDuration(CacheBuilderSpec spec, long duration, TimeUnit unit) {
    checkArgument(spec.writeExpirationTimeUnit == null, "expireAfterWrite already set");
    spec.writeExpirationDuration = duration;
    spec.writeExpirationTimeUnit = unit;
  }
}
```

##### RefreshDurationParser

继承了DurationParser，用于解析refreshAfterWrite

```java
/** 解析”写入缓存后多久刷新“参数 */
static class RefreshDurationParser extends DurationParser {
  @Override protected void parseDuration(CacheBuilderSpec spec, long duration, TimeUnit unit) {
    checkArgument(spec.refreshTimeUnit == null, "refreshAfterWrite already set");
    spec.refreshDuration = duration;
    spec.refreshTimeUnit = unit;
  }
}
```



### 总结

本类提供了更常用的CacheBuilder的方法，并提供了多个构造器供开发者使用，其中最有趣的是根据配置字符串进行配置。这个其实并不复杂，内部就是通过各种解析器来对字符串进行拆分，然后再调用CacheBuilder的构造器来创建对象。关于构造器显示根据解析类型定义了三个解析器，然后有根据各个变量的类型来创建对应的解析器，这个一步一步的来创建解析器应该也是为了以后方便扩展。