# Guava源码之缓存(三)

### 抽象类ForwardingCache

#### 介绍

这是一个会将所有对其方法调用转发到另一缓存的对应方法的缓存。其实看到forwarding开头就知道它的作用。它的子类应该实现一个或多个方法以根据装饰器模式的需要修改后背储存的行为。

#### 关于抽象类

```java
@Beta
public abstract class ForwardingCache<K, V> extends ForwardingObject implements Cache<K, V> {
```

继承了`ForwardingObject`，这样就有了转发的功能，之前阅读Map时也有类似实现；实现了`Cache`接口，这样就拥有了缓存的基本方法

#### 重要的方法

##### delegate()

```java
@Override
protected abstract Cache<K, V> delegate();
```

这个方法对于方法的转发是很重要的，其目的是用于返回要转发到的Cache对象。但是在本类中它是抽象的，需要它的实现类根据情况而实现此方法

##### 其他方法

它实现了Cache接口的所有方法并且在方法内部都是先调用`delegate()`获取它代理的Cache对象，然后再调用代理对象的对应方法

#### 静态内部抽象类SimpleForwardingCache

```java
@Beta
public abstract static class SimpleForwardingCache<K, V> extends ForwardingCache<K, V> {
  private final Cache<K, V> delegate;

  protected SimpleForwardingCache(Cache<K, V> delegate) {
    this.delegate = Preconditions.checkNotNull(delegate);
  }

  @Override
  protected final Cache<K, V> delegate() {
    return delegate;
  }
}
```

代码非常简单，先从类声明看起。官方注释说明：这是一个`ForwardingCache`抽象类的简单实现，它可以通过构造器传入一个它的子类作为代理对象

- 使用static修饰，可以直接通过类调用；abstract修饰，说明是一个抽象类，还可以进一步实现
- 类里面声明了一个final修饰的不可变的代理Cache接口对象delegate
- 实现了父类的`delegate()`方法，返回此不可变的代理对象delegate
- 定义了构造器，并在构造器中可以传入一个实现了Cache接口的类的对象赋值给成员变量delegate，在赋值前会对传入的代理对象做非空判断



### 抽象类ForwardingLoadingCache

#### 介绍

依然是一个会将方法调用转发到另一个缓存的缓存，这点和`ForwardingCache`相同。但是需要注意的是`get()`、`getUnchecked()`和`apply()`都暴露了相同的底层功能，所以也许需要把它们看成一个组来实现

#### 关于抽象类

```java
@Beta
public abstract class ForwardingLoadingCache<K, V>
    extends ForwardingCache<K, V> implements LoadingCache<K, V> {
```

继承了`ForwardingCache`，可以使转发方法调用；实现了`LoadingCache`，使得值可以被缓存自动加载

#### 重要的方法

##### delegate()

```java
@Override
protected abstract LoadingCache<K, V> delegate();
```

重写了父类的delegate()方法，但依然是抽象的，没有具体实现

##### 其他方法

其他方法也全部都在内部先调用`delegate()`方法拿到代理对象，之后再调用代理对象的对应方法

#### 静态内部抽象类SimpleForwardingLoadingCache

```java
@Beta
public abstract static class SimpleForwardingLoadingCache<K, V>
    extends ForwardingLoadingCache<K, V> {
  private final LoadingCache<K, V> delegate;

  protected SimpleForwardingLoadingCache(LoadingCache<K, V> delegate) {
    this.delegate = Preconditions.checkNotNull(delegate);
  }

  @Override
  protected final LoadingCache<K, V> delegate() {
    return delegate;
  }
}
```

这个实现就类似于`ForwardingCache#SimpleForwardingCache`，它也是继承了自己的外部类，然后又声明了代理对象delegate，并重写了外部类的delegate()方法来获取delegate对象。对于构造器也提供了delegate的赋值

### 总结

今天这两个类的设计极其类似，所以放到一起看，他们分别是给`Cache`接口和`LoadingCache`接口扩展了代理对象功能，然后在内部又编写了一个简单的实现来确定delegate和获取delegate对象