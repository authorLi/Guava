# Guava源码之HashBiMap

### 简介

这是一个由**两个哈希桶**实现的双向Map。这个实现类**允许键与值都为空**。正反向的HashBiMap都是**允许序列化的**。

```java
@GwtCompatible(emulated = true)
public final class HashBiMap<K, V> extends IteratorBasedAbstractMap<K, V>
    implements BiMap<K, V>, Serializable {
```

此实现类是使用了`@GwtCompatible`注解修饰，实现了**BiMap接口**，算是`AbstractBiMap`的兄弟。此外还继承了`IteratorBasedAbstractMap`抽象类。

之前没见过IteratorBasedAbstractMap，简单来看一下：

##### IteratorBasedAbstractMap:

```java
  abstract static class IteratorBasedAbstractMap<K, V> extends AbstractMap<K, V> {
    @Override
    public abstract int size();//抽象方法，待实现。返回map的大小

    abstract Iterator<Entry<K, V>> entryIterator();//抽象方法，待实现。返回迭代器

    Spliterator<Entry<K, V>> entrySpliterator() {//根据本类的迭代器获取分割器
      return Spliterators.spliterator(
          entryIterator(), size(), Spliterator.SIZED | Spliterator.DISTINCT);
    }

    @Override
    public Set<Entry<K, V>> entrySet() {
      return new EntrySet<K, V>() {//这里返回一个EntrySet对象，此对象也是Maps的内部类
        @Override
        Map<K, V> map() {
          return IteratorBasedAbstractMap.this;//返回此对象
        }

        @Override
        public Iterator<Entry<K, V>> iterator() {
          return entryIterator();//返回本类的迭代器
        }

        @Override
        public Spliterator<Entry<K, V>> spliterator() {
          return entrySpliterator();//返回本类的分割器
        }

        @Override
        public void forEach(Consumer<? super Entry<K, V>> action) {
          forEachEntry(action);//遍历键值对，这里使用了Consumer，说明可以使用Lambda表达式来调用
        }
      };
    }

    void forEachEntry(Consumer<? super Entry<K, V>> action) {
      entryIterator().forEachRemaining(action);//使用本类的迭代器遍历键值对
    }

    @Override
    public void clear() {
      Iterators.clear(entryIterator());//删除所有元素
    }
  }
```

首先它是Maps工具类的内部类，Maps改天再细看。。。其次它是一个抽象类继承于AbstractMap。分析了上面的代码可以看到此抽象类定义了很多关于迭代器的方法，并且在重写的`entrySet()`方法中返回了一个EntrySet类，在new这个类时又重写了很多EntrySet的方法，其目的是用自己同样作用的方法覆盖原EntrySet的方法。

那么HashBiMap继承了它应该就是用了它来处理迭代器相关的操作。

接下来正式的来看HashBiMap类:

### 类概览

##### 属性：

```java
  private static final double LOAD_FACTOR = 1.0;//负载因子，用于判断是否需要重新分配空间

  private transient BiEntry<K, V>[] hashTableKToV;//存储“键-值”的哈希桶
  private transient BiEntry<K, V>[] hashTableVToK;//存储“值-键”的哈希桶
  private transient @Nullable BiEntry<K, V> firstInKeyInsertionOrder;//可以看作引用，插入的第一个键值对
  private transient @Nullable BiEntry<K, V> lastInKeyInsertionOrder;//可以看作引用，插入的最后一个键值对
  private transient int size;//HashBiMap的元素个数
  private transient int mask;//初始大小为哈希桶大小减1，用来计算哈希值
  private transient int modCount;//修改操作时的元素个数
```

##### 构造器：

```java
  private HashBiMap(int expectedSize) {
    init(expectedSize);
  }

  private void init(int expectedSize) {
    checkNonnegative(expectedSize, "expectedSize");//检测是否为非负数，Guava里面这类判断被写成方法的好多啊
    int tableSize = Hashing.closedTableSize(expectedSize, LOAD_FACTOR);//设置哈希桶的大小，调用此方法使得大小尽量接近于2的幂次方。顺带一提，HashMap也会进行这种判断来修正传入的初始哈希桶大小。
    this.hashTableKToV = createTable(tableSize);//返回指定大小的BiEntry的数组
    this.hashTableVToK = createTable(tableSize);//返回指定大小的BiEntry的数组
    this.firstInKeyInsertionOrder = null;
    this.lastInKeyInsertionOrder = null;
    this.size = 0;//初始化时HashBiMap的元素个数为0
    this.mask = tableSize - 1;//指定mask大小
    this.modCount = 0;//修改时的元素数量
  }
```

