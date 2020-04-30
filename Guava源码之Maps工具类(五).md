# Guava源码之Maps工具类(五)

### 内部类——AbstractFilteredMap

```java
private abstract static class AbstractFilteredMap<K, V>
    extends ImprovedAbstractMap<K, V> {
  final Map<K, V> unfiltered;
  final Predicate<? super Entry<K, V>> predicate;

  AbstractFilteredMap(
      Map<K, V> unfiltered, Predicate<? super Entry<K, V>> predicate) {
    this.unfiltered = unfiltered;
    this.predicate = predicate;
  }

  boolean apply(@Nullable Object key, @Nullable V value) {
    // This method is called only when the key is in the map, implying that
    // key is a K.
    @SuppressWarnings("unchecked")
    K k = (K) key;
    return predicate.apply(Maps.immutableEntry(k, value));
  }

  @Override public V put(K key, V value) {
    checkArgument(apply(key, value));
    return unfiltered.put(key, value);
  }

  @Override public void putAll(Map<? extends K, ? extends V> map) {
    for (Entry<? extends K, ? extends V> entry : map.entrySet()) {
      checkArgument(apply(entry.getKey(), entry.getValue()));
    }
    unfiltered.putAll(map);
  }

  @Override public boolean containsKey(Object key) {
    return unfiltered.containsKey(key) && apply(key, unfiltered.get(key));
  }

  @Override public V get(Object key) {
    V value = unfiltered.get(key);
    return ((value != null) && apply(key, value)) ? value : null;
  }

  @Override public boolean isEmpty() {
    return entrySet().isEmpty();
  }

  @Override public V remove(Object key) {
    return containsKey(key) ? unfiltered.remove(key) : null;
  }

  @Override
  Collection<V> createValues() {
    return new FilteredMapValues<K, V>(this, unfiltered, predicate);
  }
}
```

- 首先它是继承了`ImprovedAbstractMap`，是在父类的基础上进行了扩展
- 定义了两个属性：Map类型的unfiltered；和Predicate类型的predicate。这里再说明一下，这个Predicate是Guava中定义的一个接口，它有一个`apply()`方法，这个方法会根据我们输入的对象返回一个true或者false
- 类中定义了一个`apply()`方法：它会先将key强转为K类型，然后使用输入的key和value构建一个`ImmutableEntry`，最后将此entry对象作为参数调用predicate的apply()方法进行判断，最后返回一个true或者false。这个方法可以理解为根据某种条件进行判断，判断key与value组成的entry是否符合预期
- `put()`和`putAll()`方法都是先调用apply()方法来验证键和值是否符合条件，只有当所有的键和值都符合条件，那么才会被加入到unfiltered这个成员变量中
- 还需要注意的是`containsKey()`和`get()`方法也会调用apply()来进行判断，只有当unfiltered中包含此key，并且它们的键和值符合了条件财货返回true；get()方法也同样会判断，仅当存在与key映射的值并且键和值是符合条件的才会返回key映射的那个值，否则也不会报错而是返回一个null
- `remove()`方法则是在删除之前先使用containsKey来进行判断，仅当存在此键才会去执行删除



### 内部类——FilteredMapValues

```java
private static final class FilteredMapValues<K, V> extends Maps.Values<K, V> {
  Map<K, V> unfiltered;
  Predicate<? super Entry<K, V>> predicate;

  FilteredMapValues(Map<K, V> filteredMap, Map<K, V> unfiltered,
      Predicate<? super Entry<K, V>> predicate) {
    super(filteredMap);
    this.unfiltered = unfiltered;
    this.predicate = predicate;
  }

  @Override public boolean remove(Object o) {
    return Iterables.removeFirstMatching(unfiltered.entrySet(),
        Predicates.<Entry<K, V>>and(predicate, Maps.<V>valuePredicateOnEntries(equalTo(o))))
        != null;
  }

  private boolean removeIf(Predicate<? super V> valuePredicate) {
    return Iterables.removeIf(unfiltered.entrySet(), Predicates.<Entry<K, V>>and(
        predicate, Maps.<V>valuePredicateOnEntries(valuePredicate)));
  }

  @Override public boolean removeAll(Collection<?> collection) {
    return removeIf(in(collection));
  }

  @Override public boolean retainAll(Collection<?> collection) {
    return removeIf(not(in(collection)));
  }

  @Override public Object[] toArray() {
    // creating an ArrayList so filtering happens once
    return Lists.newArrayList(iterator()).toArray();
  }

  @Override public <T> T[] toArray(T[] array) {
    return Lists.newArrayList(iterator()).toArray(array);
  }
}
```

