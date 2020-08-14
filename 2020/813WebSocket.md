# WebSocket

## 为什么诞生WebSocket?

因为HTTP是无状态协议，只能有客户端向服务端发起请求，所以说HTTP是单向协议。想象这样一种场景，当服务端频繁发生变化时，客户端要获取信息就非常困难。传统的做法是利用ajax轮询，每隔一段时间向服务器发起请求，以获取状态变更。为了解决这个问题，WebSocket诞生了。

## 简介

WebSocket是一款基于TCP的协议，诞生于2008年，2011年成为国际标准，目前所有浏览器都已经支持了。

WebSocket是建立在TCP连接上的、能够实现全双工通信的协议。客户端可以发起请求到服务器，服务器也可以主动推送内容给客户端，属于服务器推送的一种。WebSocket的典型应用场景是聊天室。

WebSocket具有如下特点：

1. 建立在 TCP 协议之上，服务器端的实现比较容易。

2. 与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP 代理服务器。

3. 数据格式比较轻量，性能开销小，通信高效。

4. 可以发送文本，也可以发送二进制数据。

5. 没有同源限制，客户端可以与任意服务器通信。

6. 协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。

## API

### WebSocket()构造函数

WebSocket()构造函器会返回一个 WebSocket 对象。

语法：

> var aWebSocket = new WebSocket(url [, protocols]);

参数：

- url

要连接的URL；这应该是WebSocket服务器将响应的URL。

- protocols（可选）

一个协议字符串或者一个包含协议字符串的数组。这些字符串用于指定子协议，这样单个服务器可以实现多个WebSocket子协议（例如，您可能希望一台服务器能够根据指定的协议（protocol）处理不同类型的交互）。如果不指定协议字符串，则假定为空字符串。

### 