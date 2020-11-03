### OkHttp

#### OkHttpClient类中的建造者模式

OkHttpClient实例化：

```java
//方法1 默认参数
OkHttpClient okHttpClient = new OkHttpClient();

//方法2 使用建造者模式创建OkHttpClient实例
OkHttpClient okHttpClient = new OkHttpClient.Builder()
                    .connectTimeout(60, TimeUnit.SECONDS)
                    .readTimeout(60, TimeUnit.SECONDS)
                    .addInterceptor(new HeaderInterceptor())
                    .addInterceptor(new LoggingInterceptor())
                    .build();
```

OkHttpClient源码：

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocketCall.Factory {
  // 构造函数1
  public OkHttpClient() {
    this(new Builder()); // 调用构造函数2
  }  
  // 构造函数2
  private OkHttpClient(Builder builder) {
    ......
  }  
  public Builder newBuilder() {
    return new Builder(this);
  }
  // Builder类
  public static final class Builder { 
    int callTimeout;
    int connectTimeout;
    int readTimeout;
    int writeTimeout;
    int pingInterval;  
     //·····很多参数
      
    public Builder() {
    	this.callTimeout = 0;
        this.connectTimeout = 10000;
        this.readTimeout = 10000;
        this.writeTimeout = 10000;
        this.pingInterval = 0;
        //·····默认参数
    }   
      
	public OkHttpClient.Builder callTimeout(long timeout, TimeUnit unit) {
        this.callTimeout = Util.checkDuration("timeout", timeout, unit);
        return this;
    }
      
    //·····其他设置参数的方法
      
    //通过build方法生成OkHttpClient实例
    public OkHttpClient build() {
      return new OkHttpClient(this);
    }  
   }  
}
```



#### OkHttp中的职责链模式

拦截器：

```java
public interface Interceptor {
  //拦截Chain,并触发下一个拦截器的调用
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    //返回请求
    Request request();
    //对请求进行处理
    Response proceed(Request request) throws IOException;
    ......
  }
}
```

拦截器的使用：

```java
class LoggingInterceptor implements Interceptor {
  	@Override 
    public Response intercept(Interceptor.Chain chain) throws IOException {
        Request request = chain.request();

        long t1 = System.nanoTime();
        logger.info(String.format("Sending request %s on %s%n%s",
            request.url(), chain.connection(), request.headers()));

        Response response = chain.proceed(request);

        long t2 = System.nanoTime();
        logger.info(String.format("Received response for %s in %.1fms%n%s",
            response.request().url(), (t2 - t1) / 1e6d, response.headers()));

        return response;
  }
}
```

进行网络请求：

```java
Response response = getResponseWithInterceptorChain();

Response getResponseWithInterceptorChain() throws IOException {
    // 创建拦截器的list
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());//库使用者自己定义的拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    // 创建RealInterceptorChain，传入的index索引为0
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
}
```

拦截器链中的处理：

```java
public Response proceed(Request request) throws IOException {
    return proceed(request, transmitter, exchange);
}

public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
    RealConnection connection) throws IOException {
  ......
  // 创建下一个RealInterceptorChain，将index+1（下一个拦截器索引）传入
  RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
      connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
      writeTimeout);
  //获取当前的拦截器
  Interceptor interceptor = interceptors.get(index);
  //通过Interceptor的intercept进行处理
  Response response = interceptor.intercept(next);
  ......
  return response;
}
```

- 创建下一个RealInterceptorChain对象，并将当前RealInterceptorChain中的变量当成参数传入，并将索引index+1传入。
- 获取当前index位置上的拦截器，第一次创建时传入的index为0，表示获取第一个拦截器，后面会将index+1进入传入，用于获取下一个拦截器。
- 在Interceptor的intercept中，将下一个RealInterceptorChain传入，内部会调用下一个RealInterceptorChain的proceed方法

![](https://raw.githubusercontent.com/hejinalex/notes/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/OkHttp%23Interceptor.png)

#### OkHttp中的享元模式

OkHttp中的调度器使用的线程池，线程池是享元模式：

```Java
public final class Dispatcher {
  private @Nullable ExecutorService executorService;

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
}
```

这个线程池的特点是：

- 重用之前的线程(线程池中未被销毁的线程)
- 如果没有可用的线程，则创建一个新线程并添加到池中
- 池中线程默认为60s未使用就被终止和移除
- 长期闲置的池将会不消耗任何资源
- 任务立即执行，线程自动回收

执行execute方法时，首先会先执行SynchronousQueue的offer方法提交任务，并查询线程池中是否有空闲线程来执行SynchronousQueue的poll方法来移除任务。如果有，则配对成功，将任务交给这个空闲线程。否则，配对失败，创建新的线程去处理任务；当线程池中的线程空闲时，会执行SynchronousQueue的poll方法等待执行SynchronousQueue中新提交的任务。若超过60s依然没有任务提交到SynchronousQueue，这个空闲线程就会终止；因为maximumPoolSize是无界的，所以提交任务的速度 > 线程池中线程处理任务的速度就要不断创建新线程；每次提交任务，都会立即有线程去处理，因此适用于处理大量、耗时少的任务。