构造器为私有由**private**修饰，这就注定了他不能直接被“new出来”。而私有构造器内部又调用了`init()`方法来对内部属性做一些初始化，具体看上面代码中的注释。

但是HashBiMap也需要有外界可以调用的构造器啊，所以：

```java
  public static <K, V> HashBiMap<K, V> create() {
    return create(16);
  }

  public static <K, V> HashBiMap<K, V> create(int expectedSize) {
    return new HashBiMap<>(expectedSize);
  }

  public static <K, V> HashBiMap<K, V> create(Map<? extends K, ? extends V> map) {
    HashBiMap<K, V> bimap = create(map.size());
    bimap.putAll(map);
    return bimap;
  }
```

出现了这三个重载的`create()`方法。

1. 第一个可以看到它调用了第二个create()方法并且参数为16，这显而易见是默认的构造方法，类似的HashMap的默认大小也为16。
2. 第三个create()方法可以看到它是传入了一个Map进去，调用了第二个create()方法并且将传入Map的大小作为参数来确定HashBiMap的大小并将键值对都写入到创建的HashBiMap
3. 第二个create()是会直接调用私有构造器的方法，传入的值为期望的大小值，当然最后还要通过别的方法来最后确定最终大小。

##### 内部类：

###### BiEntry：

```java
private static final class BiEntry<K, V> extends ImmutableEntry<K, V> {
    final int keyHash;//key的哈希值
    final int valueHash;//value的哈希值

    @Nullable BiEntry<K, V> nextInKToVBucket;//指向下一个“键-值”桶的引用
    @Nullable BiEntry<K, V> nextInVToKBucket;//指向下一个“值-键”桶的引用

    @Nullable BiEntry<K, V> nextInKeyInsertionOrder;//指向下一个要插入的引用
    @Nullable BiEntry<K, V> prevInKeyInsertionOrder;//指向上一个要插入的引用

    //BiEntry的构造器
    BiEntry(K key, int keyHash, V value, int valueHash) {
      super(key, value);//这里会调用ImmutableEntry的构造器，使key与value都变为不可变值(final修饰)
      this.keyHash = keyHash;
      this.valueHash = valueHash;
    }
}
```

首先BiEntry继承了`ImmutableEntry`，字面意思就是不可变得Entry，简单看了一下此类，它的内部提供了获取键和获取值的方法，虽然提供了一个设置值的方法，但是此方法会直接抛出一个`UnsupportedOperationException`异常，另外这三个方法还是使用**`final`**修饰的，所以此三个方法不可被重写。其属性也是**`final`**修饰，并且他们都允许为空。

类内部除了构造方法外没有任何方法，主要定义了很多属性，这些属性多为引用。

###### Itr：

```java
abstract class Itr<T> implements Iterator<T> {
    BiEntry<K, V> next = firstInKeyInsertionOrder;//下一个元素的指针
    BiEntry<K, V> toRemove = null;//准备删除的元素
    int expectedModCount = modCount;//修改时的个数，用来判断是否会发生并发修改异常
    int remaining = size();//哈希桶的元素个数

    //此方法判断迭代时是否有下一个值存在
    @Override
    public boolean hasNext() {
      if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
      }
      return next != null && remaining > 0;
    }

    @Override
    public T next() {
      if (!hasNext()) {
        throw new NoSuchElementException();
      }

      BiEntry<K, V> entry = next;//拿到下一个元素
      next = entry.nextInKeyInsertionOrder;//指针向后移一位
      toRemove = entry;//将引用指向当前遍历元素，便于删除操作
      remaining--;//剩余遍历元素个数
      return output(entry);//根据情况自定义想返回的值
    }

    @Override
    public void remove() {
      if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
      }
      checkRemove(toRemove != null);//要删除的元素为空则抛出异常
      delete(toRemove);//调用delete()方法执行删除
      expectedModCount = modCount;//为了避免发生并发修改异常，将两个值统一
      toRemove = null;//执行完删除，引用置空
    }

    //返回自定义的与entry相关的值，抽象方法，待实现
    abstract T output(BiEntry<K, V> entry);
  }
```

