# EventBus源码分析

## 注册

```java
EventBus.getDefault().register(this);
```

### 单例模式获取EventBus实例

其中`getDefault()`是一个双重校验锁的单例方法：

```java
static volatile EventBus defaultInstance;

/** Convenience singleton for apps using a process-wide EventBus instance. */
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```
构造方法：

```java

private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
/**
 * Creates a new EventBus instance; each instance is a separate scope in which events are delivered. To use a
 * central bus, consider {@link #getDefault()}.
 */
public EventBus() {
    this(DEFAULT_BUILDER);
}
```

单例模式的构造方法一般都是private类型的，但是EventBus的构造方法是public类型的，因为EventBus 在代码使用过程中不仅仅只有一条总线，还有其他的订阅总线，订阅者可以注册到不同的 EventBus 总线，然后通过不同的 EventBus 总线发送数据。使用默认的中心总线，考虑使用`getDefault()`。

### Builder 模式构建 EventBus

`this(DEFAULT_BUILDER)`中的`DEFAULT_BUILDER`是一个`EventBusBuilder`，EventBus是使用建造者模式进行构建的：

```java
EventBus(EventBusBuilder builder) {
    logger = builder.getLogger();
    
    //Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType
    subscriptionsByEventType = new HashMap<>();
    //Map<Object, List<Class<?>>> typesBySubscriber
    typesBySubscriber = new HashMap<>();
    //Map<Class<?>, Object> stickyEvents
    stickyEvents = new ConcurrentHashMap<>();
    
    /**
    用于线程间调度
    **/
    mainThreadSupport = builder.getMainThreadSupport();
    mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
    backgroundPoster = new BackgroundPoster(this);
    asyncPoster = new AsyncPoster(this);
    //用于记录event生成索引
    indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
    //对已经注解过的Method的查找器，会对所设定过 @Subscriber 注解的的方法查找相应的Event
    subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
            builder.strictMethodVerification, builder.ignoreGeneratedIndex);
	
    //当调用事件处理函数发生异常是否需要打印Log
    logSubscriberExceptions = builder.logSubscriberExceptions;
    //当没有订阅者订阅这个消息的时候是否打印Log
    logNoSubscriberMessages = builder.logNoSubscriberMessages;
    //当调用事件处理函数，如果异常，是否需要发送Subscriber这个事件
    sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
    //当没有事件处理函数时，对事件处理是否需要发送sendNoSubscriberEvent这个标志
    sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
    //是否需要抛出SubscriberException
    throwSubscriberException = builder.throwSubscriberException;
    
    //与Event有继承关系的类是否都需要发送
    eventInheritance = builder.eventInheritance;
    //线程池 Executors.newCachedThreadPool()
    executorService = builder.executorService;
}
```

### 注册方法

```java
public void register(Object subscriber) {
    //1. 通过反射获取到订阅者的Class对象
    Class<?> subscriberClass = subscriber.getClass();
    //2. 通过Class对象找到对应的订阅者方法集合
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    //3. 遍历订阅者方法集合，将订阅者和订阅者方法订阅起来。
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

`List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);`

这是通过`subscriberMethodFinder`对象，来找到加了`@Subscriber`注解，接收事件的方法集合的：

```java
//订阅者的 Class 对象为 key，订阅者中的订阅方法 List 为 value
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    //1. 首先在 METHOD_CACHE 中查找该 Event 对应的订阅者集合是否已经存在，如果有直接返回
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    //2. 根据订阅者类 subscriberClass 查找相应的订阅方法
    if (ignoreGeneratedIndex) {//是否忽略生成 index
        //通过反射获取
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        //通过 SubscriberIndex 方式获取
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    
    //若订阅者中没有订阅方法，则抛异常
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        // 缓存订阅者的订阅方法 List
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}

//封装了EventBus中的参数，就是一个EventBus订阅方法包装类
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    String methodString;
}

