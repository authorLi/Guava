# Guava源码之Maps工具类(三)

### 内部类——UnModifiableEntries

```java
  static class UnmodifiableEntries<K, V>
      extends ForwardingCollection<Entry<K, V>> {
    private final Collection<Entry<K, V>> entries;

    UnmodifiableEntries(Collection<Entry<K, V>> entries) {
      this.entries = entries;
    }

    @Override protected Collection<Entry<K, V>> delegate() {
      return entries;
    }

    @Override public Iterator<Entry<K, V>> iterator() {
      final Iterator<Entry<K, V>> delegate = super.iterator();
      return new UnmodifiableIterator<Entry<K, V>>() {
        @Override
        public boolean hasNext() {
          return delegate.hasNext();
        }

        @Override public Entry<K, V> next() {
          return unmodifiableEntry(delegate.next());
        }
      };
    }

    // See java.util.Collections.UnmodifiableEntrySet for details on attacks.

    @Override public Object[] toArray() {
      return standardToArray();
    }

    @Override public <T> T[] toArray(T[] array) {
      return standardToArray(array);
    }
  }
```

- 此类实现了`ForwardingCollection`抽象类，也就是说它也支持把请求转发到另一个代理对象上。
- 内部定义了一个**final**修饰的Collection类型的对象entries，带有泛型“Entry<K,V>”，表示它与Map的结构类似。
- delegate()方法直接返回了entries对象。
- iterator()方法先是获取了entries对象的迭代器，然后返回了一个`UnModifiableIterator`对象，且重写了`hasNext()`和`next()`方法，让它们都去调用entries对象的对应方法
- 两个重载的`toArray()`方法都调用了父类的方法，无参的方法最后将调用entries对象的toArray()方法；而有参的方法最后将会调用`ObjectArrays`工具类的方法将迭代器中的数据放到一个数组里并返回



### 内部类——UnModifiableEntrySet

```java
  static class UnmodifiableEntrySet<K, V>
      extends UnmodifiableEntries<K, V> implements Set<Entry<K, V>> {
    UnmodifiableEntrySet(Set<Entry<K, V>> entries) {
      super(entries);
    }

    @Override public boolean equals(@Nullable Object object) {
      return Sets.equalsImpl(this, object);
    }

    @Override public int hashCode() {
      return Sets.hashCodeImpl(this);
    }
  }
```

- 继承了`UnModifiableEntries`类和`Set<Entry<K, V>>`接口
- equals()方法调用了`Sets`工具类的`equalsImpl()`方法，它会比对此对象中的**内容和大小**是否与指定的object相等
- hashCode()方法同样调用了`Sets`工具类的`hashCodeImpl()`方法，这是Sets工具类的哈希值实现。

在Maps中存在UnModifiableEntrySet的静态构造器：

```java
static <K, V> Set<Entry<K, V>> unmodifiableEntrySet(
    Set<Entry<K, V>> entrySet) {
  return new UnmodifiableEntrySet<K, V>(
      Collections.unmodifiableSet(entrySet));
}
```

它的内部是使用`Collections`工具类的方法实现的，如下：

```java
public static <T> Set<T> unmodifiableSet(Set<? extends T> s) {
    return new UnmodifiableSet<>(s);
}
```

可以看到内部是创建了一个`UnModifiableSet`对象，而此类是Collections的一个内部类:

```java
static class UnmodifiableSet<E> extends UnmodifiableCollection<E>
                             implements Set<E>, Serializable {
    private static final long serialVersionUID = -9215047833775013803L;

    UnmodifiableSet(Set<? extends E> s)     {super(s);}
    public boolean equals(Object o) {return o == this || c.equals(o);}
    public int hashCode()           {return c.hashCode();}
}
```

而它的构造器是直接调用的父类的构造方法，`UnModifiableSet`的父类为`UnModifiableCollection`，它同样是Collections的内部类：

```java
static class UnmodifiableCollection<E> implements Collection<E>, Serializable {
        private static final long serialVersionUID = 1820017752578914078L;

        final Collection<? extends E> c;

        UnmodifiableCollection(Collection<? extends E> c) {
            if (c==null)
                throw new NullPointerException();
            this.c = c;
        }

        public int size()                   {return c.size();}
        public boolean isEmpty()            {return c.isEmpty();}
        public boolean contains(Object o)   {return c.contains(o);}
        public Object[] toArray()           {return c.toArray();}
        public <T> T[] toArray(T[] a)       {return c.toArray(a);}
        public String toString()            {return c.toString();}

        public Iterator<E> iterator() {
            return new Iterator<E>() {
                private final Iterator<? extends E> i = c.iterator();

                public boolean hasNext() {return i.hasNext();}
                public E next()          {return i.next();}
                public void remove() {
                    throw new UnsupportedOperationException();
                }
                @Override
                public void forEachRemaining(Consumer<? super E> action) {
                    // Use backing collection version
                    i.forEachRemaining(action);
                }
            };
        }

        public boolean add(E e) {
            throw new UnsupportedOperationException();
        }
        public boolean remove(Object o) {
            throw new UnsupportedOperationException();
        }

        public boolean containsAll(Collection<?> coll) {
            return c.containsAll(coll);
        }
        public boolean addAll(Collection<? extends E> coll) {
            throw new UnsupportedOperationException();
        }
        public boolean removeAll(Collection<?> coll) {
            throw new UnsupportedOperationException();
        }
        public boolean retainAll(Collection<?> coll) {
            throw new UnsupportedOperationException();
        }
        public void clear() {
            throw new UnsupportedOperationException();
        }

        // Override default methods in Collection
        @Override
        public void forEach(Consumer<? super E> action) {
            c.forEach(action);
        }
        @Override
        public boolean removeIf(Predicate<? super E> filter) {
            throw new UnsupportedOperationException();
        }
        @SuppressWarnings("unchecked")
        @Override
        public Spliterator<E> spliterator() {
            return (Spliterator<E>)c.spliterator();
        }
        @SuppressWarnings("unchecked")
        @Override
        public Stream<E> stream() {
            return (Stream<E>)c.stream();
        }
        @SuppressWarnings("unchecked")
        @Override
        public Stream<E> parallelStream() {
            return (Stream<E>)c.parallelStream();
        }
    }
```

- 此类的内部定义了一个Collection<? extends E>类型的属性
- 里面的很多方法都是直接调用了属性的同名方法；或者由于其不可修改的特性，也有一部分方法被调用时会抛出异常，类对不可更改的方法的设置不是不写此方法，而是在方法内部抛出异常，这样的设计之前也介绍过
- iterator()：返回迭代器的方法。它先是获取到指定好的类属性“c”的构造器对象“i”，然后`hasNext()`、`next()`和`forEachRemaining()`方法都直接调用“i”对象的方法。`remove()`会因为不可修改特性而抛出异常。