- 继承了Maps工具类的Values，整体设计和`AbstractFilteredMap`类似，依旧定义了Map类型的unfiltered属性和Predicate类型的predicate属性使用这两个属性来进行过滤
- 类里面主要定义了与**删除**相关的方法，其中：
  - `remove(Object o)`会去查找与此对象相同的那个键值对然后删除。主要是调用了Iterables工具类的方法
  - `removeIf(Predicate<? super V> predicate)`：会根据我们输入的“条件”来删除键值对。主要也是调用了Iterables工具类的方法
  - `removeAll()`和`retainAll()`则是直接调用现成的removeIf()方法，将两种条件传入得出结果



### 内部类——FilteredKeyMap

```java
private static class FilteredKeyMap<K, V> extends AbstractFilteredMap<K, V> {
  Predicate<? super K> keyPredicate;

  FilteredKeyMap(Map<K, V> unfiltered, Predicate<? super K> keyPredicate,
      Predicate<? super Entry<K, V>> entryPredicate) {
    super(unfiltered, entryPredicate);
    this.keyPredicate = keyPredicate;
  }

  @Override
  protected Set<Entry<K, V>> createEntrySet() {
    return Sets.filter(unfiltered.entrySet(), predicate);
  }

  @Override
  Set<K> createKeySet() {
    return Sets.filter(unfiltered.keySet(), keyPredicate);
  }

  // The cast is called only when the key is in the unfiltered map, implying
  // that key is a K.
  @Override
  @SuppressWarnings("unchecked")
  public boolean containsKey(Object key) {
    return unfiltered.containsKey(key) && keyPredicate.apply((K) key);
  }
}
```

- 它是直接继承了`AbstractFilteredMap`，在其父类的基础上又定义了新的成员变量**keyPredicate**来代表key的条件

- 它内部重写了`ImprovedAbstractMap`的createKeySet()方法，直接通过Sets工具类的filter()方法获取到过滤后的键值对集合，条件使用的是父类的predicate属性(因为是要对键值对进行过滤)

- 同样重写了createKeySet方法，并使用Sets工具类的filter()方法获取过滤后的键的集合，与上面的方法不同的是，获取键的集合使用的是keyPredicate这个与键相关的条件

  - 关于Sets的filter()方法：

    ```java
    public static <E> Set<E> filter(
        Set<E> unfiltered, Predicate<? super E> predicate) {
      if (unfiltered instanceof SortedSet) {
        return filter((SortedSet<E>) unfiltered, predicate);
      }
      if (unfiltered instanceof FilteredSet) {
        // Support clear(), removeAll(), and retainAll() when filtering a filtered
        // collection.
        FilteredSet<E> filtered = (FilteredSet<E>) unfiltered;
        Predicate<E> combinedPredicate
            = Predicates.<E>and(filtered.predicate, predicate);
        return new FilteredSet<E>(
            (Set<E>) filtered.unfiltered, combinedPredicate);
      }
    
      return new FilteredSet<E>(
          checkNotNull(unfiltered), checkNotNull(predicate));
    }
    ```

    可以看到它里面对SortedSet和FilteredSet进行了不同的处理

- 另外对`containsKey()`也进行了重写，这次使用了keyPredicate这个针对key的策略进行过滤

