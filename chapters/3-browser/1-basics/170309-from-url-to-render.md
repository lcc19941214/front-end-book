# 从输入URL到页面加载完成的过程中都发生了什么事情？

## 一、 浏览器处理用户输入

### 1.1 识别URL

完整的URL包括以下几个部分，其中有的必须的，有的是可选的：

- **协议**（e.g. `http`，`ftp`）
- **域名**（e.g. `www.gogole.com`）
- **端口**（e.g. `:80`）
- **文件路径**（e.g. `/htm_data/20/1510/1441477.htm`）
- **参数**（e.g. `?key1=value1&key2=value2`）
- **锚点**（e.g. `#first-paragraph`）

浏览器会根据识别出的协议发送请求。

当协议或主机名不合法时，浏览器会将地址栏中输入的文字传给**默认的搜索引擎**（即把用户输入看作是**关键字**）。大部分情况下，在把文字传递给搜索引擎的时候，URL会带有特定的一串字符，用来告诉搜索引擎这次搜索来自这个特定浏览器。



### 1.2 检查HSTS Preload List

HSTS的全称是**HTTP Strict Transport Security**（**HTTP严格传输安全**），它是一个Web安全策略机制（web security policy mechanism）。

在使用HTTP发送请求前，浏览器会检查自带的HSTS列表，这个列表里包含了那些请求浏览器只使用HTTPS进行连接的网站，如果网站在这个列表里，浏览器会使用 HTTPS 而不是 HTTP 协议，否则，最初的请求会使用HTTP协议发送

