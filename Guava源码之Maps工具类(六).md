# Guava源码之Maps工具类(六)

### 内部类——FilteredEntryMap

```java
static class FilteredEntryMap<K, V> extends AbstractFilteredMap<K, V> {
  final Set<Entry<K, V>> filteredEntrySet;

  FilteredEntryMap(
      Map<K, V> unfiltered, Predicate<? super Entry<K, V>> entryPredicate) {
    super(unfiltered, entryPredicate);
    filteredEntrySet = Sets.filter(unfiltered.entrySet(), predicate);
  }

  @Override
  protected Set<Entry<K, V>> createEntrySet() {
    return new EntrySet();
  }

  private class EntrySet extends ForwardingSet<Entry<K, V>> {
    @Override protected Set<Entry<K, V>> delegate() {
      return filteredEntrySet;
    }

    @Override public Iterator<Entry<K, V>> iterator() {
      return new TransformedIterator<Entry<K, V>, Entry<K, V>>(filteredEntrySet.iterator()) {
        @Override
        Entry<K, V> transform(final Entry<K, V> entry) {
          return new ForwardingMapEntry<K, V>() {
            @Override
            protected Entry<K, V> delegate() {
              return entry;
            }

            @Override
            public V setValue(V newValue) {
              checkArgument(apply(getKey(), newValue));
              return super.setValue(newValue);
            }
          };
        }
      };
    }
  }

  @Override
  Set<K> createKeySet() {
    return new KeySet();
  }

  class KeySet extends Maps.KeySet<K, V> {
    KeySet() {
      super(FilteredEntryMap.this);
    }

    @Override public boolean remove(Object o) {
      if (containsKey(o)) {
        unfiltered.remove(o);
        return true;
      }
      return false;
    }

    private boolean removeIf(Predicate<? super K> keyPredicate) {
      return Iterables.removeIf(unfiltered.entrySet(), Predicates.<Entry<K, V>>and(
          predicate, Maps.<K>keyPredicateOnEntries(keyPredicate)));
    }

    @Override
    public boolean removeAll(Collection<?> c) {
      return removeIf(in(c));
    }

    @Override
    public boolean retainAll(Collection<?> c) {
      return removeIf(not(in(c)));
    }

    @Override public Object[] toArray() {
      // creating an ArrayList so filtering happens once
      return Lists.newArrayList(iterator()).toArray();
    }

    @Override public <T> T[] toArray(T[] array) {
      return Lists.newArrayList(iterator()).toArray(array);
    }
  }
}
```

- 此类继承了`AbstractFilteredMap`，主要还是为了使用其父类的unfiltered和predicate来进行过滤。

- 除此之外此类还提供了一个用来存储键值对的集合filteredEntrySet

- 在构造器中不仅调用了父类的方法设置了unfiltered和predicate，还使用了Sets工具类的`filter()`方法根据predicate的条件取出unfiltered的键值对集合并进行过滤，最终赋值给filteredEntrySet成员变量

- 重写了父类中的`createEntrySet()`方法，直接返回了一个定义在本类中的内部类EnterySet类的对象

- EntrySet内部类：

  ```java
  private class EntrySet extends ForwardingSet<Entry<K, V>> {
    @Override protected Set<Entry<K, V>> delegate() {
      return filteredEntrySet;
    }
  
    @Override public Iterator<Entry<K, V>> iterator() {
      return new TransformedIterator<Entry<K, V>, Entry<K, V>>(filteredEntrySet.iterator()) {
        @Override
        Entry<K, V> transform(final Entry<K, V> entry) {
          return new ForwardingMapEntry<K, V>() {
            @Override
            protected Entry<K, V> delegate() {
              return entry;
            }
  
            @Override
            public V setValue(V newValue) {
              checkArgument(apply(getKey(), newValue));
              return super.setValue(newValue);
            }
          };
        }
      };
    }
  }
  ```

  继承了`forwardingSet`并重写了`delegate()`方法来返回filteredEntrySet和重写`iterator()`方法来返回一个自定义的迭代器此迭代器可以遍历反视图

- 重写了父类中的`createKeySet()`方法，返回了一个在本类中定义的内部类KeySet的对象

