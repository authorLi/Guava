# Guava源码之ForwardingFuture

### 类简介

此类会将一个Future 的所有方法都转移到另一个Future，它的子类可以根据装饰者模式来重写它的一个或多个方法。此类中有一个抽象的静态内部类**SimpleForwardingFuture**。通常都去实现此子类。

```java
public abstract class ForwardingFuture<V> extends ForwardingObject
    implements Future<V> {
```

可以看到它是继承了**ForwardingObject**，拥有了其能够把此类的方法转移到另一个类的方法上的功能。

另外它实现了**Future**类，它就拥有了进行异步运算的相关功能。

### 构造器

```java
protected ForwardingFuture() {}
```

仅有一个构造器，其权限修饰符为**protected**

### 方法

##### delegate()

```java
@Override protected abstract Future<V> delegate();
```

##### cancel()

```java
@Override
public boolean cancel(boolean mayInterruptIfRunning) {
  return delegate().cancel(mayInterruptIfRunning);
}
```

##### isCancelled()

```java
@Override
public boolean isCancelled() {
  return delegate().isCancelled();
}
```

##### isDone()

```java
@Override
public boolean isDone() {
  return delegate().isDone();
}
```

##### get()

```java
@Override
public V get() throws InterruptedException, ExecutionException {
  return delegate().get();
}
```

##### get(long timeout, TimeUnit unit)

```java
@Override
public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
  return delegate().get(timeout, unit);
}
```

### 静态内部抽象子类

##### SimpleForwardingFuture

```java
public abstract static class SimpleForwardingFuture<V> 
    extends ForwardingFuture<V> {
  private final Future<V> delegate;

  protected SimpleForwardingFuture(Future<V> delegate) {
    this.delegate = Preconditions.checkNotNull(delegate);
  }

  @Override
  protected final Future<V> delegate() {
    return delegate;
  }
  
}
```

此子类中定义了一个Future类对象作为代理类又使用final作为修饰，并在其内部的构造器中对代理对象进行了初始化，最后重写了**delegate()**方法，返回了代理对象。

所以此子类相对于其父类主要是对代理对象进行了一个初始化赋值和提供代理对象的操作





# Future类

这里也来看一下Future类的代码，与ForwardingFuture类不同，Future接口是JUC包下的，也就是说它是JDK自带的接口

### 类简介

```java
public interface Future<V> {
```

- Future类代表了一个异步计算的结果。
- 它的方法用于检验`计算是否完成`、`等待计算完成`和`检索计算结果`。
- 它的计算结果只能在计算完成后通过`get()`方法获取，必要时它会阻塞住，直到它计算完成准备就绪。
- 可以使用`cancel()`方法来进行取消计算的操作。
- 它还额外提供了方法来查询异步计算是**正常计算结束**还是**被取消**。
- 如果只是想提供一个可以被取消的方法而不在意其返回的计算结果来使用Future，可以使用Future的泛型”Future<?>“并在结束后返回”null“作为标志即可。

### 小例子

```java
interface ArchiveSearcher { String search(String target); }
class App {
  ExecutorService executor = ...
  ArchiveSearcher searcher = ...
  void showSearch(final String target)
      throws InterruptedException {
    Future<String> future
      = executor.submit(new Callable<String>() {
        public String call() {
            return searcher.search(target);
        }});
    displayOtherThings(); // do other things while searching
    try {
      displayText(future.get()); // use future
    } catch (ExecutionException ex) { cleanup(); return; }
  }
}}
```

### 子接口RunnableFuture

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```

此子接口不仅继承了Future，而且还继承了Runnable接口。接口内部定义了`run()`方法来进行异步计算，除非任务被取消。

### 子类FutureTask

FutureTask是RunnableFuture的子接口，他重写了其父接口的`run()`方法，这就使得它可以被**Executor**来执行。上面的示例代码的执行部分就可以被替换为：

```java
FutureTask<String> future =
  new FutureTask<String>(new Callable<String>() {
    public String call() {
      return searcher.search(target);
  }});
executor.execute(future);
```

### 方法

##### cancel()

```java
boolean cancel(boolean mayInterruptIfRunning);
```

- **尝试**去取消异步计算，如果**计算已经完成**或**计算已经被取消**或**优于其他原因无法被取消**，则此尝试会失败。
- 如果**已经成功**或者**调用此方法之前，异步计算还未开始**，那么异步计算将永远不会开始
- 如果异步计算已经开始，那么传入的参数`mayInterruptIfRunning`将决定是否应当中断执行此任务的线程以尝试停止该异步计算
- 此方法返回后，再调用`isDone()`方法将会永远返回true来表示任务完成
- 如果此方法返回true，那么在此方法执行结束后再调用`isCancelled()`方法将永远返回true

##### isCancelled()

```java
boolean isCancelled();
```

- 如果在异步计算正常完成之前此任务被取消，则返回true

##### isDone()

```java
boolean isDone();
```

- 当任务完成时会返回true
- 任务完成指的是：
  - 异步计算正常完成
  - 出现异常导致任务退出
  - 任务被取消

##### get()

```java
V get() throws InterruptedException, ExecutionException;
```

- 如果有必要，那么等待异步计算完成并检索其结果
- 可能会抛出以下三种异常：
  - CancellationException：当异步计算被取消时抛出
  - ExecutionException：当异步计算过程中出错抛出
  - InterruptedException：当执行异步计算的线程被中断时抛出

##### get(long timeout, @NotNull TimeUnit unit)

- 如果有必要，最多等待给定的时间以完成异步计算并检索其值(如果返回值可用)
- 可能会抛出以下四种异常：
  - CancellationException：当异步计算被取消时抛出
  - ExecutionException：当异步计算过程中出错抛出
  - InterruptedException：当执行异步计算的线程被中断时抛出
  - TimeoutException：如果等待超过指定时间时抛出

