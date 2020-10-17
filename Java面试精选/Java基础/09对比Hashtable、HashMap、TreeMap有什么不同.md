### 对比Hashtable、HashMap、TreeMap有什么不同

#### 典型回答

Hashtable、HashMap、TreeMap 都是最常见的一些 Map 实现，是以键值对的形式存储和操作数据的容器类型。

Hashtable 是早期 Java 类库提供的一个哈希表实现，本身是同步的，不支持 null 键和值，由于同步导致的性能开销，所以已经很少被推荐使用。

HashMap 是应用更加广泛的哈希表实现，行为上大致上与 HashTable 一致，主要区别在于 HashMap 不是同步的，支持 null 键和值等。通常情况下，HashMap 进行 put 或者 get 操作，可以达到常数时间的性能，所以它是绝大部分利用键值对存取场景的首选，比如，实现一个用户 ID 和用户信息对应的运行时存储结构。

TreeMap 则是基于红黑树的一种提供顺序访问的 Map，和 HashMap 不同，它的 get、put、remove 之类操作都是 O（log(n)）的时间复杂度，具体顺序可以由指定的 Comparator 来决定，或者根据键的自然顺序来判断。

#### 知识扩展

__Map的整体结构__

![Map类图](https://github.com/hejinalex/notes/blob/master/Java%E9%9D%A2%E8%AF%95%E7%B2%BE%E9%80%89/Java%E5%9F%BA%E7%A1%80/Map%E7%B1%BB%E5%9B%BE.png?raw=true)

Hashtable 比较特别，作为类似 Vector、Stack 的早期集合相关类型，它是扩展了 Dictionary 类的，类结构上与 HashMap 之类明显不同。

HashMap 等其他 Map 实现则是都扩展了 AbstractMap，里面包含了通用方法抽象。不同 Map 的用途，从类图结构就能体现出来，设计目的已经体现在不同接口上。

大部分使用 Map 的场景，通常就是放入、访问或者删除，而对顺序没有特别要求，HashMap 在这种情况下基本是最好的选择。__HashMap 的性能表现非常依赖于哈希码的有效性，请务必掌握 hashCode 和 equals 的一些基本约定__，比如：

- equals 相等，hashCode 一定要相等。
- 重写了 hashCode 也要重写 equals。
- hashCode 需要保持一致性，状态改变返回的哈希值仍然要一致。
- equals 的对称、反射、传递等特性。



虽然 LinkedHashMap 和 TreeMap 都可以保证某种顺序，但二者还是非常不同的：

- LinkedHashMap 通常提供的是遍历顺序符合插入顺序，它的实现是通过为条目（键值对）维护一个双向链表。注意，通过特定构造函数，我们可以创建反映访问顺序的实例，所谓的 put、get、compute 等，都算作“访问”。

  这种行为适用于一些特定应用场景，例如，我们构建一个空间占用敏感的资源池，希望可以自动将最不常被访问的对象释放掉，这就可以利用 LinkedHashMap 提供的机制来实现，参考下面的示例：

  ```java
  import java.util.LinkedHashMap;
  import java.util.Map;  
  public class LinkedHashMapSample {
      public static void main(String[] args) {
          LinkedHashMap<String, String> accessOrderedMap = new LinkedHashMap<String, String>(16, 0.75F, true){
              @Override
              protected boolean removeEldestEntry(Map.Entry<String, String> eldest) { // 实现自定义删除策略，否则行为就和普遍Map没有区别
                  return size() > 3;
              }
          };
          accessOrderedMap.put("Project1", "Valhalla");
          accessOrderedMap.put("Project2", "Panama");
          accessOrderedMap.put("Project3", "Loom");
          accessOrderedMap.forEach( (k,v) -> {
              System.out.println(k +":" + v);
          });
          // 模拟访问
          accessOrderedMap.get("Project2");
          accessOrderedMap.get("Project2");
          accessOrderedMap.get("Project3");
          System.out.println("Iterate over should be not affected:");
          accessOrderedMap.forEach( (k,v) -> {
              System.out.println(k +":" + v);
          });
          // 触发删除
          accessOrderedMap.put("Project4", "Mission Control");
          System.out.println("Oldest entry should be removed:");
          accessOrderedMap.forEach( (k,v) -> {// 遍历顺序不变
              System.out.println(k +":" + v);
          });
      }
  }
  
  ```
  
- 对于 TreeMap，它的整体顺序是由键的顺序关系决定的，通过 Comparator 或 Comparable（自然顺序）来决定。

  构建一个具有优先级的调度系统的问题，其本质就是个典型的优先队列场景，Java 标准库提供了基于二叉堆实现的 PriorityQueue，它们都是依赖于同一种排序机制，当然也包括 TreeMap 的马甲 TreeSet。

  类似 hashCode 和 equals 的约定，为了避免模棱两可的情况，自然顺序同样需要符合一个约定，就是 compareTo 的返回值需要和 equals 一致，否则就会出现模棱两可情况。

  TreeMap的put方法实现：

  ```java
  public V put(K key, V value) {
      Entry<K,V> t = …
      cmp = k.compareTo(t.key);
      if (cmp < 0)
          t = t.left;
      else if (cmp > 0)
          t = t.right;
      else
          return t.setValue(value);
          // ...
     }
  ```

  当我不遵守约定时，两个不符合唯一性（equals）要求的对象被当作是同一个（因为，compareTo 返回 0），这会导致歧义的行为表现。

  

__HashMap 源码分析__

主要分析点：

- HashMap 内部实现基本点分析。
- 容量（capacity）和负载系数（load factor）。
- 树化 。



