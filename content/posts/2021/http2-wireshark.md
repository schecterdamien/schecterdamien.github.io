---
title: HTTP/2 抓包遇到的坑
date: 2021-12-23 22:35:18
tags: ["web"]
menu: "posts"
---

&nbsp;&nbsp;&nbsp;&nbsp; [上一篇文章](/posts/2021/http2/)介绍了http2协议相关的细节。因为是为了准备分享，所以为了尽可能直观的展示协议通信（特别是握手协商）的整个过程，还是决定准备一下小的demo然后抓包演示下。在这个过程也遇到了一些问题，耽搁了不少时间，所以记录一下整个过程

### h2c抓包 ###
&nbsp;&nbsp;&nbsp;&nbsp; 分别对HTTP2的两种建立链接的方式进行抓包（h2c和h2），先演示h2c的连接建立过程，准备的server端的代码如下：

```
package main

import (
	"fmt"
	"log"
	"net/http"

	"golang.org/x/net/http2"
	"golang.org/x/net/http2/h2c"
)

// http2 h2c
func main() {
	h2s := &http2.Server{}

	handler := http.NewServeMux()
	handler.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "pong")
	})

	s := &http.Server{
		Addr:    ":5005",
		Handler: h2c.NewHandler(handler, h2s),
	}

	http2.ConfigureServer(s, &http2.Server{})
	log.Fatal(s.ListenAndServe())
}
```

&nbsp;&nbsp;&nbsp;&nbsp; 然后打开wireshark，监听本地的5005端口，使用curl请求server，这里需要指定--http2使用http2协议，不然的话curl默认使用http1.1：

```
curl --http2 http://localhost:5005/ping
```
&nbsp;&nbsp;&nbsp;&nbsp; 可以在wireshark里看到建立连接的过程：

