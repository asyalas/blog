### 目录

- SPDY
- HTTP@
  - 二进制格式传输
  - 多路复用
  - HPACK 首部压缩
  - server push
  - 强制加密
- TCP和UDP的区别
- TCP的三次握手／四次挥手
- QUIC
  - 简介
- 生成证书


### SPDY

SPDY 并不是新的一种协议，而是在 HTTP 之前做了一层会话层。

### http2

#### 二进制格式传输

HTTP/1.1的头信息肯定是文本（ASCII编码），数据体可以是文本，也可以是二进制（需要做自己做额外的转换，协议本身并不会转换）。

而在HTTP/2中，新增了二进制分帧层，将数据转换成二进制，也就是说HTTP/2中所有的内容都是采用二进制传输。

使用二进制可以定义额外的帧，默认定义了十种帧，如数据帧、头部帧、PING帧、SETTING帧、优先级帧和PUSH_PROMISE帧等

#### 多路复用

二进制分帧层把数据转换为二进制的同时，也把数据分成了一个一个的帧。帧是HTTP/2中数据传输的最小单位；每个帧都有stream_ID字段，表示这个帧属于哪个流，接收方把stream_ID相同的所有帧组合到一起就是被传输的内容了

在一个TCP连接中，可以同时传输多个流，即可以同时传输多个HTTP请求和响应，这种同时传输不需要遵循先入先出等规定，因此也不会产生阻塞（线头阻塞），效率极高。

#### HPACK 首部压缩

具体规则可以描述为：

- 通信双方共同维护了一份静态表，包含了常见的头部名称与值的组合
- 根据先入先出的原则，维护一份可动态添加内容的动态表
- 用基于该静态哈夫曼码表的哈夫曼编码数据

#### server push

而HTTP/2的server push允许服务器在未收到请求时就向浏览器推送资源从而利用并发的优势减少时间。

#### 强制加密

虽然加密不是强制的，但是多数浏览器都是通过TLS(HTTPS)实现的HTTP/2

### tcp和udp的区别

- TCP

TCP 协议是面向`有连接`的协议，也就是说在使用 TCP 协议传输数据之前一定要在发送方和接收方之间建立连接。一般情况下建立连接需要`三次握手`，关闭连接需要`四次挥手`。

建立 TCP 连接后，由于有数据重传、流量控制等功能，TCP 协议能够正确处理丢包问题，保证接收方能够收到数据，与此同时还能够有效利用网络带宽。

- UDP

UDP 协议是面向`无连接`的协议，它只会把数据传递给接收端，但是不会关注接收端是否真的收到了数据。但是这种特性反而适合多播，实时的视频和音频传输。因为个别数据包的丢失并不会影响视频和音频的整体效果。

### tcp的三次握手／四次挥手

参考  [输入url按回车后发生的一系列不可描述的事情](../../../2018/blog/输入url按回车后发生的一系列不可描述的事情.md)

### QUIC

#### 简介

QUIC（Quick UDP Internet Connections），直译过来就是“快速的 UDP 互联网连接”，是 Google 基于 UDP 提出的一种改进的通信协议，作为传统 HTTP over TCP 的替代品

#### QUIC 的优点

- 在发生丢包时，QUIC使用了前向纠错算法，通过连续的几个数据包的校验和，可以直接恢复出丢失的包内容，而不需要重传。
- 之前连接过，那么之后可以不用重复握手而直接开始传送数据，以实现 0-RTT 往返时延。
- 通过 ID 来标识用户（而不是 IP + 端口），在连接切换后直接恢复之前的连接会话，

在移动端表现更好：用户的网络环境并不稳定，Wi-Fi、4G、3G、2G 之间来回变化，IP 一旦发生变化，TCP 的连接是不可能保持的。而 QUIC 就不存在这样的问题，通过 ID 来标识用户（而不是 IP + 端口），在连接切换后直接恢复之前的连接会话。

### 生成证书

```shell
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048 
openssl rsa -passin pass:x -in server.pass.key -out server.key
openssl req -new -key server.key -out server.csr
# ...
# 输入证书相关信息（随意填写）
# ...
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
rm server.pass.key
```

### 参考

- [TCP 与 UDP 协议简介](https://mp.weixin.qq.com/s?__biz=MzI5ODY1NTU4Ng==&mid=2247483846&idx=1&sn=2972d306df88f8167ac040725f2a6d48&chksm=eca3cbcbdbd442dde344c8c15182cad9f9e1c518f36865db2369c8387083ab42752dcbe8e325&token=1174004427&lang=zh_CN#rd)