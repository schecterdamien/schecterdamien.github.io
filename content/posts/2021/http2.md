---
title: HTTP/2 协议
date: 2021-07-25 21:27:17
tags: ["web"]
menu: "posts"
---

&nbsp;&nbsp;&nbsp;&nbsp; 用grpc很久了，但是一直都仅限于使用，对于比较深层次的细节还是缺乏了解，想找个时间好好学习下这个黑盒底下的一系列技术，最近刚好要准备一个技术分享，所以就决定分享HTTP/2协议相关的细节，因为grpc的通信就是用的HTTP/2协议，这篇文章算是对准备的HTTP/2分享的一个记录

&nbsp;&nbsp;&nbsp;&nbsp; http应该大家都不陌生，目前使用最多的应该还是HTTP/1.1版本的http协议，那么HTTP/1.1到底有些什么样的问题，导致需要HTTP/2呢？主要有两点

+ 头阻塞：HTTP/1.1中只有收到当前请求响应后才能重用当前tcp连接发送下一次请求。虽然HTTP/1.1提出了`pipeline`，旨在缓解这个问题。但是一方面pipeline在复杂的网络环境下很难实现和普及，其次就算都用上了`pipeline`，`pipeline`也只是使得可以在本次响应完成之前可以发送下次请求，但响应依然要严格按照顺序返回，也就是如果前一个响应被阻塞，后边的响应将不会到来，头阻塞问题还是没有完全解决。目前客户端（比如浏览器）一般会同时打开多个tcp连接来绕开头阻塞的问题，chrome默认会打开6个tcp链接，并发请求各种类型数据。但是这就带来了一个矛盾，tcp链接越多，肯定对资源的浪费是越大的，链接越少，头阻塞的问题就会越突出
+ 重复的未压缩头数据的传输：自HTTP/1.1之后，HTTP请求中通常带有大量ASCII编码的头部，这些头部通常大部分不会变化，需要每次请求都携带（尤其像是User Agent、Cookie这些值比较长的头部字段），会给本来就拥挤的网络带来很大的压力

除此之外还有一些问题，比如TCP 的慢启动，起初会限制连接的最大速度，如果数据成功传输，会随着时间的推移提高传输的速度，而大量http的业务请求其实本身都只是短链接的，所以导致tcp的流控算法没有很好的应用。为此，HTTP/2的出现就很有必要了

### HTTP/2 概述 ###


先说下HTTP/2的发展历程:

+ 2009年，Google提出了一项实验性的协议SPDY（读音同speedy），旨在开发者不修改当前网站实现的前提下，提高页面加载速度。SPDY提出后，Chrome、Firefox、Opera等主流浏览器先后给出了实现，很多大型网站（如Google、Twitter、Facebook等）分别提供了其对SPDY会话的实现
+ 2012年，HTTP-WG提出了在SPDY基础上构建HTTP/2的草案
+ 2013年给出了第一个对HTTP/2的实现，自此HTTP/2、SPDY并行发展，在客户端和服务器上进行了广泛可靠的测试
+ 2015年，Google 宣布放弃对SPDY的继续支持，标志着HTTP/2正式登上历史舞台

那么HTTP/2有些什么特点呢：

+ HTTP/2 可以让我们的应用更快、更简单、更稳定
+ 通过有效压缩 HTTP 标头字段将协议开销降至最低，同时增加对请求优先级和服务器推送的支持
+ 带来了大量其他协议层面的辅助实现，例如新的流控制、错误处理和升级机制
+ HTTP/2 没有改动 HTTP 的应用语义，而是修改了数据的格式和传输方式，只需要升级基础设施（代理、web框架等），上层的业务代码都可以不必修改而在新协议下运行

>>> 上一个HTTP版本是1.1，为什么这之后不是 HTTP/1.2呢？因为 HTTP/2 引入了一个新的二进制分帧层，该层无法与之前的 HTTP/1.x 服务器和客户端兼容，因此协议的主版本提升到 HTTP/2，且HTTP/2后HTTP协议没有小版本号了，后面直接就是HTTP3