首先看到它是实现了`Iterator`这个接口的，所以它实际上是一个构造器类。其次它重写了`hasNext()`、`next()`和`remove()`方法。最重要的是它定义了一个抽象方法`output()`通过传入一个**BiEntry**对象来返回自定义的和此对象相关的值。

###### KeySet:

```java
  @WeakOuter
  private final class KeySet extends Maps.KeySet<K, V> {
    KeySet() {
      super(HashBiMap.this);
    }

    //这里返回了一个构造器，定义了遍历时返回元素的“键”，所以这个可以理解为“键”的迭代器
    @Override
    public Iterator<K> iterator() {
      return new Itr<K>() {
        @Override
        K output(BiEntry<K, V> entry) {
          return entry.key;
        }
      };
    }

    @Override
    public boolean remove(@Nullable Object o) {
      BiEntry<K, V> entry = seekByKey(o, smearedHash(o));//o为key，smearedHash(o)方法会返回key的哈希值。通过传入这两个值来寻找对应的BiEntry
      if (entry == null) {
        return false;
      } else {
        delete(entry);//同时删除“键-值”和“值-键”中的对应键值对
        entry.prevInKeyInsertionOrder = null;//将引用置空
        entry.nextInKeyInsertionOrder = null;//将引用置空
        return true;
      }
    }
  }
```

继承了Maps工具类的KeySet。定义了一系列对于键的集合的操作，包括构造器(直接调用父类构造函数)、迭代器(返回“键”的迭代器)和删除方法(同时删除两个视图的对应键值对)

###### Inverse：

```java
  private final class Inverse extends IteratorBasedAbstractMap<V, K>
      implements BiMap<V, K>, Serializable {
    //正向视图
    BiMap<K, V> forward() {
      return HashBiMap.this;
    }

    @Override
    public int size() {
      return size;
    }

    @Override
    public void clear() {
      forward().clear();
    }

    @Override
    public boolean containsKey(@Nullable Object value) {
      return forward().containsValue(value);
    }

    @Override
    public K get(@Nullable Object value) {//根据“值”获取对应的“键”
      return Maps.keyOrNull(seekByValue(value, smearedHash(value)));
    }

    @CanIgnoreReturnValue
    @Override
    @Nullable
    public K put(@Nullable V value, @Nullable K key) {//插入一个键值对，值为键，键为值，不强制
      return putInverse(value, key, false);
    }

    @Override
    @Nullable
    public K forcePut(@Nullable V value, @Nullable K key) {//插入一个键值对，值为键，键为值，相同则强制
      return putInverse(value, key, true);
    }

    @Override
    @Nullable
    public K remove(@Nullable Object value) {
      BiEntry<K, V> entry = seekByValue(value, smearedHash(value));//找到对应的“值-键”对
      if (entry == null) {
        return null;
      } else {
        delete(entry);//同时删除“键-值”和“值-键”哈希桶中对应的BiEntry
        entry.prevInKeyInsertionOrder = null;
        entry.nextInKeyInsertionOrder = null;
        return entry.key;
      }
    }

    @Override
    public BiMap<K, V> inverse() {
      return forward();
    }

    @Override
    public Set<V> keySet() {
      return new InverseKeySet();
    }

    //内部类的内部类，这个内部类和HashBiMap的KeySet也是相同的，用来装反向的KeySet
    @WeakOuter
    private final class InverseKeySet extends Maps.KeySet<V, K> {
      InverseKeySet() {
        super(Inverse.this);
      }

      @Override
      public boolean remove(@Nullable Object o) {
        BiEntry<K, V> entry = seekByValue(o, smearedHash(o));
        if (entry == null) {
          return false;
        } else {
          delete(entry);
          return true;
        }
      }

      @Override
      public Iterator<V> iterator() {//自定义迭代器返回值(对正向来说是键)
        return new Itr<V>() {
          @Override
          V output(BiEntry<K, V> entry) {
            return entry.value;
          }
        };
      }
    }

    @Override
    public Set<K> values() {
      return forward().keySet();
    }

    @Override
    Iterator<Entry<V, K>> entryIterator() {
      return new Itr<Entry<V, K>>() {
        @Override
        Entry<V, K> output(BiEntry<K, V> entry) {
          return new InverseEntry(entry);
        }

        class InverseEntry extends AbstractMapEntry<V, K> {//出现了！内部类的内部类的内部类，还定义在方法里
          BiEntry<K, V> delegate;

          InverseEntry(BiEntry<K, V> entry) {
            this.delegate = entry;
          }

          @Override
          public V getKey() {
            return delegate.value;
          }

          @Override
          public K getValue() {
            return delegate.key;
          }

          @Override
          public K setValue(K key) {//删除旧的BiEntry，新建一个新的BiEntry
            K oldKey = delegate.key;
            int keyHash = smearedHash(key);
            if (keyHash == delegate.keyHash && Objects.equal(key, oldKey)) {
              return key;
            }
            checkArgument(seekByKey(key, keyHash) == null, "value already present: %s", key);
            delete(delegate);
            BiEntry<K, V> newEntry =
                new BiEntry<>(key, keyHash, delegate.value, delegate.valueHash);
            delegate = newEntry;
            insert(newEntry, null);
            expectedModCount = modCount;
            return oldKey;
          }
        }
      };
    }

    @Override
    public void forEach(BiConsumer<? super V, ? super K> action) {
      checkNotNull(action);
      HashBiMap.this.forEach((k, v) -> action.accept(v, k));
    }

    @Override
    public void replaceAll(BiFunction<? super V, ? super K, ? extends K> function) {
      checkNotNull(function);
      BiEntry<K, V> oldFirst = firstInKeyInsertionOrder;
      clear();
      for (BiEntry<K, V> entry = oldFirst; entry != null; entry = entry.nextInKeyInsertionOrder) {
        put(entry.value, function.apply(entry.value, entry.key));
      }
    }

    Object writeReplace() {
      return new InverseSerializedForm<>(HashBiMap.this);
    }
  }
```

