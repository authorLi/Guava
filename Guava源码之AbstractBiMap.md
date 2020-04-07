# Guava源码之AbstractBiMap

### 简介

它是BiMap的实现类，但是它是一个抽象类。它是一种通用的利用两个Map实现的BiMap

```java
@GwtCompatible(emulated = true)
abstract class AbstractBiMap<K, V> extends ForwardingMap<K, V>
    implements BiMap<K, V>, Serializable {
```

它同样是使用@GwtCompatible注解修饰，并且emulated为true。此抽象类除了实现了BiMap接口，它还继承了`ForwardingMap`抽象类，此抽象类继承了`ForwardingObject`抽象类并实现了Map接口。这两个Forward开头的抽象类也同样被@GwtCompatible修饰。

#### ForwardingObject抽象类

这是一个用于实现装饰器模式的抽象基类，必须重写delegate方法来返回正在装饰的实例。这里可以知道Guava竟然为装饰器模式这种设计模式设计了一个抽象的基类。它拥有：

- protected修饰的构造器
- delegate方法，这是一个抽象方法由protected修饰，它会返回要被装饰的实例，具体实现要由子类完成
- toString方法，内部调用了被装饰对象的toString方法

#### ForwardingMap抽象类

根据官方解释，这是一个可以将一个Map的所有方法的调用转发到另一Map的抽象类，如下节选的部分代码所示：

```java
@Override
  protected abstract Map<K, V> delegate();

  @Override
  public int size() {
    return delegate().size();
  }

  @Override
  public boolean isEmpty() {
    return delegate().isEmpty();
  }

  @CanIgnoreReturnValue
  @Override
  public V remove(Object object) {
    return delegate().remove(object);
  }

  @Override
  public void clear() {
    delegate().clear();
  }
  ...
```

> 它是直接在自己的方法内部调用了delegate()方法返回的对象的同名方法

它还有一些`standard*`开头的方法，这些方法很多都是调用了Maps、Sets等工具类实现的方法

### 类概览

##### 属性：

```java
private transient @MonotonicNonNull Map<K, V> delegate;//正向的Map

@MonotonicNonNull @RetainedWith transient AbstractBiMap<V, K> inverse;//反向Map

private transient @MonotonicNonNull Set<K> keySet;//key的集合

private transient @MonotonicNonNull Set<V> valueSet;//value的集合

private transient @MonotonicNonNull Set<Entry<K, V>> entrySet;//键值对的集合

@GwtIncompatible // Not needed in emulated source.
private static final long serialVersionUID = 0;
```

前面也说了这个类使用了两个键值对调的Map来实现，这就正好对应了两个属性`delegate`和`inverse`

##### 构造器：

```java
  AbstractBiMap(Map<K, V> forward, Map<V, K> backward) {
    setDelegates(forward, backward);
  }

  //此方法用于设置delegate与inverse
  void setDelegates(Map<K, V> forward, Map<V, K> backward) {
    checkState(delegate == null);//检查delegate是否为空
    checkState(inverse == null);//检查inverse是否为空
    checkArgument(forward.isEmpty());//检查forward是否为空
    checkArgument(backward.isEmpty());//检查backward是否为空
    checkArgument(forward != backward);//检查forward与backward是否相同
    delegate = forward;
    inverse = makeInverse(backward);
  }

  AbstractBiMap<V, K> makeInverse(Map<V, K> backward) {
    return new Inverse<>(backward, this);
  }
```

而Inverse类是AbstractBiMap的内部类：

```java
  static class Inverse<K, V> extends AbstractBiMap<K, V> {
    Inverse(Map<K, V> backward, AbstractBiMap<V, K> forward) {
      super(backward, forward);
    }
    ...
  }
```

可以看到Inverse继承了AbstractBiMap，调用的此构造器最终还是会调用AbstractBiMap的**私有**构造方法：

```java
  private AbstractBiMap(Map<K, V> backward, AbstractBiMap<V, K> forward) {
    delegate = backward;
    inverse = forward;
  }
```

