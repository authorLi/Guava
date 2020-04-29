# Guava源码之Maps工具类(四)

### 内部类——TransformedEntriesMap

```java
static class TransformedEntriesMap<K, V1, V2>
    extends ImprovedAbstractMap<K, V2> {
  final Map<K, V1> fromMap;
  final EntryTransformer<? super K, ? super V1, V2> transformer;

  TransformedEntriesMap(
      Map<K, V1> fromMap,
      EntryTransformer<? super K, ? super V1, V2> transformer) {
    this.fromMap = checkNotNull(fromMap);
    this.transformer = checkNotNull(transformer);
  }

  @Override public int size() {
    return fromMap.size();
  }

  @Override public boolean containsKey(Object key) {
    return fromMap.containsKey(key);
  }

  // safe as long as the user followed the <b>Warning</b> in the javadoc
  @SuppressWarnings("unchecked")
  @Override public V2 get(Object key) {
    V1 value = fromMap.get(key);
    return (value != null || fromMap.containsKey(key))
        ? transformer.transformEntry((K) key, value)
        : null;
  }

  // safe as long as the user followed the <b>Warning</b> in the javadoc
  @SuppressWarnings("unchecked")
  @Override public V2 remove(Object key) {
    return fromMap.containsKey(key)
        ? transformer.transformEntry((K) key, fromMap.remove(key))
        : null;
  }

  @Override public void clear() {
    fromMap.clear();
  }

  @Override public Set<K> keySet() {
    return fromMap.keySet();
  }

  @Override
  protected Set<Entry<K, V2>> createEntrySet() {
    return new EntrySet<K, V2>() {
      @Override Map<K, V2> map() {
        return TransformedEntriesMap.this;
      }

      @Override public Iterator<Entry<K, V2>> iterator() {
        return Iterators.transform(fromMap.entrySet().iterator(),
            Maps.<K, V1, V2>asEntryToEntryFunction(transformer));
      }
    };
  }
}
```

- 首先是继承了`ImprovedAbstractMap`,前面说过这个类是在AbstractMap基础上对键值对、键集合和值集合做了重写

- 里面定义了两个属性：一个是Map类型的fromMap，就是传入的待我们对其进行操作的Map；还有一个就是EntryTransformer类型的transformer。这个类型是Maps工具类中定义的一个接口，其内部只有一个方法，就是根据输入的键和值去获得另一个值，具体设计如下：

  - ```java
    public interface EntryTransformer<K, V1, V2> {
        V2 transformEntry(@Nullable K key, @Nullable V1 value);//传入的键和值必须非空
      }
    ```

- 类里面定义了一个构造器，依旧是在赋值前会进行非空的判断

- 由于存在属性fromMap，所以类内部重写了`size()`、`containsKey()`、`clear()`、`keySet()`这几个方法，让它们都去调用fromMap的对应方法

- `remove()`方法也被重写了，但是返回的并不是键所对应的值，而是会调用transformer的方法返回根据键和值所得出的值

- 与remove对应，`get()`方法也不再是直接返回键所对应的值，而是也会去调用transformer的方法返回根据键和值所得出的那个值

- 还有一点，它重写了其父类中的抽象方法`createEntrySet()`，返回的EntrySet对象中自定义了`map()`方法，返回的是`TransformedEntriesMap`类型的当前对象；同样也自定义了`iterator()`方法，根据fromMap和transformer返回了一个迭代器，只要是建立键K与通过键和值所计算的值V2的关系



### 内部类——TransformedEntriesSortedMap

```java
static class TransformedEntriesSortedMap<K, V1, V2>
    extends TransformedEntriesMap<K, V1, V2> implements SortedMap<K, V2> {

  protected SortedMap<K, V1> fromMap() {
    return (SortedMap<K, V1>) fromMap;
  }

  TransformedEntriesSortedMap(SortedMap<K, V1> fromMap,
      EntryTransformer<? super K, ? super V1, V2> transformer) {
    super(fromMap, transformer);
  }

  @Override public Comparator<? super K> comparator() {
    return fromMap().comparator();
  }

  @Override public K firstKey() {
    return fromMap().firstKey();
  }

  @Override public SortedMap<K, V2> headMap(K toKey) {
    return transformEntries(fromMap().headMap(toKey), transformer);
  }

  @Override public K lastKey() {
    return fromMap().lastKey();
  }

  @Override public SortedMap<K, V2> subMap(K fromKey, K toKey) {
    return transformEntries(
        fromMap().subMap(fromKey, toKey), transformer);
  }

  @Override public SortedMap<K, V2> tailMap(K fromKey) {
    return transformEntries(fromMap().tailMap(fromKey), transformer);
  }
}
```

- 此类继承了`TransformedEntriesMap`和`SortedMap`于是具备了两个父类的功能

- 里面的方法多是重写了`SortedMap`接口的方法

- 定义了`fromMap()`方法，强转并返回了父类的属性fromMap

- `comparator()`、`firstKey()`、`lastKey()`这些方法都是先调用了fromMap()方法获得属性后再调用fromMap的对应方法

