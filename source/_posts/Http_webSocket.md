---
title: HTTP和WebSocket协议的区别
date: 2020-04-11 15:30:00
categories: 网络
tags: [HTTP,WebSocket,TCP]
---

最近需要实现一个可以和客户端进行实时通讯的后端服务，由于后端可以主动发送消息给客户端，如果使用`HTTP`的话服务端是不能主动向客户端发送信息的，之前对`Socket`只是一个大概的了解。所以查了一些相关的资料，记录一下`HTTP`和`WebSocket`的区别。

<!--more-->

### HTTP
HTTP是一个Request/Response请求和响应的模式，两种类型都是由请求行、请求头、空行、请求主体四部分组成。这里将不详细介绍，可看我之前的文章。
只有客户端向服务端发送一个请求后，服务端才能向客户端回复，客户端在没有请求服务端时，服务端是不能主动向客户端发送数据的。

#### 短连接
举个栗子：
比如说客户端想实时地知道某个状态是否**发生**变化时，于是请求了服务端，但是服务端此时又**没有发生**变化。遇到这种情况下客户端只能不断地想服务端发送请求，直到返回满意的结果。这也就是我们所说的**轮询**。这是一个很浪费资源的，因为每个资源都需要建立一个新的连接，而HTTP底层使用的是`TCP`，每次都需要三次握手建立连接，所以会造成资源浪费。

于是由引出了一个新的概念，`HTTP`长连接，这里的长连接和`socket`的长连接是有区别的，这个稍后再说。

#### 长连接
所谓的长连接就是客户端向服务端发送请求，服务端不需要马上回复客户端，只有等到服务端有结果了才向客户端回复。相比上面的短连接可以减少连接次数和压力。

<center>![avatar](/static/blogImg/HTTP_connect.png)WebSocket.png</center>

在一个TCP连接上也可以传输多个Request/Response消息对，但是HTTP的基本模型还是一个Request对应一个Response。


### WebSocket
WebSocket 协议在2008年诞生，2011年成为国际标准。所有浏览器都已经支持了。

它的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于服务器推送技术的一种。
<center>![avatar](/static/blogImg/WebSocket.png)</center>

其他特点包括：
1. 建立在 TCP 协议之上，服务器端的实现比较容易。
2. 与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此不容易屏蔽，能通过各种 HTTP 代理服务器。
3. 数据格式比较轻量，性能开销小，通信高效。
4. 可以发送文本，也可以发送二进制数据。
4. 没有同源限制，客户端可以与任意服务器通信。
5. 协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。

```
ws://example.com:80/some/path
```

<center>![avatar](/static/blogImg/tcp_http_ws.png)</center>

#### WebSocket协议的规范
以下是一个典型的WebSocket发起请求到响应请求的示例：
```
客户端到服务端：
GET / HTTP/1.1
Connection:Upgrade
Host:127.0.0.1:8088
Origin:null
Sec-WebSocket-Extensions:x-webkit-deflate-frame
Sec-WebSocket-Key:puVOuWb7rel6z2AVZBKnfw==
Sec-WebSocket-Version:13
Upgrade:websocket

服务端到客户端：
HTTP/1.1 101 Switching Protocols
Connection:Upgrade
Server:beetle websocket server
Upgrade:WebSocket
date: Thu, 10 May 2018 07:32:25 GMT
Access-Control-Allow-Credentials:true
Access-Control-Allow-Headers:content-type
Sec-WebSocket-Accept:FCKgUr8c7OsDsLFeJTWrJw6WO8Q=
```

我们可以看到，WebSocket协议和HTTP协议乍看并没有太大的区别，但细看下来，区别还是有些的，这其实是一个握手的http请求，首先请求和响应的，”Upgrade:WebSocket”表示请求的目的就是要将客户端和服务器端的通讯协议从 HTTP 协议升级到 WebSocket协议。从客户端到服务器端请求的信息里包含有”Sec-WebSocket-Extensions”、“Sec-WebSocket-Key”这样的头信息。这是客户端浏览器需要向服务器端提供的握手信息，服务器端解析这些头信息，并在握手的过程中依据这些信息生成一个28位的安全密钥并返回给客户端，以表明服务器端获取了客户端的请求，同意创建 WebSocket 连接。

当握手成功后，这个时候TCP连接就已经建立了，客户端与服务端就能够直接通过WebSocket直接进行数据传递。不过服务端还需要判断一次数据请求是什么时候开始的和什么时候是请求的结束的。在WebSocket中，由于浏览端和服务端已经打好招呼，如我发送的内容为utf-8编码，如果我发送0x00,表示包的开始，如果发送了0xFF，就表示包的结束了。这就解决了黏包的问题。

#### 与HTTP的相同点
1. 都是基于TCP的应用层协议。
2. 都使用Request/Response模型进行连接的建立。
3. 在连接的建立过程中对错误的处理方式相同，在这个阶段WS可能返回和HTTP相同的返回码。
4. 都可以在网络中传输数据。

#### 与HTTP的不同点
1. WS使用HTTP来建立连接，但是定义了一系列新的header域，这些域在HTTP中并不会使用。
2. WS的连接不能通过中间人来转发，它必须是一个直接连接。
3. WS连接建立之后，通信双方都可以在任何时刻向另一方发送数据。
4. WS连接建立之后，数据的传输使用帧来传递，不再需要Request消息。
5. WS的数据帧有序。