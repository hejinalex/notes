### Java有几种文件拷贝方式？哪一种最高效？

#### 典型回答

Java 有多种比较典型的文件拷贝实现方式，比如：

利用 java.io 类库，直接为源文件构建一个 FileInputStream 读取，然后再为目标文件构建一个 FileOutputStream，完成写入工作。

```java
public static void copyFileByStream(File source, File dest) throws
        IOException {
    try (InputStream is = new FileInputStream(source);
         OutputStream os = new FileOutputStream(dest);){
        byte[] buffer = new byte[1024];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    }
 }

```

或者，利用 java.nio 类库提供的 transferTo 或 transferFrom 方法实现。

```java
public static void copyFileByChannel(File source, File dest) throws
        IOException {
    try (FileChannel sourceChannel = new FileInputStream(source).getChannel();
         FileChannel targetChannel = new FileOutputStream(dest).getChannel();){
        for (long count = sourceChannel.size(); count>0; ) {
            long transferred = sourceChannel.transferTo(sourceChannel.position(), count, targetChannel);
            sourceChannel.position(sourceChannel.position() + transferred);
            count -= transferred;
        }
    }
 }
```

当然，Java 标准类库本身已经提供了几种 Files.copy 的实现。

对于 Copy 的效率，这个其实与操作系统和配置等情况相关，总体上来说，NIO transferTo/From 的方式可能更快，因为它更能利用现代操作系统底层机制，避免不必要拷贝和上下文切换。

#### 考点分析

- 不同的 copy 方式，底层机制有什么区别？
- 为什么零拷贝（zero-copy）可能有性能优势？
- Buffer 分类与使用。
- Direct Buffer 对垃圾收集等方面的影响与实践选择。

#### 知识扩展