```

`METHOD_CACHE `是一个 `ConcurrentHashMap`，以订阅者的 Class 对象为 key，订阅者中的订阅方法 List 为 value，缓存了注册过的订阅方法。如果有缓存则返回返回缓存，如果没有则继续往下执行。

这里看到 `ignoreGeneratedIndex `这个属性，意思为是否忽略生成 index，是在构造 `SubscriberMethodFinder `通过 `EventBusBuilder `的同名属性赋值的，默认为 false，当为 true 时，表示以反射的方式获取订阅者中的订阅方法，当为 false 时，则以 `Subscriber Index` 的方式获取。

`ignoreGeneratedIndex`为true：

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    // 创建并初始化 FindState 对象
    FindState findState = prepareFindState();
    // findState 与 subscriberClass 关联
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        // 使用反射的方式获取单个类的订阅方法
        findUsingReflectionInSingleClass(findState);
        // 使 findState.clazz 指向父类的 Class，继续获取
        findState.moveToSuperclass();
    }
    // 返回订阅者及其父类的订阅方法 List，并释放资源
    return getMethodsAndRelease(findState);
}
```

`ignoreGeneratedIndex`为false

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    //1.通过 prepareFindState 获取到 FindState(保存找到的注解过的方法的状态)
    FindState findState = prepareFindState();
     //2.findState 与 subscriberClass 关联
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        //获取订阅者信息
        //通过 SubscriberIndex 获取 findState.clazz 对应的 SubscriberInfo
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    // 逐个添加进 findState.subscriberMethods
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            // 使用反射的方式获取单个类的订阅方法
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}

// FindState 封装了所有的订阅者和订阅方法的集合。
static class FindState {
    //保存所有订阅方法
    final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
    //事件类型为Key，订阅方法为Value
    final Map<Class, Object> anyMethodByEventType = new HashMap<>();
    //订阅方法为Key，订阅者的Class对象为Value
    final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
    final StringBuilder methodKeyBuilder = new StringBuilder(128);

    Class<?> subscriberClass;
    Class<?> clazz;
    boolean skipSuperClasses;
    SubscriberInfo subscriberInfo;
    //......
}

```

它会先从缓存中读是否有缓存，没有的话接着对`ignoreGeneratedIndex`进行判断，通过默认的方式获取Eventbus，注册订阅者的话，`ignoreGeneratedIndex`的值为false，所以这里调用`findUsingInfo`：

通过 prepareFindState 获取到 FindState 对象，根据 FindState 对象可以进行下一步判断：

```java
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
private FindState prepareFindState() {
    //找到 FIND_STATE_POOL 对象池
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            //当找到了对应的FindState
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {//FindState 非空表示已经找到
                FIND_STATE_POOL[i] = null; //清空找到的这个FindState，为了下一次能接着复用这个FIND_STATE_POOL池
                return state;//返回该 FindState
            }
        }
    }
    //如果依然没找到，则创建一个新的 FindState
    return new FindState();
}
```



调用`initForSubscriber()`将订阅者的class对象与`findState.clazz`绑定，进入while循环。`getSubscriberInfo()`返回null，不成立，会进入`findUsingReflectionInSingleClass(findState)`方法查找`subscribeMethods`：

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // This is faster than getMethods, especially when subscribers are fat classes like Activities
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        //忽略非 public 和 static 的方法
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            // 获取订阅方法的所有参数
            Class<?>[] parameterTypes = method.getParameterTypes();
            // 订阅方法只能有一个参数，否则忽略
            if (parameterTypes.length == 1) {
                // 获取有 Subscribe 的注解
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    // 获取第一个参数
                    Class<?> eventType = parameterTypes[0];
                    // 检查 eventType 决定是否订阅，通常订阅者不能有多个 eventType 相同的订阅方法
                    if (findState.checkAdd(method, eventType)) {
                        // 获取线程模式
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        // 添加订阅方法进 List
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode, subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

它通过反射的方法来把订阅者中的订阅方法给找到，并且对订阅方法的信息进行包装后（封装成`SubscriberMethod`），添加至容器中。

`if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0)`

这句代码表明创建订阅者方法时，方法修饰符必须是public，不能是static的，abstract的。

`Class<?>[] parameterTypes = method.getParameterTypes();
if (parameterTypes.length == 1）`

这句代码表明订阅者方法的参数必须是1个。

在成功找到订阅者方法时，还有一个`findState.checkAdd(method, eventType)`方法：

```java
//事件类型为Key，订阅方法为Value
final Map<Class, Object> anyMethodByEventType = new HashMap<>();

boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
    		//put()方法执行之后，返回的是之前put的值
            Object existing = anyMethodByEventType.put(eventType, method);
            if (existing == null) {
                return true;
            } else {
                if (existing instanceof Method) {
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);
                }
                //根据方法签名来检查
                return checkAddWithMethodSignature(method, eventType);
            }
        }

		//订阅方法为Key，订阅者的Class对象为Value
		final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();

        private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());

            String methodKey = methodKeyBuilder.toString();
            Class<?> methodClass = method.getDeclaringClass();
            //put方法返回的是put之前的对象
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
            //如果methodClassOld不存在或者是methodClassOld的父类的话，则表明是它的父类，直接返回true。否则，就表明在它的子类中也找到了相应的订阅，执行的 put 操作是一个 revert 操作，put 进去的是 methodClassOld，而不是 methodClass
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
                // Only add if not already found in a sub class
                return true;
            } else {
                //这里是一个revert操作，所以如果找到了它的子类也订阅了该方法，则不允许父类和子类都同时订阅该事件，put 的是之前的那个 methodClassOld，就是将以前的那个 methodClassOld 存入 HashMap 去覆盖相同的订阅者。
        		//不允许出现一个订阅者有多个相同方法订阅同一个事件
                // Revert the put, old class is further down the class hierarchy
                subscriberClassByMethodKey.put(methodKey, methodClassOld);
                return false;
            }
        }