![](/img/http2-wireshark/w1.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 上图是客户端(curl)发送的http1.1的协议升级请求。这里能看到上篇文章提到的协议升级的关键的header。`Connection`指定header有哪些逐跳头部，`Upgrade`指定客户端希望升级到http2协议，`HTTP2-Settings`则指定了连接的初始参数。

![](/img/http2-wireshark/w2.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 上图是服务端的http response。返回了`Upgrade`表示服务端同意升级到http2协议，然后客户端和服务端就可以使用http2通信了。

##### h2c抓包遇到的问题 ##### 

&nbsp;&nbsp;&nbsp;&nbsp; 虽然在上文已经提到了结论：

+ go在1.6开始支持HTTP/2，这个时候只支持HTTP/2 over TLS，也就是h2。只要是TLS部署，则http库就会默认进行HTTPS/2协商，协商失败则蜕化为HTTPS/1
+ go在1.8开始支持 HTTP/2 server push
+ go在1.11开始支持 HTTP/2 h2c
+ gin目前只支持h2方式的HTTP/2，不支持h2c方式的HTTP/2

&nbsp;&nbsp;&nbsp;&nbsp; 但是在一开始对于go里是怎么支持http2是不太清楚的，是在第三方库里支持还是net/http直接支持？是h2c和h2两种方式都支持还是只支持h2？google的时候很多文章又语焉不详，所以在准备一段h2c server代码的时候比较曲折。   
&nbsp;&nbsp;&nbsp;&nbsp; 先是直接写了一个简单的gin的demo，然后用curl --http2抓包，发现server端总是不识别升级请求，后续还是使用http1.1通信，然后找gin的文档，只是写到gin支持http2 push（那说明gin肯定还是支持http2呗，但是h2c就是无法成功），后来在gin的源码里搜索h2c，无果，看issue，找到结果了，有人提了[相关pr](https://github.com/gin-gonic/gin/pull/1398)，然后又看了几篇其他文章才明白go本身官方库net里就支持http2了，h2和h2c两种方式都支持，是在gin这一层不支持h2c。然后看到了[这篇文章](https://colobu.com/2018/09/06/Go-http2-%E5%92%8C-h2c/)，改了下终于能跑一个支持h2c的server了

### h2抓包 ###
&nbsp;&nbsp;&nbsp;&nbsp; h2是通过tls协商建立http2连接，这种方式更具有实际意义，因为h2是更提倡的也是更安全普遍的建立http2的方式，先附上server端代码：

```
package main

import (
	"fmt"
	"log"
	"net/http"

	"golang.org/x/net/http2"
)

// http2 h2
func main() {
	http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "pong")
	})

	srv := &http.Server{
		Addr: ":5005",
	}
    
   // 下面这一行代码可以省略，因为net/http会自动进行协议协商，优先选择建立http2链接
	http2.ConfigureServer(srv, &http2.Server{})
	log.Fatal(srv.ListenAndServeTLS("/Users/wujinjing/projects/360/go/stack.crt", "/Users/wujinjing/projects/360/go/stack.key"))
}
```
&nbsp;&nbsp;&nbsp;&nbsp; 然后打开wireshark，监听本地的5005端口，使用curl请求server：
```
curl --http2 http://localhost:5005/ping -k
```
&nbsp;&nbsp;&nbsp;&nbsp; 发现抓到的包是这样的：

![](/img/http2-wireshark/w3.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 看不到http2连接的报文，协议也显示的是TLSv1.2，而不是http2。刚开始我一直以为是不是哪里有问题，server代码有问题或者client有问题，后来才突然想到，应用数据是被加密了，客户端和服务端两端在TLS上协商密钥加密通信，wireshark肯定就不知道TLS上应用层数据都是些什么，要想看到过程，必须要能解密流量。那么问题来了，wireshark怎么解密TLS协议之上的流量呢？

&nbsp;&nbsp;&nbsp;&nbsp; 然后又看了几篇文章（可以看看参考连接），根据TLS密钥协商的过程，wireshark想要解密TLS流量大概有两种办法，一种是拿到服务端的证书私钥，wireshark能直接用私钥解密出会话密钥，然后可以解密流量。一种是某些客户端支持把TLS握手过程中生成的密钥保存在外部文件中，可供wireshark解密使用。

#### 尝试第一种方式 ####
&nbsp;&nbsp;&nbsp;&nbsp; 先尝试用直接导入证书私钥的方式来解密流量，重复检查配置了好几次，发现抓到的包还是没办法解密。然后google了下，发现这种方式只对RSA算法有用，因为如果是通过 RSA 算法交换秘钥，那么客户端会加密并发送给服务端，这样只需要知道服务端的私钥就可以解密出 PreMasterSecret 值。但是目前比较主流的方式都是使用DH类算法交换密钥，而DH类算法中的 PreMasterSecret 是由服务端和客户端各自计算出来，而且没有保存到磁盘，没有通过网络传输，这样就导致无法进行解密。    

然后看了下Server Hello里服务端选择的Cipher Suite：`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`。刚好是ECDHE算法，这样的话，这种配置证书私钥的方式就失效了啊，怎么办呢，略加思索....要不指定Cipher Suite？指定成RSA方式来交换密钥不就可以了吗，然后就开始在curl的时候指定tls的Cipher Suite，结果试了好几个，发现server都不支持，tls链接建立不起来，握手都过不了，一拍脑子才想起来，curl指定的是Client HELLO的Cipher Suite，是提供给server端选择的，这样直接指定一个很可能server不支持。那要不在curl默认的Client Hello里的Cipher Suite列表里找到一个RSA密钥交换类的，然后在server端来指定这个Cipher Suite？于是又开始漫长的找go里tls源码的旅途，最后是找到了，但是折腾一番，指定了好几个，发现还是会有问题，可能这些rsa的cipher suites都在go的http2实现中都不支持了？没有细细探究，感觉此路不通了，换种方式吧

#### 尝试第二种方式 ####
&nbsp;&nbsp;&nbsp;&nbsp; 第二种方式是客户端把握手过程中生成的密钥保存在外部文件中供Wireshark解密使用，那最主要的便是客户端是否支持这种方式。亲测Chrom和Curl是支持的，首先在命令行`export SSLKEYLOGFILE=~/tls/sslkeylog.log`导入环境变量SSLKEYLOGFILE指定密钥文件的路径，然后再open Chrome，或者执行curl命令，都会自动把握手过程中的密钥都保存到这个文件中。接下来需要在wireshark里指定密钥文件：[Edit]->[Preference]->[Protocols]->[TSL] ，准备完毕就可以开始解密抓包了，至此终于能够查看http2的h2建立连接的过程了

![](/img/http2-wireshark/w4.jpg)

tips：
&nbsp;&nbsp;&nbsp;&nbsp; 补充下，最近还遇到另外一个问题，需要排查某个服务到另外一个服务的https链接问题，需要抓包查看包内容，客户端是go的代码，其实也是可以配置SSLKEYLOGFILE的，这样wireshark就可以解密抓到的https流量了

```
f, err := os.OpenFile("keylog.txt", os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0664)
if err != nil {
	log.Fatal(err)
}

client := http.Client{
	Transport: &http.Transport{
		TLSClientConfig: &tls.Config{
			KeyLogWriter:       f,
			InsecureSkipVerify: true,
		},
	},
}
```

#### 继续抓包 ####
&nbsp;&nbsp;&nbsp;&nbsp; 配置好了curl的`SSLKEYLOGFILE`环境变量和wireshark的相关配置，就可以观察到以下流量

![](/img/http2-wireshark/w5.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 可以看到抓到的包里出现了http2协议的包了，说明wireshark已经解密出了TLS上层流量，能识别上层协议了。再来看下h2协议协商的过程。在上文提到过，h2的协议协商是借助TLS的一个扩展协议，ALPN（Application Layer Protocol Negotiation，应用层协议协商）来进行的。从上图可以看到，首先是在TLS的Client Hello里，客户端在扩展里添加了ALPN的内容，并且在里面指定了上层协议使用h2

![](/img/http2-wireshark/w6.jpg)

&nbsp;&nbsp;&nbsp;&nbsp; 后面Server Hello里，服务端在扩展里添加了ALPN相关内容，确认了后续使用h2进行通信，到此为止借助tls本身的握手机制，h2的协议协商基本完成，后面就可以正常通信了



### 参考链接

+ [https://gohalo.me/post/decrypt-tls-ssl-with-wireshark.html](https://gohalo.me/post/decrypt-tls-ssl-with-wireshark.html)
+ [https://blog.wonter.net/posts/f288be00/](https://blog.wonter.net/posts/f288be00/)
+ [https://imququ.com/post/http2-traffic-in-wireshark.html](https://imququ.com/post/http2-traffic-in-wireshark.html)