Inverse类也是继承了IteratorBasedAbstractMap抽象类，这也不难理解，因为反向类嘛，肯定与原来的类相同的。里面还定义了一个内部类，还是定义在方法中的。

###### InverseSerializedForm：

```java
  private static final class InverseSerializedForm<K, V> implements Serializable {
    private final HashBiMap<K, V> bimap;

    InverseSerializedForm(HashBiMap<K, V> bimap) {
      this.bimap = bimap;
    }

    Object readResolve() {
      return bimap.inverse();
    }
  }
```

继承了Serializable，表示它可以序列化。它里面定义了readResolver()方法来返回本HashBiMap对象的inverse属性，即反视图。



##### 其他方法：

###### putInverse()：

```java
  private @Nullable K putInverse(@Nullable V value, @Nullable K key, boolean force) {
    int valueHash = smearedHash(value);//获取值的哈希值
    int keyHash = smearedHash(key);//获取键的哈希值

    BiEntry<K, V> oldEntryForValue = seekByValue(value, valueHash);//根据值获取BiEntry
    BiEntry<K, V> oldEntryForKey = seekByKey(key, keyHash);//根据键获取BiEntry
    if (oldEntryForValue != null
        && keyHash == oldEntryForValue.keyHash
        && Objects.equal(key, oldEntryForValue.key)) {
      return key;
    } else if (oldEntryForKey != null && !force) {
      throw new IllegalArgumentException("key already present: " + key);
    }

    /*
     * The ordering here is important: if we deleted the key entry and then the value entry,
     * the key entry's prev or next pointer might point to the dead value entry, and when we
     * put the new entry in the key entry's position in iteration order, it might invalidate
     * the linked list.
     */

    if (oldEntryForValue != null) {
      delete(oldEntryForValue);
    }

    if (oldEntryForKey != null) {
      delete(oldEntryForKey);
    }

    BiEntry<K, V> newEntry = new BiEntry<>(key, keyHash, value, valueHash);
    insert(newEntry, oldEntryForKey);

    if (oldEntryForKey != null) {
      oldEntryForKey.prevInKeyInsertionOrder = null;
      oldEntryForKey.nextInKeyInsertionOrder = null;
    }
    if (oldEntryForValue != null) {
      oldEntryForValue.prevInKeyInsertionOrder = null;
      oldEntryForValue.nextInKeyInsertionOrder = null;
    }
    rehashIfNecessary();
    return Maps.keyOrNull(oldEntryForValue);
  }
```

###### delete()：