```

两层检查，找到订阅者方法，封装成`SubscriberMethod`类，添加到容器中。

`findUsingReflectionInSingleClass(findState)`找完之后，会接着判断订阅者有没有父类，如果有，它会向父类查找订阅方法。

```java
void moveToSuperclass() {
    if (skipSuperClasses) {
        clazz = null;
    } else {
        clazz = clazz.getSuperclass();
        String clazzName = clazz.getName();
        /** Skip system classes, this just degrades performance. */
        if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
            clazz = null;
        }
    }
}
```

查找订阅方法的流程就结束了，所查找到的订阅方法会放在一个集合容器中，并缓存起来，最后return出去。

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    	...
        ...
	if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the 	@Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
 }
```

再看注册方法：

```java
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            //迭代每个 Subscribe 方法，调用 subscribe() 传入 subscriber(订阅者) 和 subscriberMethod(订阅方法) 完成订阅，
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

通过`subscriberMethodFinder.findSubscriberMethods`方法找到订阅方法集合后，还需要对每个订阅方法进行一个`subscribe`方法操作：

```java
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    // 创建 Subscription 封装订阅者和订阅方法信息
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    //可并发读写的ArrayList，key为EventType，value为Subscriptions
    //根据事件类型从 subscriptionsByEventType 这个 Map 中获取 Subscription 集合
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    //如果为 null，表示还没有订阅过，创建并 put 进 Map
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        //若subscriptions中已经包含newSubscription，表示该newSubscription已经被订阅过，抛出异常
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }
    // 按照优先级插入subscriptions
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    //key为订阅者，value为eventType，用来存放订阅者中的事件类型
    //private final Map<Object, List<Class<?>>> typesBySubscriber;
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    //将EventType放入subscribedEvents的集合中
    subscribedEvents.add(eventType);
    //判断是否为Sticky事件
    if (subscriberMethod.sticky) {
        //判断是否设置了事件继承
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            //获取到所有Sticky事件的Set集合
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            //遍历所有Sticky事件
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                //判断当前事件类型是否为黏性事件或者其子类
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    // 执行设置了 sticky 模式的订阅方法
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

首先会根据`subscriber`和`subscriberMethod`来构建一个`Subscription`对象，接着在`subscriptionsByEventType `Map容器中找到有没有对应key的value，然后走找不到等于null的逻辑。对两个容器的添加Value值，`subscriptions`容器添加封装好的`subscription`，`subscribedEvents`容器添加`eventType`。还有一个就是对于黏性事件的处理，黏性事件对应于`postSticky`等sticky系列方法的。

![register](https://raw.githubusercontent.com/hejinalex/notes/master/%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/EventBus%20register.png)



## POST

```java
//currentPostingThreadState 线程独有的
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };

/** Posts the given event to the event bus. */
public void post(Object event) {
    //获取当前线程的 posting 状态
    PostingThreadState postingState = currentPostingThreadState.get();
    //获取当前事件队列
    List<Object> eventQueue = postingState.eventQueue;
    //将事件添加进当前线程的事件队列
    eventQueue.add(event);
    //判断是否正在posting
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        //如果已经取消，则抛出异常
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            //发送事件
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            //状态复原
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}

//发送事件的线程封装类
final static class PostingThreadState {
    final List<Object> eventQueue = new ArrayList<>();//事件队列
    boolean isPosting;//是否正在 posting
    boolean isMainThread;//是否为主线程
    Subscription subscription;
    Object event;
    boolean canceled;//是否已经取消
}

```

EventBus 用 `ThreadLocal `存储每个线程的 `PostingThreadState`，一个存储了事件发布状态的类，当 post 一个事件时，添加到事件队列末尾，等待前面的事件发布完毕后再拿出来发布，这里看事件发布的关键代码`postSingleEvent()`。

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        //查找到所有继承关系的事件类型，将该类的父类全部放入集合中
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            //判断是否找到订阅者订阅了该事件
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            //发送没有订阅者订阅该事件
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

首先看 `eventInheritance `这个属性，是否开启事件继承，若是，找出发布事件的所有父类，也就是 `lookupAllEventTypes()`，然后遍历每个事件类型进行发布。若不是，则直接发布该事件。

如果需要发布的事件没有找到任何匹配的订阅信息，则发布一个 `NoSubscriberEvent `事件。这里只看发布事件的关键代码 `postSingleEventForEventType()`：

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    //获取到 Subscription 的集合
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                //调用 postToSubscription 发送
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

来到这里，开始根据事件类型匹配出订阅信息，如果该事件有订阅信息，则执行 `postToSubscription()`，发布事件到每个订阅者，返回 true，若没有，则返回 false。继续追踪发布事件到具体订阅者的代码 `postToSubscription()`：

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        //订阅线程跟随发布线程，EventBus 默认的订阅方式
        case POSTING:
            // 订阅线程和发布线程相同，直接订阅
            invokeSubscriber(subscription, event);
            break;
        // 订阅线程为主线程
        case MAIN:
            //如果在 UI 线程，直接调用 invokeSubscriber
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
     			//如果不在 UI 线程，用 mainThreadPoster 进行调度，将订阅线程切换到主线程订阅
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        // 订阅线程为主线程
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        // 订阅线程为后台线程
        case BACKGROUND:
            //如果在 UI 线程，则将 subscription 添加到后台线程的线程池
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
            //不在UI线程，直接分发
                invokeSubscriber(subscription, event);
            }
            break;
        // 订阅线程为异步线程
        case ASYNC:
            // 使用线程池线程订阅
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

订阅者五种线程模式的特点对应的就是以上代码，简单来讲就是订阅者指定了在哪个线程订阅事件，无论发布者在哪个线程，它都会将事件发布到订阅者指定的线程。这里我们继续看实现订阅者的方法：

```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

`mainThreadPoster`：`mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;`

`mainThreadSupport.createPoster(this)` 返回的是`HandlerPoster(eventBus, looper, 10)`，looper 是一个 `MainThread MainThread `的 Looper。

`HandlerPoster `其实就是 `Handler `的实现，内部维护了一个 `PendingPostQueue `的消息队列，在 `enqueue(Subscription subscription, Object event)` 方法中不断从 `pendingPostPool `的 `ArrayList `缓存池中获取 `PendingPost `添加到 `PendingPostQueue `队列中，并将该 `PendingPost `事件发送到 `Handler `中处理。

在 `handleMessage `中，通过一个 while 死循环，不断从 `PendingPostQueue `中取出 `PendingPost `出来执行，获取到 post 之后由 eventBus 通过该 post 查找相应的 `Subscriber `处理事件。

**while 退出的条件有两个**

1. 获取到的 PendingPost 为 null，即是 PendingPostQueue 已经没有消息可处理。
2. 每个 PendingPost 在 Handler 中执行的时间超过了最大的执行时间。

```java
public class HandlerPoster extends Handler implements Poster {

    private final PendingPostQueue queue;//存放待执行的 Post Events 的事件队列
    private final int maxMillisInsideHandleMessage;//Post 事件在 handleMessage 中执行的最大的时间值，超过这个时间值则会抛异常
    private final EventBus eventBus;
    private boolean handlerActive;//标识 Handler 是否被运行起来

   protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        //从 pendingPostPool 的 ArrayList 缓存池中获取 PendingPost 添加到 PendingPostQueue 队列中，并将该 PendingPost 事件发送到 Handler 中处理。
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);//添加到队列中
            if (!handlerActive) {//标记 Handler 为活跃状态
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {//死循环，不断从 PendingPost 队列中取出 post 事件执行
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {//如果为 null，表示队列中没有 post 事件，此时标记 Handelr 关闭，并退出 while 循环
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false; //标记 Handler 为非活跃状态
                            return;
                        }
                    }
                }
                //获取到 post 之后由 eventBus 通过该 post 查找相应的 Subscriber 处理事件
                eventBus.invokeSubscriber(pendingPost);
                //计算每个事件在 handleMessage 中执行的时间
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}