这样就成功的利用了两个Map实现了BiMap，其中这个内部类很有意思它是直接继承了其外部类，然后在自己的构造器中将正向和反向这两个Map颠倒过来，所以从上面的属性那里也可以看出来delegate是Map类型，而inverse是AbstractBiMap类型，它们之间的关系就像一个相互的死循环。

##### 内部类：

```java
  @WeakOuter
  private class KeySet extends ForwardingSet<K> {
    @Override
    protected Set<K> delegate() {
      return delegate.keySet();
    }

    @Override
    public void clear() {
      AbstractBiMap.this.clear();
    }

    @Override
    public boolean remove(Object key) {
      if (!contains(key)) {
        return false;
      }
      removeFromBothMaps(key);
      return true;
    }

    @Override
    public boolean removeAll(Collection<?> keysToRemove) {
      return standardRemoveAll(keysToRemove);
    }

    @Override
    public boolean retainAll(Collection<?> keysToRetain) {
      return standardRetainAll(keysToRetain);
    }

    @Override
    public Iterator<K> iterator() {
      return Maps.keyIterator(entrySet().iterator());
    }
  }
```

首先说一下这个@WeakOuter注解，这是一个标示性注解，表示此类与其所属类关系很弱。KeySet类顾名思义说明他是所有键的集合，此类继承了ForwardingSet类，可以猜测此类使用了两个Set来维护分别对应了AbstractBiMap的正向和反向的Map的键的集合。此类的`clear()`方法和`remove()`方法都是调用了AbstractBiMap的方法实现了正向与反向Map的同时操作。

```java
  @WeakOuter
  private class ValueSet extends ForwardingSet<V> {
    final Set<V> valuesDelegate = inverse.keySet();

    @Override
    protected Set<V> delegate() {
      return valuesDelegate;
    }

    @Override
    public Iterator<V> iterator() {
      return Maps.valueIterator(entrySet().iterator());
    }

    @Override
    public Object[] toArray() {
      return standardToArray();
    }

    @Override
    public <T> T[] toArray(T[] array) {
      return standardToArray(array);
    }

    @Override
    public String toString() {
      return standardToString();
    }
  }
```

同样使用@WeakOuter修饰，同样继承于ForwardingSet，它与KeySet相同，是所有“值”的集合。它内部有一个`valuesDelegate`属性，这个属性获取了AbstractBiMap的反视图的keySet来作为正向视图的valueSet

```java
  @WeakOuter
  private class EntrySet extends ForwardingSet<Entry<K, V>> {
    final Set<Entry<K, V>> esDelegate = delegate.entrySet();

    @Override
    protected Set<Entry<K, V>> delegate() {
      return esDelegate;
    }

    @Override
    public void clear() {
      AbstractBiMap.this.clear();
    }

    @Override
    public boolean remove(Object object) {
      if (!esDelegate.contains(object)) {
        return false;
      }

      // safe because esDelegate.contains(object).
      Entry<?, ?> entry = (Entry<?, ?>) object;
      inverse.delegate.remove(entry.getValue());
      esDelegate.remove(entry);
      return true;
    }

    @Override
    public Iterator<Entry<K, V>> iterator() {
      return entrySetIterator();
    }

    // See java.util.Collections.CheckedEntrySet for details on attacks.

    @Override
    public Object[] toArray() {
      return standardToArray();
    }

    @Override
    public <T> T[] toArray(T[] array) {
      return standardToArray(array);
    }

    @Override
    public boolean contains(Object o) {
      return Maps.containsEntryImpl(delegate(), o);
    }

    @Override
    public boolean containsAll(Collection<?> c) {
      return standardContainsAll(c);
    }

    @Override
    public boolean removeAll(Collection<?> c) {
      return standardRemoveAll(c);
    }

    @Override
    public boolean retainAll(Collection<?> c) {
      return standardRetainAll(c);
    }
  }
```