- KeySet：

  ```java
  class KeySet extends Maps.KeySet<K, V> {
    KeySet() {
      super(FilteredEntryMap.this);
    }
  
    @Override public boolean remove(Object o) {
      if (containsKey(o)) {
        unfiltered.remove(o);
        return true;
      }
      return false;
    }
  
    private boolean removeIf(Predicate<? super K> keyPredicate) {
      return Iterables.removeIf(unfiltered.entrySet(), Predicates.<Entry<K, V>>and(
          predicate, Maps.<K>keyPredicateOnEntries(keyPredicate)));
    }
  
    @Override
    public boolean removeAll(Collection<?> c) {
      return removeIf(in(c));
    }
  
    @Override
    public boolean retainAll(Collection<?> c) {
      return removeIf(not(in(c)));
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

  继承了Maps工具类的KeySet。构造器传入的是当前的FilteredEntryMap对象。`remove()`方法是直接查看是否包含，然后直接删除。其他方法与**FilteredMapValues**的同名方法相同



### 内部类——FilteredEntrySortedMap

```java
private static class FilteredEntrySortedMap<K, V>
      extends FilteredEntryMap<K, V> implements SortedMap<K, V> {

    FilteredEntrySortedMap(SortedMap<K, V> unfiltered,
        Predicate<? super Entry<K, V>> entryPredicate) {
      super(unfiltered, entryPredicate);
    }

    SortedMap<K, V> sortedMap() {
      return (SortedMap<K, V>) unfiltered;
    }

    @Override public SortedSet<K> keySet() {
      return (SortedSet<K>) super.keySet();
    }

    @Override
    SortedSet<K> createKeySet() {
      return new SortedKeySet();
    }

    class SortedKeySet extends KeySet implements SortedSet<K> {
      @Override
      public Comparator<? super K> comparator() {
        return sortedMap().comparator();
      }

      @Override
      public SortedSet<K> subSet(K fromElement, K toElement) {
        return (SortedSet<K>) subMap(fromElement, toElement).keySet();
      }

      @Override
      public SortedSet<K> headSet(K toElement) {
        return (SortedSet<K>) headMap(toElement).keySet();
      }

      @Override
      public SortedSet<K> tailSet(K fromElement) {
        return (SortedSet<K>) tailMap(fromElement).keySet();
      }

      @Override
      public K first() {
        return firstKey();
      }

      @Override
      public K last() {
        return lastKey();
      }
    }

    @Override public Comparator<? super K> comparator() {
      return sortedMap().comparator();
    }

    @Override public K firstKey() {
      // correctly throws NoSuchElementException when filtered map is empty.
      return keySet().iterator().next();
    }

    @Override public K lastKey() {
      SortedMap<K, V> headMap = sortedMap();
      while (true) {
        // correctly throws NoSuchElementException when filtered map is empty.
        K key = headMap.lastKey();
        if (apply(key, unfiltered.get(key))) {
          return key;
        }
        headMap = sortedMap().headMap(key);
      }
    }

    @Override public SortedMap<K, V> headMap(K toKey) {
      return new FilteredEntrySortedMap<K, V>(sortedMap().headMap(toKey), predicate);
    }

    @Override public SortedMap<K, V> subMap(K fromKey, K toKey) {
      return new FilteredEntrySortedMap<K, V>(
          sortedMap().subMap(fromKey, toKey), predicate);
    }

    @Override public SortedMap<K, V> tailMap(K fromKey) {
      return new FilteredEntrySortedMap<K, V>(
          sortedMap().tailMap(fromKey), predicate);
    }
  }
```

- 在继承了`FilteredEntryMap`的基础上又实现了`SortedMap`接口，这样就在原来的基础上又扩展了对Map的排序

- 其中`sortedMap()`这个方法是很重要的，虽然它内部仅仅只是将父类的unfiltered强转为了SortedMap类型

- 其他大部分方法都是直接通过调用sortedMap()这个方法来获得已排序的unfiltered对象，然后再进行对应的操作

- 此类也对`createKeySet()`方法进行的重写，返回了一个本类自定义的内部类SortedKeySet类的对象

- 内部类SortedKeySet：

  ```java
  class SortedKeySet extends KeySet implements SortedSet<K> {
    @Override
    public Comparator<? super K> comparator() {
      return sortedMap().comparator();
    }
  
    @Override
    public SortedSet<K> subSet(K fromElement, K toElement) {
      return (SortedSet<K>) subMap(fromElement, toElement).keySet();
    }
  
    @Override
    public SortedSet<K> headSet(K toElement) {
      return (SortedSet<K>) headMap(toElement).keySet();
    }
  
    @Override
    public SortedSet<K> tailSet(K fromElement) {
      return (SortedSet<K>) tailMap(fromElement).keySet();
    }
  
    @Override
    public K first() {
      return firstKey();
    }
  
    @Override
    public K last() {
      return lastKey();
    }
  }
  ```

  可以看到它在继承了Maps工具类的KeySet的基础上又实现了SortedSet这个接口，同样也使其变得有序，方法都是通过调用其外部类的方法实现的



### 内部类——FilteredEntryNavigableMap

```java
@GwtIncompatible("NavigableMap")
private static class FilteredEntryNavigableMap<K, V> extends AbstractNavigableMap<K, V> {
  private final NavigableMap<K, V> unfiltered;
  private final Predicate<? super Entry<K, V>> entryPredicate;
  private final Map<K, V> filteredDelegate;

