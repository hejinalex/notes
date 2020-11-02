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

