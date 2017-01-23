---
title: WebRTC通信初探
date: 2017-01-23 10:17:37
tags: [HTML5,WebRTC]
categories: 
- HTML5
---
WebRTC是由谷歌收购GIPS公司开源的一项用于方便实现音频和视频实时P2P通信的技术。除了音频和视频通信外，还支持其他包括字符串，Blob,ArrayBuffer,ArrayBufferView等数据类型的传输。

与之前聊过的WebSocket应用层协议不同，虽然都能实现实时通信，但是ws实现server-client的全双工通信，而WebRTC实现了端对端的通信技术，不依赖于服务器，在视频直播会议，游戏，文件传输等领域发挥着重要作用。

**文章目录**
1. WebRTC底层架构
2. MediaStream获取音视频数据流
3. RTCPeerConnection实现P2P通信
4. NAT/防火墙 穿越技术
5. RTCDataChannel API
6. 遇到的问题总结

![WebRTC](http://7xsi10.com1.z0.glb.clouddn.com/f1d3e86b156c3c663fab85434ee07858_b.png)
<!--more-->

## 底层架构
上图实为WebRTC的底层架构，可以其用到的技术非常多的，包括视频音频处理以及网络传输，防火墙穿越等技术。

---
WebRTC有三个模块，Voice Engine(音频引擎)，Video Engine(视频引擎)，Transport。Voice Engine包含iSAC/iLBC Codec(音频编解码器，前者是针对宽带和超宽带，后者是针对窄带)，NetEQ for voice(处理网络抖动和语音包丢失)，Echo Canceler(回声消除器)，Noise Reduction(噪声抑制)；Video Engine包含VP8 Codec(视频图像编解码器)，Video jitter buffer(视频抖动缓冲器，处理视频抖动和视频信息包丢失)，Image enhancements(图像质量增强)。Transport包含SRTP(安全的实时传输协议，用以音视频流传输)，Multiplexing(多路复用)，P2P，STUN+TURN+ICE(用于NAT网络和防火墙穿越的)。除此之外，安全传输可能还会用到DTLS(数据报安全传输)，用于加密传输和密钥协商。整个WebRTC通信是基于UDP的

## MediaStream获取音视频数据流

### getUserMedia(constraints,successCallback,errCallback)

navigator上的方法，用于获取用户授权提供的音频视频数据流，三个参数分别为约束对象，成功的回调函数，发送错误的回调函数。

### 浏览器兼容性

```js
const getUserMedia = (navigator.getUserMedia || 
                    navigator.webkitGetUserMedia || 
                    navigator.mozGetUserMedia || 
                    navigator.msGetUserMedia);
```

### 上栗子：MediaStream和Canvas实现拍照功能
```js
const getUserMedia = (navigator.getUserMedia || 
                    navigator.webkitGetUserMedia || 
                    navigator.mozGetUserMedia || 
                    navigator.msGetUserMedia);
let video = document.getElementById('video'),
	canvas = document.getElementById('canvas'),
	ctx = canvas.getContext('2d'),
	localStream = null;
getUserMedia({video:true,audio:true},stream=>{
    video.src = window.URL.createObjectURL(stream);
    localStream = stream;
},err=>{
    console.log('get stream failed!');
});
video.addEventListener('click',()=>{
    if(localStream){
        ctx.drawImage(video,0,0);
        document.getElementById('img').src = canvas.toDataURL('image/png');
    }
});
```
## RTCPeerConnection实现P2P通信

P2P通信基于UDP传输协议，更加注重传输实时性。P2P建立过程是比较复杂的。主要是交换SDP和ICE信息。
![P2P](http://images.cnitblog.com/blog2015/57211/201503/250050215053233.png)

上图所示为利用信令服务器实现P2P通信的流程图，其中还包含了STUN服务器（非WebRTC实现)

### P2P通信建立过程

#### 交换SDP
SDP是一种会话描述协议（Session Description Protocol），包含了一系列信息包括会话使用的媒体种类，双方ip和port,带宽，会话属性等。

SDP交换采用Offer/Answer形式。

1. 首先Offer方通过new RTCPeerConnection(config)建立PeerConnection
2. Offer方通过createOffer生成sessionDescription，设置localDescription,并通过信令服务器发送给Answer方
3. Answer方收到offer,发现并没有与之对应的peerConnection,新建peerConnection,并设置remoteDescription
4. Answer方通过createAnswer生成sessionDescripton,设置localDescription,并通过信令服务器发送answer
5. Offer方收到answer,设置remoteDescription
6. SDP交换结束

#### 交换ICE

ICE是一种用于实现NAT/防火墙穿越的协议，可以实现:
1. P2P直接通信
2. 使用STUN服务器实现突破NAT的P2P通信
3. 使用TURN中继服务器实现突破防火墙的中继通信

交换过程：
1. 双方在new RTCPeerConnection(config)建立连接后，当网络候选者可用时，会触发icecandidate事件
2. 在onicecandidate事件处理程序中将candidate通过信令服务器发送给对方
3. 双方在接受到彼此的candidate后，通过addIceCandidate将对方的candidate加入到PeerConnection实例中。

在连接建立前或者建立后调用peerConnection.addStream()方法将本地视频/音频数据流加入到connection当中，当对方接受到视频流时会触发addStream事件，在其处理程序中我们可以接受数据流将其显示。

### 基于Socket.io和WebRTC的两人视频聊天室

上述过程的信令服务器可以使用WebSocket服务，而Node.js可以方便的实现ws服务，socket.io更是封装了一系列API，可以方便的实现多人视频聊天室，多人视频聊天室有需要注意一些其他诸如SDP冲突问题，这里先以两人通信为例来更深入理解整个过程。

```js
//代码较多这里省略
```


## NAT/防火墙穿越技术
之所以将NAT/防火墙单独出来，是因为NAT/防火墙问题是建立端对端通信的一个重要问题
![STUN](http://lingyu.wang/img/WebRTC/3.png)
### NAT
NAT将连接到公网的全局ip转换为内网ip，实现多个终端通信，防止受到来自外网的攻击，有效节省IPV4数量。WebRTC必须穿越NAT进行通信。ICE可以通过STUN技术穿越NAT。

NAT的实现方式有三种，即静态转换Static Nat、动态转换Dynamic Nat和端口多路复用OverLoad。目前用得比较多的就是端口多路复用。

STUN服务器可以是自己搭建的，也可以是直接使用现成的，比如谷歌的stun服务:stun:stun.l.google.com:19302

自己搭建STUN服务器比较简单，这里篇幅有限，省略

配置好STUN服务，以此建立RTCPeerConnection,配置方法就是上面的config对象:

```js
const config = {
	"iceServers":[{"url":"stun:stun.yourdomain.com:3478"}]
	}
let peer = new RTCPeerConnection(config);
```

### 防火墙

对于防火墙，需要依靠TURN服务器来进行通信。起一各TURN服务器监听在某个端口时，需要设置防火墙开发这个端口。搭建TURN服务器也比较简单，在这里也省略。

同样也要配置好config:

```js
const config = {
	"iceServers":[
		{"url":"stun:stun.yourdomain.com:3478"},{"url":"turn:turn.yourdomain.com:3478","username":"yourid","credential:"yourpassword"}]
	}
let peer = new RTCPeerConnection(config);
```

## RTCDataChannel API

既然可以实现音频视频的实时通信，为何不可以实现文本，文件等数据的传输呢？RTCDataChannel API就提供了这个功能。它通过将数据直接从一个浏览器发送到另一个浏览器，不需要将数据通过服务器来进行中转发送，简化了过程，保证实时性，同时还确保数据的安全私密性。

RTCDataChannel 与RTCPeerConnection API相结合，使用SCTP（流控制传输协议）实现稳定有序的数据传递。其仍然需要信令服务器的参与。以及STUN和TURN服务器来穿越NAT/防火墙。

RTCDataChannel支持两种模式运行：不可信赖模式（类似UDP)和可信赖模式（类似于TCP)。

### 流程分析
1. 通过config建立RTCPeerConnection
2. 通过peerConnection.createDataChannel(label,dataChannelConfig)获取dataChannel对象
3. 按照上述P2P建立流程完成SDP和ICE信息交换
4. 调用send()方法发送消息
5. 接收方监听message事件，获取数据

```js
let dataChannel = peerConnection.createDataChannel('dataChannel',{'reliable:false'});
```
详细API可查看WebRTC官网
## 遇到的问题总结