这个内部类还是@WeakOuter修饰，继承于ForwardingSet。它的属性`esDelegate`就是AbstractBiMap的正向视图的entrySet。其`clear()`方法是调用了AbstractBiMap的方法实现而`remove()`方法则是分别调用了Map的remove方法(反视图)直接根据键也就是所传入对象的“值”来删除和Set的remove方法(正向视图)。还有`iterator()`方法，这个方法在内部调用了AbstractBiMap的方法，它返回一个自定义的构造器，此构造器允许在遍历过程中同时删除正反两个视图。

```java
  class BiMapEntry extends ForwardingMapEntry<K, V> {
    private final Entry<K, V> delegate;

    BiMapEntry(Entry<K, V> delegate) {
      this.delegate = delegate;
    }

    @Override
    protected Entry<K, V> delegate() {
      return delegate;
    }

    @Override
    public V setValue(V value) {
      checkValue(value);//value不能为空
      checkState(entrySet().contains(this), "entry no longer in map");//调用对象需要在AbstractBiMap的entrySet中
      if (Objects.equal(value, getValue())) {
        return value;
      }
      checkArgument(!containsValue(value), "value already present: %s", value);
      V oldValue = delegate.setValue(value);
      checkState(Objects.equal(value, get(getKey())), "entry no longer in map");
      updateInverseMap(getKey(), true, oldValue, value);
      return oldValue;
    }
  }
```

这个类不同，它继承于ForwardingMapEntry类。其`setValue()`方法会同时更新正反视图的entrySet

##### 方法：

```java
  @CanIgnoreReturnValue
  @Override
  public V forcePut(@Nullable K key, @Nullable V value) {
    return putInBothMaps(key, value, true);
  }

  private V putInBothMaps(@Nullable K key, @Nullable V value, boolean force) {
    checkKey(key);//检查key是否为空
    checkValue(value);//检查value是否为空
    boolean containedKey = containsKey(key);
    if (containedKey && Objects.equal(value, get(key))) {//是否正视图包含此key并且value相同
      return value;
    }
    if (force) {
      inverse().remove(value);//在反视图中把相同的key删除
    } else {
      checkArgument(!containsValue(value), "value already present: %s", value);
    }
    V oldValue = delegate.put(key, value);//插入正视图
    updateInverseMap(key, containedKey, oldValue, value);//更新反视图
    return oldValue;
  }
```

这里着重看一下`forcePut()`方法，可以看到内部调用了另一方法`putInBothMaps()`，并把第三个参数是否强制设置为了true。

```java
  @Override
  public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
    this.delegate.replaceAll(function);
    inverse.delegate.clear();
    Entry<K, V> broken = null;
    Iterator<Entry<K, V>> itr = this.delegate.entrySet().iterator();
    while (itr.hasNext()) {
      Entry<K, V> entry = itr.next();
      K k = entry.getKey();
      V v = entry.getValue();
      K conflict = inverse.delegate.putIfAbsent(v, k);
      if (conflict != null) {
        broken = entry;
        // We're definitely going to throw, but we'll try to keep the BiMap in an internally
        // consistent state by removing the bad entry.
        itr.remove();
      }
    }
    if (broken != null) {
      throw new IllegalArgumentException("value already present: " + broken.getValue());
    }
  }
```

这个replaceAll是先调用Map类的replaceAll方法将正视图替换，然后再遍历正视图，取出key与value反着将其插入反视图。

AbstractBiMap的其他方法都是用于操作正视图或反视图或同时操作正反视图的方法。

### 总结

AbstractBiMap类运用了两个Map来实现正视图与反视图，在构造器中巧妙地调用了继承了AbstractBiMap的内部类Inverse作为反视图，然而根本上还是调用了AbstractBiMap的私有构造器。

其他内部类均继承了Forwarding开头的一系列抽象类用于操作正反两视图。

其内部方法也用`delegate`和`inverse`这两个属性来操作正反视图。