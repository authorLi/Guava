# Guava源码之EnumHashBiMap

### 简介

这是一个使用EnumMap和HashMap实现的BiMap，其中EnumMap用于存储`键-值`，HashMap用于存储`值-键`。EnumHashBiMap的键不允许为空，但是值允许为空！正反向的EnumHashBiMap都可以进行序列化。

### 类概览

```java
@GwtCompatible(emulated = true)
public final class EnumHashBiMap<K extends Enum<K>, V> extends AbstractBiMap<K, V> {
```

此类使用@GwtCompatible修饰，并且与它的“兄弟”EnumBiMap相同，它也是由**final**修饰，是一个**不可变类**。

##### 属性：

```java
  private transient Class<K> keyType;

  @GwtIncompatible // only needed in emulated source.
  private static final long serialVersionUID = 0;
```

看类声明是`EnumHashBiMap<K extends Enum<K>, V>`所以，keyType就是确定了键的Class对象，即确定了键的类型

##### 构造器：

```java
  private EnumHashBiMap(Class<K> keyType) {
    super(
        new EnumMap<K, V>(keyType),
        Maps.<V, K>newHashMapWithExpectedSize(keyType.getEnumConstants().length));
    this.keyType = keyType;
  }
```

如上所示，此构造器如同它的兄弟EnumBiMap一样，都是私有的，内部也是调用了AbstractBiMap的构造器来创建实例的，共传入了两个参数第一个参数是EnumMap，它用来存储`键-值`，第二个参数是HashMap，它用来存储`值-键`。比较有意思的是它在创建HashMap时使用了Maps工具类，通过传入一个期望的容量大小(这里是枚举值的数量)来创建HashMap实例，这还是比较符合规范的，阿里巴巴规范告诉我们创建HashMap最好是传入一个初始容量，最好是2的幂次方，这样有利于哈希的均匀分配。

由于构造器是私有的，所以它也提供了两个`create()`方法：

```java
  public static <K extends Enum<K>, V> EnumHashBiMap<K, V> create(Map<K, ? extends V> map) {
    EnumHashBiMap<K, V> bimap = create(EnumBiMap.inferKeyType(map));
    bimap.putAll(map);//放入Map
    return bimap;
  }

  static <K extends Enum<K>> Class<K> inferKeyType(Map<K, ?> map) {
    if (map instanceof EnumBiMap) {//是否为EnumBiMap类型
      return ((EnumBiMap<K, ?>) map).keyType();
    }
    if (map instanceof EnumHashBiMap) {//是否为EnumHashBiMap类型
      return ((EnumHashBiMap<K, ?>) map).keyType();
    }
    checkArgument(!map.isEmpty());//map非空验证
    return map.keySet().iterator().next().getDeclaringClass();//都不是
  }

  public static <K extends Enum<K>, V> EnumHashBiMap<K, V> create(Class<K> keyType) {
    return new EnumHashBiMap<>(keyType);//调用底层私有构造器
  }
```

很有既视感啊，和EnumBiMap很像啊，也是第一个create()方法内部调用第二个create()方法，最后还是调用私有的构造器。并且第一个create()方法调用inferKeyType来确定keyType的类型，然后调用第二个create()方法。这个只要看过EnumBiMap就很眼熟了。

##### 其他方法：

```java
  @GwtIncompatible // java.io.ObjectOutputStream
  private void writeObject(ObjectOutputStream stream) throws IOException {
    stream.defaultWriteObject();
    stream.writeObject(keyType);
    Serialization.writeMap(this, stream);
  }

  @SuppressWarnings("unchecked") // reading field populated by writeObject
  @GwtIncompatible // java.io.ObjectInputStream
  private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {
    stream.defaultReadObject();
    keyType = (Class<K>) stream.readObject();
    setDelegates(
        new EnumMap<K, V>(keyType), new HashMap<V, K>(keyType.getEnumConstants().length * 3 / 2));
    Serialization.populateMap(this, stream);
  }
```

这两个方法也看着很眼熟啊，由于EnumHashBiMap只是用了EnumMap作键，所以writeObject就只设了一个keyType。存储格式为：键的类型，键值对数量，第一个键，第一个值，第二个键，第二个值...

而readObject则略有不同。这里在调用setDelegates()方法构造正反视图时是直接“new”了一个HashMap注意这里的泛型是`值所为键，键作为值`。并且HashMap的初始容量是枚举值数量的1.5倍，然后再将Map的键值对写入到调用这个方法的对象中。

这里它还重写了put()和forcePut()两个方法，也值得注意一下：

```java
  @CanIgnoreReturnValue
  @Override
  public V put(K key, @Nullable V value) {
    return super.put(key, value);
  }

  @CanIgnoreReturnValue
  @Override
  public V forcePut(K key, @Nullable V value) {
    return super.forcePut(key, value);
  }
```

可以看到它内部还是调用了AbstractBiMap的方法，**只不过是在参数传入的时候做了key不能为空的限制**。

### 总结

EnumBiMap和EnumHashBiMap是实现了AbstractBiMap的唯二的两个类，他们有相似之处也有不同之处，他们都是用了EnumMap作为键，所以它们的键都不可为空！它们都有自己的writeObject()和readObject()方法。它们自己定义的方法很少，如果调一般都会调用它们的父类AbstractBiMap的方法，这也在侧面证明了AbstractBiMap的重要性。