- __拷贝实现机制分析__

  需要理解用户态空间（User Space）和内核态空间（Kernel Space），这是操作系统层面的基本概念，操作系统内核、硬件驱动等运行在内核态空间，具有相对高的特权；而用户态空间，则是给普通应用和服务使用。

  当我们使用输入输出流进行读写时，实际上是进行了多次上下文切换，比如应用读取数据时，先在内核态将数据从磁盘读取到内核缓存，再切换到用户态将数据从内核缓存读取到用户缓存。

  ![](https://raw.githubusercontent.com/hejinalex/notes/master/Java%E9%9D%A2%E8%AF%95%E7%B2%BE%E9%80%89/Java%E5%9F%BA%E7%A1%80/%E8%AF%BB%E5%8F%96%E5%86%99%E5%85%A5.png)

  这种方式会带来一定的额外开销，可能会降低 IO 效率。

  而基于 NIO transferTo 的实现方式，在 Linux 和 Unix 上，则会使用到零拷贝技术，数据传输并不需要用户态参与，省去了上下文切换的开销和不必要的内存拷贝，进而可能提高应用拷贝性能。注意，transferTo 不仅仅是可以用在文件拷贝中，与其类似的，例如读取磁盘文件，然后进行 Socket 发送，同样可以享受这种机制带来的性能和扩展性提高。

  ![](https://raw.githubusercontent.com/hejinalex/notes/master/Java%E9%9D%A2%E8%AF%95%E7%B2%BE%E9%80%89/Java%E5%9F%BA%E7%A1%80/transferTo%20%E7%9A%84%E4%BC%A0%E8%BE%93%E8%BF%87%E7%A8%8B.png)

- __Java IO/NIO 源码结构__

  Java 标准库也提供了文件拷贝方法（java.nio.file.Files.copy）,有几个不同的 copy 方法。

  ```java
  public static Path copy(Path source, Path target, CopyOption... options)
      throws IOException
      
  public static long copy(InputStream in, Path target, CopyOption... options)
      throws IOException
      
  public static long copy(Path source, OutputStream out) 
  throws IOException
  
  ```

  copy 不仅仅是支持文件之间操作，没有人限定输入输出流一定是针对文件的，这是两个很实用的工具方法。

  后面两种 copy 实现，能够在方法实现里直接看到使用的是 InputStream.transferTo()，其内部实现其实是 stream 在用户态的读写。

  第一种方法的分析可以参考下面片段：

  ```java
  public static Path copy(Path source, Path target, CopyOption... options)
      throws IOException
   {
      FileSystemProvider provider = provider(source);
      if (provider(target) == provider) {
          // same provider
          provider.copy(source, target, options);//这是本文分析的路径
      } else {
          // different providers
          CopyMoveHelper.copyToForeignTarget(source, target, options);
      }
      return target;
  }
  ```

  

- __提高拷贝等IO操作的性能__
  - 在程序中，使用缓存等机制，合理减少 IO 次数（在网络通信中，如 TCP 传输，window 大小也可以看作是类似思路）。
  - 使用 transferTo 等机制，减少上下文切换和额外 IO 操作。
  - 尽量减少不必要的转换过程，比如编解码；对象序列化和反序列化，比如操作文本文件或者网络通信，如果不是过程中需要使用文本信息，可以考虑不要将二进制信息转换成字符串，直接传输二进制信息。

- __掌握 NIO Buffer__

  Buffer 是 NIO 操作数据的基本工具，Java 为每种原始数据类型都提供了相应的 Buffer 实现（布尔除外），所以掌握和使用 Buffer 是十分必要的，尤其是涉及 Direct Buffer 等使用，因为其在垃圾收集等方面的特殊性，更要重点掌握。

  Buffer 有几个基本属性：

  - capacity，它反映这个 Buffer 到底有多大，也就是数组的长度。
  - position，要操作的数据起始位置。
  - limit，相当于操作的限额。在读取或者写入时，limit 的意义很明显是不一样的。比如，读取操作时，很可能将 limit 设置到所容纳数据的上限；而在写入时，则会设置容量或容量以下的可写限度。
  - mark，记录上一次 postion 的位置，默认是 0，算是一个便利性的考虑，往往不是必须的。

  Buffer 的基本操作：

  - 我们创建了一个 ByteBuffer，准备放入数据，capacity 当然就是缓冲区大小，而 position 就是 0，limit 默认就是 capacity 的大小。
  - 当我们写入几个字节的数据时，position 就会跟着水涨船高，但是它不可能超过 limit 的大小。
  - 如果我们想把前面写入的数据读出来，需要调用 flip 方法，将 position 设置为 0，limit 设置为以前的 position 那里。
  - 如果还想从头再读一遍，可以调用 rewind，让 limit 不变，position 再次设置为 0。

- __Direct Buffer 和垃圾收集__

  两种特别的 Buffer:

  - Direct Buffer：如果我们看 Buffer 的方法定义，你会发现它定义了 isDirect() 方法，返回当前 Buffer 是否是 Direct 类型。这是因为 Java 提供了堆内和堆外（Direct）Buffer，我们可以以它的 allocate 或者 allocateDirect 方法直接创建。
  - MappedByteBuffer：它将文件按照指定大小直接映射为内存区域，当程序访问这个内存区域时将直接操作这块儿文件数据，省去了将数据从内核空间向用户空间传输的损耗。我们可以使用FileChannel.map创建 MappedByteBuffer，它本质上也是种 Direct Buffer。

  在实际使用中，Java 会尽量对 Direct Buffer 仅做本地 IO 操作，对于很多大数据量的 IO 密集操作，可能会带来非常大的性能优势，因为：

  - Direct Buffer 生命周期内内存地址都不会再发生更改，进而内核可以安全地对其进行访问，很多 IO 操作会很高效。
  - 减少了堆内对象存储的可能额外维护工作，所以访问效率可能有所提高。

  但是请注意，Direct Buffer 创建和销毁过程中，都会比一般的堆内 Buffer 增加部分开销，所以通常都建议用于长期使用、数据较大的场景。

  大多数垃圾收集过程中，都不会主动收集 Direct Buffer，它的垃圾收集过程，就是Cleaner（一个内部实现）和幻象引用（PhantomReference）机制，其本身不是 public 类型，内部实现了一个 Deallocator 负责销毁的逻辑。对它的销毁往往要拖到 full GC 的时候，所以使用不当很容易导致 OutOfMemoryError。

  对于 Direct Buffer 的回收，有几个建议：

  - 在应用程序中，显式地调用 System.gc() 来强制触发。
  - 另外一种思路是，在大量使用 Direct Buffer 的部分框架中，框架会自己在程序中调用释放方法，Netty 就是这么做的，有兴趣可以参考其实现（PlatformDependent0）。
  - 重复使用 Direct Buffer。