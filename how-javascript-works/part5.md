# WebSockets 和 HTTP/2 + SSE

## 长轮训

```js
function poll() {
    setTimeout(function() {
        $.ajax({
            url: 'https://api.example.com/endpoint',
            success: function(data) {
                // 使用 data 做一些事儿
                
                // 开始下一次轮询
                poll();
            }
        })
    }, 10000)
}
poll();
```

## WebSocket

1. 客户端创建 WebSocket 连接：

   ```js
   var socket = new WebSocket('ws://websocket.example.com')
   ```

   这一步，初始化了请求头：

   ```http
   GET ws://websocket.example.com/ HTTP/1.1
   Origin: http://example.com
   Connection: Upgrade
   Host: websocket.example.com
   Upgrade: websocket
   ```

   HTTP header 中的 Upgrade 将 HTTP 协议提升为了 WebSocket 协议

2. 在服务端，假定我们使用 Node.js 创建 WebSockteServer：

   ```js
   var WebSocketServer = require('websocket').server;
   var http = require('http');

   var server = http.createServer(function(request, response){
       // 处理 HTTP 请求
   });

   server.listen(1337, function() {});

   // 创建 WebSocket server
   wsServer = new WebSocketServer({
       httpServer: server
   });

   wsServer.on('request', function(request){
       var connection = request.accept(null, request.origin);
       
       connection.on('message', function(message) {
         // 处理消息  
       })
       
       connection.on('close', function(message){
           // 连接关闭时的操作
       })
   })

   ```

   在连接建立以后，服务器通过响应头的 `Upgrade` 进行回复：

   ```http
   HTTP/1.1 101 Switching Protocols
   Date: Wed, 25 Oct 2017 10:07:34 GMT
   Connection: Upgrade
   Upgrade: WebSocket
   ```

3. 一旦连接建立，客户端下 WebSocket 实例的 `open` 事件将会被触发：

   ```js
   var socket = new WebSocket('ws://websocket.example.com')

   socket.onopen = function(event) {
       console.log('WebSocket is connected.')
   }
   ```

握手完成时，HTTP 协议被提升为了 WebSocket 协议，客户端和服务器任一方都可以发送数据了。

在客户端发送数据：

```js
socket.onopen = function(event) {
    socket.send('some message')
}
```

在服务端发送数据：

```js
socket.onmessage = function(event) {
    var message = event.data;
    console.log(message);
}
```

## WebSocket 帧

WebSocket 是基于帧的，亦即每段数据含有多个帧，帧的构成如下：

```

0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

- `fin`: 1 bits，指出了当前帧是消息的最后一帧。由于绝大多数消息都不足一帧，因此该位通常会被设置。

- `rsv1`、`rsv2`、`rsv3`：各自都是 1 bits，如果没有扩展协议去定义它们取非零值时的含义，那么它们就该被设置为 0。如果没有扩展协议定义非零值含义，则它们取非零值时，消息接收端会认为连接失败。

- `opcode`：4 bits，描述了帧的含义：

  - `0x00`：当前帧继续传输上一帧的内容。
  - `0x01`：当前帧为文本数据。
  - `0x02`：当前帧为二进制数据。
  - `0x08`：当前帧终止了连接。
  - `0x09`：当前帧为 ping。
  - `0x0a`：当前帧为 pong。

- `mask`：1 bits，指示了连接是否被掩码。一般而言，每一条从客户端到服务器的消息都需要被掩码，否则连接就会被终止。

- `payload_len`：7 bits，payload 长度。WebSocket 的帧长度区间为：

  如果是 0–125，则直接指示了 payload 长度。如果是 126，则意味着接下来 2 个字节（16 bits）将指明长度，如果是 127，则意味着接下来 8 个字节（64 bits）将指明长度。所以，一个 payload 的长度将可能是 7 bits、16 bits 或者 64 bits 以内。

- `masking-key`：32 bits，对帧做掩码处理。

- `payload`：消息内容，由 `payload_len` 指明了长度。

## 分片

消息可以分为多个帧进行发送。接收端会缓存这些帧，知道收到了 `fin` 位被设置的帧。例如，我们可以用 11 个帧传输 “Hello World” 字符串，每个帧的大小为 6（头部）+1 字节。

合并各个帧的逻辑如下：

* 收到第一帧
* 记住第一帧 `opcode`，即消息格式
* 连接各个帧的 payload，知道遇到 `fin` 位被设置的帧
* 后续帧的 `opcode` 为 0，即各个帧都是继续传输上一帧

## 心跳

握手完成之后的任意时刻，客户端或者服务器都能够发送一个 ping 到对面。当 ping 被接收以后，接收方必须尽快回送一个 pong。这就是一次心跳，你可以通过这个机制来确保客户端仍处于连接状态。

ping 和 pong 都是**控制帧**。ping 的 opcode 为 `0x9`，pong 则为 `0xA`。当你收到了一个 ping，你回送的 pong 需要和 ping 具有一样的 payload data（ping 和 pong 允许的最大 payload 长度为 **125**）。如果你收到了没有和一个 ping 结对的 pong ，那么直接忽略即可。

## 错误处理与连接关闭

在客户端，我们可以这样进行错误处理：

```js
var socket = new WebSocket('ws://websocket.example.com');