- `headMap()`、`subMap()`和`tailMap()`这三个方法则是：

  - 先去调用Maps工具类的`transformEntries()`方法，将fromMap的对应方法的值和属性transforer作为参数传入

    ```java
    public static <K, V1, V2> SortedMap<K, V2> transformEntries(
        SortedMap<K, V1> fromMap,
        EntryTransformer<? super K, ? super V1, V2> transformer) {
      return Platform.mapsTransformEntriesSortedMap(fromMap, transformer);
    }
    ```

  - 然后它内部又调用了Platform的`mapsTransformEntriesSortedMap()`方法，这个方法会在内部判断fromMap的类型然后创建对应的对象

    ```java
    static <K, V1, V2> SortedMap<K, V2> mapsTransformEntriesSortedMap(
        SortedMap<K, V1> fromMap,
        EntryTransformer<? super K, ? super V1, V2> transformer) {
      return (fromMap instanceof NavigableMap)
          ? Maps.transformEntries((NavigableMap<K, V1>) fromMap, transformer)
          : Maps.transformEntriesIgnoreNavigable(fromMap, transformer);
    }
    ```

  - 最后可能是会创建两种对象：

    ```java
    @GwtIncompatible("NavigableMap")
    public static <K, V1, V2> NavigableMap<K, V2> transformEntries(
        final NavigableMap<K, V1> fromMap,
        EntryTransformer<? super K, ? super V1, V2> transformer) {
      return new TransformedEntriesNavigableMap<K, V1, V2>(fromMap, transformer);
    }
    
    static <K, V1, V2> SortedMap<K, V2> transformEntriesIgnoreNavigable(
        SortedMap<K, V1> fromMap,
        EntryTransformer<? super K, ? super V1, V2> transformer) {
      return new TransformedEntriesSortedMap<K, V1, V2>(fromMap, transformer);
    }
    ```



### 内部类——TransformedEntriesNavigableMap

```java
@GwtIncompatible("NavigableMap")
private static class TransformedEntriesNavigableMap<K, V1, V2>
    extends TransformedEntriesSortedMap<K, V1, V2>
    implements NavigableMap<K, V2> {

  TransformedEntriesNavigableMap(NavigableMap<K, V1> fromMap,
      EntryTransformer<? super K, ? super V1, V2> transformer) {
    super(fromMap, transformer);
  }

  @Override public Entry<K, V2> ceilingEntry(K key) {
    return transformEntry(fromMap().ceilingEntry(key));
  }

  @Override public K ceilingKey(K key) {
    return fromMap().ceilingKey(key);
  }

  @Override public NavigableSet<K> descendingKeySet() {
    return fromMap().descendingKeySet();
  }

  @Override public NavigableMap<K, V2> descendingMap() {
    return transformEntries(fromMap().descendingMap(), transformer);
  }

  @Override public Entry<K, V2> firstEntry() {
    return transformEntry(fromMap().firstEntry());
  }
  @Override public Entry<K, V2> floorEntry(K key) {
    return transformEntry(fromMap().floorEntry(key));
  }

  @Override public K floorKey(K key) {
    return fromMap().floorKey(key);
  }

  @Override public NavigableMap<K, V2> headMap(K toKey) {
    return headMap(toKey, false);
  }

  @Override public NavigableMap<K, V2> headMap(K toKey, boolean inclusive) {
    return transformEntries(
        fromMap().headMap(toKey, inclusive), transformer);
  }

  @Override public Entry<K, V2> higherEntry(K key) {
    return transformEntry(fromMap().higherEntry(key));
  }

  @Override public K higherKey(K key) {
    return fromMap().higherKey(key);
  }

  @Override public Entry<K, V2> lastEntry() {
    return transformEntry(fromMap().lastEntry());
  }

  @Override public Entry<K, V2> lowerEntry(K key) {
    return transformEntry(fromMap().lowerEntry(key));
  }

  @Override public K lowerKey(K key) {
    return fromMap().lowerKey(key);
  }

  @Override public NavigableSet<K> navigableKeySet() {
    return fromMap().navigableKeySet();
  }

  @Override public Entry<K, V2> pollFirstEntry() {
    return transformEntry(fromMap().pollFirstEntry());
  }

  @Override public Entry<K, V2> pollLastEntry() {
    return transformEntry(fromMap().pollLastEntry());
  }

  @Override public NavigableMap<K, V2> subMap(
      K fromKey, boolean fromInclusive, K toKey, boolean toInclusive) {
    return transformEntries(
        fromMap().subMap(fromKey, fromInclusive, toKey, toInclusive),
        transformer);
  }

  @Override public NavigableMap<K, V2> subMap(K fromKey, K toKey) {
    return subMap(fromKey, true, toKey, false);
  }

  @Override public NavigableMap<K, V2> tailMap(K fromKey) {
    return tailMap(fromKey, true);
  }

  @Override public NavigableMap<K, V2> tailMap(K fromKey, boolean inclusive) {
    return transformEntries(
        fromMap().tailMap(fromKey, inclusive), transformer);
  }

  @Nullable
  private Entry<K, V2> transformEntry(@Nullable Entry<K, V1> entry) {
    return (entry == null) ? null : Maps.transformEntry(transformer, entry);
  }

  @Override protected NavigableMap<K, V1> fromMap() {
    return (NavigableMap<K, V1>) super.fromMap();
  }
}
```

- 此类继承了`TransformedEntriesSortedMap`和`NavigableMap`于是便拥有了两个父类的特性
- 构造器依然是指定fromMap和transformer这两个属性
- 在此类中大多是重写了NavigableMap的方法且很多方法都是直接根据父类`TransformedEntriesSortedMap`的各种方法运算的，代码没什么特殊的，关键是功能的实现思路吧

### 总结

这次看了Transformed开头的三个内部类，它们之间是依次继承的关系，且每次子类继承时都会多实现一种接口来扩展新的特性，这个设计就像是Spring中ApplicationContext的设计：每实现一个功能就继承父类和一个功能接口来扩展父类的功能特性，这个设计还是很好的。