# Guava源码之Maps工具类(一)

### 枚举——EntryFunction

```java
  private enum EntryFunction implements Function<Entry<?, ?>, Object> {
    KEY {
      @Override
      @Nullable
      public Object apply(Entry<?, ?> entry) {
        return entry.getKey();
      }
    },
    VALUE {
      @Override
      @Nullable
      public Object apply(Entry<?, ?> entry) {
        return entry.getValue();
      }
    };
  }
```

- private修饰，表示是私有的。
- 继承了Function接口，可以根据输入值决定输出值。
- 枚举只有两个：`key`与`value`，且它们都实现了`apply()`方法，作用分别是获得输入Entry的键和值。



### 内部类——MapDifferenceImpl

```java
  static class MapDifferenceImpl<K, V> implements MapDifference<K, V> {
    final Map<K, V> onlyOnLeft;
    final Map<K, V> onlyOnRight;
    final Map<K, V> onBoth;
    final Map<K, ValueDifference<V>> differences;

    MapDifferenceImpl(Map<K, V> onlyOnLeft,
        Map<K, V> onlyOnRight, Map<K, V> onBoth,
        Map<K, ValueDifference<V>> differences) {
      this.onlyOnLeft = unmodifiableMap(onlyOnLeft);
      this.onlyOnRight = unmodifiableMap(onlyOnRight);
      this.onBoth = unmodifiableMap(onBoth);
      this.differences = unmodifiableMap(differences);
    }

    @Override
    public boolean areEqual() {
      return onlyOnLeft.isEmpty() && onlyOnRight.isEmpty() && differences.isEmpty();
    }

    @Override
    public Map<K, V> entriesOnlyOnLeft() {
      return onlyOnLeft;
    }

    @Override
    public Map<K, V> entriesOnlyOnRight() {
      return onlyOnRight;
    }

    @Override
    public Map<K, V> entriesInCommon() {
      return onBoth;
    }

    @Override
    public Map<K, ValueDifference<V>> entriesDiffering() {
      return differences;
    }

    @Override public boolean equals(Object object) {
      if (object == this) {
        return true;
      }
      if (object instanceof MapDifference) {
        MapDifference<?, ?> other = (MapDifference<?, ?>) object;
        return entriesOnlyOnLeft().equals(other.entriesOnlyOnLeft())
            && entriesOnlyOnRight().equals(other.entriesOnlyOnRight())
            && entriesInCommon().equals(other.entriesInCommon())
            && entriesDiffering().equals(other.entriesDiffering());
      }
      return false;
    }

    @Override public int hashCode() {
      return Objects.hashCode(entriesOnlyOnLeft(), entriesOnlyOnRight(),
          entriesInCommon(), entriesDiffering());
    }

    @Override public String toString() {
      if (areEqual()) {
        return "equal";
      }

      StringBuilder result = new StringBuilder("not equal");
      if (!onlyOnLeft.isEmpty()) {
        result.append(": only on left=").append(onlyOnLeft);
      }
      if (!onlyOnRight.isEmpty()) {
        result.append(": only on right=").append(onlyOnRight);
      }
      if (!differences.isEmpty()) {
        result.append(": value differences=").append(differences);
      }
      return result.toString();
    }
  }
```

##### 父接口——MapDifference

介绍这个类要先看其父接口：`MapDifference`，这里只做简单介绍：

- areEqual()：判断两个Map是否相等
- entriesOnlyOnLeft()：返回一个包含左边的Map拥有但是右边的Map没有的所有元素的不可变Map对象
- entriesOnlyOnRight()：返回一个包含左边的Map没有但是右边的Map拥有的所有元素的不可变Map对象
- entriesInCommon()：返回一个包含两个Map都拥有的所有元素的不可变Map对象**（取交集）**
- entriesDiffering()：返回一个包含两个Map都拥有的**键相同，但是值不同**的所有元素的不可变Map对象
- equals()：比较给定对象与该实例的相等性。如果给定对象也是MapDifference类型，那么如果调用四个`entries`开头的方法结果都一样的话，那么就返回true
- hashcode()：返回该实例的哈希值
- 子接口`ValueDifference`:
  - leftValue()：返回一个左边的Map的值，此值可能为空
  - rightValue()：返回一个右边的Map的值，此值可能为空
  - equals()：如果调用leftValue()和rightValue()两个方法值都相同，那么会返回true
  - hashCode()：返回哈希值，此值等于`Arrays.asList(leftValue(), rightValue()).hashCode()`

##### MapDifferenceImpl

下面正式来看`MapDifferenceImpl`类：

