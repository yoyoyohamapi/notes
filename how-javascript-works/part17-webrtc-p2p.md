# WebRTC 以及 P2P 网络

**RTC**：Real Time Communication。早先，桌面应用使用了 P2P 技术来做一些 web 做不到的事儿，例如即时聊天应用。但 WebRTC 改变了这个局面。

WebRTC 允许 web 应用创建**点对点（Peer-To-Peer）** 通信。

## P2P 通信

为了通过 web 浏览器和其他 peer 进行通信，浏览器需要进行下面几步：

- 允许通信开始
- 知晓如何定位到对方 peer 的位置
- 通过安全和防火墙保护
- 进行实时的多媒体通信

基于浏览器的 P2P 通信最大的革新就在于知道如何定位并且和另外一个浏览器建立 socket 连接，以此实现双向数据传输。

当应用想要传输数据或资源时，它会从具有这些资源的 server 上获取。然而，如果你想要创建一个 P2P 的视频聊天，就需要直连到对方的浏览器，然而你却不知道对方浏览器的地址。

## 防火墙以及 NAT 穿透

我们的电脑一般来说不会被分配到一个静态 IP，原因是处于防火墙及 NAT（network access translation）设备之下。

NAT 设备是能够将防火墙内的私有 IP 转换为公网 IP。例如，我们的计算机连入了企业 WiFi，就会被分配一个只存在于企业 NAT 下的 IP。假定这个 IP 是 172.0.23.4，这是一个私有 I，其对应的公网 IP 是 164.53.27.98。因此，对于外部世界来说，看到的请求将是发送自 164.53.27.98，NAT 通过映射表，保证了响应到来时，内容能被送到 172.0.23.4。

这其中，涉及到的技术还有 **S**ession **T**raversal **U**tilities for **N**AT (STUN) 和 **T**raversal **U**sing **R**elays around **N**AT (TURN) 。为了让 WebRTC 能够工作，首先你要到 STUN server 查询自己的公网 IP（Who am I ?）。

假定这个过程成功，你将收到你的公网 IP 和 端口，这样你就能告诉其他 peer 自己的位置，从而进行 socket 连接。

## Signaling（信令）、Session 和 Protocol

信令涉及网络发现，NAT 穿透，session 的创建和管理，通信安全，媒体能力的元信息和调制，以及错误处理。

为了让连接能够工作，节点将本地媒体状况（例如解析，编码能力）俘获到元信息，并为应用 host 取得可能的网络地址。负责来回传递这些关键信息的信令机制没有被收纳到 WebRTC API 中。

信令机制不在 WebRTC 标准的声明中，并且也没有对应的 API 实现，这是为了保证技术和协议的灵活性。

当你的基于 WebRTC 的应用使用 STUN 知道了自己的公网 IP 后，接下来就是和别的 peer 协商并且建立 session 连接了。

最初的 session 协商和建立使用的在多媒体通信中的 信令/通信 协议。该协议也承担了 session 的管理及终止规则。

这个协议之一就是 Session Initiation Protocal（SIP 会话初始化协议）。由于 WebRTC 信令机制的灵活性，因此，SIP 不是唯一可供使用的信令协议。我们所选择的信令协议必须能够和应用层的 Session Description Protocal（SDP）协议工作。所有的多媒体功能元信息都通过 SDP 进行传输。

任何一个试图与其他 peer 通信的节点都会生成一系列的 **I**nteractive **C**onnectivity **E**stablishment protocol (ICE) 候选者。每个候选者都是 IP，端口和传输协议的组合。

