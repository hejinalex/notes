### HTTP的Header和Body

#### Header ⾸部

作⽤：HTTP 消息的 metadata(元数据)

- Host

  目标主机。注意：不是在⽹络上⽤于寻址的，⽽是在⽬标服务器上⽤于定位⼦服务
  器的。

- Content-Type

  指定 Body 的类型。主要有四类：

  1. text/html

     请求 Web ⻚⾯是返回响应的类型，Body 中返回 html ⽂本。格式如下：

     ```
     HTTP/1.1 200 OK
     Content-Type: text/html; charset=utf-8
     Content-Length: 853
     
     <!DOCTYPE html>
     <html>
     <head>
     <meta charset="utf-8">
     ......
     ```

  2. x-www-form-urlencoded

     Web ⻚⾯纯⽂本表单的提交⽅式。

     ![](https://raw.githubusercontent.com/hejinalex/notes/master/Hencoder%20Plus/x-www-form-urlencoded-1.png)

     ![](https://raw.githubusercontent.com/hejinalex/notes/master/Hencoder%20Plus/x-www-form-urlencoded-2.png)

     格式如下：

     ```
     POST /users HTTP/1.1
     Host: api.github.com
     Content-Type: application/x-www-form-urlencoded
     Content-Length: 27
     
     name=rengwuxian&gender=male
     ```

     对应 Retrofit 的代码：

     ```java
     @FormUrlEncoded
     @POST("/users")
     Call<User> addUser(@Field("name") String name,
     @Field("gender") String gender);
     ```

  3. multipart/form-data

     Web ⻚⾯含有⼆进制⽂件时的提交⽅式。

     ![](https://raw.githubusercontent.com/hejinalex/notes/master/Hencoder%20Plus/multipart-form-data-1.png)

     ![](https://raw.githubusercontent.com/hejinalex/notes/master/Hencoder%20Plus/multipart-form-data-2.png)

     格式如下：

     boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

     boundary作为分界

     ```
     POST /users HTTP/1.1
     Host: hencoder.com
     Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
     Content-Length: 2382
     
     ------WebKitFormBoundary7MA4YWxkTrZu0gW
     Content-Disposition: form-data; name="name"
     
     rengwuxian
     ------WebKitFormBoundary7MA4YWxkTrZu0gW
     Content-Disposition: form-data; name="avatar";
     filename="avatar.jpg"
     Content-Type: image/jpeg
     
     JFIFHHvOwX9jximQrWa......
     ------WebKitFormBoundary7MA4YWxkTrZu0gW--
     ```

     对应 Retrofit 的代码：

     ```java
     @Multipart
     @POST("/users")
     Call<User> addUser(@Part("name") RequestBody name,
     @Part("avatar") RequestBody avatar);
     
     ...
         
     RequestBody namePart = RequestBody.create(MediaType.parse("text/plain"), nameStr);
     RequestBody avatarPart = RequestBody.create(MediaType.parse("image/jpeg"), avatarFile);
     api.addUser(namePart, avatarPart);
     ```

  4. application/json , image/jpeg , application/zip ...

     单项内容（⽂本或⾮⽂本都可以），⽤于 Web Api 的响应或者 POST / PUT 的请求

     请求中提交 JSON：

     ```
     POST /users HTTP/1.1
     Host: hencoder.com
     Content-Type: application/json; charset=utf-8
     Content-Length: 38
     
     {"name":"rengwuxian","gender":"male"}
     ```

     对应 Retrofit 的代码：

     ```java
     @POST("/users")
     Call<User> addUser(@Body("user") User user);
     
     ...
     
     // 需要使⽤ JSON 相关的 Converter
     api.addUser(user);
     ```

     响应中返回 JSON：

     ```
     HTTP/1.1 200 OK
     content-type: application/json; charset=utf-8
     content-length: 234
     
     [{"login":"mojombo","id":1,"node_id":"MDQ6VXNl
     cjE=","avatar_url":"https://avatars0.githubuse
     rcontent.com/u/1?v=4","gravat......
     ```

     请求中提交⼆进制内容：

     ```
     POST /user/1/avatar HTTP/1.1
     Host: hencoder.com
     Content-Type: image/jpeg
     Content-Length: 1575
     
     JFIFHH9......
     ```

     对应 Retrofit 的代码：

     ```java
     @POST("users/{id}/avatar")
     Call<User> updateAvatar(@Path("id") String id, @Body
     RequestBody avatar);
     
     ...
         
     RequestBody avatarBody =
     RequestBody.create(MediaType.parse("image/jpeg"),
     avatarFile);
     api.updateAvatar(id, avatarBody)
     ```

     相应中返回⼆进制内容：

     ```
     HTTP/1.1 200 OK
     content-type: image/jpeg
     content-length: 1575
     
     JFIFHH9......
     ```

- Content-Length

  指定 Body 的⻓度（字节）。

- Transfer: chunked (分块传输编码 Chunked Transfer Encoding)

  ⽤于当响应发起时，内容⻓度还没能确定的情况下。和 Content-Length 不同时使
  ⽤。⽤途是尽早给出响应，减少⽤户等待。

  格式：

  ```
  HTTP/1.1 200 OK
  Content-Type: text/html
  Transfer-Encoding: chunked
  
  4
  Chun
  9
  ked Trans
  12
  fer Encoding
  0
  
  ```
  
最后传输0表示内容结束
  
- Location

  指定重定向的⽬标 URL

- User-Agent

  ⽤户代理，即是谁实际发送请求、接受响应的，例如⼿机浏览器、某款⼿机 App。

- Range / Accept-Range

  按范围取数据

  Accept-Range: bytes 响应报⽂中出现，表示服务器⽀持按字节来取范围数据

  Range: bytes=<start>-<end> 请求报⽂中出现，表示要取哪段数据

  Content-Range:<start>-<end>/total 响应报⽂中出现，表示发送的是哪段数据

  作⽤：断点续传、多线程下载。

- Cache

  作⽤：在客户端或中间⽹络节点缓存数据，降低从服务器取数据的频率，以提⾼⽹络性能。

- 其他 Headers

  Accept: 客户端能接受的数据类型。如 text/html

  Accept-Charset: 客户端接受的字符集。如 utf-8

  Accept-Encoding: 客户端接受的压缩编码类型。如 gzip

  Content-Encoding：压缩类型。如 gzip

