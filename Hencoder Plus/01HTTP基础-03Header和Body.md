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

     