final class PendingPost {
    //通过ArrayList来实现PendingPost的添加和删除
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();

    Object event;
    Subscription subscription;
    PendingPost next;

    private PendingPost(Object event, Subscription subscription) {
        this.event = event;
        this.subscription = subscription;
    }

    //获取 PendingPost
    static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            int size = pendingPostPool.size();
            if (size > 0) {
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
            }
        }
        return new PendingPost(event, subscription);
    }

    //释放 PendingPost
    static void releasePendingPost(PendingPost pendingPost) {
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        synchronized (pendingPostPool) {
            // Don't let the pool grow indefinitely
            if (pendingPostPool.size() < 10000) {
                pendingPostPool.add(pendingPost);
            }
        }
    }

}
```

`eventBus.invokeSubscriber(pendingPost)`：

最终执行的还是上面的`invokeSubscriber(subscription, event);`

```java
void invokeSubscriber(PendingPost pendingPost) {
    Object event = pendingPost.event;
    Subscription subscription = pendingPost.subscription;
    PendingPost.releasePendingPost(pendingPost);
    if (subscription.active) {
        invokeSubscriber(subscription, event);
    }
}
```

`backgroundPoster # BackGroundPoster`：

`BackgroundPoster `实现了 `Runnable `和 Poster，`enqueue()` 和 `HandlerPoster `中实现一样，在上文中已经讲过，这里不再赘述。