要申请将自己的站点加入到HSTS当中，可以访问[https://hstspreload.org/](https://hstspreload.org/)



> 参考
>
> - [什么是URL？](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)
> - [HTTP Strict Transport Security](https://developer.mozilla.org/zh-CN/docs/Security/HTTP_Strict_Transport_Security)
> - [HSTS详解](http://www.jianshu.com/p/caa80c7ad45c)



##二、 浏览器发送请求

在发送请求前，浏览器首先检查Web缓存，然后调用网络请求的方法。



###2.1 DNS解析

####读取本地DNS缓存

浏览器检查域名是否在缓存当中。如果使用本地缓存，则无DNS 查询，使用持久连接也是。

要查看Chrome中的DNS缓存，可以打开`chrome://net-internals/#dns`。



#### 读取本地hosts文件

**hosts文件**文件负责将主机名称映射到相应的IP地址，通常用于补充或替换DNS的功能。用户可以直接对hosts文件进行控制。

要查看Linux中的hosts文件，可以访问`/etc/hosts`。



#### 发送DNS解析请求

1. 以`www.google.com`为例，首先，客户端会向**本地域名服务器**发送DNS查询报文
2. 本地域名转发请求给**根域名服务器**，后者会解析根域名`com`，并返回对应**顶级域名服务器**的IP地址给前者
3. 本地域名服务器访问负责**com域名服务器**，后者会解析顶级域名`google`，然后返回对应**权威域名服务器**的IP地址给前者
4. 本地余名服务器从负责解析`google`的权威域名服务器处获取到`www.google.com`的IP地址，返回给客户端


DNS的解析是一个逐步缩小范围的查找过程。



### 2.2 建立连接

有了 IP 地址，就可以通过选择 TCP 或 UDP 协议来发送数据了。

这里需要补充一点的是，光靠 IP 地址是无法进行通信的，因为 IP 地址并不和某台设备绑定（比如使用了[DHCP](https://zh.wikipedia.org/zh-cn/%E5%8A%A8%E6%80%81%E4%B8%BB%E6%9C%BA%E8%AE%BE%E7%BD%AE%E5%8D%8F%E8%AE%AE)）。所以在底层通信时需要使用一个固定的地址，即结合MAC协议来建立连接。



#### 建立TCP连接

![TCP三次握手](/assets/images/2018-07-20-17-38-49.png)

第一次握手:

- 客户端发送一个SYN标志位为1的包，指明客户打算连接的服务器的端口
- 设置初始序号ISN(即 J)，保存在包头的序列号(Sequence Number)字段里​

第二次握手:

- 服务器发回确认包(ACK)应答，即SYN标志位和ACK标志位均为1
- 设置序列号ISN（即K）
- 将确认序号ack设置为客户的ISN+1（即J+1）

第三次握手：

- 客户端再次发送确认包(ACK) SYN标志位为0，ACK标志位为1
- 把服务器发来ACK的序号字段+1（即K+1），放在确定字段ack中发送给对方
- 设置发送序号ISN（即J+1）


TCP的关键在于双方都需要确认自己的发信和收信功能正常，收信功能通过接收对方信息得到确认，发信功能需要发出信息—>对方回复信息得到确认。

要了解TCP如何断开连接，可以查看[TCP三次握手四次挥手详解](http://www.cnblogs.com/zmlctt/p/3690998.html)。



#### TLS握手

- 客户端发送一个 `Client hello` 消息到服务器端，消息中同时包含了它的TLS版本，可用的加密算法和压缩算法。

- 服务器端向客户端返回一个 `Server hello` 消息，消息中包含了服务器端的TLS版本，服务器选择了哪个加密和压缩算法，以及服务器的公开证书，证书中包含了公钥。客户端会使用这个公钥加密接下来的握手过程，直到协商生成一个新的对称密钥

- 客户端根据自己的信任CA列表，验证服务器端的证书是否有效。如果有效，客户端会生成一串伪随机数，使用服务器的公钥加密它。这串随机数会被用于生成新的对称密钥

- 服务器端使用自己的私钥解密上面提到的随机数，然后使用这串随机数生成自己的对称主密钥

- 客户端发送一个 `Finished` 消息给服务器端，使用对称密钥加密这次通讯的一个散列值

- 服务器端生成自己的 hash 值，然后解密客户端发送来的信息，检查这两个值是否对应。如果对应，就向客户端发送一个 `Finished` 消息，也使用协商好的对称密钥加密

- 从现在开始，接下来整个 TLS 会话都使用对称秘钥进行加密，传输应用层（HTTP）内容

  ​
#### SSL握手

  


### 2.3 发送请求

TCP连接建立后，浏览器就可以利用HTTP／HTTPS协议向服务器发送请求了。

#### HTTP协议

协议相关的内容这里不做展开介绍，要了解HTTP的相关内容可以参考MDN中[HTTP](https://developer.mozilla.org/zh-CN/docs/Web/HTTP)的相关文档。



#### Web缓存

Web缓存是指一个Web资源（如html页面，图片，js，数据等）存在于Web服务器和客户端（浏览器）之间的副本。缓存会根据进来的请求保存输出内容的副本。

**当下一个请求来到的时候，如果是相同的URL，缓存会根据缓存机制决定是直接使用副本响应访问请求，还是向源服务器再次发送请求。**比较常见的就是浏览器会缓存访问过网站的网页，当再次访问这个URL地址的时候，如果网页没有更新，就不会再次下载网页，而是直接使用本地缓存的网页。只有当网站明确标识资源已经更新，浏览器才会再次下载网页。



##### 浏览器端的缓存规则

对于浏览器端的缓存来讲，他们分别从**新鲜度**和**校验值**两个维度来规定浏览器是否可以直接使用缓存中的副本，还是需要去源服务器获取更新的版本。

**新鲜度（过期机制）**：也就是缓存副本有效期。一个缓存副本必须满足以下条件，浏览器会认为它是有效的，足够新的：

1. 含有完整的过期时间控制头信息（HTTP协议报头），并且仍在有效期内；
2. 浏览器已经使用过这个缓存副本，并且在一个会话中已经检查过新鲜度；

满足以上两个情况的一种，浏览器会直接从缓存中获取副本并渲染。

**校验值（验证机制）：**服务器返回资源的时候有时在控制头信息带上这个资源的实体标签Etag（Entity Tag），它可以用来作为浏览器再次请求过程的校验标识。如过发现校验标识不匹配，说明资源已经被修改或过期，浏览器需求重新获取资源内容。

在HTTP请求和响应的消息报头中，常见的与缓存有关的消息报头有：

![HTTP请求和响应的消息报头中，常见的与缓存有关的消息报头](/assets/images/2018-07-20-17-39-49.png)



##### 用户操作行为与缓存

用户在使用浏览器的时候，会有各种操作，这些行为会对缓存如下：

![用户操作与缓存](/assets/images/2018-07-20-17-40-07.png)

一般情况下，同时配置了Cache-Control/Expires和Last-Modified/ETag，前者的优先级要高于后者。但是当用户在按F5进行刷新的时候，浏览器会忽略Expires/Cache-Control的设置，再次发送请求去服务器请求，而Last-Modified/Etag还是有效的，服务器会根据情况判断返回304还是200；而当用户使用Ctrl+F5进行强制刷新的时候，只是所有的缓存机制都将失效，重新从服务器拉去资源。



> 参考
>
> - [hosts文件](https://zh.wikipedia.org/wiki/Hosts%E6%96%87%E4%BB%B6)
> - [在浏览器地址栏输入一个URL后回车，背后会进行哪些技术步骤？（@一骑讨、@何伟鹏关于DNS解析的回答）](https://www.zhihu.com/question/34873227) 
> - [TCP三次握手与四次挥手](http://www.jianshu.com/p/8a40f99da882)
> - [【Web缓存机制系列】2 – Web浏览器的缓存机制（by AlloyTeam）](http://www.alloyteam.com/2012/03/web-cache-2-browser-cache/)



## 三、 服务端处理响应

### 3.1 负载均衡

请求在进入到真正的应用服务器前，可能还会先经过负责负载均衡的机器，它的作用是将请求合理地分配到多个服务器上，同时具备具备防攻击等功能。

负载均衡具体实现有很多种，有直接基于硬件的 F5，有操作系统传输层(TCP)上的 [LVS](http://www.linuxvirtualserver.org/)，也有在应用层(HTTP)实现的反向代理（也叫七层代理），下面简单介绍一下。

#### LVS

LVS 的作用是从对外看来只有一个 IP，而实际上这个 IP 后面对应是多台机器，因此也被成为 Virtual IP。

前面提到的 NAT 也是一种 LVS 中的工作模式，除此之外还有 DR 和 TUNNEL，具体细节这里就不展开了，它们的缺点是无法跨网段，所以百度自己开发了 BVS 系统。



#### 反向代理

反向代理是工作在 HTTP 上的，具体实现可以基于 HAProxy 或 Nginx，因为反向代理能理解 HTTP 协议，所以能做非常多的事情，比如：

- 进行很多统一处理，比如防攻击策略、防抓取、SSL、gzip、自动性能优化等
- 应用层的分流策略都能在这里做，比如对 /xx 路径的请求分到 a 服务器，对 /yy 路径的请求分到 b 服务器，或者按照 cookie 进行小流量测试等
- 缓存，并在后端服务挂掉的时候显示友好的 404 页面
- 监控后端服务是否异常

  ​

### 3.2 Web Server

请求经过前面的负载均衡后，将进入到对应服务器上的 Web Server／HTTP Daemon，比如 Apache、Tomcat、Node.JS 等。

以 Apache 为例，在接收到请求后会交给一个独立的进程来处理，可以通过编写 Apache 或调用 PHP 等脚本语言来进行处理。一般网站都会基于某个 Web 框架来开发，因此在后端语言执行时首先进入 Web 框架的代码，然后由框架再去调用应用的实现代码。

大致过程如下：

- HTTPD 接收请求

- 服务器把请求拆分为以下几个参数：
- HTTP 请求方法(`GET`, `POST`, `HEAD`, `PUT`, `DELETE`, `CONNECT`, `OPTIONS`, 或者 `TRACE`)。直接在地址栏中输入 URL 这种情况下，使用的是 GET 方法
- 域名：google.com
- 请求路径/页面：/ (我们没有请求google.com下的指定的页面，因此 / 是默认的路径)

- 服务器验证其上已经配置了 google.com 的虚拟主机

- 服务器验证 google.com 接受 GET 方法

- 服务器验证该用户可以使用 GET 方法(根据 IP 地址，身份信息等)

- 如果服务器安装了 URL 重写模块（例如 Apache 的 mod_rewrite 和 IIS 的 URL Rewrite），服务器会尝试匹配重写规则，如果匹配上的话，服务器会按照规则重写这个请求

- 服务器根据请求信息获取相应的响应内容，这种情况下由于访问路径是 "/" ,会访问首页文件（你可以重写这个规则，但是这个是最常用的）。

- 服务器会使用指定的处理程序分析处理这个文件，假如 Google 使用 PHP，服务器会使用 PHP 解析 index 文件，并捕获输出，把 PHP 的输出结果返回给请求者

Web Server还会涉及到数据的读写，这部分就不做展开说明了。




> 参考
>
> - [反向代理为何叫反向代理？](https://www.zhihu.com/question/24723688)



## 四、浏览器接收并处理响应

浏览器的功能是从服务器上取回请求的资源，然后展示在浏览器窗口当中。资源通常是 HTML 文件，也可能是 PDF，图片，或者其他类型的内容。下面以请求一个网页为例进行介绍。

![performance api对应的资源请求并加载的过程](/assets/images/2018-07-20-17-41-14.png)
### 4.1 浏览器接收响应

#### 内容编码

浏览器发送请求时，会通过 Accept-Encoding 带上自己支持的内容编码格式列表；服务端从中挑选一种用来对正文进行编码，并通过 Content-Encoding 响应头指明选定的格式。

浏览器拿到响应正文后， Content-Encoding 进行解压。当然，服务端也可以返回未压缩的正文，但这种情况不允许返回 Content-Encoding。这个过程就是 HTTP 的内容编码机制。

在 HTTP/1 中，头部始终是以 ASCII 文本传输，没有经过任何压缩，因此内容编码仅仅是针对请求的正文部分。这个问题[在 HTTP/2 中得以解决](https://imququ.com/post/header-compression-in-http2.html)。

要了解HTTP内容编码，可以参考[HTTP压缩，浏览器是如何解析的](http://caibaojian.com/http-gzip.html)。

***注：不要把内容编码（Conent-Encoding）与传输编码（[Transfer-Encoding](https://imququ.com/post/transfer-encoding-header-in-http.html)）混淆了***



### 4.2 浏览器基本结构

在说明浏览器如何解析资源之前，先对浏览器的基本组件做一个简单的介绍：

- **用户界面** 
- **浏览器引擎**  在用户界面和呈现引擎之间传送指令
  - **[呈现引擎](https://zh.wikipedia.org/wiki/%E6%8E%92%E7%89%88%E5%BC%95%E6%93%8E)**  也称为浏览器内核、排版引擎。负责渲染请求的内容，以HTML为例，呈现引擎会解析HTML和CSS并显示到屏幕上。
  - **[JS引擎](https://zh.wikipedia.org/wiki/JavaScript%E5%BC%95%E6%93%8E)**  用于解析和执行 JavaScript 代码
- **网络组件 ** 用于网络调用，比如 HTTP 请求。其接口与平台无关，并为所有平台提供底层实现
- **用户界面后端**  用于绘制基本的窗口小部件，比如组合框和窗口。其公开了与平台无关的通用接口，而在底层使用操作系统的用户界面方法
- **数据存储**  浏览器需要在硬盘上保存各种数据，例如 Cookie。新的 HTML 规范 (HTML5) 定义了“网络数据库”，这是一个完整（但是轻便）的浏览器内数据库

![](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/layers.png)



### 4.3 解析和DOM树构建

#### 主流程

当服务器响应资源之后（HTML，CSS，JS，图片等），呈现引擎会从网络组件获取请求文档的内容，内容的大小一般限制在 8000 个块（8kb）以内。接着浏览器会执行以下操作：

- 解析 —— HTML，CSS，JS

- 渲染 —— 构建 DOM 树 -> 构建呈现树 -> 布局 -> 绘制

  ​

主流程如图所示：

![](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/flow.png)



呈现引擎将解析 HTML 文档，并将各标记逐个转化成DOM tree上的 DOM 节点。同时也会解析外部 CSS 文件以及样式元素中的样式数据。HTML 中这些带有视觉指令的样式信息将用于创建呈现树（render tree）。

呈现树包含多个带有视觉属性（如颜色和尺寸）的矩形。这些矩形的排列顺序就是它们将在屏幕上显示的顺序。

呈现树构建完毕之后，进入“布局”处理阶段，也就是为每个节点分配一个应出现在屏幕上的确切坐标。下一个阶段是绘制 - 呈现引擎会遍历呈现树，由用户界面后端层将每个节点绘制出来。

需要着重指出的是，这是一个渐进的过程。为达到更好的用户体验，呈现引擎会力求尽快将内容显示在屏幕上。它不必等到整个 HTML 文档解析完毕之后，就会开始构建呈现树和设置布局。在不断接收和处理来自网络的其余内容的同时，呈现引擎会将部分内容解析并显示出来。所以，会看到这种情况：网页的HTML代码还没下载完，但浏览器已经显示出内容了。



##### 呈现引擎的线程

呈现引擎采用了单线程。几乎所有操作（除了网络操作）都是在单线程中进行的。在 Firefox 和 Safari 中，该线程就是浏览器的主线程。而在 Chrome 浏览器中，该线程是标签进程的主线程。 
网络操作可由多个并行线程执行。并行连接数是有限的（通常为 2 至 6 个，以 Firefox 3 为例是 6 个）。



#### 解析

解析的过程可以分成两个子过程：词法分析和语法分析。解析器通常将解析工作分给以下两个组件来处理：**词法分析器**（有时也称为标记生成器），负责将输入内容分解成一个个有效标记；而**语法解析器**负责根据语言的语法规则分析文档的结构，从而构建解析树。



##### HTML解析

HTML解析起输出的结果是由 DOM 元素和属性节点构成的树结构。DOM 是文档对象模型 (Document Object Model) 的缩写。它是 HTML 文档的对象表示，同时也是外部内容（例如 JavaScript）与 HTML 元素之间的接口。 解析树的根节点是“Document”对象。

DOM 与标记之间几乎是一一对应的关系。

###### 解析算法

[在 HTML5 规范中查看标记化和树构建的完整算法](http://www.w3.org/TR/html5/syntax.html#html-parser)

###### 解析结束之后

在此阶段，浏览器会将文档标注为交互状态（interactive），并开始解析那些处于“deferred”模式的脚本，也就是那些应在文档解析完成后才执行的脚本。然后，文档状态将设置为complete，onload事件将随之触发。

要注意文档解析始终是在呈现引擎主线程中进行，解释并执行脚本会在另外的线程中进行。



##### CSS解析

[在CSS 规范中查看 CSS 的词法和语法](http://www.w3.org/TR/CSS2/grammar.html)

每个CSS文件都被解析成一个样式表对象，这个对象里包含了带有选择器的CSS规则，和对应CSS语法的对象（CSSOM）

##### 脚本和样式表的加载顺序

网络的模型是同步的。一般来说解析器遇到 `<script>` 标记时会立即解析并执行脚本。文档的解析将停止，直到脚本执行完毕。具体步骤如下：

1. 浏览器一边下载HTML网页，一边开始解析
2. 解析过程中，发现`<script>`标签
3. 暂停解析，网页渲染的控制权转交给js引擎
4. 如果`<script>`标签引用了外部脚本，就下载该脚本，否则就直接执行
5. 执行完毕，控制权交还渲染引擎，恢复往下解析HTML网页

加载外部脚本时声明`defer`属性时，脚本会在文档解析完成后执行。而在`async`属性时，脚本会异步被异步执行。

（两者的下载过程不会被延迟）

![](/assets/images/2018-07-20-17-42-11.png)

##### 预解析

[在MDN中了解什么是页面预解析以及如何对预解析进行优化](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Optimizing_your_pages_for_speculative_parsing)

在传统的浏览器中，HTML 解析器运行于主线程之中，并且在遇到 </script> 标签后会被阻塞，直到脚本从网络中被获取和执行。页面预解析技术是指，当脚本在获取和执行的过程中，浏览器支持提前解析HTML文档。



##### JS解析



### 4.4 呈现树的构建

在 DOM 树构建的同时，浏览器还会构建另一个树结构：呈现树（Render tree／Frame tree）。这是文档的可视化表示，个人认为呈现树即是一个满足可视化要求的DOM树的子集。

Firefox 将呈现树中的元素称为“框架”。WebKit 使用的术语是呈现器或呈现对象。 
呈现器知道如何布局并将自身及其子元素绘制出来。 

##### 呈现树和DOM树

呈现器是和 DOM 元素相对应的，但并非一一对应。非可视化的 DOM 元素不会插入呈现树中，例如“head”元素。如果元素的 display 属性值为“none”，那么也不会显示在呈现树中（但是 visibility 属性值为“hidden”的元素仍会显示）。

##### 构建呈现树的流程

##### 样式计算

样式的计算只包括呈现器对应的DOM元素应该采用哪一条CSS Rule，即层叠的规则计算。



### 4.5 布局与绘制

##### 布局

呈现器在创建完成并添加到呈现树时，并不包含位置和大小信息。计算这些值的过程称为布局（layout）或重排（reflow）。

HTML 采用基于流的布局模型，这意味着大多数情况下只要一次遍历就能计算出几何信息。处于流中靠后位置元素通常不会影响靠前位置元素的几何特征，因此布局可以按从左至右、从上至下的顺序遍历文档。但是也有例外情况，比如 HTML 表格，以及采用了`float`，`absolute`，`relative`等属性的元素，在计算时就需要不止一次的遍历 。

坐标系是相对于根框架而建立的，使用的是上坐标和左坐标。

布局是一个递归的过程。它从根呈现器（对应于 HTML 文档的 `<html>` 元素）开始，然后递归遍历部分或所有的框架层次结构，为每一个需要计算的呈现器计算几何信息。

###### 宽度计算

###### 换行

##### 绘制

- 创建layer(层)来表示页面中的哪些部分可以成组的被绘制，而不用被重新栅格化处理。每个帧对象都被分配给一个层
- 页面上的每个层都被分配了纹理(?)
- 每个层的帧对象都会被遍历，计算机执行绘图命令绘制各个层，此过程可能由CPU执行栅格化处理，或者直接通过D2D/SkiaGL在GPU上绘制
- 上面所有步骤都可能利用到最近一次页面渲染时计算出来的各个值，这样可以减少不少计算量
- 计算出各个层的最终位置，一组命令由 Direct3D/OpenGL发出，GPU命令缓冲区清空，命令传至GPU并异步渲染，帧被送到Window Server。

---

## 参考
> - [HTTP 协议中的 Content-Encoding](https://imququ.com/post/content-encoding-header-in-http.html)
>
> - [How Browsers Work: Behind the scenes of modern web browsers](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)
>
> - [当...时发生了什么](https://github.com/skyline75489/what-happens-when-zh_CN/blob/master/README.rst)
>
> - [从输入 URL 到页面加载完成的过程中都发生了什么事情？（百度FEXmwind的博客）](http://fex.baidu.com/blog/2014/05/what-happen/)
>
> - [javascript标准参考教程-浏览器环境概述 阮一峰](http://javascript.ruanyifeng.com/bom/engine.html#toc11)
>
> - [前端知识普及之页面加载](https://segmentfault.com/a/1190000004466407)
