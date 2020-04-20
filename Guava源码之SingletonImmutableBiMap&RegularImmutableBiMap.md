# Guava源码之SingletonImmutableBiMap&RegularImmutableBiMap

### SingletonImmutableBiMap

首先来看`SingletonImmutableBiMap`类，如其名称它是一个“只保存一个Entry的不可变双向Map的实现”。由于其内部只保存一个Entry，所以这个类的代码也很少。完整代码如下：

```java
@GwtCompatible(serializable = true, emulated = true)
@SuppressWarnings("serial") // uses writeReplace(), not default serialization
final class SingletonImmutableBiMap<K, V> extends ImmutableBiMap<K, V> {

  final transient K singleKey;
  final transient V singleValue;

  SingletonImmutableBiMap(K singleKey, V singleValue) {
    checkEntryNotNull(singleKey, singleValue);
    this.singleKey = singleKey;
    this.singleValue = singleValue;
  }

  private SingletonImmutableBiMap(K singleKey, V singleValue, ImmutableBiMap<V, K> inverse) {
    this.singleKey = singleKey;
    this.singleValue = singleValue;
    this.inverse = inverse;
  }

  @Override
  public V get(@Nullable Object key) {
    return singleKey.equals(key) ? singleValue : null;
  }

  @Override
  public int size() {
    return 1;
  }

  @Override
  public void forEach(BiConsumer<? super K, ? super V> action) {
    checkNotNull(action).accept(singleKey, singleValue);
  }

  @Override
  public boolean containsKey(@Nullable Object key) {
    return singleKey.equals(key);
  }

  @Override
  public boolean containsValue(@Nullable Object value) {
    return singleValue.equals(value);
  }

  @Override
  boolean isPartialView() {
    return false;
  }

  @Override
  ImmutableSet<Entry<K, V>> createEntrySet() {
    return ImmutableSet.of(Maps.immutableEntry(singleKey, singleValue));
  }

  @Override
  ImmutableSet<K> createKeySet() {
    return ImmutableSet.of(singleKey);
  }

  @LazyInit @RetainedWith transient ImmutableBiMap<V, K> inverse;

  @Override
  public ImmutableBiMap<V, K> inverse() {
    // racy single-check idiom
    ImmutableBiMap<V, K> result = inverse;
    if (result == null) {
      return inverse = new SingletonImmutableBiMap<>(singleValue, singleKey, this);
    } else {
      return result;
    }
  }
}
```

通过看class的注释可以看出：

1. 它是一个最终实现类，使用final修饰。
2. 它仅仅继承了`ImmutableBiMap`类，没有实现其他接口。

再看其属性，仅仅只有两个**singleKey**和**singleValue**，用于保存唯一Entry的键和值。

它一共有两个构造器，一个只有两个参数即键和值，另一个在他们的基础上多了一个**inverse**反视图。

它的所有方法都是重写了父类的方法来适应自己“仅保存自己一个Entry”的特点的。每个方法的实现都很简单。甚至在inverse()方法中直接new了一个SingletonImmutableBiMap来作为反视图。

### RegularImmutableBiMap

这个相对于上面那个类就复杂多了，还记得在它们的父类ImmutableBiMap中，只有在构造仅包含一个Entry的情况下才会用到SingletonImmutableBiMap，其他的情况下基本都是使用RegularImmutableBiMap来构造，这也注定了此类比上一个类更复杂。

下面正式开始介绍：

```java
@GwtCompatible(serializable = true, emulated = true)
@SuppressWarnings("serial") // uses writeReplace(), not default serialization
class RegularImmutableBiMap<K, V> extends ImmutableBiMap<K, V> {
```

首先它是一个包含零个或多个映射的双向Map。在类的声明上面它也是仅仅继承了`ImmutableBiMap`这一个类，并没有实现其他接口。

##### 属性：

```java
  static final RegularImmutableBiMap<Object, Object> EMPTY =
      new RegularImmutableBiMap<>(
          null, null, (Entry<Object, Object>[]) ImmutableMap.EMPTY_ENTRY_ARRAY, 0, 0);

  static final double MAX_LOAD_FACTOR = 1.2;

  private final transient ImmutableMapEntry<K, V>[] keyTable;
  private final transient ImmutableMapEntry<K, V>[] valueTable;
  @VisibleForTesting final transient Entry<K, V>[] entries;
  private final transient int mask;
  private final transient int hashCode;
  @LazyInit @RetainedWith private transient ImmutableBiMap<V, K> inverse;
```

