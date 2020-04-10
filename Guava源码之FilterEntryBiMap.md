# Guava源码之FilterEntryBiMap

### 简介

这是一个BiMap的实现类。它是Maps工具类的内部类

```java
  static final class FilteredEntryBiMap<K, V> extends FilteredEntryMap<K, V>
      implements BiMap<K, V> {
```

可以看到它继承了`FilteredEntryMap`，实现了`BiMap`。

先来看一下FilteredEntryMap：

```java
  static class FilteredEntryMap<K, V> extends AbstractFilteredMap<K, V> {

    final Set<Entry<K, V>> filteredEntrySet;//存储的键值对

    FilteredEntryMap(Map<K, V> unfiltered, Predicate<? super Entry<K, V>> entryPredicate) {
      super(unfiltered, entryPredicate);//调用父类方法设置未过滤的Map和条件
      filteredEntrySet = Sets.filter(unfiltered.entrySet(), predicate);//调用了Sets工具类的过滤方法拿到过滤后的Map
    }

    @Override
    protected Set<Entry<K, V>> createEntrySet() {
      return new EntrySet();
    }

    @WeakOuter
    private class EntrySet extends ForwardingSet<Entry<K, V>> {//这里继承了ForwardingSet抽象类，又是一个私有内部类
      @Override
      protected Set<Entry<K, V>> delegate() {//重写delegate()方法
        return filteredEntrySet;
      }

      @Override
      public Iterator<Entry<K, V>> iterator() {//返回迭代器
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
      return new KeySet();//返回一个KeySet类的实例
    }

    static <K, V> boolean removeAllKeys(//删除某些符合条件又存在于keyCollection的键值对
        Map<K, V> map, Predicate<? super Entry<K, V>> entryPredicate, Collection<?> keyCollection) {
      Iterator<Entry<K, V>> entryItr = map.entrySet().iterator();
      boolean result = false;
      while (entryItr.hasNext()) {
        Entry<K, V> entry = entryItr.next();
        if (entryPredicate.apply(entry) && keyCollection.contains(entry.getKey())) {
          entryItr.remove();
          result = true;
        }
      }
      return result;
    }

    static <K, V> boolean retainAllKeys(//删除某些符合条件又不存在于keyCollection的键值对
        Map<K, V> map, Predicate<? super Entry<K, V>> entryPredicate, Collection<?> keyCollection) {
      Iterator<Entry<K, V>> entryItr = map.entrySet().iterator();
      boolean result = false;
      while (entryItr.hasNext()) {
        Entry<K, V> entry = entryItr.next();
        if (entryPredicate.apply(entry) && !keyCollection.contains(entry.getKey())) {
          entryItr.remove();
          result = true;
        }
      }
      return result;
    }

    @WeakOuter
    class KeySet extends Maps.KeySet<K, V> {//又一个内部类
      KeySet() {
        super(FilteredEntryMap.this);
      }

      @Override
      public boolean remove(Object o) {
        if (containsKey(o)) {
          unfiltered.remove(o);
          return true;
        }
        return false;
      }

      @Override
      public boolean removeAll(Collection<?> collection) {
        return removeAllKeys(unfiltered, predicate, collection);
      }

      @Override
      public boolean retainAll(Collection<?> collection) {
        return retainAllKeys(unfiltered, predicate, collection);
      }

      @Override
      public Object[] toArray() {
        // creating an ArrayList so filtering happens once
        return Lists.newArrayList(iterator()).toArray();
      }

      @Override
      public <T> T[] toArray(T[] array) {
        return Lists.newArrayList(iterator()).toArray(array);
      }
    }
  }
```

浏览了上面的类可以发现，此内部类中又有两个内部类：`EntrySet`和`Keyset`，此外提供了一个构造器和两个返回EntrySet和KeySet的方法。另外还提供了根据条件来删除和保留的`removeAllKeys()`和`retainAllKeys()`这两个方法。其实这个类根据名字也可以分析出来这是一个带过滤功能的EntryMap

### 类概览：

##### 属性：

```java
@RetainedWith private final BiMap<V, K> inverse;
```

它自己本身只提供了一个反视图。

##### 构造器：

```java
  FilteredEntryBiMap(BiMap<K, V> delegate, Predicate<? super Entry<K, V>> predicate) {
      super(delegate, predicate);
      this.inverse =
          new FilteredEntryBiMap<>(delegate.inverse(), inversePredicate(predicate), this);
    }

  private FilteredEntryBiMap(
        BiMap<K, V> delegate, Predicate<? super Entry<K, V>> predicate, BiMap<V, K> inverse) {
      super(delegate, predicate);//调用FilteredEntryMap的构造器，在里面已完成过滤
      this.inverse = inverse;
    }

  private static <K, V> Predicate<Entry<V, K>> inversePredicate(
        final Predicate<? super Entry<K, V>> forwardPredicate) {
      return new Predicate<Entry<V, K>>() {
        @Override
        public boolean apply(Entry<V, K> input) {
          return forwardPredicate.apply(Maps.immutableEntry(input.getValue(), input.getKey()));
        }
      };
    }
```

它共提供了两个构造器，其中一个是私有的。非私有构造器内部先是调用了其父类的方法将待过滤Map按照条件过滤，然后是设置其反视图。这里是调用了私有构造器，将原Map的反视图，相反的条件以及正视图作为参数，这样就设置好了正反视图。

##### 其他方法：

```java
BiMap<K, V> unfiltered() {//获取unfiltered对象，此为其父类AbstractFilteredMap的属性
   return (BiMap<K, V>) unfiltered;
}

@Override
public V forcePut(@Nullable K key, @Nullable V value) {//重写了强制插入的方法，这里调用的是unfiltered对象的强制插入方法
   checkArgument(apply(key, value));
   return unfiltered().forcePut(key, value);
}

@Override
public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {//重写了替换所有的方法，调用的是unfiltered对象的方法
 unfiltered()
    .replaceAll(
        (key, value) ->
            predicate.apply(Maps.immutableEntry(key, value))
                ? function.apply(key, value)
                : value);
}
@Override
public BiMap<V, K> inverse() {//返回反视图
  return inverse;
}

@Override
public Set<V> values() {//返回所有的值的集合
  return inverse.keySet();
}
```

### 总结：

此类虽然是Maps的内部类，但是也挺复杂，并且其父类FilteredEntryMap的设计也很特殊，是直接又写了两个内部类EntrySet和KeySet，它是直接将这两个属性重新定义了两个类，然后本类里面只留了两个用来创建这两个类的方法。FilteredEntryBiMap的话，自己的实现更少一些，还是调用到其父类的方法较多。果然是越底层的类越是掌握着更多等关键的方法和思想，真正拿来给人用的那些类往往都只提供了很少的方法，其内部还是会调用底层父类的方法。