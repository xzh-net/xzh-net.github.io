# 网络协议详解

## 1. HTTP协议格式

### 1.1 请求格式

请求报文由请求行、请求头部、空行和请求包体4个部分组成

![](../../assets/_images/java//network/net/http1.jpg)

```html
POST /examples/default.jsp HTTP/1.1 
Accept: text/plain; text/html 
Accept-Language: en-gb 
Connection: Keep-Alive 
Host: localhost 
User-Agent: Mozilla/4.0 (compatible; MSIE 4.01; Windows 98)
Content-Length: 33 
Content-Type: application/x-www-form-urlencoded 

lastName=Franks&firstName=Michael  

```

请求行

请求行由方法字段、URL 字段 和HTTP 协议版本字段 3 个部分组成，他们之间使用空格隔开。常用的 HTTP 请求方法有 GET、POST、HEAD、PUT、DELETE、OPTIONS、TRACE、CONNECT

请求头

请求头部由关键字/值对组成，每行一对，关键字和值用英文冒号“:”分隔。请求头部通知服务器有关于客户端请求的信息，典型的请求头有：

```
Header	解释	示例
Accept	指定客户端能够接收的内容类型	Accept: text/plain, text/html
Accept-Charset	浏览器可以接受的字符编码集。	Accept-Charset: iso-8859-5
Accept-Encoding	指定浏览器可以支持的web服务器返回内容压缩编码类型。	Accept-Encoding: compress, gzip
Accept-Language	浏览器可接受的语言	Accept-Language: en,zh
Accept-Ranges	可以请求网页实体的一个或者多个子范围字段	Accept-Ranges: bytes
Authorization	HTTP授权的授权证书	Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
Cache-Control	指定请求和响应遵循的缓存机制	Cache-Control: no-cache
Connection	表示是否需要持久连接。（HTTP 1.1默认进行持久连接）	Connection: close
Cookie	HTTP请求发送时，会把保存在该请求域名下的所有cookie值一起发送给web服务器。	Cookie: $Version=1; Skin=new;
Content-Length	请求的内容长度	Content-Length: 348
Content-Type	请求的与实体对应的MIME信息	Content-Type: application/x-www-form-urlencoded
Date	请求发送的日期和时间	Date: Tue, 15 Nov 2010 08:12:31 GMT
Expect	请求的特定的服务器行为	Expect: 100-continue
From	发出请求的用户的Email	From: user@email.com
Host	指定请求的服务器的域名和端口号	Host: www.zcmhi.com
If-Match	只有请求内容与实体相匹配才有效	If-Match: “737060cd8c284d8af7ad3082f209582d”
If-Modified-Since	如果请求的部分在指定时间之后被修改则请求成功，未被修改则返回304代码	If-Modified-Since: Sat, 29 Oct 2010 19:43:31 GMT
If-None-Match	如果内容未改变返回304代码，参数为服务器先前发送的Etag，与服务器回应的Etag比较判断是否改变	If-None-Match: “737060cd8c284d8af7ad3082f209582d”
If-Range	如果实体未改变，服务器发送客户端丢失的部分，否则发送整个实体。参数也为Etag	If-Range: “737060cd8c284d8af7ad3082f209582d”
If-Unmodified-Since	只在实体在指定时间之后未被修改才请求成功	If-Unmodified-Since: Sat, 29 Oct 2010 19:43:31 GMT
Max-Forwards	限制信息通过代理和网关传送的时间	Max-Forwards: 10
Pragma	用来包含实现特定的指令	Pragma: no-cache
Proxy-Authorization	连接到代理的授权证书	Proxy-Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
Range	只请求实体的一部分，指定范围	Range: bytes=500-999
Referer	先前网页的地址，当前请求网页紧随其后,即来路	Referer: http://www.zcmhi.com/archives/71.html
TE	客户端愿意接受的传输编码，并通知服务器接受接受尾加头信息	TE: trailers,deflate;q=0.5
Upgrade	向服务器指定某种传输协议以便服务器进行转换（如果支持）	Upgrade: HTTP/2.0, SHTTP/1.3, IRC/6.9, RTA/x11
User-Agent	User-Agent的内容包含发出请求的用户信息	User-Agent: Mozilla/5.0 (Linux; X11)
Via	通知中间网关或代理服务器地址，通信协议	Via: 1.0 fred, 1.1 nowhere.com (Apache/1.1)
Warning	关于消息实体的警告信息	Warn: 199 Miscellaneous warning
```

空行

最后一个请求头之后是一个空行，发送回车符和换行符，通知服务器以下不再有请求头;

请求体

请求包体不在 GET 方法中使用，而是在POST 方法中使用。POST 方法适用于需要客户填写表单的场合。与请求包体相关的最常使用的是包体类型 Content-Type 和包体长度 Content-Length;



### 1.2 响应格式

响应报文由状态行、响应头部、空行和响应包体4个部分组成

![](../../assets/_images/java//network/net/http2.jpg)

```html
HTTP/1.1 200 OK 
Server: Microsoft-IIS/4.0 
Date: Mon, 5 Jan 2004 13:13:33 GMT 
Content-Type: text/html 
Last-Modified: Mon, 5 Jan 2004 13:13:12 GMT 
Content-Length: 112 

<html> 
    <head> 
        <title>HTTP Response Example</title> 
    </head> 
    <body> 
        Welcome to Brainy Software 
    </body> 
</html> 

```

状态行

状态行由 HTTP 协议版本字段、状态码和状态码的描述文本 3 个部分组成，他们之间使用空格隔开;

状态码由三位数字组成，第一位数字表示响应的类型，常用的状态码有五大类如下所示：
 - 1xx：表示服务器已接收了客户端请求，客户端可继续发送请求;
 - 2xx：表示服务器已成功接收到请求并进行处理;
 - 3xx：表示服务器要求客户端重定向;
 - 4xx：表示客户端的请求有非法内容;
 - 5xx：表示服务器未能正常处理客户端的请求而出现意外错误;

状态码描述文本有如下取值：
 - 200 OK：表示客户端请求成功;
 - 400 Bad Request：表示客户端请求有语法错误，不能被服务器所理解;
 - 401 Unauthonzed：表示请求未经授权，该状态代码必须与 WWW-Authenticate 报头域一起使用;
 - 403 Forbidden：表示服务器收到请求，但是拒绝提供服务，通常会在响应正文中给出不提供服务的原因;
 - 404 Not Found：请求的资源不存在，例如，输入了错误的URL;
 - 500 Internal Server Error：表示服务器发生不可预期的错误，导致无法完成客户端的请求;
 - 503 Service Unavailable：表示服务器当前不能够处理客户端的请求，在一段时间之后，服务器可能会恢复正常;

响应头部

```
Header	解释	示例
Accept-Ranges	表明服务器是否支持指定范围请求及哪种类型的分段请求	Accept-Ranges: bytes
Age	从原始服务器到代理缓存形成的估算时间（以秒计，非负）	Age: 12
Allow	对某网络资源的有效的请求行为，不允许则返回405	Allow: GET, HEAD
Cache-Control	告诉所有的缓存机制是否可以缓存及哪种类型	Cache-Control: no-cache
Content-Encoding	web服务器支持的返回内容压缩编码类型。	Content-Encoding: gzip
Content-Language	响应体的语言	Content-Language: en,zh
Content-Length	响应体的长度	Content-Length: 348
Content-Location	请求资源可替代的备用的另一地址	Content-Location: /index.htm
Content-MD5	返回资源的MD5校验值	Content-MD5: Q2hlY2sgSW50ZWdyaXR5IQ==
Content-Range	在整个返回体中本部分的字节位置	Content-Range: bytes 21010-47021/47022
Content-Type	返回内容的MIME类型	Content-Type: text/html; charset=utf-8
Date	原始服务器消息发出的时间	Date: Tue, 15 Nov 2010 08:12:31 GMT
ETag	请求变量的实体标签的当前值	ETag: “737060cd8c284d8af7ad3082f209582d”
Expires	响应过期的日期和时间	Expires: Thu, 01 Dec 2010 16:00:00 GMT
Last-Modified	请求资源的最后修改时间	Last-Modified: Tue, 15 Nov 2010 12:45:26 GMT
Location	用来重定向接收方到非请求URL的位置来完成请求或标识新的资源	Location: http://www.zcmhi.com/archives/94.html
Pragma	包括实现特定的指令，它可应用到响应链上的任何接收方	Pragma: no-cache
Proxy-Authenticate	它指出认证方案和可应用到代理的该URL上的参数	Proxy-Authenticate: Basic
refresh	应用于重定向或一个新的资源被创造，在5秒之后重定向（由网景提出，被大部分浏览器支持）	Refresh: 5; url=http://www.zcmhi.com/archives/94.html
Retry-After	如果实体暂时不可取，通知客户端在指定时间之后再次尝试	Retry-After: 120
Server	web服务器软件名称	Server: Apache/1.3.27 (Unix) (Red-Hat/Linux)
Set-Cookie	设置Http Cookie	Set-Cookie: UserID=JohnDoe; Max-Age=3600; Version=1
Trailer	指出头域在分块传输编码的尾部存在	Trailer: Max-Forwards
Transfer-Encoding	文件传输编码	Transfer-Encoding:chunked
Vary	告诉下游代理是使用缓存响应还是从原始服务器请求	Vary: *
Via	告知代理客户端响应是通过哪里发送的	Via: 1.0 fred, 1.1 nowhere.com (Apache/1.1)
Warning	警告实体可能存在的问题	Warning: 199 Miscellaneous warning
WWW-Authenticate	表明客户端请求实体应该使用的授权方案	WWW-Authenticate: Basic
```

空行

最后一个响应头部之后是一个空行，发送回车符和换行符，通知服务器以下不再有响应头部。

响应体

服务器返回给客户端的文本信息;


HTTP 工作原理

HTTP 协议采用请求/响应模型。客户端向服务器发送一个请求报文，服务器以一个状态作为响应。
 - 客户端连接到web服务器：HTTP 客户端与web服务器建立一个TCP连接;
 - 客户端向服务器发起 HTTP 请求：通过已建立的TCP连接，客户端向服务器发送一个请求报文;
 - 服务器接收HTTP请求并返回HTTP响应：服务器解析请求，定位请求资源，服务器将资源副本写到TCP连接，由客户端读取;
 - 释放TCP连接：若connection 模式为close，则服务器主动关闭TCP连接，客户端被动关闭连接，释放TCP连接;若connection模式为keepalive，则该连接会保持一段时间，在该时间内可以继续接收请求;
 - 客户端浏览器解析HTML内容：客户端将服务器响应的html文本解析并显示;

HTTP 无状态性

HTTP 协议是无状态的(stateless)。也就是说，同一个客户端第二次访问同一个服务器上的页面时，服务器无法知道这个客户端曾经访问过，服务器也无法分辨不同的客户端。HTTP的无状态特性简化了服务器的设计，使服务器更容易支持大量并发的HTTP 请求。

HTTP 持久连接

HTTP1.0 使用的是非持久连接，主要缺点是客户端必须为每一个待请求的对象建立并维护一个新的连接，即每请求一个文档就要有两倍RTT 的开销。因为同一个页面可能存在多个对象，所以非持久连接可能使一个页面的下载变得十分缓慢，而且这种短连接增加了传输的负担。HTTP1.1 使用持久连接keepalive，所谓持久连接，就是服务器在发送响应后仍然在一段时间内保持这条连接，允许在同一个连接中存在多次数据请求和响应，即在持久连接情况下，服务器在发送完响应后并不关闭TCP 连接，而客户端可以通过这个连接继续请求其他对象。

## 2. HTTP转发与重定向

1、首先转发是服务器的操作，而重定向是客户端的操作。

2、重定向时，客户端浏览器的地址栏有变化，而转发没有

3、转发可以携带数据，重定向不行

4、重定向发生在客户端!转发发生在服务端,客户端是不知道

5、重定向时，客户端发送了两个请求，而转发只发送了一个请求（本质区别）

## 3. HTTP的Cookie机制

Cookie是解决HTTP无状态性的有效手段，服务器可以设置或读取Cookie中所包含的信息。当用户登录后，服务器会发送包含登录凭据的Cookie到用户浏览器客户端

一个Cookie包含以下信息：

1. name    Cookie名称
2. value    Cookie值，Cookie值同理Cookie的名称，可以进行转码和加密。
3. domain    域，指定在该子域下才可以访问Cookie，例如要让Cookie在a.test.com下可以访问，但在b.test.com下不能访问，则可将domain设置成a.test.com。
4. Path    一个路径，在这个路径下面的页面才可以访问该Cookie，一般设为“/”，以表示同一个站点的所有页面都可以访问这个Cookie。
5. expires/Max-Age    过期日期，一个GMT格式的时间，当过了这个日期之后，浏览器就会将这个Cookie删除掉，当不设置这个的时候，Cookie在浏览器关闭后消失。
6. Size    cookie大小
7. Secure    安全性，指定Cookie是否只能通过https协议访问，一般的Cookie使用HTTP协议既可访问，如果设置了Secure（没有值），则只有当使用https协议连接时cookie才可以被页面访问。
8. HttpOnly    如果在Cookie中设置了"HttpOnly"属性，那么通过程序(JS脚本、Applet等)将无法读取到Cookie信息。

## 4. HTTP缓存控制和代理服务

浏览器缓存分为强缓存和协商缓存，浏览器加载一个页面的简单流程如下：

- 浏览器先根据这个资源的http头信息来判断是否命中强缓存。如果命中则直接加在缓存中的资源，并不会将请求发送到服务器。
- 如果未命中强缓存，则浏览器会将资源加载请求发送到服务器。服务器来判断浏览器本地缓存是否失效。若可以使用，则服务器并不会返回资源信息，浏览器继续从缓存加载资源。
- 如果未命中协商缓存，则服务器会将完整的资源返回给浏览器，浏览器加载新资源，并更新缓存。

web缓存器，也叫作代理服务器

web缓存代理服务器的工作方式：

- 用户A浏览器建立一条到该web缓存服务器中的TCP连接，并向web缓存服务器中的该对象发送一个HTTP请求。
- 该web缓存服务器先检查该对象在本地有没有副本。如果有，那么该web缓存服务器就用HTTP响应报文想客户机浏览器返回该对象。
- 如果该web缓存服务器没有该对象的副本，他就与该对象的初始服务器建立一条TCP连接，并发送一个该对象的HTTP请求。在收到该请求后初始web服务器向该web缓存服务器发送具有该对象的HTTP响应报文。
- 当该web缓存服务器收到该对象时，他在本地存储空间存储了一个副本，并用HTTP响应报文向用户A浏览器发送该副本。
- 如果其他用户的浏览器也恰好要访问该对象，那么久能迅速从web缓存服务器中得到它，并且不会增加网络流量。

HTTP代理分为正向代理、反向代理和透明代理；
 - 正向代理：服务器代理客户端去访问其他的服务器，这个服务器时存在于客户端和源始服务器之间，客户端隐藏与代理服务器的后面充当幕后主使。源始服务器所看到的是它在代理服务器服务器进行交互，而不知道客户端的存在。一般用在web缓存技术中。场景如下：
   - 访问原来无法访问的资源，如google
   - 可以做缓存，加速访问资源
   - 对客户端访问授权，上网进行认证
   - 代理可以记录用户访问记录（上网行为管理），对外隐藏用户信息
 - 反向代理：与正向代理相反，此时是源始服务器藏在代理服务器后面，客户端看到的是自己在与代理服务器交互，而不知道源始服务器的存在。可以隐藏保护源始服务器资源。在负载均衡中使用反向代理技术来减小服务器压力。在CDN（分布式集群管理）的核心技术中也用到了反向代理。
   - 保证内网的安全，阻止web攻击，大型网站，通常将反向代理作为公网访问地址，Web服务器是内网
   - 负载均衡，通过反向代理服务器来优化网站的负载
 - 透明代理：客户端根本不需要知道有代理服务器的存在它改变你的request fields（报文），并会传送真实IP，多用于路由器的NAT转发中。注意，加密的透明代理则是属于匿名代理，意思是不用设置使用代理了。当用户服务器提交请求的时候，透明代理设备根据自身策略拦截并修改用户的报文，并作为实际的请求方，向服务器送请求，当接收信息回传，透明代理再根据自身的设置把允许的报文发回至用户

>总结：正向代理即是客户端代理, 代理客户端, 服务端不知道实际发起请求的客户端.
>反向代理即是服务端代理, 代理服务端, 客户端不知道实际提供服务的服务端

## 5. HTTPS是什么？SSL/TLS又是什么？ 

## 6. 对称加密与非对称加密 、数字签名与证书 

## 7. OSI七层网络协议

OSI（Open System Interconnect），即开放式系统互联。 一般都叫OSI参考模型，是ISO（国际标准化组织）组织在1985年研究的网络互连模型。ISO为了更好的使网络应用更为普及，推出了OSI参考模型。其含义就是推荐所有公司使用这个规范来控制网络。这样所有公司都有相同的规范，就能互联了。
OSI定义了网络互连的七层框架（物理层、数据链路层、网络层、传输层、会话层、表示层、应用层），即ISO开放互连系统参考模型

![](../../assets/_images/java//network/net/osi.png)

OSI七层和TCP/IP四层的关系

OSI引入了服务、接口、协议、分层的概念，TCP/IP借鉴了OSI的这些概念建立TCP/IP模型。

OSI先有模型，后有协议，先有标准，后进行实践；而TCP/IP则相反，先有协议和应用再提出了模型，且是参照的OSI模型。

OSI是一种理论下的模型，而TCP/IP已被广泛使用，成为网络互联事实上的标准。
 - TCP：transmission control protocol 传输控制协议
 - UDP：user data protocol 用户数据报协议

![](../../assets/_images/java//network/net/osi7.png)

OSI七层和TCP/IP的区别

 - TCP/IP他是一个协议簇；而OSI（开放系统互联）则是一个模型，且TCP/IP的开发时间在OSI之前。
 - TCP/IP是由一些交互性的模块做成的分层次的协议，其中每个模块提供特定的功能；OSi则指定了哪个功能是属于哪一层的。
 - TCP/IP是五层结构，而OSI是七层结构。OSI的最高三层在TCP中用应用层表示。

![](../../assets/_images/java//network/net/osiall.png)

## 8. TCP协议详解

传输控制协议（TCP，Transmission Control Protocol）是一种面向连接的、可靠的、基于字节流的传输层通信协议


## 9. TCP协议的流量控制详解

零窗口探测报文

## 10. TCP协议的可靠性是如何保障的

数据链路层流量控制技术，目标：提高信道利用率

停止-等待协议SW
 - 每发完一个数据帧就停止发送，等待对方确认
 - 收到确认后再发送下一个数据帧
 - 每发送完一个数据帧就启动一个相应的重传计时器，当计时器超时却未收到确认时就重传该数据帧

这种可靠性传输协议被称为`自动重传请求ARQ（Aytomatic Repeat reQuest）`

ARQ表明重传的请求式发送方自动进行的，接收方不需要请求方重传某个出错的数据帧

后退N帧协议GBN
 - 基于互动窗口的流量控制技术
 - 发送方可在未收到接收方确认帧的情况下，将序号落在发送窗口内的多个数据帧全部发送出去，接收方必须具备累计确认和捎带确认
 - 发生方只有收到确认帧时，发送窗口才能向前滑动
 - 当发送窗口内某个已经发送的数据帧出现超时重发时，其后续在发送窗口内已经发送的数据帧必须全部重传，这是后退N帧协议名称的由来

选择重传协议SR
 - 接收方有差错处理策略，一旦收到误码数据帧或者怀疑数据帧立即发送相应的否定确认帧`NAK`,不必等到计时器超
 - 发送发只重传出错的数据帧或者超时的数据帧

## 11. TCP协议可靠传输的实现

以字节为单位的时间窗口
 - 虽然发送方的发送窗口式根据接收方的接收窗口设置的，但在同一时刻发送方的窗口并不总是和接收方的接收窗口一样大
 - 对于不按序到达的数据如何处理，TCP并无明确要求，先到先入缓存到齐后一并提交应用进程
 - TCP接收方必须有积累确认和捎带确认机制，但不硬过分推迟发送确认，会导致不必要的重传
 - 全双工同通信，通信中的每一方都再发送和接受报文段，每一方都有自己的发送窗口和接收窗口

## 12. TCP协议与Socket的联系
- HTTP是应用层协议，定义的是传输数据的内容以及格式的规范
- TCP是底层通讯协议，定义的是数据传输和连接方式的规范
- Socket可以支持不同的传输层协议（TCP或UDP），当使用TCP协议进行连接时，该Socket连接就是一个TCP连接，Socket是发动机，提供了网络通信的能力

我们在传输数据时，可以只使用（传输层）TCP/IP协议，但是那样的话，如果没有应用层，便无法识别数据内容，如果想要使传输的数据有意义，则必须使用到应用层协议，应用层协议有很多，比如HTTP、FTP、TELNET等，也可以自己定义应用层协议。WEB使用HTTP协议作应用层协议，以封装HTTP文本信息，然后使用TCP/IP做传输层协议将它发到网络上

TCP/IP只是一个协议栈，就像操作系统的运行机制一样，必须要具体实现，同时还要提供对外的操作接口。这个就像操作系统会提供标准的编程接口，比如win32编程接口一样，TCP/IP也要提供可供程序员做网络开发所用的接口，这就是Socket编程接口

## 12. MQTT协议

MQTT（Message Queuing Telemetry Transport，消息队列遥测传输协议），是一种基于发布/订阅模式的"轻量级"通讯协议，该协议构建于TCP/IP协议上，由IBM在1999年发布

在MQTT协议中，一个MQTT数据包由：固定头（Fixed header）、可变头（Variable header）、消息体（payload）三部分构成。MQTT数据包结构如下：

1. 固定头（Fixed header）。存在于所有MQTT数据包中，表示数据包类型及数据包的分组类标识，如连接，发布，订阅，心跳等。其中固定头是必须的，所有类型的MQTT协议中，都必须包含固定头。
2. 可变头（Variable header）。存在于部分MQTT数据包中，数据包类型决定了可变头是否存在及其具体内容。可变头部不是可选的意思，而是指这部分在有些协议类型中存在，在有些协议中不存在。
3. 消息体（Payload）。存在于部分MQTT数据包中，表示客户端收到的具体内容。 与可变头一样，在有些协议类型中有消息内容，有些协议类型中没有消息内容。


| 报文类型 | 字段值 |数据方向 |描述 |7-4bit值 |
| ----- | ----- | ----- |----- |----- |
| 保留|  0 | 禁用 | 保留 | 0000 |
| CONNECT|  1 | Client--->Server|  客户端连接到服务器| 0001 |
| CONNACK | 2 | Server ---> Client | 连接确认|  0010 |
| PUBLISH | 3 | Client <--> Server | 发布消息|  0011 |
| PUBACK | 4 | Client <--> Server | 发布确认(QoS1)|  0100 |
| PUBREC | 5 | Client <--> Server | 消息已接收(Qos2 第一阶段)|  0101 |
| PUBREL | 6 | Client <--> Server | 消息释放(QoS2 第二阶段)|  0110 |
| PUBCOMP | 7 | Client <--> Server | 发布结束(QoS2 第三阶段)|  0111 |
| SUBSCRIBE | 8 | Client ---> Server | 客户端订阅请求|  1000 |
| SUBACK | 9 | Server--->Client | 服务端订阅确认|  1001 |
| UNSUBACRIBE | 10 | Client ---> Server | 客户端取消订阅|  1010 |
| UNSUBACK | 11 | Server ---> Client | 服务端取消订阅确认|  1011 |
| PINGREQ | 12 | Client--->Server | 客户端发送心跳|  1100 |
| PINGRESP | 13 | Server ---> Client | 服务端回复心跳|  1101 |
| DISCONNECT | 14 | Client--->Server | 客户端断开连接请求|  1110 |
| AUTH（V5.0使用） | 15 | Client <--> Server | 认证数据交换|  1111 |

![](../../assets/_images/java//network/net/mqtt2.png)

| 报文类型 | 是否需要payload |
| ----- | ----- |
| CONNECT | Required |
| CONNACK | None |
| PUBLISH | Optional |
| PUBACK | None |
| PUBREC | None |
| PUBREL | None |
| PUBCOMP | None |
| SUBSCRIBE | Required |
| SUBACK | Required |
| UNSUBSCRIBE | Required |
| UNSUBACK | Required |
| PINGREQ | None |
| PINGRESP | None |
| DISCONNECT | None |


MQTT消息QoS

- Qos0消息发布订阅
![](../../assets/_images/java//network/net/mqtt_qos0.png)

- Qos1消息发布订阅
![](../../assets/_images/java//network/net/mqtt_qos1.png)

- Qos2消息发布订阅
![](../../assets/_images/java//network/net/mqtt_qos2.png)

选型：
- Eclipse Mosquitto
- EMQX
- RabbitMq
- ActiveMq
- Mosca
- VerneMQ


`AMQP和MQTT之间的关键区别`

1. MQTT具有客户机/代理体系结构，而AMQP具有客户机或代理以及客户机或服务器体系结构。
2. MQTT遵循发布和订阅的抽象，而AMQP遵循响应或请求以及发布或订阅方法。
3. AMQP的头大小为8bytes，MQTT的头大小为2bytes。MQTT的消息大小很小且已定义，而AMQP具有可协商的和未定义的。
4. MQTT的方法是connected、publish、close、subscribe和disconnect。AMQP遵循Consume, deliver, publish, get, select, acknowledge, delete, recover, reject, open, and close.
5. MQTT部分支持缓存和代理，而AMQP则提供完全支持。
6. AMQP和MQTT都遵循TCP协议、二进制标准和开源队列系统。
7. AMQP提供的安全性是IPSec、SASL、TLS或SSL，而MQTT只提供TLS或SSL安全标准。AMQP和TCP一起使用SCTP进行传输。OASIS同时支持AMQP和MQTT。
8. MQTT提供的服务质量是fire和forget，如果QoS为0。如果QoS为1，则至少有一个，如果QoS为2，则正好有一个。AMQP提供的服务质量是与MQTT类似的结算和取消结算格式。


![](../../assets/_images/java//network/net/amqp-vs-mqtt.png)