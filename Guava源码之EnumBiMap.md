# Guava源码之EnumBiMap

### 简介

这是一个使用两个EnumMap实例来实现的BiMap。其键与值都不允许为空！

##### 关于EnumMap(参考自廖雪峰的官方网站)

简单提一句，EnumMap是java.util包下的类，它的key是Enum类型的。我们都知道HashMap是根据`hashcode()`方法来计算内部数组的索引的，这种以空间换时间的方式提高了效率。但是如果你的key是Enum(枚举)类型的话，你就可以使用EnumMap，它的内部以一种非常紧凑的数组存储value，并且可以直接根据Enum类型的key直接定位到内部数组的索引，**并不需要计算hashcode**，所以效率更高且不需要占用额外空间。

代码用例如下：

```java
public class Main {
    public static void main(String[] args) {
        Map<DayOfWeek, String> map = new EnumMap<>(DayOfWeek.class);
        map.put(DayOfWeek.MONDAY, "星期一");
        map.put(DayOfWeek.TUESDAY, "星期二");
        map.put(DayOfWeek.WEDNESDAY, "星期三");
        map.put(DayOfWeek.THURSDAY, "星期四");
        map.put(DayOfWeek.FRIDAY, "星期五");
        map.put(DayOfWeek.SATURDAY, "星期六");
        map.put(DayOfWeek.SUNDAY, "星期日");
        System.out.println(map);
        System.out.println(map.get(DayOfWeek.MONDAY));
    }
}

```

### 类概览

```java
@GwtCompatible(emulated = true)
public final class EnumBiMap<K extends Enum<K>, V extends Enum<V>> extends AbstractBiMap<K, V> {
```

此类与其父类一样使用@GwtCompatible注解修饰。比较特殊的是它还使用了**final关键字**来修饰，证明其是一个**不可变类**。另外此类的泛型告诉我们它的key与value都是**Enum(枚举)**类型。

##### 属性：

```java
  private transient Class<K> keyType;

  private transient Class<V> valueType;

  @GwtIncompatible 
  private static final long serialVersionUID = 0;
```

类属性比较简单，keyType与valueType代表了键与值的Class对象

##### 构造器：

```java
  private EnumBiMap(Class<K> keyType, Class<V> valueType) {
    super(new EnumMap<K, V>(keyType), new EnumMap<V, K>(valueType));
    this.keyType = keyType;
    this.valueType = valueType;
  }
```

可以看到这个构造器是**私有的**，他不能直接被调用，为了创建EnumBiMap对象，EnumBiMap提供了`create()`方法：

```java
  public static <K extends Enum<K>, V extends Enum<V>> EnumBiMap<K, V> create(Map<K, V> map) {
    EnumBiMap<K, V> bimap = create(inferKeyType(map), inferValueType(map));//调用另一个create()方法
    bimap.putAll(map);
    return bimap;
  }

  static <K extends Enum<K>> Class<K> inferKeyType(Map<K, ?> map) {
    if (map instanceof EnumBiMap) {//如果是EnumBiMap类型则返回EnumBiMap的keyType
      return ((EnumBiMap<K, ?>) map).keyType();
    }
    if (map instanceof EnumHashBiMap) {//如果是EnumHashBiMap类型(这个类也是AbstractBiMap的实现类)，则返回EnumHashBiMap的keyType
      return ((EnumHashBiMap<K, ?>) map).keyType();
    }
    checkArgument(!map.isEmpty());
    return map.keySet().iterator().next().getDeclaringClass();//都不是则返回键的Class对象
  }

  private static <V extends Enum<V>> Class<V> inferValueType(Map<?, V> map) {
    if (map instanceof EnumBiMap) {//如果是EnumBiMap类型则返回EnumBiMap的valueType
      return ((EnumBiMap<?, V>) map).valueType;
    }
    checkArgument(!map.isEmpty());
    return map.values().iterator().next().getDeclaringClass();//不满足以上条件则返回value的Class对象
  }

  public static <K extends Enum<K>, V extends Enum<V>> EnumBiMap<K, V> create(
      Class<K> keyType, Class<V> valueType) {
    return new EnumBiMap<>(keyType, valueType);//这里最终调用私有的构造器
  }
```

如上面的代码所示，create()方法有两个重载的方法

其中第一个create()方法是一个静态的方法，允许直接通过类来调用，它接收的是一个Map对象然后返回一个相同键值类型的EnumBiMap。具体步骤是先调用`inferKeyType()`和`inferValueType()`这两个方法来获取传入Map的键与值的Class对象，然后就调用第二个create()方法

这第二个create()方法则是直接调用了私有构造器完成对象的创建。

##### 其他方法：

```java
  @GwtIncompatible // java.io.ObjectOutputStream
  private void writeObject(ObjectOutputStream stream) throws IOException {
    stream.defaultWriteObject();
    stream.writeObject(keyType);
    stream.writeObject(valueType);
    Serialization.writeMap(this, stream);
  }

  @SuppressWarnings("unchecked") // reading fields populated by writeObject
  @GwtIncompatible // java.io.ObjectInputStream
  private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {
    stream.defaultReadObject();
    keyType = (Class<K>) stream.readObject();
    valueType = (Class<V>) stream.readObject();
    setDelegates(new EnumMap<K, V>(keyType), new EnumMap<V, K>(valueType));//设置好数据类型，此方法是AbstractBiMap的
    Serialization.populateMap(this, stream);//填充Map
  }
```

writeObject()与readObject()这两个方法都是由@GwtIncompatible修饰，表明与Gwt不兼容，其次这两个是序列化方法与反序列化方法。

序列化时格式为：键类型，值类型，键值对个数，第一个键，第一个值，第二个键，第二个值...

反序列化时又调用了AbstractBiMap的setDelegate()方法将两个EnumMap"连接"起来，最后再调用Serialization工具类的populateMap()方法来将数据填充到调用对象中。

### 总结：

此实现类本身的方法不是很多，主要的就是构造器和create()方法比较重要，它将构造器设置为私有避免了对构造器的直接调用，而两个create()方法来代替开发人员调用构造器来创建实例。