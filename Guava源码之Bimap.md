# Guava源码之BiMap

### 简介

先说一下这个“Bi”是什么意思，Bi是`bidirectional`的缩写，意为“双向的”。如何理解双向我们下面说。

### 类概览

#### 类

![1586071834304](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586071834304.png)

如图可以看出，这个接口是用注解`@GwtCompatible`修饰的，意思是说它是与Google的GWT Web框架兼容的。这个接口是直接继承了java包的Map接口的，所以可以看出这个BiMap的接口是所有双向Map的顶级接口，所有双向Map都是在此接口上设计的。

##### @GwtCompatible

鉴于第一次遇到这个注解，我们就来看一下这个注解。

![1586072192289](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586072192289.png)

这个注解共有两个属性：`serializable`和`emulated`分别代表了此注解修饰的类是否兼容序列化和模拟源是否与JVM相同。这两个属性的默认值都为false。这个注解实际上是一个标示性注解。

#### 方法

![1586072424328](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586072424328.png)

此接口共有5个方法：

##### put方法

![1586072529242](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586072529242.png)

因为是put所以可以忽略返回值，返回值为键值对中的“值”。并且根据注解可以看出**它的键和值都是允许为空的**。他还有一个特殊的要求，那就是不允许重复的“值”存在，即**能够保证Map中“值”的唯一性**。如果调用put方法插入同样的值那么会**报错**。那有人会问了，如果我非要插入相同的之，或者我不知道会不会有相同的值会被插入到这个Map中，但是我要保证这个插入的过程必须完成不能报错我应该怎么办？你可以使用下面这个方法：forcePut。

##### forcePut方法

![1586072875136](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586072875136.png)

这个方法同样也是可以忽略返回值的并且允许键值为空。为了解决上面的问题，Guava设计了forcePut方法，这个方法实现了插入同样的“值”不会是程序报错，但是为了保证BiMap的“值”一致性，后面的键会覆盖前面的键，即可以认为前面的键值对被BiMap删除。它是会在插入之前删除与插入的“值”相同的键值对并且不会返回删除的“值”对应的键

##### putAll方法

![1586073312608](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586073312608.png)

此方法直接继承自Map接口，此方法将传入的Map添加进BiMap，至于如果Map有重复的值该如何处理（是用put还是forcePut），这个貌似要看具体的实现。

##### values方法

![1586073608127](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586073608127.png)

此方法直接继承自Map接口，值得注意的一点是它返回的是一个Set结合，因为BiMap是能够保证“值”唯一性的，所以不必担心返回的“值”的集合中存在重复的“值”。

##### inverse方法

![1586073807783](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586073807783.png)

这个方法是BiMap中非常重要的一个方法。如源码中所说，这个方法会返回一个反视图，这个反视图也是一个BiMap对象，它之所以成为反视图是因为这个反视图中的键和值与原BiMap中的键和值是相反的，即这个反视图对于原BiMap来说是`值-键`对。更为神奇的是如果改变了原BiMap的键值对，那么此反视图也会跟着改变，我初步猜测它们是操作的同一份底层数据，这个方法的具体实现需要各个实现类来完成，并且每个实现类的实现方式并不同。

### 总体设计

此接口中声明了添加和删除键值对的方法，而为了保证“值”一致性又设计了forcePut这个方法，并且连values方法的返回值类型也由`Collection`变为了`Set`，这个是很值得关注的，另外需要关注的一点是inverse这个方法，虽说它的具体实现是依靠它的实现类来完成的，而这些实现类的实现方式又有所不同，但是根源上是一样的，都是为了实现反视图这个概念来提供`值-键`对，并且它们的改变还会互相影响。也就是说BiMap之所以是双向的，也是得益于inverse这个方法。

# 另一个包下的BiMap

在`org.apache.commons.collections`这个包下也同样存在一个名为BiMap的类。它同样也提供了双向的Map

### 类概览

![1586179503085](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586179503085.png)

此包下的BiMap同样是个接口，与Guava BiMap不同的是它是继承自`IterableMap​`

### 方法

![1586179712496](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586179712496.png)

##### mapIterator方法

![1586179971750](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586179971750.png)

此方法继承自IterableMap，提供了一种高效率的Map迭代器，它不需要使用Map Entry对象存储键值对，这可以提高性能

##### put方法

![1586179961714](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586179961714.png)

这个方法与Guava的put方法类似，它也是会在插入之前检查有没有相同的“值”，如果有，则删除之前插入的那一个键值对

##### getKey方法

![1586180066358](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586180066358.png)

它就是一个根据value值来寻找key的方法，当不存在想要查找的键时，它并不会报错，而是会**返回null**

##### removeValue方法

![1586180222773](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586180222773.png)

删除所传入“值”对应的键值对，当不存在符合条件的键值对则**返回null**

##### inverseBiMap方法

![1586180327343](C:\Users\52489\AppData\Roaming\Typora\typora-user-images\1586180327343.png)

此方法与Guava中BiMap的BiMap也很相似，就是返回一个反视图，并且返回图的改变也会导致源BiMap的改变。因为是接口所以设计是一样的，至于实现是否相同还要看两个包下的实现子类。

### 总体设计

没有丢下其父类IterableMap的mapIterator方法，除此之外也设计了一些插入和删除方法，但是与Guava不同的是，它没有强制添加的方法forcePut，不知道它遇到那些批量插入还不能报错的步骤如何实现？

# 总体差别

其实上面也说到了一些，commons没有强制插入的方法，Guava没有根据值删除键值对的功能，感觉两个类虽然整体设计一样但还是有些细微的差别，大概是两个类想要涉及的使用场景不相同吧。。。