```java
  //删除“键-值”和“值-键”哈希桶中的BiEntry
  private void delete(BiEntry<K, V> entry) {
    int keyBucket = entry.keyHash & mask;//求得entry在“键”哈希桶中的位置
    BiEntry<K, V> prevBucketEntry = null;
    for (BiEntry<K, V> bucketEntry = hashTableKToV[keyBucket];
        true;
        bucketEntry = bucketEntry.nextInKToVBucket) {
      if (bucketEntry == entry) {
        if (prevBucketEntry == null) {
          hashTableKToV[keyBucket] = entry.nextInKToVBucket;
        } else {
          prevBucketEntry.nextInKToVBucket = entry.nextInKToVBucket;
        }
        break;
      }
      prevBucketEntry = bucketEntry;
    }

    int valueBucket = entry.valueHash & mask;
    prevBucketEntry = null;
    for (BiEntry<K, V> bucketEntry = hashTableVToK[valueBucket];
        true;
        bucketEntry = bucketEntry.nextInVToKBucket) {
      if (bucketEntry == entry) {
        if (prevBucketEntry == null) {
          hashTableVToK[valueBucket] = entry.nextInVToKBucket;
        } else {
          prevBucketEntry.nextInVToKBucket = entry.nextInVToKBucket;
        }
        break;
      }
      prevBucketEntry = bucketEntry;
    }

    if (entry.prevInKeyInsertionOrder == null) {
      firstInKeyInsertionOrder = entry.nextInKeyInsertionOrder;
    } else {
      entry.prevInKeyInsertionOrder.nextInKeyInsertionOrder = entry.nextInKeyInsertionOrder;
    }

    if (entry.nextInKeyInsertionOrder == null) {
      lastInKeyInsertionOrder = entry.prevInKeyInsertionOrder;
    } else {
      entry.nextInKeyInsertionOrder.prevInKeyInsertionOrder = entry.prevInKeyInsertionOrder;
    }

    size--;
    modCount++;
  }
```

**删除步骤(大概)**：根据key的哈希值算得键在“键-值”哈希桶的位置，并开始遍历外联表，直到找到对应的BiEntry将其删除(如果是第一个则将第二个顶替上来，如果不是第一个则将其前面一个BiEntry的后置置为对应BiEntry的后置，前面你这一句就是废话，实际上就是链表中元素的删除！)。然后对值进行同样的操作。下一步将此BiEntry在key插入队列中删除(分别判断了前一个和后一个的引用是否为空)。最后元素的数量减1，修改数加1。

###### insert()：

```java
  private void insert(BiEntry<K, V> entry, @Nullable BiEntry<K, V> oldEntryForKey) {
    int keyBucket = entry.keyHash & mask;
    entry.nextInKToVBucket = hashTableKToV[keyBucket];
    hashTableKToV[keyBucket] = entry;

    int valueBucket = entry.valueHash & mask;
    entry.nextInVToKBucket = hashTableVToK[valueBucket];
    hashTableVToK[valueBucket] = entry;

    if (oldEntryForKey == null) {
      entry.prevInKeyInsertionOrder = lastInKeyInsertionOrder;
      entry.nextInKeyInsertionOrder = null;
      if (lastInKeyInsertionOrder == null) {
        firstInKeyInsertionOrder = entry;
      } else {
        lastInKeyInsertionOrder.nextInKeyInsertionOrder = entry;
      }
      lastInKeyInsertionOrder = entry;
    } else {
      entry.prevInKeyInsertionOrder = oldEntryForKey.prevInKeyInsertionOrder;
      if (entry.prevInKeyInsertionOrder == null) {
        firstInKeyInsertionOrder = entry;
      } else {
        entry.prevInKeyInsertionOrder.nextInKeyInsertionOrder = entry;
      }
      entry.nextInKeyInsertionOrder = oldEntryForKey.nextInKeyInsertionOrder;
      if (entry.nextInKeyInsertionOrder == null) {
        lastInKeyInsertionOrder = entry;
      } else {
        entry.nextInKeyInsertionOrder.prevInKeyInsertionOrder = entry;
      }
    }

    size++;
    modCount++;
  }
```

插入步骤(大概)：求得“键-值”哈希桶的位置，获得其哈希值锁定即将要操作的外联表，然后将要插入的BiEntry放到外联表的**首位**

对于“值-键”哈希表也执行同样的操作。然后判断oldEntryForKey是否为空：

- 如果为空，则将其加入插入顺序的最后，如果没有要插入的则将其作为第一个，否则将其最后一个要插入的的后置指向此BiEntry
- 如果不为空，则将entry的前置置为oldEntryForKey的前置，如果entry的前置为空，则将entry设为插入顺序的首位，否则将其前置的后置置为entry。再将entry的后置设置为oldEntryForKey的后置。再判断entry的后置是否为空，是则将插入顺序的最后一位置为entry，否则将entry的后置的前置设为entry。