- 首先可以看到它有四个属性`onlyOnLeft`、`onlyOnRight`、`OnBoth`和`differences`，这就分别对应了其父接口的那四个方法。
- 构造器并非直接给四个属性赋值，而是有一步判断是否是`SortedMap`类型，即传入Map是否有序
- 那四个方法经过重写直接返回对应的属性值
- areEqual()：此方法的实现为如果onlyOnLeft、onlyOnRight和differences的值同时为空，则返回true
- equals()：此方法的实现为：先查看是否为MapDifference类型，然后再比较四个方法的值如果都相等的话，则返回true
- hashCode()：计算哈希值，内部实现为调用Arrays.hashCode()，参数为调用四个方法的返回值
- toString()：此方法会先调用areEqual()方法查看两个Map是否完全相等，如果相等直接返回“equal”，如果不相等或不全相等，那么会将左边独有的，右边独有的和两边值不同的都展示出来



### 内部类——ValueDifferenceImpl

```java
  static class ValueDifferenceImpl<V>
      implements MapDifference.ValueDifference<V> {
    private final V left;
    private final V right;

    static <V> ValueDifference<V> create(@Nullable V left, @Nullable V right) {
      return new ValueDifferenceImpl<V>(left, right);
    }

    private ValueDifferenceImpl(@Nullable V left, @Nullable V right) {
      this.left = left;
      this.right = right;
    }

    @Override
    public V leftValue() {
      return left;
    }

    @Override
    public V rightValue() {
      return right;
    }

    @Override public boolean equals(@Nullable Object object) {
      if (object instanceof MapDifference.ValueDifference) {
        MapDifference.ValueDifference<?> that =
            (MapDifference.ValueDifference<?>) object;
        return Objects.equal(this.left, that.leftValue())
            && Objects.equal(this.right, that.rightValue());
      }
      return false;
    }

    @Override public int hashCode() {
      return Objects.hashCode(left, right);
    }

    @Override public String toString() {
      return "(" + left + ", " + right + ")";
    }
  }
```

- 实现了MapDifference的子接口ValueDifference
- 定义了两个属性`left`和`right`，分别对应了父接口的两个方法：leftValue()和rightValue()
- **私有**构造器直接给两个属性赋值，对外开放了一个`create()`方法作为构造方法，内部调用了私有构造器
- equals()：此方法直接比较了当前对象与传入对象的left与right值，仅当都相等时才返回true
- hashCode()：此方法返回哈希值，内部依然调用了Arrays.hashCode()方法，参数为left和right
- toString()：直接输出`(左边的值，右边的值)`



### 内部类——SortedMapDifferenceImpl

```java
  static class SortedMapDifferenceImpl<K, V> extends MapDifferenceImpl<K, V>
      implements SortedMapDifference<K, V> {
    SortedMapDifferenceImpl(SortedMap<K, V> onlyOnLeft,
        SortedMap<K, V> onlyOnRight, SortedMap<K, V> onBoth,
        SortedMap<K, ValueDifference<V>> differences) {
      super(onlyOnLeft, onlyOnRight, onBoth, differences);
    }

    @Override public SortedMap<K, ValueDifference<V>> entriesDiffering() {
      return (SortedMap<K, ValueDifference<V>>) super.entriesDiffering();
    }

    @Override public SortedMap<K, V> entriesInCommon() {
      return (SortedMap<K, V>) super.entriesInCommon();
    }

    @Override public SortedMap<K, V> entriesOnlyOnLeft() {
      return (SortedMap<K, V>) super.entriesOnlyOnLeft();
    }

    @Override public SortedMap<K, V> entriesOnlyOnRight() {
      return (SortedMap<K, V>) super.entriesOnlyOnRight();
    }
  }
```

- 继承了`MapDifferenceImpl`类,实现了`SortedMapDifference`接口。其父接口只定义了四种方法，依然是`entriesOnlyOnLeft()`、`entriesOnlyOnRight()`、`entriesInCommon()`和`entriesDiffering()`这四个方法，但是不同的是，其返回值都被定义为了`SortedMap`，即它们在Map中是有序的
- 构造器为直接调用父类的构造器(这下可以理解为什么在MapDifferenceImpl的构造器中要判断类型了)
- 重写了“那四个方法”，不过都是调用父类的方法，然后做一下强转(转换成SortedMap类型)



### 总结

这里只看了Maps中进行Map比较的内部类(其实只是一部分，后边还有其他类也涉及到两Map比较……)，可以看到，它的接口设计存在“左Map”和"右Map"的概念来代替当前Map对象和传入对象,并且定义了四个属性,这让人联想到数据库的左连接、右连接，还有取交集的操作。而对于两Map都有的键但值不同的键值对也设计了一个属性来接收，并专门设计了一个子接口ValueDifference，它也是利用了“左右”的概念。Maps中也给出了ValueDifference的实现类。还有就是hashCode()方法的计算方式很特殊，底层是调用了Arrays.hashCode()方法，这个可以去了解下为什么这么做。