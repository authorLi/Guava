# Guava源码之ImmutableBiMap

### 简介：

这是一个内容永远不会改变的BiMap，从其名称“Immutable”可知。

```java
@GwtCompatible(serializable = true, emulated = true)
public abstract class ImmutableBiMap<K, V> extends ImmutableBiMapFauxverideShim<K, V>
    implements BiMap<K, V> {
```

此类是一个抽象类，由@GwtCompatible修饰，继承了`ImmutableBiMapFauxverideShim`这个抽象类，点进去看这个类才发现此类的唯一拥有的两个方法也被弃用了，但它是继承了`ImmutableMap`的，所以可以认为此BiMap实现类是继承了ImmmutableMap的。

### 类概览：

##### 属性：

该类没有任何自定义的属性。

##### 构造器：

```java
ImmutableBiMap() {}//空构造器，它是可以被调用的，但是它内部什么也没做
```

除此之外它还有多个重载的`of()`方法来返回一个ImmutableBiMap对象

```java
  @SuppressWarnings("unchecked")
  public static <K, V> ImmutableBiMap<K, V> of() {
    return (ImmutableBiMap<K, V>) RegularImmutableBiMap.EMPTY;
  }

  public static <K, V> ImmutableBiMap<K, V> of(K k1, V v1) {
    return new SingletonImmutableBiMap<>(k1, v1);
  }

  public static <K, V> ImmutableBiMap<K, V> of(K k1, V v1, K k2, V v2) {
    return RegularImmutableBiMap.fromEntries(entryOf(k1, v1), entryOf(k2, v2));
  }

  public static <K, V> ImmutableBiMap<K, V> of(K k1, V v1, K k2, V v2, K k3, V v3) {
    return RegularImmutableBiMap.fromEntries(entryOf(k1, v1), entryOf(k2, v2), entryOf(k3, v3));
  }

  public static <K, V> ImmutableBiMap<K, V> of(K k1, V v1, K k2, V v2, K k3, V v3, K k4, V v4) {
    return RegularImmutableBiMap.fromEntries(
        entryOf(k1, v1), entryOf(k2, v2), entryOf(k3, v3), entryOf(k4, v4));
  }

  public static <K, V> ImmutableBiMap<K, V> of(
      K k1, V v1, K k2, V v2, K k3, V v3, K k4, V v4, K k5, V v5) {
    return RegularImmutableBiMap.fromEntries(
        entryOf(k1, v1), entryOf(k2, v2), entryOf(k3, v3), entryOf(k4, v4), entryOf(k5, v5));
  }
```

可以看到，它内部提供了两种方法来创建ImmutableBiMap对象：`RegularImmutableBiMap`和`SingletonImmutableBiMap`。

1. 第一个方法会返回RegularImmutableBiMap，然后将此对象的各种属性置为0，即返回一个空的RegularImmutableBiMap
2. 第二个方法会返回一个只有一个键和一个值的SingletonImmutableBiMap
3. 第三、四、五、六个方法会返回一个RegularImmutableBiMap，内部是调用了其`fromEntries()`方法，这个方法接收一个可变参数，所以可以传入多个Entry，而`entryOf()`方法就是将传入的键和值封装成SimpleImmutableEntry对象，当然这个肯定是实现了Entry接口。

其实看源码不难发现，`RegularImmutableBiMap`和`SingletonImmutableBiMap`都是ImmutableBiMap的子类，这里的构造方法是直接调用了这两个子类的方法来实例化的，看来这是鼓励开发者多利用ImmutableBiMap来开发，而不是直接使用两个子类？另外，这样也就知道为什么此抽象类没有任何自定义的属性了，它把自定义的属性都交给子类去定义了。

> 然后这里会有一个疑问，为什么它的of()方法最多只支持到5个Entry呢，再多就不行了吗？

实际上源码中有提示：当需要加入大于5个的Entry时，推荐使用**builder**来实现。

关于builder，它也提供了两种方法：

```java
  public static <K, V> Builder<K, V> builder() {
    return new Builder<>();//返回一个默认的Builder
  }

  @Beta
  public static <K, V> Builder<K, V> builderWithExpectedSize(int expectedSize) {
    checkNonnegative(expectedSize, "expectedSize");
    return new Builder<>(expectedSize);//返回一个指定大小的Builder
  }
```

##### 内部类：

###### Builder：