// 处理任何发生的错误。
socket.onerror = function(error) {
  console.log('WebSocket Error: ' + error);
};
```

为了关闭连接，客户端或服务端都可以发送一个 opcode 为 `0x8` 的控制帧来关闭连接。一旦收到这样一帧，另一端也需要发送一个关闭帧作为回应。接着发送端便会关闭连接。关闭连接后收到的任何数据都会被丢弃。

下面代码展示了如何在客户端中关闭连接：

```js
// 如果连接是打开的，则关闭
if (socket.readyState === WebSocket.OPEN) {
    socket.close();
}
```

通过监听 `close` 事件，你可以在在连接关闭后进行一些“善后”工作：

```js
// 做一些必要的清理
socket.onclose = function(event) {
  console.log('Disconnected from WebSocket.');
};
```

服务器也必须监听 `close` 事件，做一些它需要的处理工作：

```js
connection.on('close', function(reasonCode, description) {
    // 连接关闭了
});
```

## HTTP/2 与 WebSocket 对比：

|                              | HTTP/2                      | WebSocket        |
| ---------------------------- | --------------------------- | ---------------- |
| 头部（Headers）              | 压缩（HPACK）               | 不压缩           |
| 二进制数据（Binary）         | Yes                         | 二进制或文本数据 |
| 多路复用（Multiplexing）     | Yes                         | Yes              |
| 优先级技术（Prioritization） | Yes                         | Yes              |
| 压缩（Compression）          | Yes                         | Yes              |
| 方向（Direction）            | Client/Server + Server Push | 双向的           |
| 全双工（Full-deplex）        | Yes                         | Yes              |

## SSE（Server-Sent Events）

HTTP/2 引入了 [Server Push](https://en.wikipedia.org/wiki/Push_technology?oldformat=true) 来允许服务器主动地发送资源到客户端缓存中。但是，并不允许直接发送数据到客户端应用程序中。服务器推送的内容只能被浏览器处理，而不是客户端应用程序代码，这意味着应用中没有 API 能够感知到推送。

当客户端和服务器的连接建立后，SSE 能够让服务器异步地推送数据到客户端。之后，服务器随时都可以在准备好后发送数据。这可以被看作是**单向的**[发布-订阅](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)模型。SSE 还提供了一个叫做 [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) 的标准 JavaScript 客户端 API。

```js
// 其中，sse.php 负责产生 server sent events
var evtSource = new EventSource('sse.php');
var eventList = document.querySelector('ul');

evtSource.onmessage = function(e) {
  var newElement = document.createElement("li");

  newElement.textContent = "message: " + e.data;
  eventList.appendChild(newElement);
}
```

sse.php 中的内容可能为：

```php
date_default_timezone_set("America/New_York");
header('Cache-Control: no-cache');
header("Content-Type: text/event-stream\n\n");

$counter = rand(1, 10);
while (1) {
  // 每秒发送一个 ping event
  
  echo "event: ping\n";
  $curDate = date(DATE_ISO8601);
  echo 'data: {"time": "' . $curDate . '"}';
  echo "\n\n";
  
  // Send a simple message at random intervals.
  
  $counter--;
  
  if (!$counter) {
    echo 'data: This is a message at time ' . $curDate . "\n\n";
    $counter = rand(1, 10);
  }
  
  ob_end_flush();
  flush();
  sleep(1);
}
```



由于 SSE 是基于 HTTP 的，所以它天然亲和 HTTP/2，因此可以组合二者，以吸取各自精华：HTTP/2 通过多路复用流来提高传输层的效率，SSE 则为客户端应用程序提供了接收推送的 API。

## 抉择

WebSocket 适用场景：

- 低延迟
- 消息吞吐量大，如多人在线网游

HTTP/2 + SSE 适用场景：

- 仍舍不得脱离 HTTP 世界，Web 组件（防火墙、入侵检测、负载均衡）是基于 HTTP 来构建、维护和配置的，考虑到弹性伸缩、安全性和可扩展。有扩展性和安全性的应用可以选择 HTTP/2 + SSE