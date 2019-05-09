# EventBus的使用

## 定义事件
普通的POJO即可
```java
public class MessageEvent {
 
    public final String message;
 
    public MessageEvent(String message) {
        this.message = message;
    }
}
```
## 准备订阅者
  订阅者方法将在事件发送时调用，这些方法使用 **@Subscribe**注解进行修饰

  * 该方法有且只有一个参数。
  * 该方法必须是public修饰符修饰，不能用static关键字修饰，不能是抽象的（abstract）
  * 该方法需要用@Subscribe注解进行修饰。

```java
// This method will be called when a MessageEvent is posted (in the UI thread for Toast)
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessageEvent(MessageEvent event) {
    Toast.makeText(getActivity(), event.message, Toast.LENGTH_SHORT).show();
}
 
// This method will be called when a SomeOtherEvent is posted
@Subscribe
public void handleSomethingElse(SomeOtherEvent event) {
    doSomethingWith(event);
}
```
准备好订阅者方法后还需要向总线进行订阅和注销，这样订阅者方法才能接收到相应的事件。
对于activity和fragment，应当在生命周期中进行订阅和注销，对于大多数情况，在onStart/onStop中订阅注销是没有问题的。
```java
@Override
public void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}
 
@Override
public void onStop() {
    EventBus.getDefault().unregister(this);
    super.onStop();
}
```
## 发送事件
在代码的任意位置发送一个事件，所有与这个事件匹配的订阅者方法都将收到这个事件
```java
EventBus.getDefault().post(new MessageEvent("Hello everyone!"));
```

# @Subscribe 注解介绍

## ThreadMode
```java
public enum ThreadMode {
    POSTING,//EventBus 默认的线程模式
    MAIN,//主线程
    MAIN_ORDERED,//主线程
    BACKGROUND,//后台线程
    ASYNC//异步线程
}
```
ThreadMode 是 enum（枚举）类型，threadMode 默认值是 POSTING。
* POSTING：事件发送在什么线程，就在什么线程订阅。故它不需要切换线程来分发事件，因此开销最小。
* MAIN：如在主线程发送事件，则直接在主线程处理事件；如果在子线程发送事件，则先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
* MAIN_ORDERED：无论在那个线程发送事件，都先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
* BACKGROUND：如果发送事件的线程不是主线程，则在该线程处理，如果是主线程，则使用一个单独的后台线程处理。
* ASYNC：无论在那个线程发送事件，都将事件入队列，然后通过线程池处理。

## sticky 粘性事件
粘性事件是订阅者方法在事件发送之后才注册，依然能接收到该事件的特殊类型。

首先，发送一个粘性事件：
```java
EventBus.getDefault().postSticky(new MessageEvent("Hello everyone!"));
```
然后，打开一个activity，在注册期间，所有粘性订阅者方法将立即获得先前发布的粘性事件：
```java
@Override
public void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}

// UI updates must run on MainThread
@Subscribe(sticky = true, threadMode = ThreadMode.MAIN)
public void onEvent(MessageEvent event) {   
    textField.setText(event.message);
}

@Override
public void onStop() {
    EventBus.getDefault().unregister(this);    
    super.onStop();
}
```
最后一个粘性事件在注册时会自动传递给匹配的订阅者。

有时可能需要删除（使用）粘性事件，以便它们不再被传递。
```java
MessageEvent stickyEvent = EventBus.getDefault().getStickyEvent(MessageEvent.class);
// Better check that an event was actually posted before
if(stickyEvent != null) {
    // "Consume" the sticky event
    EventBus.getDefault().removeStickyEvent(stickyEvent);
    // Now do something with it
}
```

`removeStickyEvent`是一个被重载的方法，传入事件类时，它将返回先前持有的粘性事件。使用这个重载方法，我们可以改进前面的示例：
```java
MessageEvent stickyEvent = EventBus.getDefault().removeStickyEvent(MessageEvent.class);
// Better check that an event was actually posted before
if(stickyEvent != null) {
    // Now do something with it
}
```

## priority 

### 订阅优先级
可以通过在注册期间为订阅者方法提供优先级来更改事件传递的顺序
```java
@Subscribe(priority = 1);
public void onEvent(MessageEvent event) {
    ...
}
```
在同一传递线程（ThreadMode）中，较高优先级的订阅者方法将在优先级较低的其他订阅者方法之前接收事件。默认优先级为0。

！注意：优先级不会影响具有不同ThreadMode的订阅者的传递顺序！

### 取消传递
可以通过在订阅者方法的事件处理方法中调用`cancelEventDelivery（Object事件）`来取消事件传递过程。任何一步的事件被取消，后续订阅者方法都将不会收到该事件。
```java
// Called in the same thread (default)
@Subscribe
public void onEvent(MessageEvent event){
    // Process the event
    ...
    // Prevent delivery to other subscribers
    EventBus.getDefault().cancelEventDelivery(event) ;
}
```