  FilteredEntryNavigableMap(
      NavigableMap<K, V> unfiltered, Predicate<? super Entry<K, V>> entryPredicate) {
    this.unfiltered = checkNotNull(unfiltered);
    this.entryPredicate = entryPredicate;
    this.filteredDelegate = new FilteredEntryMap<K, V>(unfiltered, entryPredicate);
  }

  @Override
  public Comparator<? super K> comparator() {
    return unfiltered.comparator();
  }

  @Override
  public NavigableSet<K> navigableKeySet() {
    return new Maps.NavigableKeySet<K, V>(this) {
      @Override
      public boolean removeAll(Collection<?> c) {
        return Iterators.removeIf(unfiltered.entrySet().iterator(),
            Predicates.<Entry<K, V>>and(entryPredicate, Maps.<K>keyPredicateOnEntries(in(c))));
      }

      @Override
      public boolean retainAll(Collection<?> c) {
        return Iterators.removeIf(unfiltered.entrySet().iterator(), Predicates.<Entry<K, V>>and(
            entryPredicate, Maps.<K>keyPredicateOnEntries(not(in(c)))));
      }
    };
  }

  @Override
  public Collection<V> values() {
    return new FilteredMapValues<K, V>(this, unfiltered, entryPredicate);
  }

  @Override
  Iterator<Entry<K, V>> entryIterator() {
    return Iterators.filter(unfiltered.entrySet().iterator(), entryPredicate);
  }

  @Override
  Iterator<Entry<K, V>> descendingEntryIterator() {
    return Iterators.filter(unfiltered.descendingMap().entrySet().iterator(), entryPredicate);
  }

  @Override
  public int size() {
    return filteredDelegate.size();
  }

  @Override
  public boolean isEmpty() {
    return !Iterables.any(unfiltered.entrySet(), entryPredicate);
  }

  @Override
  @Nullable
  public V get(@Nullable Object key) {
    return filteredDelegate.get(key);
  }

  @Override
  public boolean containsKey(@Nullable Object key) {
    return filteredDelegate.containsKey(key);
  }

  @Override
  public V put(K key, V value) {
    return filteredDelegate.put(key, value);
  }

  @Override
  public V remove(@Nullable Object key) {
    return filteredDelegate.remove(key);
  }

  @Override
  public void putAll(Map<? extends K, ? extends V> m) {
    filteredDelegate.putAll(m);
  }

  @Override
  public void clear() {
    filteredDelegate.clear();
  }

  @Override
  public Set<Entry<K, V>> entrySet() {
    return filteredDelegate.entrySet();
  }

  @Override
  public Entry<K, V> pollFirstEntry() {
    return Iterables.removeFirstMatching(unfiltered.entrySet(), entryPredicate);
  }

  @Override
  public Entry<K, V> pollLastEntry() {
    return Iterables.removeFirstMatching(unfiltered.descendingMap().entrySet(), entryPredicate);
  }

  @Override
  public NavigableMap<K, V> descendingMap() {
    return filterEntries(unfiltered.descendingMap(), entryPredicate);
  }

  @Override
  public NavigableMap<K, V> subMap(
      K fromKey, boolean fromInclusive, K toKey, boolean toInclusive) {
    return filterEntries(
        unfiltered.subMap(fromKey, fromInclusive, toKey, toInclusive), entryPredicate);
  }

  @Override
  public NavigableMap<K, V> headMap(K toKey, boolean inclusive) {
    return filterEntries(unfiltered.headMap(toKey, inclusive), entryPredicate);
  }