- 第一个属性EMPTY是由`static`和`final`修饰的，它调用构造器并将所有参数置为空。在其父类ImmutableBiMap中当没有传入任何映射时会返回这个空的RegularImmutableBiMap。
- 第二个属性MAX_LOAD_FACTOR为最大的负载因子大小，是**1.2**，我记得HashBiMap的是1.0
- 第三第四个分别用来存储键和值的哈希桶
- 第五个属性根据注释根据其注释显示，其是用于测试使用，存储了所有的映射的数组
- 第六个属性mask用于哈希计算
- 第七个属性用于存储哈希值
- 第八个属性用于存储反视图

##### 构造器：

```java
  private RegularImmutableBiMap(
      ImmutableMapEntry<K, V>[] keyTable,
      ImmutableMapEntry<K, V>[] valueTable,
      Entry<K, V>[] entries,
      int mask,
      int hashCode) {
    this.keyTable = keyTable;
    this.valueTable = valueTable;
    this.entries = entries;
    this.mask = mask;
    this.hashCode = hashCode;
  }
```

此构造器为私有，内部仅仅只是将各个属性赋值。而调用它的方法是：

```java
 static <K, V> ImmutableBiMap<K, V> fromEntryArray(int n, Entry<K, V>[] entryArray) {
    checkPositionIndex(n, entryArray.length);
    int tableSize = Hashing.closedTableSize(n, MAX_LOAD_FACTOR);//调整为适合的桶大小
    int mask = tableSize - 1;
    ImmutableMapEntry<K, V>[] keyTable = createEntryArray(tableSize);//创建key的哈希桶
    ImmutableMapEntry<K, V>[] valueTable = createEntryArray(tableSize);//创建value的哈希桶
    Entry<K, V>[] entries;
    if (n == entryArray.length) {//根据需要创建对应大小的Entry数组
      entries = entryArray;
    } else {
      entries = createEntryArray(n);
    }
    int hashCode = 0;

    for (int i = 0; i < n; i++) {//遍历entryArray
      @SuppressWarnings("unchecked")
      Entry<K, V> entry = entryArray[i];
      K key = entry.getKey();//获取键
      V value = entry.getValue();//获取值
      checkEntryNotNull(key, value);
      int keyHash = key.hashCode();//获取键的哈希
      int valueHash = value.hashCode();//获取值的哈希
      int keyBucket = Hashing.smear(keyHash) & mask;//获取key对应的桶位置
      int valueBucket = Hashing.smear(valueHash) & mask;//获取value对应的桶位置

      ImmutableMapEntry<K, V> nextInKeyBucket = keyTable[keyBucket];//获取key桶位置上的Entry
      int keyBucketLength = checkNoConflictInKeyBucket(key, entry, nextInKeyBucket);
      ImmutableMapEntry<K, V> nextInValueBucket = valueTable[valueBucket];//获取value桶位置上的Entry
      int valueBucketLength = checkNoConflictInValueBucket(value, entry, nextInValueBucket);
      if (keyBucketLength > RegularImmutableMap.MAX_HASH_BUCKET_LENGTH
          || valueBucketLength > RegularImmutableMap.MAX_HASH_BUCKET_LENGTH) {
        return JdkBackedImmutableBiMap.create(n, entryArray);//特殊情况下使用其他类来构造
      }
      ImmutableMapEntry<K, V> newEntry =
          (nextInValueBucket == null && nextInKeyBucket == null)
              ? RegularImmutableMap.makeImmutable(entry, key, value)
              : new NonTerminalImmutableBiMapEntry<>(
                  key, value, nextInKeyBucket, nextInValueBucket);//根据情况创建Entry对象
      keyTable[keyBucket] = newEntry;//将其放入key桶对应位置
      valueTable[valueBucket] = newEntry;//将其放入value桶对应位置
      entries[i] = newEntry;//将此放入Entry数组对应位置
      hashCode += keyHash ^ valueHash;//计算hashcode
    }
    return new RegularImmutableBiMap<>(keyTable, valueTable, entries, mask, hashCode);//调用私有构造器
  }
```

这是个静态方法，可以使用类直接调用，内部调用了其私有的构造器，所以对外来说它才是构造方法。另外此类还有一个构造方法，其内部也调用了这个方法：

```java
  static <K, V> ImmutableBiMap<K, V> fromEntries(Entry<K, V>... entries) {
    return fromEntryArray(entries.length, entries);
  }
```

它同样是静态的，可通过类直接调用。这两个构造方法也在其父类的多个构造其中广泛使用。

##### 内部类：

###### Inverse：

