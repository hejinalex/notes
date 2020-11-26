### HTTP的请求方法和状态码

#### GET

- ⽤于获取资源
- 对服务器数据不进⾏修改
- 不发送 Body

```
GET /users/1 HTTP/1.1
Host: api.github.com
```

对应 Retrofit 的代码：

```java
@GET("/users/{id}")
Call<User> getUser(@Path("id") String id, @Query("gender") String gender);
```

#### POST

- ⽤于增加或修改资源
- 发送给服务器的内容写在 Body ⾥⾯

``` 
POST /users HTTP/1.1
Host: api.github.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 13

name=rengwuxian&gender=male
```

对应 Retrofit 的代码：

```java
@FormUrlEncoded
@POST("/users")
Call<User> addUser(@Field("name") String name, @Field("gender") String gender);
```

#### PUT

- ⽤于修改资源
- 发送给服务器的内容写在 Body ⾥⾯

```
PUT /users/1 HTTP/1.1
Host: api.github.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 13

gender=female
```

对应 Retrofit 的代码：

```java
@FormUrlEncoded
@PUT("/users/{id}")
Call<User> updateGender(@Path("id") String id, @Field("gender") String gender);
```

#### DELETE

- ⽤于删除资源
- 不发送 Body

```
DELETE /users/1 HTTP/1.1
Host: api.github.com
```

对应 Retrofit 的代码：

```java
@DELETE("/users/{id}")
Call<User> getUser(@Path("id") String id, @Query("gender") String gender);
```

#### HEAD

- 和 GET 使⽤⽅法完全相同
- 和 GET 唯⼀区别在于，返回的响应中没有 Body



#### Status Code 状态码

三位数字，⽤于对响应结果做出类型化描述（如「获取成功」「内容未找到」）。

- 1xx：临时性消息。如：100 （继续发送）、101（正在切换协议）
- 2xx：成功。最典型的是 200（OK）、201（创建成功）
- 3xx：重定向。如 301（永久移动）、302（暂时移动）、304（内容未改变）
- 4xx：客户端错误。如 400（客户端请求错误）、401（认证失败）、403（被禁
  ⽌）、404（找不到内容）
- 5xx：服务器错误。如 500（服务器内部错误）