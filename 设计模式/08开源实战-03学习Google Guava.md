### 学习Google Guava

Google Guava 是 Google 公司内部 Java 开发工具库的开源版本。它提供了一些 JDK 没有提供的功能，以及对 JDK 已有功能的增强功能。其中就包括：集合（Collections）、缓存（Caching）、原生类型支持（Primitives Support）、并发库（Concurrency Libraries）、通用注解（Common Annotation）、字符串处理（Strings Processing）、数学计算（Math）、I/O、事件总线（EventBus）等等。

- __如何发现通用的功能模块？__

  复用和业务无关。

- __如何开发通用的功能模块？__

  “产品意识”，“服务意识”，代码质量，不要重复造轮子。

#### Google Guava中用到的设计模式

- __Builder模式__

  使用：

  ```java
  public class CacheDemo {
    public static void main(String[] args) {
      Cache<String, String> cache = CacheBuilder.newBuilder()
              .initialCapacity(100)
              .maximumSize(1000)
              .expireAfterWrite(10, TimeUnit.MINUTES)
              .build();
  
      cache.put("key1", "value1");
      String value = cache.getIfPresent("key1");
      System.out.println(value);
    }
  }
  ```

  源码：

  ```java
  public <K1 extends K, V1 extends V> Cache<K1, V1> build() {
    this.checkWeightWithWeigher();//参数校验
    this.checkNonLoadingCache();//参数校验
    return new LocalManualCache(this);
  }
  
  private void checkNonLoadingCache() {
    Preconditions.checkState(this.refreshNanos == -1L, "refreshAfterWrite requires a LoadingCache");
  }
  
  private void checkWeightWithWeigher() {
    if (this.weigher == null) {
      Preconditions.checkState(this.maximumWeight == -1L, "maximumWeight requires weigher");
    } else if (this.strictParsing) {
      Preconditions.checkState(this.maximumWeight != -1L, "weigher requires maximumWeight");
    } else if (this.maximumWeight == -1L) {
      logger.log(Level.WARNING, "ignoring weigher specified without maximumWeight");
    }
  
  }
  ```

  checkWeightWithWeigher()方法和checkNonLoadingCache()方法进行参数校验

- __Wrapper 模式__

  代理模式、装饰器、适配器模式可以统称为 Wrapper 模式，通过 Wrapper 类二次封装原始类。

  ```java
  @GwtCompatible
  public abstract class ForwardingCollection<E> extends ForwardingObject implements Collection<E> {
    protected ForwardingCollection() {
    }
  
    protected abstract Collection<E> delegate();
  
    public Iterator<E> iterator() {
      return this.delegate().iterator();
    }
  
    public int size() {
      return this.delegate().size();
    }
  
    @CanIgnoreReturnValue
    public boolean removeAll(Collection<?> collection) {
      return this.delegate().removeAll(collection);
    }
  
    public boolean isEmpty() {
      return this.delegate().isEmpty();
    }
  
    public boolean contains(Object object) {
      return this.delegate().contains(object);
    }
  
    @CanIgnoreReturnValue
    public boolean add(E element) {
      return this.delegate().add(element);
    }
  
    @CanIgnoreReturnValue
    public boolean remove(Object object) {
      return this.delegate().remove(object);
    }
  
    public boolean containsAll(Collection<?> collection) {
      return this.delegate().containsAll(collection);
    }
  
    @CanIgnoreReturnValue
    public boolean addAll(Collection<? extends E> collection) {
      return this.delegate().addAll(collection);
    }
  
    @CanIgnoreReturnValue
    public boolean retainAll(Collection<?> collection) {
      return this.delegate().retainAll(collection);
    }
  
    public void clear() {
      this.delegate().clear();
    }
  
    public Object[] toArray() {
      return this.delegate().toArray();
    }
    
    //...省略部分代码...
  }
  
  
  public class AddLoggingCollection<E> extends ForwardingCollection<E> {
    private static final Logger logger = LoggerFactory.getLogger(AddLoggingCollection.class);
    private Collection<E> originalCollection;
  
    public AddLoggingCollection(Collection<E> originalCollection) {
      this.originalCollection = originalCollection;
    }
  
    @Override
    protected Collection delegate() {
      return this.originalCollection;
    }
  
    @Override
    public boolean add(E element) {
      logger.info("Add element: " + element);
      return this.delegate().add(element);
    }
  
    @Override
    public boolean addAll(Collection<? extends E> collection) {
      logger.info("Size of elements to add: " + collection.size());
      return this.delegate().addAll(collection);
    }
  
  }
  
  ```

  这个 ForwardingCollection 类是一个“默认 Wrapper 类”或者叫“缺省 Wrapper 类”。

  为了简化 Wrapper 模式的代码实现，Guava 提供一系列缺省的 Forwarding 类。用户在实现自己的 Wrapper 类的时候，基于缺省的 Forwarding 类来扩展，就可以只实现自己关心的方法，其他不关心的方法使用缺省 Forwarding 类的实现，就像 AddLoggingCollection 类的实现那样。

