# Guava源码之Maps工具类(二)

### 内部类——ImprovedAbstractMap

```java
  abstract static class ImprovedAbstractMap<K, V> extends AbstractMap<K, V> {
    
    abstract Set<Entry<K, V>> createEntrySet();

    private transient Set<Entry<K, V>> entrySet;

    @Override public Set<Entry<K, V>> entrySet() {
      Set<Entry<K, V>> result = entrySet;
      return (result == null) ? entrySet = createEntrySet() : result;
    }

    private transient Set<K> keySet;

    @Override public Set<K> keySet() {
      Set<K> result = keySet;
      return (result == null) ? keySet = createKeySet() : result;
    }

    Set<K> createKeySet() {
      return new KeySet<K, V>(this);
    }

    private transient Collection<V> values;

    @Override public Collection<V> values() {
      Collection<V> result = values;
      return (result == null) ? values = createValues() : result;
    }

    Collection<V> createValues() {
      return new Values<K, V>(this);
    }
  }
```

- 此类实现了AbstractMap，众所周知AbstractMap是Map接口的一种基本实现，目的是减少实现Map接口所要进行的工作。而ImprovedAbstractMap看名字也知道它是在AbstractMap的基础上对Map的进一步实现
- 此类内部自定义了三个属性：`entrySet`、`keySet`和`values`，然后根据这三个属性定义了各自的“构造”方法。但是构造方法是有两个，需要注意下。其中keySet使用了Maps的内部类KeySet，values使用了内部类Values
- 可以看到它主要是在键值对集合、键集合和值集合这三方面进行的重写和定义。



### 内部类——KeySet

```java
  static class KeySet<K, V> extends Sets.ImprovedAbstractSet<K> {
    final Map<K, V> map;

    KeySet(Map<K, V> map) {
      this.map = checkNotNull(map);
    }

    Map<K, V> map() {
      return map;
    }

    @Override public Iterator<K> iterator() {
      return keyIterator(map().entrySet().iterator());
    }

    @Override public int size() {
      return map().size();
    }

    @Override public boolean isEmpty() {
      return map().isEmpty();
    }

    @Override public boolean contains(Object o) {
      return map().containsKey(o);
    }

    @Override public boolean remove(Object o) {
      if (contains(o)) {
        map().remove(o);
        return true;
      }
      return false;
    }

    @Override public void clear() {
      map().clear();
    }
  }
```

- 此类为Maps工具类定义的键集合类,首先因为他是一个Set，所以他继承了Sets工具类的`ImprovedAbstractSet`，又在内部定义了一个Map属性，并用`final`修饰。
- 构造器里面设置了这个唯一的属性Map，并且在赋值之前还检查了指定的Map是否为空。
- 方法还定义了`size`、`isEmpty`、`contains`、`remove`和`clear`,其实内部还是调用了map属性的同样的方法。



### 内部类——Values

```java
  static class Values<K, V> extends AbstractCollection<V> {
    final Map<K, V> map;

    Values(Map<K, V> map) {
      this.map = checkNotNull(map);
    }

    final Map<K, V> map() {
      return map;
    }

    @Override public Iterator<V> iterator() {
      return valueIterator(map().entrySet().iterator());
    }

    @Override public boolean remove(Object o) {
      try {
        return super.remove(o);
      } catch (UnsupportedOperationException e) {
        for (Entry<K, V> entry : map().entrySet()) {
          if (Objects.equal(o, entry.getValue())) {
            map().remove(entry.getKey());
            return true;
          }
        }
        return false;
      }
    }

    @Override public boolean removeAll(Collection<?> c) {
      try {
        return super.removeAll(checkNotNull(c));
      } catch (UnsupportedOperationException e) {
        Set<K> toRemove = Sets.newHashSet();
        for (Entry<K, V> entry : map().entrySet()) {
          if (c.contains(entry.getValue())) {
            toRemove.add(entry.getKey());
          }
        }
        return map().keySet().removeAll(toRemove);
      }
    }

    @Override public boolean retainAll(Collection<?> c) {
      try {
        return super.retainAll(checkNotNull(c));
      } catch (UnsupportedOperationException e) {
        Set<K> toRetain = Sets.newHashSet();
        for (Entry<K, V> entry : map().entrySet()) {
          if (c.contains(entry.getValue())) {
            toRetain.add(entry.getKey());
          }
        }
        return map().keySet().retainAll(toRetain);
      }
    }

    @Override public int size() {
      return map().size();
    }

    @Override public boolean isEmpty() {
      return map().isEmpty();
    }

    @Override public boolean contains(@Nullable Object o) {
      return map().containsValue(o);
    }

    @Override public void clear() {
      map().clear();
    }
  }
```

- Values内部类继承了`AbstractCollection`抽象类，其内部和KeySet一样都定义了一个属性Map，并且都是用**final**修饰。构造器再给map赋值之前依然检查了指定map是否为空。
- `remove`、`removeAll`、`retainAll`这三个方法都是调用了父类的方法，在代码抛错的时候通过遍历的方式来操作。



### 总结

本次主要介绍了增强AbstractMap的一些内部类，主要是定义了entrySet、keySet和values的实现以供其他的类使用。