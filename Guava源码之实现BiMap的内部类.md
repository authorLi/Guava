# 实现BiMap的内部类

### ConstrainedBiMap——ConstrainedMap的内部类

```java
  private static class ConstrainedBiMap<K, V> extends ConstrainedMap<K, V>
      implements BiMap<K, V> {
    
    volatile BiMap<V, K> inverse;

    ConstrainedBiMap(BiMap<K, V> delegate, @Nullable BiMap<V, K> inverse,
        MapConstraint<? super K, ? super V> constraint) {
      super(delegate, constraint);
      this.inverse = inverse;
    }

    @Override protected BiMap<K, V> delegate() {
      return (BiMap<K, V>) super.delegate();
    }

    @Override
    public V forcePut(K key, V value) {
      constraint.checkKeyValue(key, value);
      return delegate().forcePut(key, value);
    }

    @Override
    public BiMap<V, K> inverse() {
      if (inverse == null) {
        inverse = new ConstrainedBiMap<V, K>(delegate().inverse(), this,
            new InverseConstraint<V, K>(constraint));
      }
      return inverse;
    }

    @Override public Set<V> values() {
      return delegate().values();
    }
  }
```

此类继承了`BiMap`和`ConstrainedMap`两个类，而ConstrainedMap又继承了`ForwardingMap`这个类之前也说到过，`AbstractBiMap`也继承了这个类，再说一下它是可以把调用它的方法“转发”到另一个map对象中，因为这个类里面维护了一个protected修饰的抽象map对象，而它的方法内部调用了这个抽象对象的方法。

然后关于**ConstrainedMap**，构造器中会检查参数是否非空，插入单个元素时会检查键和值，插入Map时会检查Map，实际上也是遍历每个键值对来验证。

而ConstrainedBiMap融化了两个父类的特点，在支持验证的基础上也支持正反两个视图。类中声明的唯一属性**inverse**是用**volatile**修饰保证了可见性。构造器中调用了ConstrainedMap的构造器设置了delegate和constraint，又设置了反视图inverse。forcePut()方法里也会先去检验键和值，再去调用delegate对象的forcePut()方法。inverse()方法里直接赋值inverse属性一个新的对象，此对象也是ConstrainedBiMap类型，然后调用它的构造器(这次不是继承了同一个父类的内部类了)。

### SynchronizedBiMap——Synchronized的内部类

```java
    @VisibleForTesting static class SynchronizedBiMap<K, V>
      extends SynchronizedMap<K, V> implements BiMap<K, V>, Serializable {
    private transient Set<V> valueSet;
    private transient BiMap<V, K> inverse;

    private SynchronizedBiMap(BiMap<K, V> delegate, @Nullable Object mutex,
        @Nullable BiMap<V, K> inverse) {
      super(delegate, mutex);
      this.inverse = inverse;
    }

    @Override BiMap<K, V> delegate() {
      return (BiMap<K, V>) super.delegate();
    }

    @Override public Set<V> values() {
      synchronized (mutex) {
        if (valueSet == null) {
          valueSet = set(delegate().values(), mutex);
        }
        return valueSet;
      }
    }

    @Override
    public V forcePut(K key, V value) {
      synchronized (mutex) {
        return delegate().forcePut(key, value);
      }
    }

    @Override
    public BiMap<V, K> inverse() {
      synchronized (mutex) {
        if (inverse == null) {
          inverse
              = new SynchronizedBiMap<V, K>(delegate().inverse(), mutex, this);
        }
        return inverse;
      }
    }

    private static final long serialVersionUID = 0;
  }
```

继承了`SynchronizedMap`类，实现了`BiMap`和`Serializable`接口。这也能看出来它支持的三个特性：同步、正反视图和序列化。其构造器中也是调用了父类的构造器，设置了用来同步的对象，而此对象也广泛用于本类中的`values()`、`forcePut()`和`inverse()`方法中用于做同步。

### UnModifiableBiMap——Maps的内部类

```java
  private static class UnmodifiableBiMap<K, V>
      extends ForwardingMap<K, V> implements BiMap<K, V>, Serializable {
    final Map<K, V> unmodifiableMap;
    final BiMap<? extends K, ? extends V> delegate;
    BiMap<V, K> inverse;
    transient Set<V> values;

    UnmodifiableBiMap(BiMap<? extends K, ? extends V> delegate,
        @Nullable BiMap<V, K> inverse) {
      unmodifiableMap = Collections.unmodifiableMap(delegate);
      this.delegate = delegate;
      this.inverse = inverse;
    }

    @Override protected Map<K, V> delegate() {
      return unmodifiableMap;
    }

    @Override
    public V forcePut(K key, V value) {
      throw new UnsupportedOperationException();
    }

    @Override
    public BiMap<V, K> inverse() {
      BiMap<V, K> result = inverse;
      return (result == null)
          ? inverse = new UnmodifiableBiMap<V, K>(delegate.inverse(), this)
          : result;
    }

    @Override public Set<V> values() {
      Set<V> result = values;
      return (result == null)
          ? values = Collections.unmodifiableSet(delegate.values())
          : result;
    }

    private static final long serialVersionUID = 0;
  }
```

继承了`ForwardingMap`类，实现了`BiMap`和`Serializable`接口。和ConstrainedBiMap相比都是继承了ForwardingMap，但是ConstrainedBiMap不支持序列化。看这个类的名字就知道，它是一个不可修改的双向Map，所以如果调用它的强制方法会报错。

### 总结：

其实直接和间接继承了BiMap的内部类还有很多，前面的文章中有的也介绍过。这些内部类大多数都是编写于一些工具类中，如Maps、Synchronized，这些工具类将这些应用于不同情况下的一系列列表或集合或键值对映射定义在一起，方便其他类使用。