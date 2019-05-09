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
  * Creates a new EventBus instance; each instance is a separate scope in which events are 	         * delivered. To use a
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
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

`List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);`

这是通过`subscriberMethodFinder`对象，来找到加了`@Subscriber`注解，接收事件的方法集合的。

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

它会先从缓存中读是否有缓存，没有的话接着对`ignoreGeneratedIndex`进行判断，通过默认的方式获取Eventbus，注册订阅者的话，`ignoreGeneratedIndex`的值为false，所以这里调用`findUsingInfo`。

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

里先通过`prepareFindState()`初始化一个`FindState`，调用`initForSubscriber()`将订阅者的class对象与`findState.clazz`绑定，进入while循环。`getSubscriberInfo()`返回null，不成立，会进入`findUsingReflectionInSingleClass(findState)`方法查找`subscribeMethods`。

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
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
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
boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
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
                return checkAddWithMethodSignature(method, eventType);
            }
        }

        private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());

            String methodKey = methodKeyBuilder.toString();
            Class<?> methodClass = method.getDeclaringClass();
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
                // Only add if not already found in a sub class
                return true;
            } else {
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
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

通过`subscriberMethodFinder.findSubscriberMethods`方法找到订阅方法集合后，还需要对每个订阅方法进行一个`subscribe`方法操作：

```java
// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
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