```java
  private final class Inverse extends ImmutableBiMap<V, K> {

    @Override
    public int size() {
      return inverse().size();
    }

    @Override
    public ImmutableBiMap<K, V> inverse() {
      return RegularImmutableBiMap.this;
    }

    @Override
    public void forEach(BiConsumer<? super V, ? super K> action) {
      checkNotNull(action);
      RegularImmutableBiMap.this.forEach((k, v) -> action.accept(v, k));
    }

    @Override
    public K get(@Nullable Object value) {
      if (value == null || valueTable == null) {
        return null;
      }
      int bucket = Hashing.smear(value.hashCode()) & mask;
      for (ImmutableMapEntry<K, V> entry = valueTable[bucket];
          entry != null;
          entry = entry.getNextInValueBucket()) {
        if (value.equals(entry.getValue())) {
          return entry.getKey();
        }
      }
      return null;
    }

    @Override
    ImmutableSet<V> createKeySet() {
      return new ImmutableMapKeySet<>(this);
    }

    @Override
    ImmutableSet<Entry<V, K>> createEntrySet() {
      return new InverseEntrySet();
    }

    final class InverseEntrySet extends ImmutableMapEntrySet<V, K> {
      @Override
      ImmutableMap<V, K> map() {
        return Inverse.this;
      }

      @Override
      boolean isHashCodeFast() {
        return true;
      }

      @Override
      public int hashCode() {
        return hashCode;
      }

      @Override
      public UnmodifiableIterator<Entry<V, K>> iterator() {
        return asList().iterator();
      }

      @Override
      public void forEach(Consumer<? super Entry<V, K>> action) {
        asList().forEach(action);
      }

      @Override
      ImmutableList<Entry<V, K>> createAsList() {
        return new ImmutableAsList<Entry<V, K>>() {
          @Override
          public Entry<V, K> get(int index) {
            Entry<K, V> entry = entries[index];
            return Maps.immutableEntry(entry.getValue(), entry.getKey());
          }

          @Override
          ImmutableCollection<Entry<V, K>> delegateCollection() {
            return InverseEntrySet.this;
          }
        };
      }
    }

    @Override
    boolean isPartialView() {
      return false;
    }

    @Override
    Object writeReplace() {
      return new InverseSerializedForm<>(RegularImmutableBiMap.this);
    }
  }
```

这是个继承了`ImmutableBiMap`的私有内部不可变类，其实其他双向Map都是这么设计的，反向视图都会继承自己本类继承的父类，这样也方便作为反视图来使用。其方法都很容易看懂，在这里不多做赘述。

再来看这个内部类的内部类`InverseEntrySet`：同样使用**final**修饰来表示它不可变，其次这个类继承了ImmutableMapEntrySet。重点说一下其`createAsList()`方法：它返回一个重写了get()方法和delegateCollection()方法的ImmutableAsList对象。get()方法重写的内容是找出键值对并返回ImmutableEntry对象。而delegateCollection()方法则是返回ImmutableEntrySet对象。

######  InverseSerializedForm：

```java
  private static class InverseSerializedForm<K, V> implements Serializable {
    private final ImmutableBiMap<K, V> forward;

    InverseSerializedForm(ImmutableBiMap<K, V> forward) {
      this.forward = forward;
    }

    Object readResolve() {
      return forward.inverse();
    }

    private static final long serialVersionUID = 1;
  }
```

定义了反向视图的输出格式。很多双向Map都有自己的InverseSerializedForm内部类。

##### 其他方法：

######  checkNoConflictInValueBucket：

```java
  @CanIgnoreReturnValue
  private static int checkNoConflictInValueBucket(
      Object value, Entry<?, ?> entry, @Nullable ImmutableMapEntry<?, ?> valueBucketHead) {
    int bucketSize = 0;
    for (; valueBucketHead != null; valueBucketHead = valueBucketHead.getNextInValueBucket()) {
      checkNoConflict(!value.equals(valueBucketHead.getValue()), "value", entry, valueBucketHead);
      bucketSize++;
    }
    return bucketSize;
  }
```

此方法会遍历值哈希桶中所有与给定value值不相同的个数并返回。`checkNoConflict()`这个方法会判断第一个参数是否正确，如果不正确，它会将后三个参数作为错误内容报错。

### 总结：

SingletonImmutableBiMap和RegularImmutableBiMap都是ImmutableBiMap的子类，它们分别应用于不同的情况——存储单个Entry、存储零个或多个Entry。它们也没什么独特的方法，基本功能都在父类中被定义好，自己只是根据自身情况重写了部分方法，算是用在两种环境下的不同实现吧。

