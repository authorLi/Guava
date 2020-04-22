# Guava源码之AbstractIndexedListIterator

### 简介

此抽象类提供了一种ListIterator接口的基本实现，该抽象类支持通过下标检索集合中的元素。但是它不支持`remove()`、`set()`和`add()`这三个方法。

### 类概览

此抽象类的继承情况如下：

```properties
Iterator
     |-----ListIterator------------|
     |-----UnModifiableIterator    |    
            |-----UnModifiableListIterator
                         |-----AbstractIndexedListIterator
```

这几个类具体来看看，先来看`UnModifiableIterator`：

```java
@GwtCompatible
public abstract class UnmodifiableIterator<E> implements Iterator<E> {
  /** Constructor for use by subclasses. */
  protected UnmodifiableIterator() {}
  
  @Deprecated
  @Override
  public final void remove() {
    throw new UnsupportedOperationException();
  }
}
```

它继承了`Iterator`接口，声明了一个**protected**类型的构造器。其次重写了**remove()**方法，将其内部直接抛出错误。这里官方的介绍里面说的就是：这是一个不支持remove方法的迭代器。但是很明显的这个方法被**@Deprecated**修饰，**表明它是不建议被使用的**。

再来看`UnModifiableListIterator`：

```java
@GwtCompatible
public abstract class UnmodifiableListIterator<E>
    extends UnmodifiableIterator<E> implements ListIterator<E> {
  /** Constructor for use by subclasses. */
  protected UnmodifiableListIterator() {}

  @Deprecated @Override public final void add(E e) {
    throw new UnsupportedOperationException();
  }

  @Deprecated @Override public final void set(E e) {
    throw new UnsupportedOperationException();
  }
}
```

此抽象类继承了`UnModifiableIterator`，实现了`ListIterator`，算是实现了这两个的功能，值得一提的是它把`add()`和`set()`两个方法给“**禁用**”了。这是一个在不支持删除的基础上，进一步不支持添加和替换最新返回值的迭代器

接下来正式进入`AbstractIndexedListIterator`：

##### 属性：

```java
  private final int size;
  private int position;
```

属性一共包括两个：

- size：元素的个数
- position：表示初始位置，也可以看做指针。

##### 构造器：

```java
  protected AbstractIndexedListIterator(int size) {
    this(size, 0);
  }

  protected AbstractIndexedListIterator(int size, int position) {
    checkPositionIndex(position, size);
    this.size = size;
    this.position = position;
  }
```

- 第一个构造器：只指定元素大小，起始下标默认为第一位。
- 第二个构造器：同时指定了元素大小和起始下标。

为了方便理解，这里翻译一下源码中的一段注释：

> 构造器会构建一个指定大小的指定操作起始位置的迭代器。也就是说，首次调用`nextLink()`方法会返回position的大小。对`next()`方法的首次调用将会返回该索引处的元素。

##### 方法：

- get(int index)：返回指定索引的元素。**next()方法将会调用此方法**。
- hasNext()：判断是否存在下一个元素。**使用final修饰**，内部是直接比对position和size的大小，因为迭代过程中会改变position，将其视为指针使用 。
- next()：取出当前指针指向的元素。**使用final修饰**，如果不存在下一个将会抛出`NoSuchElementException`异常。内部调用了get()方法来取值，并且会使position加一。
- nextIndex()：返回当前索引，即返回position。**使用final修饰**。
- hasPrevious()：查看是否存在上一个元素。**使用final修饰**，内部直接比较position和0，以此查看是否迭代到了第一个元素。
- previous()：返回当前指针的上一个元素。**使用final修饰**，如果不存在上一个将会抛出`NoSuchElementException`异常。内部依旧调用了get()方法取值，并且会使position减一。
- previousIndex()：返回上一个元素的索引。**使用final修饰**，内部是直接返回了(position - 1)这个值。

### 总结：

此抽象了支持了根据下表来检索并且可以在构建时直接从指定大小的集合的指定下标来操作。除此之外，实现了ListIterator接口使它可以同时向前或向后操作指针，继承了UnModifiableListIterator使它可以“禁用”`添加`、`设置`和`删除`这三个功能。