```java
public static final class Builder<K, V> extends ImmutableMap.Builder<K, V> {

    public Builder() {}//空构造器

    Builder(int size) {//期望大小的构造器
      super(size);
    }

    @CanIgnoreReturnValue
    @Override
    public Builder<K, V> put(K key, V value) {
      super.put(key, value);//调用父类的put()方法，内部就是创建一个新的Entry并添加到数组中
      return this;
    }
    
    @CanIgnoreReturnValue
    @Override
    public Builder<K, V> put(Entry<? extends K, ? extends V> entry) {
      super.put(entry);//直接放入一个Entry(省的再封装了)
      return this;
    }

    @CanIgnoreReturnValue
    @Override
    public Builder<K, V> putAll(Map<? extends K, ? extends V> map) {
      super.putAll(map);//直接放入一个Map
      return this;
    }

    @CanIgnoreReturnValue
    @Beta
    @Override
    public Builder<K, V> putAll(Iterable<? extends Entry<? extends K, ? extends V>> entries) {
      super.putAll(entries);//直接放入一个Iterable类型的对象
      return this;
    }

    @CanIgnoreReturnValue
    @Beta
    @Override
    public Builder<K, V> orderEntriesByValue(Comparator<? super V> valueComparator) {
      super.orderEntriesByValue(valueComparator);//调用父类方法
      return this;
    }

    @Override
    @CanIgnoreReturnValue
    Builder<K, V> combine(ImmutableMap.Builder<K, V> builder) {
      super.combine(builder);
      return this;
    }

    @Override
    public ImmutableBiMap<K, V> build() {
      switch (size) {//根据实际大小构建
        case 0:
          return of();
        case 1:
          return of(entries[0].getKey(), entries[0].getValue());
        default:
          if (valueComparator != null) {
            if (entriesUsed) {
              entries = Arrays.copyOf(entries, size);
            }
            Arrays.sort(
                entries,
                0,
                size,
                Ordering.from(valueComparator).onResultOf(Maps.<V>valueFunction()));
          }
          entriesUsed = true;
          return RegularImmutableBiMap.fromEntryArray(size, entries);
      }
    }

    @Override
    @VisibleForTesting
    ImmutableBiMap<K, V> buildJdkBacked() {
      checkState(
          valueComparator == null,
          "buildJdkBacked is for tests only, doesn't support orderEntriesByValue");
      switch (size) {
        case 0:
          return of();
        case 1:
          return of(entries[0].getKey(), entries[0].getValue());
        default:
          entriesUsed = true;
          return RegularImmutableBiMap.fromEntryArray(size, entries);
      }
    }
  }
```

此内部类继承了ImmutableMap的Builder内部类。可以看到其方法大部分都是调用了其父类的方法。此外，官方还给了一个例子：

```java
static final ImmutableBiMap<String, Integer> WORD_TO_INT =
    new ImmutableBiMap.Builder<String, Integer>()
        .put("one", 1)
        .put("two", 2)
        .put("three", 3)
        .build();
```

这种构造方式其实我之前看《Effective Java》看到过，感觉这样的方式来构建更加清晰明了，这应该也是一种高性能的设计方式吧。

###### SerializedForm：

```java
  private static class SerializedForm extends ImmutableMap.SerializedForm {
    SerializedForm(ImmutableBiMap<?, ?> bimap) {
      super(bimap);//就是调用了父类方法，填充keys和values数组
    }

    @Override
    Object readResolve() {
      Builder<Object, Object> builder = new Builder<>();
      return createMap(builder);//将keys和values数组的值放入到builder中完成构建
    }

    private static final long serialVersionUID = 0;
  }
```

此内部类继承了ImmutableMap的SerializedForm，这还是内部调用了父类的方法。

##### 其他方法：

###### copyOf():

```java
  public static <K, V> ImmutableBiMap<K, V> copyOf(Map<? extends K, ? extends V> map) {
    if (map instanceof ImmutableBiMap) {
      @SuppressWarnings("unchecked") // safe since map is not writable
      ImmutableBiMap<K, V> bimap = (ImmutableBiMap<K, V>) map;
      // TODO(lowasser): if we need to make a copy of a BiMap because the
      // forward map is a view, don't make a copy of the non-view delegate map
      if (!bimap.isPartialView()) {
        return bimap;
      }
    }
    return copyOf(map.entrySet());
  }

  @Beta
  public static <K, V> ImmutableBiMap<K, V> copyOf(
      Iterable<? extends Entry<? extends K, ? extends V>> entries) {
    @SuppressWarnings("unchecked") // we'll only be using getKey and getValue, which are covariant
    Entry<K, V>[] entryArray = (Entry<K, V>[]) Iterables.toArray(entries, EMPTY_ENTRY_ARRAY);
    switch (entryArray.length) {
      case 0:
        return of();
      case 1:
        Entry<K, V> entry = entryArray[0];
        return of(entry.getKey(), entry.getValue());
      default:
        /*
         * The current implementation will end up using entryArray directly, though it will write
         * over the (arbitrary, potentially mutable) Entry objects actually stored in entryArray.
         */
        return RegularImmutableBiMap.fromEntries(entryArray);
    }
  }
```

第一个copyOf()方法会调用第二个copyOf()方法。顺便插一句：`@Beta`这个注解也是个说明性注解，它并不表示功能的实现有问题，而是说此方法可能在未来由于某种原因就不被支持了。这个方法就是将Map或Iterable对象的数据复制到一个新的ImmutableBiMap对象中(里面还根据输入的大小调用了不同的构造器)。

###### inverse()：

```java
  @Override
  public abstract ImmutableBiMap<V, K> inverse();
```

获取反视图，这是个抽象方法，待实现。

###### createValue()&forcePut()：

```java
  @Override
  final ImmutableSet<V> createValues() {
    throw new AssertionError("should never be called");
  }

  @CanIgnoreReturnValue
  @Deprecated
  @Override
  public V forcePut(K key, V value) {
    throw new UnsupportedOperationException();
  }
```

这两个方法保证了当我们试图去创建值或者强制插入时会抛出错误以提醒我们这是一个“不可变的”BiMap。

### 总结：

整体看下来，感觉都是在调用父类的方法，或者其实现子类的方法，这样的类通常适合拿来进行开发，我理解就像是一个对外开放的门户吧。这么看下来它内部使用`ImmutableMap`、`RegularImmutableBiMap`和`SingletonImmutableBiMap`这三个类的方法比较多，可能它们比较重要吧。那么下次就把RegularImmutableBiMap和SingletonImmutableBiMap一起写并相互比较下吧。