![](https://cdn-images-1.medium.com/max/1600/0*SXRTlnVxy2-hE9ZX)

## 连接建立

每一个节点建立了公网 IP 后，信令数据 “channel” 就会动态创建，用以发现节点，并且支撑 P2P 的协商以及 session 建立。

这些 channels 对外界是不可知的，访问它们时，需要指定唯一的标志符。

由于 WebRTC 的灵活性，信令流程没有出现在标准中，因此 channel 机制也不是所有实现所必须的。

一旦两个或多个节点连接到了同一个 channel，彼此就能沟通 session 信息。初始节点会使用信令协议（例如 SIP 或者 SDP）发送一个 “offer” 命令，然后，他就会等待着 channel 上任意接收者回送的 “answer” 。如果收到了节点的回复，就会从各个节点聚合成的 **I**nteractive **C**onnectivity **E**stablishment protocol (ICE) 候选者选出最佳的，此时，节点间通信用的元信息、网络路由（IP 和 端口）以及媒体信息就商定了，节点间的 socket session 也就完全建立了。各个节点将创建本地数据流和数据通道断点（endpoint），最终，可以使用任何的双向通信技术进行多媒体数据的传输。

倘若由于防火墙或者是 NAT 技术，导致选择最佳候选者失败，将回退到使用 TURN server 作为替代。TURN 将扮演一个中介者，由它来转发节点间的传输的数据，因此，这不算一个完全意义上的  P2P 通信。

如果使用了 TURN，那么各个节点不再需要知道如何和对方进行数据传输，它们只需要知道哪个公共的 TURN server 需要去通信。因此，TURN server 需要足够的健壮性，并且拥有大的带宽和海量数据的处理能力。

## WebRTC API

目前具有的 WebRTC 分类大致有下面三个类型：

- **Media Capture and Streams**：允许你访问输入设备，例如麦克风和网络摄像头。这个 API 允许你获得其他节点的媒体流。
- **RTCPeerConnection**：允许你实时发送截获的音视频流到另外的 WebRTC 端点。允许你创建一个本地机器到远程节点的连接。其提供了连接到远程节点的方法，维护并且监控了连接，并再连接不再需要时关闭连接。
- **RTCDataChannel**：允许你传输任意的数据。每一个数据 channel 都关联到一个 RTCPeerConnection。

## Media Capture and Streams

[MediaDevices](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices) 的 `getUserMedia()` 方法请求用户授权使用媒体输入来导出一个 [MediaStream](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream) ，这个流包含了对应类型的轨道。例如，摄像头就将导出一个含有视频轨道信息的 media stream。

这个 API 返回一个 Promise 对象，如果用户没有授权，或者媒体设备不可用，promise 就会被一个 PermissionDeniedError 或者 NotFoundError reject。

```js
navigator.mediaDevices.getUserMedia(constraints)
.then(stream => {
	/* 使用这个 stream */
})
.then(err => {
	/* 错误处理 */
})
```

`constraints` 指明了你想要返回哪种类型的 stream。

`getUserMedia` 也能够被用作 Web Audio API 的输入节点：

```js
function gotStream(stream) {
  window.AudioContext = window.AudioContext || window.webkitAudioContext
  const audioContext = new AudioContext()
  // 从 stream 中创建一个 AudioNode
  const mediaStreamSource = audioContext.createMediaStreamSource(stream)
  // 连接节点到目的节点
  mediaStreamSource.connect(audioContext.destination)
}

navigator.getUserMedia({audio: true}, gotStream)
```

### 隐私限制

由于 API 能够获得本地设备，因此浏览器也被要求在问询用户权限时，展示一个明显的 indicator。

## RTCPeerConnection

**RTCPeerConnection** 接口描述了一个本地主机到远程节点的连接。其提供了连接到远程节点的方法，维护并且监控了连接，并再连接不再需要时关闭连接。

![](https://cdn-images-1.medium.com/max/1600/0*Nm9r_NLcAhJernmo)

从图中可以看到，`RTCPeerConnection` 给到了 Web 开发者一个抽象，屏蔽了底层的复杂性。WebRTC 所用的编解码技术和协议做了大量的工作来让实时通信成为可能，甚至在网络条件不稳定是亦然：

- 包补偿（Packet loss concealment）
- 回波消除（Echo cancellation）
- 自适应带宽（Bandwidth adaptivity）
- 视频抖动缓冲器（Dynamic jitter buffering）
- 自动增益控制（Automatic gain control）
- 降噪和噪声抑制（Noise reduction and suppression）
- 图像净化（Image “cleaning”）

## RTCDataChannel

除了音视频，WebRTC 也支持其他数据类型的实时通信。`RTCDataChannel` API 使得任意数据点对点的通信成为可能，这个 API 的使用场景有：

- 游戏
- 即时通信
- 文件传输
- 去中心化网络

API 所具有的特性充分利用到了 `RTCPeerConnection`  ，为 P2P 通信提供了强大而灵活的能力：

- 利用了 `RTCPeerConnection` 的 session setup
- 具有优先级的，多并发 channel
- 可靠及不可靠的交付语义
- 内置安全策略（DTLS）以及拥塞控制。

```js
const peerConnection = new webkitRTCPeerConnection(servers, {
  optional: [{RtpDataChannles: true}]}
})

peerConnection.ondatachannel = function(event) {
  // 获得 receive channel
  const receiveChannel = event.channel;
  receiveChannel.onmessage = function(event) {
    document.querySelector('#receiver').innerHTML = event.data;
  }
}

// 创建一个不可靠的 send channel
const sendChannel = peerConnection.createDataChannel('sendDataChannel', {reliable: false})

document.querySelector('button#send').onclick = function () {
  const data = document.querySelector('textarea#send').value
  sendChannel.send(data)
}
```



## 现实世界的 WebRTC

在现实世界中，WebRTC 需要服务器，无论架构多么简单，需要满足：

- 用户能够发现彼此，并且交换各自的细节信息
- WebRCT 客户端（peer）交换网络信息
- peer 交换媒体内容，例如视频格式和分辨率
- WebRTC 客户端穿透 NAT 网关和防火墙

换言之，WebRTC 需要四种类型的服务端能力：

- 用户发现及通信
- 信令
- NAT 及防火墙穿透
- P2P 失败时的中转（relay）服务器

ICE 所使用 TURN 协议和 STUN 协议使得 RTCPeerConnection 能够应付 NAT 穿透及其他网络异常。

ICE 协议是为了让节点（例如视频聊天的两个节点）相连。其试图通过 UDP，以最小延迟直接连接各个节点。在这个过程中，STUN server 只有一个职责：让节点查询到自己的公网地址和端口。

![](https://cdn-images-1.medium.com/max/1600/1*ONNxJHqmMTXB1Nuq3qTNXQ.png)

### 查询连接候选人

如果 UDP 失败了，ICE 将尝试使用 TCP：首先会使用 HTTP，再使用 HTTPS。如果连接失败了，可能是由于企业的 NAT 穿透和防火墙原因，ICE 将会使用 TURN server 作为中转。换言之，ICE 将首先使用 STUN 直连各个节点，失败的时候再通过 TURN 进行中转。

![](https://cdn-images-1.medium.com/max/1600/1*0REL14sYPR34hY7yua6-PA.png)

## 安全

一些实时通信的应用或者插件会在安全上有些问题：

- 未经加密的内容可能在浏览器间，或浏览器与服务器之间被截获
- 一个应用可能在用户不知情的情况下分发视频或者音频内容
- 插件或者应用自身可能携带病毒或者被病毒侵入

WebRTC 的一些特性能规避这些状况：

- WebRTC 使用了 [DTLS](http://en.wikipedia.org/wiki/Datagram_Transport_Layer_Security) 和 [SRTP](http://en.wikipedia.org/wiki/Secure_Real-time_Transport_Protocol) 安全协议
- WebRTC 的组件被强制加密
- WebRTC 不是一个插件：其组件运行在浏览器沙盒中，而不是一个分离的进程，组件不需要安装，也不需要随着浏览器的更新而更新
- 摄像头和麦克风的使用必须经过用户授权