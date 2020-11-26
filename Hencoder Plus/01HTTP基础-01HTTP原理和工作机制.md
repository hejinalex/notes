### HTTP原理和工作原理

#### HTTP 的定义

Hypertext Transfer Protocol，超⽂本传输协议，和 HTML (Hypertext Markup
Language 超⽂本标记语⾔) ⼀起诞⽣，⽤于在⽹络上请求和传输 HTML 内容。

超⽂本，即「扩展型⽂本」，指的是 HTML 中可以有链向别的⽂本的链接
（hyperlink）。

![](https://raw.githubusercontent.com/hejinalex/notes/master/Hencoder%20Plus/HTML.png)

#### HTTP 的工作方式

- 浏览器：

  ⽤户输⼊地址后回⻋或点击链接 -> 浏览器拼装 HTTP 报⽂并发送请求给服务器 -> 服
  务器处理请求后发送响应报⽂给浏览器 -> 浏览器解析响应报⽂并使⽤渲染引擎显示
  到界⾯

- ⼿机 App：

  ⽤户点击或界⾯⾃动触发联⽹需求 -> Android 代码调⽤拼装 HTTP 报⽂并发送请求
  到服务器 -> 服务器处理请求后发送响应报⽂给⼿机 -> Android 代码处理响应报⽂并
  作出相应处理（如储存数据、加⼯数据、显示数据到界⾯）

#### URL 和 HTTP 报文

##### URL 格式

三部分：协议类型、服务器地址(和端⼝号)、路径(Path)

协议类型://服务器地址[:端⼝号]路径

http://hencoder.com/users?gender=male

#### 报文格式

- 请求报⽂

  ![](https://raw.githubusercontent.com/hejinalex/notes/master/Hencoder%20Plus/Response.png)

- 响应报文

  ![](https://raw.githubusercontent.com/hejinalex/notes/master/Hencoder%20Plus/Request.png)