  @Override
  public NavigableMap<K, V> tailMap(K fromKey, boolean inclusive) {
    return filterEntries(unfiltered.tailMap(fromKey, inclusive), entryPredicate);
  }
}
```

- 继承了`AbstractNavigableMap`，这个Guava对util包下的NavigableMap的基本实现,这使得FilteredEntryMap拥有了NavigableMap的实现。也正是因为实现了AbstractNavigableMap，FilteredEntryNavigableMap无法再继承FilteredEntryMap

- 共定义了三个属性：NavigableMap类型的**unfiltered**、Predicate类型的**entryPredicate**还有Map类型的**filteredDelegate**

- 构造器中主要是检查了unfiltered中的参数并赋值、赋值entryPredicate还有最关键的就是用这两个属性创建了一个新的`FilteredEntryMap`对象并赋值给filteredDelegate

- `navigableKeySet()`方法会返回一个Maps工具类的内部类NavigableSet对象并使用unfiltered和entryPredicate重写其`removeAll()`和`retainAll()`两个方法

- `values()`方法会根据当前对象、unfiltered和entryPredicate返回一个FilteredMapValues的对象

- `entryIterator()`和`descendingEntryIterator()`这两个获取升降序构造器的方法都是调用了Iterators工具类的`filter()`方法根据unfiltered和entryPredicate的过滤来获得的

- `isEmpty()`这个方法也是调用Iterators工具类的`any()`方法再传入unfiltered和entryPredicate来判断的

- `pollFirstEntry()`和`pollLastEntry()`这两个方法都是通过Iterators工具类的`removeFirstMatching()`方法传入升序和降序的Map再根据entryPredicate来实现的

- `descendingMap()`、`subMap()`、`headMap()`和`tailMap()`这些方法则都通过Maps工具类的`filterEntries()`方法来重新构建了一个新的FilteredEntryNavigableMap对象

  ```java
  @GwtIncompatible("NavigableMap")
  public static <K, V> NavigableMap<K, V> filterEntries(
      NavigableMap<K, V> unfiltered,
      Predicate<? super Entry<K, V>> entryPredicate) {
    checkNotNull(entryPredicate);
    return (unfiltered instanceof FilteredEntryNavigableMap)
        ? filterFiltered((FilteredEntryNavigableMap<K, V>) unfiltered, entryPredicate)
        : new FilteredEntryNavigableMap<K, V>(checkNotNull(unfiltered), entryPredicate);
  }
  ```

  如果传入的unfiltered是FilteredEntryNavigableMap类型，那么将unfiltered的条件和传入的条件合并然后返回一个新的FilteredEntryNavigableMap对象其内容为传入的unfiltered，否知直接返回一个FilteredEntryNavigableMap对象内容为传入的unfiltered

- 其余方法都是直接调用filteredDelegate对象的同名方法



### 内部类——FilteredEntryBiMap

```java
static final class FilteredEntryBiMap<K, V> extends FilteredEntryMap<K, V>
    implements BiMap<K, V> {
  private final BiMap<V, K> inverse;

  private static <K, V> Predicate<Entry<V, K>> inversePredicate(
      final Predicate<? super Entry<K, V>> forwardPredicate) {
    return new Predicate<Entry<V, K>>() {
      @Override
      public boolean apply(Entry<V, K> input) {
        return forwardPredicate.apply(
            Maps.immutableEntry(input.getValue(), input.getKey()));
      }
    };
  }

  FilteredEntryBiMap(BiMap<K, V> delegate,
      Predicate<? super Entry<K, V>> predicate) {
    super(delegate, predicate);
    this.inverse = new FilteredEntryBiMap<V, K>(
        delegate.inverse(), inversePredicate(predicate), this);
  }

  private FilteredEntryBiMap(
      BiMap<K, V> delegate, Predicate<? super Entry<K, V>> predicate,
      BiMap<V, K> inverse) {
    super(delegate, predicate);
    this.inverse = inverse;
  }

  BiMap<K, V> unfiltered() {
    return (BiMap<K, V>) unfiltered;
  }

  @Override
  public V forcePut(@Nullable K key, @Nullable V value) {
    checkArgument(apply(key, value));
    return unfiltered().forcePut(key, value);
  }

  @Override
  public BiMap<V, K> inverse() {
    return inverse;
  }

  @Override
  public Set<V> values() {
    return inverse.keySet();
  }
}
```

- 继承了`FilteredEntryMap`，同时实现了`BiMap`接口，在原来的基础上扩展了正反视图的功能
- 内部定义了一个反视图inverse
- `inversePredicate()`方法返回了一个新的Predicate对象重写了`apply()`方法，内部直接调用forwardPredicate的apply()方法
- 构造器在给反视图赋值时是直接创建一个新的FilteredEntryBiMap对象，传入delegate的反视图和调用inversePredicate()方法传入过滤规则
- 它还有一个私有的构造器，这个私有的构造器会多传入一个inverse反视图，这样它里面就可以直接赋值
- `unfiltered()`方法会将unfiltered强转为BiMap类型，然后直接返回
- `forcePut()`会先检查键和值是否符合条件，然后再调用unfiltered()方法获取unfiltered再调用unfiltered的forcePut()方法
- `inverse()`方法会直接返回反视图inverse
- `values()`方法会去调用inverse()拿到反视图，然后调用反视图的keySet()方法



### 总结

这些内部类都是在`FilteredEntryMap`类之上的分别对有序、可导航和正反视图方向的扩展。其中FilteredEntryNavigableMap因为继承了AbstractNavigableMap而无法再继承FilteredEntryMap，所以其内部又多设置了一个成员变量来存放根据unfiltered和entryPredicate获得的FilterEntryMap对象，这是很巧妙的