- __Immutable模式（不变模式）__

  不变模式可以分为两类，一类是普通不变模式，另一类是深度不变模式（Deeply Immutable Pattern）。普通的不变模式指的是，对象中包含的引用对象是可以改变的。如果不特别说明，通常我们所说的不变模式，指的就是普通的不变模式。深度不变模式指的是，对象包含的引用对象也不可变。

  ```java
  // 普通不变模式
  public class User {
    private String name;
    private int age;
    private Address addr;
    
    public User(String name, int age, Address addr) {
      this.name = name;
      this.age = age;
      this.addr = addr;
    }
    // 只有getter方法，无setter方法...
  }
  
  public class Address {
    private String province;
    private String city;
    public Address(String province, String city) {
      this.province = province;
      this.city= city;
    }
    // 有getter方法，也有setter方法...
  }
  
  // 深度不变模式
  public class User {
    private String name;
    private int age;
    private Address addr;
    
    public User(String name, int age, Address addr) {
      this.name = name;
      this.age = age;
      this.addr = addr;
    }
    // 只有getter方法，无setter方法...
  }
  
  public class Address {
    private String province;
    private String city;
    public Address(String province, String city) {
      this.province = province;
      this.city= city;
    }
    // 只有getter方法，无setter方法..
  }
  ```

  

  Google Guava 针对集合类（Collection、List、Set、Map…）提供了对应的不变集合类（ImmutableCollection、ImmutableList、ImmutableSet、ImmutableMap…）。

  属于普通不变模式，集合中的对象不会增删，但是对象的成员变量（或叫属性值）是可以改变的。

  ```java
  public class ImmutableDemo {
    public static void main(String[] args) {
      List<String> originalList = new ArrayList<>();
      originalList.add("a");
      originalList.add("b");
      originalList.add("c");
  
      List<String> jdkUnmodifiableList = Collections.unmodifiableList(originalList);
      List<String> guavaImmutableList = ImmutableList.copyOf(originalList);
  
      //jdkUnmodifiableList.add("d"); // 抛出UnsupportedOperationException
      // guavaImmutableList.add("d"); // 抛出UnsupportedOperationException
      originalList.add("d");
  
      print(originalList); // a b c d
      print(jdkUnmodifiableList); // a b c d
      print(guavaImmutableList); // a b c
    }
  
    private static void print(List<String> list) {
      for (String s : list) {
        System.out.print(s + " ");
      }
      System.out.println();
    }
  }
  ```

  JDK的Collections.unmodifiableList()方法保存了原始集合的引用

  ```java
  public static <T> List<T> unmodifiableList(List<? extends T> list) {
          return (list instanceof RandomAccess ?
                  new UnmodifiableRandomAccessList<>(list) :
                  new UnmodifiableList<>(list));
      }
  static class UnmodifiableList<E> extends UnmodifiableCollection<E>
                                    implements List<E> {
          
          final List<? extends E> list;
  
          UnmodifiableList(List<? extends E> list) {
              super(list);
              this.list = list;//直接使用原始集合的引用
          }
  
          ....
      }
  ```

  

  Google Guava的ImmutableList.copyOf()会拷贝原始集合的值。

  ```java
    public static <E> ImmutableList<E> copyOf(Collection<? extends E> elements) {
      return construct(elements.toArray());// 将原始集合的值拷贝为数组来保存了
    }
  ```

  

#### 函数式编程

```java
List<Integer> list = Lists.newArrayList(0, 1, 2, 3, 4, 5, 6, 7, 8, 9);
List<String> strings = Lists.transform(list, new Function<Integer, String>() {
    @Override
    public String apply(Integer input) {
        return String.valueOf(input);
    }
});
```