最后将元素总数加1，修改数也加1。

###### put()&forcePut()：

```java
  @CanIgnoreReturnValue
  @Override
  public V put(@Nullable K key, @Nullable V value) {
    return put(key, value, false);
  }

  @CanIgnoreReturnValue
  @Override
  @Nullable
  public V forcePut(@Nullable K key, @Nullable V value) {
    return put(key, value, true);
  }

  private V put(@Nullable K key, @Nullable V value, boolean force) {
    int keyHash = smearedHash(key);//key的哈希值
    int valueHash = smearedHash(value);//value的哈希值

    BiEntry<K, V> oldEntryForKey = seekByKey(key, keyHash);//根据key找到对应的BiEntry
    if (oldEntryForKey != null
        && valueHash == oldEntryForKey.valueHash
        && Objects.equal(value, oldEntryForKey.value)) {//插入了相同的值
      return value;
    }

    BiEntry<K, V> oldEntryForValue = seekByValue(value, valueHash);//根据值(键)获取对应的BiEntry
    if (oldEntryForValue != null) {//然而对应的“值-键”哈希桶中有相同的键值对
      if (force) {//是否强制插入？是则要删除老的BiEntry，否则因为插入了相同的值报错
        delete(oldEntryForValue);
      } else {
        throw new IllegalArgumentException("value already present: " + value);
      }
    }

    BiEntry<K, V> newEntry = new BiEntry<>(key, keyHash, value, valueHash);//构造新的BiEntry
    if (oldEntryForKey != null) {
      delete(oldEntryForKey);//删除旧的
      insert(newEntry, oldEntryForKey);//插入新的
      oldEntryForKey.prevInKeyInsertionOrder = null;
      oldEntryForKey.nextInKeyInsertionOrder = null;
      return oldEntryForKey.value;
    } else {
      insert(newEntry, null);//直接插入新的
      rehashIfNecessary();
      return null;
    }
  }
```

可以看到put()和forcePut()两个方法都是内部调用了另一put()方法，区别是第三个参数：“是否强制”。

###### rehashIfNecessary()：

```java
  private void rehashIfNecessary() {
    BiEntry<K, V>[] oldKToV = hashTableKToV;
    if (Hashing.needsResizing(size, oldKToV.length, LOAD_FACTOR)) {//判断是否需要rehash
      int newTableSize = oldKToV.length * 2;//将新的哈希桶的大小设置为原来的两倍，相当于还是2的幂次方大小

      this.hashTableKToV = createTable(newTableSize);//为“键-值”创建新的BiEntry哈希桶
      this.hashTableVToK = createTable(newTableSize);//为“值-键”创建新的BiEntry哈希桶
      this.mask = newTableSize - 1;//重新设置mask
      this.size = 0;//重新设置元素个数

      for (BiEntry<K, V> entry = firstInKeyInsertionOrder;
          entry != null;
          entry = entry.nextInKeyInsertionOrder) {
        insert(entry, entry);//逐个重新插入到新的哈希桶
      }
      this.modCount++;//修改数加1
    }
  }
```

看名字也知道，这是在需要的时候对整个“键-值”和“值-键”的哈希桶进行重新哈希分配。

### 总结：

前面说了，截止到现在看的代码，功能的设定一般被定义在父类中，子类直接继承父类来继承这些方法甚至不需要重写，因为每个子类对于父类的实现方式不同，子类内部的方法更多的是在为自己的例如：构造函数、内部类等服务。说到内部类，发现Guava里拥有内部类的大类还挺多的，Maps这种工具类尤其多，这里也要在看的同时多思考下内部类创建的意义。同时，类里面有内部类，然后内部类里还有内部类，这个看起来真的很麻烦！这样的设计可读性有点。。。还有就是Guava将类似判断是否为负数这样的判断语句封装成方法，这大概也是对于面向对象思想的体现。

HashBiMap里面由于是链表实现(HashMap也是)，所以什么插入啦、删除啦都是链表的一些基本操作，所以可以看出数据结构也很重要啊。

这个HashBiMap真的是有点乱，只看一遍就算很认真的看也无法记住所有，需要反复多次的去阅读才能真正掌握，这个源码起码还要在读3遍吧。

所以这只是第一版，里面可能有不对的或者理解的不对的地方，后面会陆续更新版本来加深体会和改正原文中的不对之处。