&nbsp;&nbsp;&nbsp;&nbsp; 那HTTP/2是怎么做到加入了这么多功能和优化，但是上层业务不需要改代码呢？一切都是因为引入了一个二进制分帧层，并且HTTP/2的新特性也都是建立在这个基础之上的。这又印证了那句话，“计算机中任何问题都可以通过增加一个中间层来解决“。

![](/img/http2/layer.png)


&nbsp;&nbsp;&nbsp;&nbsp; 如上图，HTTP/2.0在应用层（HTTP）和传输层（TCP或者TLS）之间增加一个二进制分帧层，HTTP/2.0通信都在一个TCP连接上完成，这个连接可以承载任意数量的双向数据流，相应的每个数据流以消息的形式发送。而消息由一或多个帧组成，这些帧可以乱序发送，然后根据每个帧首部的流标识符重新组装

![](/img/http2/layer2.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 这是另一张描述连接（connection）、流（stream）、消息（message）、帧（frame）关系的图，这里解释下这几个概念。     

+ 帧（Frame）：
    + 帧是HTTP/2通信的最小单位
    + 请求和响应都分为首部帧和消息帧单独传输，初此之外还有一些其他的类型的帧，下面会介绍帧结构的时候会说 
+ 消息（Message）：
    + 指逻辑上的HTTP消息，比如一次请求、一次响应等。
    + 由一个或多个帧组成，以二进制压缩格式存放 HTTP/1 中的头部。    
+ 流（Stream）：
    + 是一条逻辑上的连接，是TCP连接中的一个虚拟通道（每个Tcp连接会建立多个steam）
    + 可以承载双向的消息，每个流都有一个唯一的整数标识符，帧会记录Stream 的id
    + 不同 Stream 的帧是可以乱序发送的（因此可以并发不同的 Stream ），因为每个帧的头部会携带 Stream ID 信息，所以接收端可以通过 Stream ID 有序组装成 HTTP 消息。同一 Stream 内部的帧必须是严格有序的。
    + 客户端和服务器双方都可以建立 Stream， Stream ID 也是有区别的，客户端建立的 Stream 必须是奇数号，而服务器建立的 Stream 必须是偶数号。
    + 同一个连接中的 Stream ID 是不能复用的，只能顺序递增，所以当 Stream ID 耗尽时，需要发一个控制帧 GOAWAY，用来关闭 TCP 连接
+ 连接（Connection）：
    + 即TCP连接

&nbsp;&nbsp;&nbsp;&nbsp; 这种在一个TCP连接中划分多个逻辑链接，把所有两端的流量都集中在一个TCP连接的做法，减少了服务端的压力，虽然流量还是这么多流量，但是创建的socket会明显减少，内存占用更少，每个连接吞吐量更大。并且HTTP 性能优化的关键并不在于高带宽，而是低延迟，这种做法避免了TCP的慢启动，能更有效的利用到TCP的流控算法。


### 帧结构 ###

![](/img/http2/frame_format.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 让我们再来看一下帧的结构，帧由帧头（Frame Header）和帧负载（Frame Payload）构成，如上是帧头的结构，帧头总共9个字节，包括帧长度、帧类型、标识位、保留位和流标识符。    
&nbsp;&nbsp;&nbsp;&nbsp; 其中帧长度为3个字节，表示帧负载（Frame Playload）的长度（不包括帧头9字节），这个长度的最大值通过SETTING类型的帧中携带的SETTINGS_MAX_FRAME_SIZE参数来设置。假如一个HEADERS帧不够传输所有的HEADER字段，则会把剩余的部分放到后续的CONTINUATION类型的帧来传输   
&nbsp;&nbsp;&nbsp;&nbsp; 帧类型，1个字节，表示帧的类型，目前HTTP/2 总共定义了 10 种类型的帧。

帧类型 | 类型编码 | 用途 | 
--- | --- | ---
DATA | 0x0 | 传递HTTP包体 | 
HEADERS | 0x1 | 传递HTTP头部 | 
PRIORITY | 0x2 | 指定STREAM流的优先级 | 
RST_STREAM | 0x3 | 终止STREAM流 | 
SETTINGS | 0x4 | 初始化或修改连接或者STREAM流的配置 | 
PUSH_PROMISE | 0x5 | 服务端推送资源时描述请求的帧 | 
PING | 0x6 | 探活、兼具计算RTT往返时间 | 
GOAWAY | 0x7 | 优雅终止连接并通知错误 | 
WINDOW_UPDATE | 0x8 | 用于流控，指定窗口大小的包 | 
CONTINUATION | 0x9 | 传递较大HTTP头部的持续帧 | 

&nbsp;&nbsp;&nbsp;&nbsp; TCP本身有探活的机制，为什么HTTP/2里又要重新定义一个PING的帧呢？因为TCP协议栈其实是内核里实现的，收到包后会自动ack把数据缓存到内核缓存区里，然后再复制到应用程序的进程空间，所以其实TCP的探活不能反映应用程序是否正常，可能应用程序就是因为某些原因卡住了，但是进程又没挂，ack还是正常的。所以在应用层实现一种探活机制是很有必要的。    
&nbsp;&nbsp;&nbsp;&nbsp; 标识位，一字节，总共可以保存 8 个标志位，用于携带简单的控制信息，这里同一位在不同类型的帧里的语义可能不太一样，目前HTTP/2总共只定义了有5种标志

标志位 | 位置 | 含义 | 此标志位对哪些帧有效 | 
--- | --- | --- | ---
ACK | 0x01 | 表示这个帧是回复确认帧（某些帧需要接受方回复一个同样类型的确认帧） | SETTINGS、PING |
END_STREAM | 0x01 | 表示这个帧是这个流的最后一个帧 | DATA、HEADERS |
END_HEADERS | 0x04 | 表示这个帧是承载http头的最后一个帧 | HEADERS、PUSH_PROMISE、CONTINUATION |
PADDED | 0x08 | 表示这个帧的payload里存在padding | DATA、HEADERS、PUSH_PROMISE、CONTINUATION |
PRIORITY | 0x20 | 表示帧的payload里存在优先级信息相关字段  | HEADERS |

&nbsp;&nbsp;&nbsp;&nbsp; 可以发现ACK和END_STREAM在标识位的里的位置一样，都是0x01，第一位，但是在不类型的帧里，这一位可能表示是ACK可能表示是END_STREAM。有些帧是没有定义标志位（字段还是存在的，只是为0x00），包括PRIORITY、RST_STREAM、GOAWAY、WINDOW_UPDATE这些帧

&nbsp;&nbsp;&nbsp;&nbsp; 标志位后面是1位的保留位，暂时未定义

&nbsp;&nbsp;&nbsp;&nbsp; 保留位后面是31位的流标识符，所以流标识符的最大值是 2^31，大约是 21 亿。这个值表示帧属于的流，某些帧的这个值必须为0，比如SETTINGS、PING、GOAWAY，因为这些帧并不属于任何流，是和整个连接相关的。WINDOW_UPDATE的这个值可以为0，可以不为0，为0则表示这个帧的payload描述的是整个连接的流控信息，不为0则是描述的指定的这个流的流控信息。其他类型的帧流标识符不能为0，这张图很好的描述的不同类型的帧的标志位和流标示符的情况：

![](/img/http2/frame_flag.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 帧体则在不同类型的帧里的结构都不一样，这里不再赘述，感兴趣的可以看下参考链接

### HTTP/2特性 ###

&nbsp;&nbsp;&nbsp;&nbsp; 相比HTTP1，HTTP/2除了上面说的分帧层，HTTP/2还有如下特性：首部压缩、流优先级、服务端推送、流量控制，下面一一介绍。

#### 首部压缩 ####

&nbsp;&nbsp;&nbsp;&nbsp; HTTP/1.1包含很多固定字段，比如Cookie、User Agent、Accept 等，这些字段加起来也高达几百字节甚至上千字节，HTTP/2对这些内容进行了二进制压缩，所以header在HTTP/2里不是直接可读的格式。同时因为大量请求响应报文里很多header都是重复的，这些会占用额外带宽和cpu。HTTP/2把header的字段在客户端和服务端分别给缓存了。    
&nbsp;&nbsp;&nbsp;&nbsp; HTTP/2处于安全考虑，没有使用常见的压缩算法比如gzip，而是定义了一个名叫的HPACK的算法，由静态表、动态表、哈夫曼编码等方式构成，更详细的可以看下参考链接

#### 请求优先级 ####

&nbsp;&nbsp;&nbsp;&nbsp; 将 HTTP 消息分解为很多独立的帧之后，我们就可以复用一个连接，客户端和服务器交错发送和传输这些帧的顺序就成为关键的性能决定因素。而对于浏览器来说，这是一种用于提升浏览性能的关键功能，网络中拥有多种资源类型，它们的依赖关系和权重各不相同，比如可以定义传输HTML的流的优先级比传输图片文件的流的优先级更高，来让服务端优先传输HTML

![](/img/http2/priority.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 如上图，HTTP/2定义了每个流的依赖关系，依赖于另一个流的流是依赖流，被依赖的是父流。兄弟节点则会定一个一个权重值，这个权重是个介于1至256之间的整数，兄弟节点依靠权重来分配资源。这个模型构建了一个虚拟的流，作为所有流的祖先节点，是这个依赖树的根节点（其实就是id为0的流，当然这个流是不存在的）。流节点不能依赖自身。

&nbsp;&nbsp;&nbsp;&nbsp; 请求先级这个特性主要还是客户端用来调整服务端发送资源的顺序，比如浏览器建议服务端返回资源的优先级。但是，请求优先级只是一个建议，而不是要求，更不是保证，流优先级不保证流相对于任何其他流的任何特定处理或传输顺序。也就是说，客户端定义了流优先级，并把这个信息通过HEADERS或者PRIORITY帧传输给服务端，但是服务端会不会按照这个定义的优先级顺序来进行处理和返回，则完全没有保证，可能服务端没有实现任何和流优先级相关的功能，也可能实现了，一旦服务端实现了，就可以利用客户端给予的信息来做一些优化。

#### 服务器推送 ####

&nbsp;&nbsp;&nbsp;&nbsp; 服务器推送这个机制允许服务端主动向客户端在新开的流上推送一些资源，典型的场景是浏览器请求一个html页面，html里会引用css和相关图片，当服务端收到html页面的请求时，除了返回html文件，还会直接把相关的css和图片文件都返回，这样浏览器就不用请求多次再来获取html页面里引用的资源了。服务器推送在语义上等同于服务器响应一个请求，为了不改变HTTP的语义怎么来做呢，服务端会先返回一个帧作为请求，然后在一个新打开的流上，返回一个帧作为响应，大概过程是：

+ Push Requests：当收到一个客户端请求时，服务端决定推送一些资源，此时服务端会发送一个PUSH_PROMISE帧给客户端，这个PUSH_PROMISE帧包含了后面返回的响应帧所在的流的id，还包含了响应的header的信息。客户端在接收到 PUSH_PROMISE帧后，它可以根据自身情况选择拒绝响应流（发送RST_STREAM帧），例如，如果资源已经位于缓存中，便可能会发生这种情况。同时这个PUSH_PROMISE在语义上视作客户端对服务端的一个请求  
+ Push Responses：发送PUSH_PROMISE帧后，服务端会直接发送数据帧到之前PUSH_PROMISE承诺的流上，作为响应，（因为header已经在PUSH_PROMISE传输了，这里只要传输body即可）也就是这里的数据帧在语义上相当于对PUSH_PROMISE帧的一个响应

![](/img/http2/push.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 服务端推送只能由服务端进行，客户端（发起连接的一方）无法推送，也就是只能是单向的。所以在第一眼看到服务端推送这个功能时，还以为和我们常规理解的服务端推送技术一样，还想着是否在HTTP/2时代就不需要websocket了，现在看来还是不太一样，正是因为这个推送是单向的，所以还是受到了很大的限制，场景也比较有限

#### 流量控制 ####

&nbsp;&nbsp;&nbsp;&nbsp; 由于 HTTP/2 数据流在一个 TCP 连接内复用，TCP 流控制既不够精细，也无法提供必要的应用级 API 来调节各个数据流的传输。为了解决这一问题，HTTP/2 提供了一组简单的机制，允许客户端和服务器实现其自己的数据流和连接级流控制。

&nbsp;&nbsp;&nbsp;&nbsp; 流量控制基于WINDOW_UPDATE帧，发送者通告他们准备在 stream 流，或者整个连接上接收多少个八位字节，流量控制是定向的，发送者提供信息，接收者控制自己的速度。对于一个新的stream流和整个连接，流量控制窗口的初始值为 65,535 个八位字节。并且只有DATA帧才会受到流控的限制，因为其他帧都不大，而且也比较重要，这确保了重要的控制帧不会被流量控制阻挡。无法禁用流量控制。建立 HTTP/2 连接后，客户端将与服务器交换 SETTINGS 帧，这会在两个方向上设置流控制窗口

&nbsp;&nbsp;&nbsp;&nbsp; HTTP/2 仅定义 WINDOW_UPDATE 帧的格式和语义。未规定发送方何时发送此帧或其发送的值，也未规定接受方如何选择发送数据包。实现方能够选择任何适合其需求的算法，也就是HTTP/2的流量控制只是定义了流控信息的交换格式，至于怎么利用这些信息来进行流控，客户端和服务端可以有不同的实现

### HTTP/2带来的一些变化 ###

&nbsp;&nbsp;&nbsp;&nbsp; 那么HTTP/2到底给我们带来了哪些改变呢？最主要的一点就是更快了，上面的请求优先级、服务端推送，流量控制其实都在某种程度上让HTTP/2的请求响应更快了，并且因为浏览器可以在一个TCP上多路复用并发请求资源，不再局限于每个域只能打开一定的TCP连接，所以不会出现请求超过这个数目的资源时，后续的请求只能阻塞等待前面的请求完成的情况。在这个网站（[https://HTTP/2.akamai.com/demo](https://HTTP/2.akamai.com/demo)）上可以感受下HTTP/1.1和HTTP/2速度的差异。除此之外，有一些HTTP/1.1里的做法在HTTP/2里就不太适用了

+ 域名分片：HTTP/1.x中，因为浏览器会对并发连接有限制。为了突破这个限制，通常会把请求的资源置于不同的域名下（如 shard1.example.org, shard2.example.org）。而在HTTP/2中，因为不需要新开连接来解决头阻塞问题，直接可以在一个连接上多路复用，所以这种做法没什么必要了

+ 雪碧图：HTTP/1.x中，为了缓解请求多个图片带来的头阻塞问题，通常采用把多个图片拼接成一个大图，然后一个请求将所有图片加载在浏览器中，然后通过css或者js将所需要的部分按需展示出来，这种做法增加了代码的复杂性，并且浏览器渲染过程中，内存需要加载更多的图片，同上，因为HTTP/2支持连接多路复用，这种做法也没必要了

+ 拼接js、css：类似雪碧图，拼接JavaScript、CSS的做法同样是为了减少请求数，以避免潜在的头阻塞问题

+ 资源内联：为了防止在请求页面后，还需要多次请求页面里引用的css和图片，有时会把css、base64编码的图片直接内连到页面里一起返回，HTTP/2的服务端推送（server push）正好解决了这个问题

&nbsp;&nbsp;&nbsp;&nbsp; 当然，HTTP/2也不是那么完美，也仍然会存在一些问题，比如

+ 解决了HTTP1.1的队头阻塞问题，但是TCP 的队头阻塞并没有彻底解决，HTTP/2 出现丢包时，整个 TCP 都要等待重传，那么就会阻塞该 TCP 连接中的所有请求
+ TCP 连接断掉会导致所有的Stream断掉，而在HTTP/1.1里断掉一个TCP连接影响不会这么大
+ 这篇文章([https://www.lucidchart.com/techblog/2019/04/10/why-turning-on-http2-was-a-mistake/](https://www.lucidchart.com/techblog/2019/04/10/why-turning-on-http2-was-a-mistake/))描述了一个问题，因为浏览器不在像之前那样限制并发的请求（通过限制TCP连接来实现），导致服务器的压力更大，并且会出现更多毛刺现象，更多尖锐的流量，监控上的流量不像之前HTTP/1.1那么平滑，需要重新调整下负载均衡的设置（虽然HTTP2也会通过SETTING帧来设置最大的并发Stream数，但是默认是100，还是比较大）


### HTTP/2协议协商 ###

&nbsp;&nbsp;&nbsp;&nbsp; 对于客户端或者服务端来说，怎么确认对端是否支持HTTP/2呢？这里就要说下HTTP/2的协议协商机制。HTTP/2 使用HTTP/1.1相同"http"和"https" URI scheme。且共享相同的默认端口号: "http" 为 80，"https" 为 443。因此，对于客户端来说首先需要发现上游服务器是否支持 HTTP/2，对于 "http" 和 "https" URI，确定支持 HTTP/2 的方式是不同的。对此，RFC中定义了两个不同的标识符：

+ 字符串"h2"标识：表示HTTP/2使用传输层安全性(TLS)协议，在TLS上加密通信，对应https
+ 字符串"h2c"标识：表示HTTP/2，不使用TLS，直接建立在TCP明文通信，对应http

#### h2c的协议协商 ####

&nbsp;&nbsp;&nbsp;&nbsp; h2c，也就是直接建立在TCP上的HTTP/2的协议协商，简单来说就是客户端会先用HTTP/1.1协议的格式开始通信，然后协商升级到HTTP/2，这依赖于HTTP/1.1的协议升级机制。HTTP/1.1（也仅限1.1版本）提供了一种特殊的机制，这一机制允许将一个已建立的连接升级成新的、不相容的协议，比如HTTP/2，比如websocket。通常来说这一机制总是由客户端发起的，服务端可以选择是否要升级到新协议。借助这一技术，连接可以以常用的协议启动（如HTTP/1.1），随后再升级到HTTP/2甚至是WebSockets，大致过程如下

&nbsp;&nbsp;&nbsp;&nbsp; 当客户端支持HTTP/2，试图升级到HTTP/2，可以先发送一个普通的请求（GET，POST等），这个请求需要添加三项额外的header：

+ Connection: Upgrade    // 设置 Connection 头的值为 "Upgrade" 来指示这是一个升级请求.
+ Upgrade: h2c     // 设置指定要升级的协议名称，这里就是h2c
+ HTTP/2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>    // 这个header包含了base64url编码的settings帧，作为客户端的初始化配置
 
&nbsp;&nbsp;&nbsp;&nbsp; 服务端收到请求，如果服务端不支持h2c，会忽略客户端发送的 "Upgrade 头部字段，返回一个常规的响应：例如一个200 OK，此时客户端就明白服务端不支持h2c，之后将会降级成HTTP/1.1进行通信。如果服务端支持h2c，决定升级这个连接，则会返回一个 101 Switching Protocols 响应状态码，并且原样返回`Connection: Upgrade`和`Upgrade: h2c`。并且在body里就会返回一个HTTP/2的settings帧，来初始化配置，作为服务端的连接序言，之后服务端可以和客户端开始通信了

&nbsp;&nbsp;&nbsp;&nbsp; 客户端在接收到101响应后，也必须发送一个连接序言（包括一个MAGIC帧和SETTINGS帧），然后可以开始传输其他帧，和服务端通信了


#### h2的协议协商 ####

&nbsp;&nbsp;&nbsp;&nbsp; HTTP/2 over TLS 使用 "h2"作为协议标识符，因为h2是建立在TLS之上，而TLS本身在握手的时候就会有一个协议协商的过程，在ClientHello中，客户端会发送自己支持的一系列协议或者算法（虽然都是和后续的TLS建立有关），然后服务端会在这其中选择一组协议或者算法来作为后续建立TLS的依据，并在ServerHello里返回。并且TLS还提供了扩展机制，借着这个扩展机制，可以在TLS握手阶段做一些额外的事情，h2的协议协商就是利用TLS的扩展机制来完成的。

&nbsp;&nbsp;&nbsp;&nbsp; h2使用名为ALPN（Application Layer Protocol Negotiation，应用层协议协商）的TLS扩展协议进行协商。正如HTTP/2前身是SPDY协议，Goggle在当时也开发了一个名为NPN（Next Protocol Negotiation，下一代协议协商协议）的TLS扩展，这就是ALPN的前身。顾名思义，APLN就是用来在TLS里协商上层协议使用什么协议的一个扩展。在HTTP/2以及之后，上文提到的HTTP/1.1的协议升级机制就被废除了，基于TLS的应用层协议均可以通过ALPN来进行协商。    
&nbsp;&nbsp;&nbsp;&nbsp; h2协商大概的过程，就是客户端TLS握手时候，会在ClientHello中带上ALPN扩展，在里面会指定客户端支持的应用层协议（一般能看到HTTP/1.1、h2、h2c三种），服务端收到后会返回ServerHello，如果不能识别ALPN，则结果里不会包括ALPN扩展，如果支持h2，则会在ALPN扩展返回h2。


&nbsp;&nbsp;&nbsp;&nbsp; HTTP/2 协议本身并没有要求它必须基于 HTTPS（TLS）部署，但是出于以下原因，实际使用中，HTTP/2 和 HTTPS 几乎都是捆绑在一起。一方面，HTTPS可以保证数据传输的安全性，另一方面因为TLS是建立在TCP之上的，对中间节点是保密的，所以具备更好的连通性，基于HTTPS部署的新协议具有更高的连接成功率。然后最重要的，当前的主流浏览器都只支持HTTPS访问HTTP/2服务，并且很多web框架也只支持h2方式的HTTP/2请求，比如go的gin。这也是个好事，可以进一步推动大家都来使用HTTPS。


### HTTP/2的支持情况 ###

&nbsp;&nbsp;&nbsp;&nbsp; 目前HTTP/2已经很成熟了，在互联网上已经比较普及了。客户端侧，主流的浏览器都已经支持了HTTP/2，在caniuse上可以看到浏览器对HTTP/2的支持情况：

![](/img/http2/support.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 服务端侧，nginx、haproxy等主流的反向代理也都支持了HTTP/2，很多语言的web框架也都能通过某些方式来支持HTTP/2，另外一个杀手级应用grpc就是使用HTTP/2通信的。golang对HTTP/2支持的比较早：

+ go在1.6开始支持HTTP/2，这个时候只支持HTTP/2 over TLS，也就是h2。只要是TLS部署，则http库就会默认进行HTTPS/2协商，协商失败则蜕化为HTTPS/1
+ go在1.8开始支持 HTTP/2 server push
+ go在1.11开始支持 HTTP/2 h2c（因为一般是不建议使用h2c这种方式来暴露HTTP/2服务的，所以社区经过不断讨论最后才决定合并对h2c的支持）
+ gin目前只支持h2方式的HTTP/2，不支持h2c方式的HTTP/2（看到有相关的pr，不知道未来是否会支持）

&nbsp;&nbsp;&nbsp;&nbsp; 如果想让服务升级到HTTP/2，目前大概有几种方式，如果只有静态资源，比如只是静态页面，像是博客这种，可以直接托管到支持HTTP/2的cdn或者类似的托管网站上。另外一种是服务端直接支持HTTP/2，需要看下框架是否支持，如果框架支持的话其实基本不需要更改业务层代码，因为HTTP/2保持语义不变，只是传输方式改变，这些复杂的事情一般web框架都会直接给处理了，对上层暴露的接口则保持了兼容（比如在go里，官方的net库直接就支持了HTTP/2，可能已经用上了都不知道），最后一种办法则是在反向代理这一层做协议转换，比如使用haproxy，haproxy和用户侧使用HTTP/2通信，和后端服务之间则是使用HTTP/1.1通信

&nbsp;&nbsp;&nbsp;&nbsp; 下一篇文章会介绍HTTP/2抓包的过程，以及途中遇到的坑....


### 参考链接

+ [https://developers.google.com/web/fundamentals/performance/http2/?hl=zh-cn](https://developers.google.com/web/fundamentals/performance/http2/?hl=zh-cn)
+ [https://datatracker.ietf.org/doc/html/rfc7540](https://datatracker.ietf.org/doc/html/rfc7540)
+ [https://imququ.com/post/protocol-negotiation-in-http2.html](https://imququ.com/post/protocol-negotiation-in-http2.html)
+ [https://halfrost.com/http2-header-compression/](https://halfrost.com/http2-header-compression/)
+ [https://halfrost.com/http2-http-frames-definitions/](https://halfrost.com/http2-http-frames-definitions/)
+ [https://zhuanlan.zhihu.com/p/359920955](https://zhuanlan.zhihu.com/p/359920955)
+ [https://github.com/amandakelake/blog/issues/35](https://github.com/amandakelake/blog/issues/35)