我们来看下 run() 方法中的实现，不断从 `PendingPostQueue `中取出 `pendingPost `到 EventBus 中分发，这里注意外部是 while() 死循环，意味着 `PendingPostQueue `中**所有**的 `pendingPost `都将分发出去。而 AsyncPoster 只是取出一个。

```java
final class BackgroundPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);//添加到队列中
            if (!executorRunning) {
                executorRunning = true;
                //在线程池中执行这个 pendingPost
                eventBus.getExecutorService().execute(this);//Executors.newCachedThreadPool()
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                //不断循环从 PendingPostQueue 取出 pendingPost 到 eventBus 执行
                while (true) {
                    //在 1000 毫秒内从 PendingPostQueue 中获取 pendingPost
                    PendingPost pendingPost = queue.poll(1000);
                    //双重校验锁判断 pendingPost 是否为 null
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();//再次尝试获取 pendingPost
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    //将 pendingPost 通过 EventBus 分发出去
                   //这里会将PendingPostQueue中【所有】的pendingPost都会分发，这里区别于AsyncPoster
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }
}
```

`asyncPoster # AsyncPoster`：

```java
class AsyncPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);//Executors.newCachedThreadPool()
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }
}
```

![register](https://raw.githubusercontent.com/hejinalex/notes/master/%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/EventBus%20post.png)



## 注销

```java
EventBus.getDefault().unregister(this);
```

跟踪 `unregister()`：

```java
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

注册过程我们就知道 `typesBySubscriber `是保存订阅者的所有订阅事件类型的一个 Map，这里根据订阅者拿到订阅事件类型 List，然后逐个取消订阅，最后 `typesBySubscriber `移除该订阅者,。这里只需要关注它是如果取消订阅的，跟踪 `unsubscribeByEventType()`。

```java
/** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

`subscriptionsByEventType `是存储事件类型对应订阅信息的 Map，代码逻辑非常清晰，找出某事件类型的订阅信息 List，遍历订阅信息，将要取消订阅的订阅者和订阅信息封装的订阅者比对，如果是同一个，则说明该订阅信息是将要失效的，于是将该订阅信息移除。

参考：

> [Android EventBus源码分析，基于最新3.1.1版本,看这一篇就够了！！](<https://blog.csdn.net/qq_34902522/article/details/85013185>)
>
> [EventBus 3.0+ 源码详解（史上最详细图文讲解）](<https://blog.csdn.net/WBST5/article/details/81089710>)
