# 网络是如何工作的

> 译者: 本文在 google 翻译的基础上进行意译, 并保留了一部分专有名词的英文方便查询其解释
> 2023.3.18

当我们在浏览器中输入 google.com 时，幕后发生了什么？

**目录**

- ["google"的“g”键被按下](#"google" 的“g”键被按下)
- [当你按下“Enter”键时](#当你按下“Enter”键时)
- [解析 URL](#解析 URL)
- [检查 HSTS 列表（已弃用）](#检查 HSTS 列表（已弃用）)
- [DNS 查询](#DNS 查询)
- [打开套接字(socket) + TLS 握手](#打开套接字(socket) + TLS 握手)
- [HTTP 协议](#HTTP 协议)
- [HTTP 服务器请求句柄(handle)](#HTTP 服务器请求句柄(handle))
- [服务器响应](#服务器响应)
- [浏览器的幕后](#浏览器的幕后)
- [浏览器的高层结构](#浏览器的高层结构)
- [渲染(rendering)引擎](#渲染(rendering)引擎)
- [主要流程](#主要流程)
- [解析(parsing)基础](#解析(parsing)基础)
- [DOM 树](#DOM 树)
   - [为什么 DOM 很慢？](#为什么 DOM 很慢？)
- [渲染树](#渲染树)
- [渲染树与 DOM 树的关系](#渲染树与 DOM 树的关系)
- [CSS 解析](#CSS 解析)
- [布局(layout)](#布局(layout))
- [绘图(painting)](#绘图(painting))
- [冷知识(trivia)](#冷知识(trivia))
   - [网络的诞生](#网络的诞生)

## "google" 的“g”键被按下

当你按下“g”时，浏览器就会收到事件，整个自动完成机制就会启动。 根据您浏览器的算法以及您是否处于隐私/隐身模式，URL 栏下方的下拉框中会显示各种建议。 大多数这些算法根据搜索历史和书签对结果进行优先排序。 您将输入“google.com”，所以这些都无关紧要，但是在您到这一步之前会运行很多代码，并且每次按键都会改变显示的建议。 它甚至可能会在您键入之前显示“google.com”。

## 当你按下“Enter”键时

要选择一个出发点，让我们选择键盘上的 Enter 键按下时。 此时，回车键的电路闭合（直接闭合或电容闭合）。 这允许少量电流流入键盘的逻辑电路，该逻辑电路扫描每个按键开关的状态，消除开关快速间歇闭合的电噪声，并将其转换为键码整数 13(译: keycode integer应该就是每个按键会对应一个整数)，键盘控制器然后对键码进行编码。 现在几乎普遍通过通用串行总线 (USB) 或蓝牙连接。

如果是 USB 键盘：

* 生成的键码由内部键盘电路存储器存储在称为“端点”(endpoint)的寄存器中。
* 主机 USB 控制器每 ~10 毫秒轮询一次该“端点”，获取存储在其上的键码值。
* 该值以最高 1.5 Mb/s (USB 2.0) 的速度发送到 USB SIE（串行接口引擎 serial interface engine）。
* 该串行信号随后在计算机的主机 USB 控制器上解码，并由计算机的人机接口设备 (HID, human interface device) 通用键盘设备驱动进行解释。
* 然后将按键的值传递到操作系统的硬件抽象层。

对于触摸屏键盘：

* 当用户将手指放在现代电容式触摸屏上时，少量电流会传输到手指。 这通过导电层的静电场导通了电路，并在屏幕上的那个点产生了电压降。 屏幕控制器然后引发中断报告“点击”的坐标。
* 然后移动操作系统通知当前的应用程序在这个 GUI 元素中的单击事件。（现在是虚拟键盘应用程序按钮）
* 虚拟键盘现在可以引发软件中断，用于将“按键按下”消息发送回操作系统。
* 此中断将“按键按下”事件通知当前聚焦的应用程序。

## 解析 URL

浏览器在 URL（统一资源定位符）中包含以下信息：

- 协议“http”：使用“超文本传输协议”

* 资源“/”：检索主（索引）页面

当没有给出协议或有效域名时，浏览器继续将地址框中给出的文本提供给浏览器的默认网络搜索引擎。

## 检查 HSTS 列表（已弃用）

* ~浏览器检查其“预加载的 HSTS（HTTP 严格传输安全）”列表。 这是已请求仅通过 HTTPS 联系的网站列表。~
* ~如果网站在列表中，浏览器将通过 HTTPS 而不是 HTTP 发送请求。 否则，初始请求通过HTTP发送。~

注意：该网站不在 HSTS 列表中，仍然可以使用 HSTS 策略。 用户对网站的第一个 HTTP 请求将收到一个响应，要求用户只发送 HTTPS 请求。 然而，这个单一的 HTTP 请求可能会让用户容易受到[降级攻击](http://www.yourdictionary.com/downgrade-attack)，这就是现代网络浏览器中包含 HSTS 列表的原因。

现代浏览器首先请求 https


## DNS 查询

浏览器试图找出输入域的 IP 地址。 DNS 查找过程如下：

* **浏览器缓存：**浏览器缓存 DNS 记录一段时间。 有趣的是，操作系统不会告诉浏览器每条 DNS 记录的生存时间，因此浏览器会将它们缓存一段固定的时间（因浏览器而异，2 - 30 分钟）。
* **OS 缓存：** 如果浏览器缓存不包含所需的记录，浏览器将进行系统调用（Windows 中的 gethostbyname）。 操作系统有自己的缓存。
* **路由器缓存：**请求继续到您的路由器，路由器通常有自己的 DNS 缓存。
* **ISP DNS 缓存：** 下一个检查的地方是 ISP 的 DNS 服务器的缓存。 
* **递归搜索：**您的 ISP 的 DNS 服务器开始递归搜索，从根名称服务器，通过 .com 顶级名称服务器，到 Google 的名称服务器。 通常，DNS 服务器缓存中包含 .com 域名服务器的名称，因此无需访问根域名服务器。

这是递归 DNS 搜索的示意图：

<p align="center">
  <img src="img/Example_of_an_iterative_DNS_resolver.svg" alt="Recursive DNS search"/>
</p>


关于 DNS 的一个令人担忧的事情是，整个域（如 wikipedia.org 或 facebook.com）似乎映射到一个单一的 IP 地址。 幸运的是，有一些方法可以缓解瓶颈：

* **轮询(round-robin) DNS** 是一种解决方案，其中 DNS 查找返回多个 IP 地址，而不仅仅是一个。 例如，facebook.com 实际上映射到四个 IP 地址。
* **负载均衡器** 是一种硬件，它侦听特定的 IP 地址并将请求转发到其他服务器。 主要站点通常会使用昂贵的高性能负载平衡器。
* **地理 DNS** 通过将域名映射到不同的 IP 地址来提高可扩展性，具体取决于客户端的地理位置。 这对于托管静态内容非常有用，这样不同的服务器就不必更新共享状态。
* **任播(anycast)** 是一种路由技术，其中单个 IP 地址映射到多个物理服务器。 遗憾的是，任播与 TCP 不相适应，并且很少用于这种情况。

大多数 DNS 服务器本身使用任播来实现 DNS 查找的高可用性和低延迟。 任播服务（DNS 是一个很好的例子）的用户将始终连接到“最近的”（从路由协议的角度来看）DNS 服务器。 这减少了延迟，并提供了一定程度的负载平衡（假设您的消费者均匀分布在您的网络中）。

## 打开套接字(socket) + TLS 握手

* 一旦浏览器接收到目标服务器的 IP 地址，它就用它和给定的端口号（HTTP 协议默认为端口 80，HTTPS 为端口 443），并调用名为socket 的系统库函数并请求 [TCP](http://www.webopedia.com/TERM/T/TCP.html) [套接字](http://www.webopedia.com/TERM/S/socket.html) 流。
* 客户端计算机向服务器发送一条 ClientHello 消息，其中包含其 TLS(Transport Layer Security) 版本、可用的密码算法和压缩方法列表。
* 服务器向客户端回复一条 ServerHello 消息，其中包含 TLS 版本、选择的密码、选择的压缩方法和由 CA（证书颁发机构 Certificate Authority）签署的公共证书。 该证书包含一个公钥，客户端将使用该公钥加密握手的其余部分，直到可以就对称密钥达成一致。
* 客户端根据其受信任的 CA 列表验证服务器数字证书。 如果可以基于 CA 建立信任，则客户端生成一串伪随机字节并使用服务器的公钥对其进行加密。 这些随机字节可用于确定对称密钥。
* 服务器使用其私钥解密随机字节，并使用这些字节生成自己的对称主密钥副本。
* 客户端向服务器发送 Finished 消息，使用对称密钥加密传输的哈希值。
* 服务器生成自己的哈希，然后解密客户端发送的哈希以验证它是否匹配。 如果是，它将自己的 Finished 消息发送给客户端，该消息也使用对称密钥加密。
* 从现在开始，TLS 会话传输使用约定的对称密钥加密的应用程序 (HTTP) 数据。

## HTTP 协议

您可以非常确定不会从浏览器缓存中找到诸如 Facebook/Gmail 之类的动态站点，因为动态页面会很快或立即过期（过期日期设置为过去）。

如果使用的 Web 浏览器是由 Google 编写的，那么它不会发送 HTTP 请求来检索页面，而是发送请求以尝试与服务器协商从 HTTP 到 SPDY 协议的“升级”。 请注意，在最新版本的 Chrome 中，SPDY 已被弃用，取而代之的是 HTTP/2。

```http
GET http://www.google.com/ HTTP/1.1
Accept: application/x-ms-application, image/jpeg, application/xaml+xml, [...]
User-Agent: Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1; WOW64; [...]
Accept-Encoding: gzip, deflate
Connection: Keep-Alive
Host: google.com
Cookie: datr=1265876274-[...]; locale=en_US; lsd=WW[...]; c_user=2101[...]
```

GET 请求指定要获取的 URL：“[http://www.google.com/](http://www.google.com/)”。 浏览器标识自己（User-Agent header），并说明它将接受什么类型的响应（Accept 和 Accept-Encoding header）。 Connection header 要求服务器保持 TCP 连接打开以供进一步请求。

该请求还包含浏览器对该域的 cookie。 您可能已经知道，cookie 是键值对，用于在不同的页面请求之间跟踪网站的状态。 因此 cookie 存储登录用户的名称、服务器分配给用户的秘密数字、用户的一些设置等。cookie 将存储在客户端的文本文件中，并发送到每个请求的服务器。

HTTP/1.1 为发送方定义了“close”连接选项，以表示连接将在响应完成后关闭。 例如，`Connection: close`.

发送请求和 header 后，Web 浏览器向服务器发送一个空白换行符，表示请求内容已完成。 服务器以表示请求状态的响应代码进行响应，并以以下形式的响应进行响应：**200 OK [response headers]**

接着是一个换行符，然后发送 www.google.com 的 HTML 内容负载。 然后服务器可以关闭连接，或者如果客户端发送的 header 要求，则保持连接打开以供进一步请求重用。

如果 Web 浏览器发送的 HTTP header 包含足够的信息，以便 Web 服务器确定 Web 浏览器缓存的文件版本自上次检索以来是否未被修改（即，如果 Web 浏览器包含 ETag header），则它可能会以以下形式的请求进行响应：**304 Not Modified [response headers]** 并且没有有效负载，并且 Web 浏览器会从其缓存中检索 HTML。

解析 HTML 后，Web 浏览器（和服务器）对 HTML 页面引用的每个资源（图像、CSS、favicon.ico 等）重复此过程，除了 GET / HTTP/1.1 以外，请求将是 **GET /$(URL relative to www.google.com) HTTP/1.1.**

如果 HTML 引用了与 www.google.com 不同的域上的资源，Web 浏览器将返回到解析其他域所涉及的步骤，并执行该域的所有步骤。 请求中的主机header将设置为适当的服务器名称，而不是 google.com。

**陷阱：**

* URL“[http://facebook.com/](http://facebook.com/)”中的尾部斜杠很重要。 在这种情况下，浏览器可以安全地添加斜线。 对于http://example.com/folderOrFile 形式的URL，浏览器无法自动添加斜杠，因为无法明确folderOrFile 是文件夹还是文件。 在这种情况下，浏览器将访问不带斜杠的 URL，服务器将以重定向响应，从而导致不必要的往返。
* 服务器可能会响应 301 Moved Permanently 响应，告诉浏览器转到“[http://www.google.com/](http://www.google.com/)”而不是“[http://google.com/](http://google.com/)”。 服务器坚持重定向而不是立即响应用户想要查看的网页的原因很有趣。
   原因之一与搜索引擎排名有关。 如果同一个页面有两个 URL，比如 http://www.vasanth.com/ 和 http://vasanth.com/，搜索引擎可能会认为它们是两个不同的站点，每个站点都有较少的传入链接, 因此排名较低。 搜索引擎理解永久重定向 (301)，并将来自两个来源的传入链接合并到一个排名(ranking)中。
   此外，同一内容的多个 URL 不缓存友好。 当一段内容有多个名称时，它可能会在缓存中出现多次。

**注意：**
HTTP 响应以服务器返回的状态代码开始。 以下是状态代码表示的非常简短的摘要：

   * 1xx 仅表示信息性消息
   * 2xx 表示某种成功
   * 3xx 将客户端重定向到另一个 URL
   * 4xx 表示客户端错误
   * 5xx 表示服务器端出错

## HTTP 服务器请求句柄(handle)

HTTPD（HTTP 守护进程）服务器是在服务器端处理请求/响应的服务器。 最常见的 HTTPD 服务器是用于 Linux 的 Apache 或 nginx 和用于 Windows 的 IIS。

* HTTPD（HTTP 守护进程）接收请求。
* 服务器将请求分解为以下参数：
   * HTTP 请求方法（GET、POST、HEAD、PUT 和 DELETE）。 对于直接在地址栏中输入的 URL，这将是 GET。
   * 域(domain)，在本例中为 google.com。
   * 请求的路径/页面，在本例中为 `/` (即没有特定的路径/页面被请求, `/`是默认路径
   * 服务器验证在服务器上配置了与google.com对应的虚拟主机。
* 服务器验证 google.com 可以接受 GET 请求。
* 服务器验证是否允许客户端使用此方法（通过IP、身份验证等）。
* 如果服务器安装了重写模块（如 Apache 的 mod_rewrite 或 IIS 的 URL Rewrite），它会尝试将请求与配置的规则之一相匹配。 如果找到匹配规则，服务器将使用该规则重写请求。
* 服务器去拉取与请求对应的内容，在我们的例子中它会回退到索引文件，因为“/”是主文件（有些情况可以覆盖这个，但这是最常见的方法）。
* 服务器根据请求处理器(handler)解析文件。 请求处理器是一个程序（在 ASP.NET中, PHP、Ruby 等），它读取请求并为响应生成 HTML。 如果 Google 在 PHP 上运行，则服务器使用 PHP 来解释索引文件，并将输出流式传输到客户端。

注意：每个动态网站面临的一个有趣的困难是如何存储数据。 较小的站点通常只有一个 SQL 数据库来存储它们的数据，但是存储大量数据和/或有很多访问者的站点必须找到一种方法将数据库拆分到多台机器上。 解决方案包括分片(sharding)（根据主键将表拆分到多个数据库）、重复副本, 和使用一致性语义较弱的简化数据库。

## 服务器响应

这是服务器生成并发回的响应：

```http
HTTP/1.1 200 OK
Cache-Control: private, no-store, no-cache, must-revalidate, post-check=0,
    pre-check=0
Expires: Sat, 01 Jan 2000 00:00:00 GMT
P3P: CP="DSP LAW"
Pragma: no-cache
Content-Encoding: gzip
Content-Type: text/html; charset=utf-8
X-Cnection: close
Transfer-Encoding: chunked
Date: Fri, 12 Feb 2010 09:05:55 GMT

2b3
��������T�n�@����[...]
```

整个响应是 36 kB，其中大部分在我修剪的末尾中。

**Content-Encoding** header告诉浏览器响应主体是使用 gzip 算法压缩的。 解压缩 blob 后，您将看到您期望的 HTML：

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
       "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="zh-cn"
       lang="zh-cn" id="google" class="no_js">
<头>
<meta http-equiv="Content-type" content="text/html; charset=utf-8" />
<meta http-equiv="Content-language" content="en" />
...
```

请注意将 Content-Type 设置为 text/html 的header。 header指示浏览器将响应内容呈现为 HTML，而不是说将其下载为文件。 浏览器将使用header来决定如何解释响应，但也会考虑其他因素，例如 URL 的扩展名。

## 浏览器的幕后

一旦服务器向浏览器提供资源（HTML、CSS、JS、图像等），它就会经历以下过程：

* 解析 - HTML、CSS、JS
* 渲染 - 构建 DOM 树 → 渲染树 → 渲染树布局 → 绘制渲染树

## 浏览器的高层结构

1. **用户界面：** 包括地址栏、后退/前进按钮、书签菜单等。除了您看到请求页面的窗口外，浏览器显示的每个部分。

2. **浏览器引擎：** UI 和渲染引擎之间的序列化([Marshals](http://stackoverflow.com/a/5600887/1672655))操作。

3. **渲染引擎：**负责显示请求的内容。 例如。 渲染引擎解析HTML和CSS，并将解析后的内容显示在屏幕上。

4. **网络：**对于HTTP请求等网络调用，针对不同的平台使用不同的实现（在平台无关的接口后面）。

5. **UI 后端：** 用于绘制组合框和窗口等基本小部件。 该后端公开了一个非特定于平台的通用接口。 在它下面使用操作系统用户界面方法。

6. **JavaScript 引擎：** 用于解析和执行 JavaScript 代码的解释器。

7. **数据存储：**这是一个持久层。 浏览器可能需要在本地保存数据，例如 cookie。 浏览器还支持存储机制，例如 [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)、[IndexedDB](https://developer.mozilla.org/ en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB) 和 [文件系统](https://developer.chrome.com/apps/fileSystem)。

<p align="center">
  <img src="img/layers.png" alt="Browser Components"/>
</p>


让我们从最简单的情况开始：一个带有一些文本和一张图片的纯 HTML 页面。 浏览器需要做什么来处理这个简单的页面？

1. **转换：**浏览器从磁盘或网络读取 HTML 的原始字节并将它们转换为单独的基于文件指定编码的字符（例如 UTF-8）。

2. **标记化(tokenizing)：**浏览器将字符串转换为 W3C HTML5 标准指定的不同标记 - 例如 “<html>”、“<body>”等“尖括号”内的字符串。 每个标记都有特殊的含义和一组规则。

3. **词法分析(Lexing)：**发出的标记被转换为定义其属性和规则的“对象”。

4. **DOM 构造：** 最后，由于 HTML 标记定义了不同标签之间的关系（一些标签包含在标签中），因此创建的对象链接在树数据结构中，该结构还捕获原始文件标记中定义的父子关系 ：HTML 对象是主体对象的父对象，主体是段落对象的父对象，等等。

   <p align="center">
     <img src="img/full-process.png" alt="DOM Construction Process"/>
   </p>


   整个过程的最终输出是文档对象模型，或者我们简单页面的“DOM”，浏览器使用它来对页面进行所有进一步处理。

   每次浏览器必须处理 HTML 标记时，它必须逐步完成上述所有步骤：将字节转换为字符、识别标记、将标记转换为节点以及构建 DOM 树。 整个过程可能需要一些时间，尤其是当我们要处理大量 HTML 时。

   <p align="center">
     <img src="img/dom-timeline.png" alt="Tracing DOM construction in DevTools"/>
   </p>


   如果您打开 Chrome DevTools 并在页面加载时记录时间线，您可以看到执行此步骤所花费的实际时间——在上面的示例中，我们花了大约 5 毫秒将一大块 HTML 字节转换为 DOM 树。 当然，如果页面较大（大多数页面都是这样），则此过程可能需要更长的时间。 在我们以后关于创建流畅动画的章节中，您将看到如果浏览器必须处理大量 HTML，这很容易成为您的瓶颈。

## 渲染(rendering)引擎

渲染引擎是一个软件组件，它获取标记的内容（如 HTML、XML、图像文件等）和格式化信息（如 CSS、XSL 等），并将格式化后的内容显示在屏幕上。

| Browser           |          Engine           |
| ----------------- | :-----------------------: |
| Chrome            | Blink (a fork of WebKit)  |
| Firefox           |           Gecko           |
| Safari            |          Webkit           |
| Opera             |  Blink (Presto if < v15)  |
| Internet Explorer |          Trident          |
| Edge              | Blink (EdgeHTML if < v79) |

WebKit 是一个开源渲染引擎，最初是作为 Linux 平台的引擎，后来被 Apple 修改以支持 Mac 和 Windows。

渲染引擎是单线程的。 除了网络操作之外，几乎所有的事情都发生在一个线程中。 在 Firefox 和 Safari 中，这是浏览器的主线程。 在 Chrome 中，它是选项卡进程主线程。
网络操作可以由多个并行线程执行。 并行连接的数量是有限的（通常每个主机名有 6-13 个连接）。

浏览器主线程是一个事件循环。 这是一个无限循环，使进程保持活动状态。 它等待事件（如布局和绘制事件）并处理它们。

注意：Chrome 等浏览器运行渲染引擎的多个实例：每个选项卡一个。 每个选项卡都在单独的进程中运行。

## 主要流程

渲染引擎将开始从网络层获取请求文档的内容。 这通常以 8KB 块的形式完成。

之后渲染引擎的基本流程是：

<p align="center">
  <img src="img/flow.png" alt="Rendering engine basic flow"/>
</p>


呈现引擎将开始解析 HTML 文档并将元素转换为称为 **“内容树”** 的树中的 [DOM](http://domenlightenment.com/) 节点。

引擎将解析样式数据，包括外部 CSS 文件和样式元素。 HTML 中的样式信息和视觉指令将用于创建另一棵树：**渲染树**。
渲染树包含具有视觉属性（如颜色和尺寸）的矩形。 矩形以正确的顺序显示在屏幕上。

在构建渲染树之后，它会经历一个**“布局”**过程。 这意味着为每个节点提供它应该出现在屏幕上的确切坐标。

下一阶段是**绘图**——将遍历渲染树并使用 UI 后端层绘制每个节点。

重要的是要了解这是一个渐进的过程。 为了更好的用户体验，渲染引擎会尝试尽快将内容显示在屏幕上。 它不会等到所有 HTML 都被解析后才开始构建和布局渲染树。 部分内容将被解析和显示，同时继续处理来自网络的其余内容。

下面给出的是 Webkit 的流程：

<p align="center">
  <img src="img/webkitflow.png" alt="Webkit main flow"/>
</p>


## 解析(parsing)基础

**解析：** 将文档转换为代码可以使用的结构。 解析的结果通常是代表文档结构的节点树。

**语法：** 解析基于文档遵循的语法规则：编写它的语言或格式。您可以解析的每种格式都必须具有由词汇和语法规则组成的确定性语法。 它被称为**上下文无关文法**。

解析可以分为两个子过程：词法分析和句法分析。

**词法分析：**将输入分解为标记的过程。 标记是语言词汇：有效构建块的集合。

**语法分析：**语言语法规则的应用。

解析器通常将工作分为两个部分：负责将输入分解为有效标记的词法分析器（有时称为分词器），以及负责根据语言语法规则分析文档结构来构建解析树的解析器。 词法分析器知道如何去除不相关的字符，如空格和换行符。

<p align="center">
  <img src="img/image011.png" alt="Source document to parse tree"/>
</p>


解析过程是迭代的。 解析器通常会向词法分析器询问一个新的标记，并尝试将该标记与其中一个语法规则相匹配。 如果一条规则被匹配，一个与该标记对应的节点将被添加到解析树中，解析器将请求另一个标记。

如果没有规则匹配，解析器将在内部存储标记，并不断请求标记，直到找到匹配所有内部存储标记的规则。 如果未找到规则，则解析器将引发异常。 这意味着该文档无效并且包含语法错误。

HTML 解析器的工作是将 HTML 标记解析为解析树。 HTML 定义采用 DTD（文档类型定义Document Type Definition）格式。 此格式用于定义 SGML 家族的语言。 该格式包含所有允许的元素、它们的属性和层次结构的定义。 正如我们之前看到的，HTML DTD 没有形成上下文无关语法。

HTML 解析算法包括两个阶段：标记化和树构造。

**标记化**是词法分析，将输入解析为标记。 HTML 标记包括开始标记、结束标记、属性名称和属性值。 tokenizer 识别 token，将其交给树构造函数，并使用下一个字符来识别下一个 token，依此类推，直到输入结束。

<p align="center">
  <img src="img/image017.png" alt="HTML parsing flow"/>
</p>


## DOM 树

输出树（“解析树”）是 DOM 元素和属性节点的树。 DOM 是文档对象模型(Document Object Model)的缩写。 它是 HTML 文档的对象表示，是 HTML 元素与 JavaScript 一样对外的接口。 树的根是“文档”对象。

DOM 与标记几乎是一对一的关系。 例如：

```html
<html>
  <body>
    <p>
      Hello World
    </p>
    <div> <img src="example.png"/></div>
  </body>
</html>
```

此标记将被转换为以下 DOM 树：

<p align="center">
  <img src="img/image015.png" alt="DOM Tree"/>
</p>


### 为什么 DOM 很慢？

简短的回答是 DOM 并不慢。 添加和删除 DOM 节点只是一些指针交换，只不过是在 JS 对象上设置一个属性。

但是，布局(layout)很慢。 当您以任何方式接触 DOM 时，您都会在整棵树上设置一个脏位，告诉浏览器它需要重新弄清楚一切都去哪里了。 当 JS 将控制权交还给浏览器时，它会调用其布局算法（或者更专业地说，它会调用其 CSS 重新计算算法，然后进行布局，然后重新绘制，然后重新合成）以重新绘制屏幕。 布局算法非常复杂 - 阅读 CSS 规范以了解一些规则 - 这意味着它通常必须做出非局部决策。

更糟糕的是，布局是通过访问某些属性同步触发的。 其中包括 getComputedStyleValue()、getBoundingClientWidth()、.offsetWidth、.offsetHeight 等，这使得它们非常容易撞上。 完整列表在 [此处](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)。
正因为如此，很多 Angular 和 JQuery 代码都非常慢。 一种布局将耗尽您在移动设备上的整个框架预算。 当我测试 Google Instant c 2013 时。 它在一次查询中导致了13个布局，在移动设备上锁屏近2秒。 （此后已加快速度。）

React 无助于加速布局——如果你想在移动网络浏览器上获得流畅的动画，你需要求助于其他技术，比如将你在一个帧中所做的一切限制为可以在 GPU 上执行的操作。 但真正有效的是确保每次更新页面状态时最多执行一个布局。 那经常对现状有相当大的改善。

## 渲染树

在构建 DOM 树的同时，浏览器会构建另一棵树，即渲染树。 这棵树是按显示顺序排列的视觉元素。 它是文档的可视化表示。 这棵树的目的是能够以正确的顺序绘制内容。

渲染器知道如何布局和绘制自身及其子项。 每个渲染器代表一个矩形区域，通常对应于节点的 CSS box。

## 渲染树与 DOM 树的关系

渲染器对应于 DOM 元素，但不是一对一的关系。 非可视化的 DOM 元素不会被插入到渲染树中。 一个例子是“head”元素。 显示值被分配为“none”的元素也不会出现在树中（而具有“hidden”可见性的元素将出现在树中）。

有对应于多个可视对象的 DOM 元素。 这些通常是具有复杂结构的元素，无法用单个矩形来描述。 例如，“select”元素有三个渲染器：一个用于显示区域，一个用于下拉列表框，一个用于按钮。 此外，当文本因一行的宽度不够而被分成多行时，新行将作为额外的渲染器添加。

一些渲染对象对应于一个 DOM 节点，但不在树中的同一位置。 浮动元素和绝对定位元素脱离主流，被放置在树的不同部分，并映射到真实的框架。 占位符框架是他们应该在的地方。

<p align="center">
  <img src="img/image025.png" alt="The render tree and the corresponding DOM tree"/>
</p>


在 WebKit 中，解析样式和创建渲染器的过程称为“attachment”。 每个 DOM 节点都有一个“attach”方法。 attachment是同步的，节点插入 DOM 树调用新节点的“attach”方法。

构建渲染树需要计算每个渲染对象的视觉属性。 这是通过计算每个元素的样式属性来完成的。 样式包括各种来源的样式表、内联样式元素和 HTML 中的视觉属性（如“bgcolor”属性）。后者会被翻译来匹配 CSS 样式属性。

## CSS 解析

CSS 选择器(selector)由浏览器引擎从右到左匹配。 请记住，当浏览器进行选择器匹配时，它有一个元素（它试图为其确定样式的那个）和所有规则及其选择器，它需要找到哪些规则与该元素匹配。 这与通常的 jQuery 不同，例如，您只有一个选择器，您需要找到与该选择器匹配的所有元素。

选择器的确定是这样计算的：

* 如果它来自的声明是“style”属性而不是带有选择器的规则，则计数 1，否则计数 0 (= a)
* 统计选择器中ID选择器的个数(= b)
* 统计选择器中类选择器、属性选择器、伪类(class selectors, attributes selectors, and pseudo-classes)的个数(= c)
* 统计选择器中元素名和伪元素的个数(=d)
* 忽略通用选择器

连接三个数字 a-b-c-d（在具有大基数的数字系统中）给出了特异性。 您需要使用的基数由您在 a、b、c 和 d 之一中的最高计数定义。

例子：

``` 文本文件
*               /* a=0 b=0 c=0 -> specificity =   0 */
LI              /* a=0 b=0 c=1 -> specificity =   1 */
UL LI           /* a=0 b=0 c=2 -> specificity =   2 */
UL OL+LI        /* a=0 b=0 c=3 -> specificity =   3 */
H1 + *[REL=up]  /* a=0 b=1 c=1 -> specificity =  11 */
UL OL LI.red    /* a=0 b=1 c=3 -> specificity =  13 */
LI.red.level    /* a=0 b=2 c=1 -> specificity =  21 */
#x34y           /* a=1 b=0 c=0 -> specificity = 100 */
#s12:not(FOO)   /* a=1 b=0 c=1 -> specificity = 101 */
```

为什么 CSSOM 有树状结构？ 当计算页面上任何对象的最终样式集时，浏览器从适用于该节点的最通用规则开始（例如，如果它是 body 元素的子元素，则应用所有 body 样式），然后递归地优化计算出的样式 通过应用更具体的规则——即“向下级联(cascade down)”的规则。

WebKit 使用一个标志来标记所有顶级样式表（包括@imports）是否已加载。 如果附加时样式未完全加载，则使用占位符并在文档中标记，一旦加载样式表将重新计算它们。

## 布局(layout)

创建渲染器并将其添加到树中时，它没有位置和大小。 计算这些值称为布局或回流(reflow)。

HTML 使用基于流的布局模型，这意味着大多数时候可以一次计算几何图形。 “流中”较晚的元素通常不会影响“流中”较早的元素的几何形状，因此布局可以从左到右、从上到下贯穿整个文档。 这个坐标系是相对于根框架的。 使用顶部和左侧坐标。

布局是一个递归过程。 它从根渲染器开始，对应于 HTML 文档的 <html> 元素。 布局通过部分或全部帧层次结构递归地继续，为需要它的每个渲染器计算几何信息。

根渲染器的位置是 0,0，它的尺寸是视口(viewpoint)——浏览器窗口的可见部分。 所有渲染器都有一个“layout”或“reflow”方法，每个渲染器调用其需要布局的子级的布局方法。

为了不为每一个小的变化做一个完整的布局，浏览器使用了一个“脏位”系统。 更改或添加的渲染器将其自身及其子项标记为“脏”：需要布局。 有两个标志：“dirty”和“children are dirty”，这意味着尽管渲染器本身可能没问题，但它至少有一个子项需要布局。

布局通常具有以下模式：

- 父渲染器决定自己的宽度。
- 父母越过孩子并且：
   - 放置子渲染器（设置它的 x 和 y）。
   - 如果需要调用子布局——它们是脏的或者我们处于全局布局中，或者出于某些其他原因——计算子布局的高度。
- Parent 使用 children 的累积高度和 margins 和 padding 的高度来设置自己的高度——这将被 parent renderer 的 parent 使用。
- 将其脏位设置为false。

另请注意，布局抖动(layout thrashing)是指 Web 浏览器在“加载”页面之前必须多次回流或重新绘制网页。 在 JavaScript 流行之前的日子里，网站通常只回流和绘制一次，但现在越来越普遍的是，JavaScript 在页面加载时运行，这可能会导致对 DOM 的修改，从而导致额外的回流或重绘。 根据回流次数和网页的复杂性，加载页面时可能会造成严重延迟，尤其是在手机或平板电脑等低功率设备上。

## 绘图(painting)

在绘图阶段，遍历渲染树，调用渲染器的“paint()”方法在屏幕上显示内容。 绘图使用 UI 基础结构组件。

像布局一样，绘制也可以是全局的——整个树都被绘制——或增量的。 在增量绘图中，一些渲染器的变化不会影响整个树。 更改后的渲染器使其在屏幕上的矩形无效。 这会导致操作系统将其视为“脏区”并生成“绘制”事件。 操作系统巧妙地做到了这一点，并将多个区域合并为一个区域。

在重新绘制之前，WebKit 将旧矩形保存为位图。 然后它只绘制新旧矩形之间的增量。 浏览器会尝试执行最少的可能操作来响应更改。 所以改变元素的颜色只会导致元素的重绘。 元素位置的更改将导致元素及其子元素和可能的兄弟元素的布局和重绘。 添加 DOM 节点将导致节点的布局和重绘。 重大更改，如增加“html”元素的字体大小，将导致缓存失效、重新布局和重新绘制整个树。

存在三种不同的定位方案：

* **普通：**对象根据其在文档中的位置定位。 这意味着它在渲染树中的位置就像它在 DOM 树中的位置一样，并根据其框类型和尺寸进行布局
* **浮动：**对象首先像正常流一样布局，然后尽可能向左或向右移动
* **绝对：**对象在渲染树中的位置不同于 DOM 树中的位置

定位方案由“position”属性和“float”属性设置。

- 静态和相对导致正常流动
- absolute 和 fixed 导致绝对定位

在静态定位中，没有定义任何位置，而是使用默认定位。 在其他方案中，作者指定了位置：top, bottom, left, right。

**图层(layers)** 由 z-index CSS 属性指定。 它表示盒子的第三维：它沿“z 轴”的位置。

这些框被分成堆栈（称为堆栈上下文 stacking contexts）。 在每个堆栈中，后面的元素将首先绘制，前面的元素在顶部，更靠近用户。 在重叠的情况下，最前面的元素将隐藏前一个元素。 堆栈根据 z-index 属性排序。 具有“z-index”属性的框形成本地堆栈。

## 冷知识(trivia)

### 网络的诞生

CERN 的英国科学家 Tim Berners-Lee 于 1989 年发明了万维网 (World Wide Web, WWW)。Web 最初的构想和开发是为了满足世界各地大学和研究所的科学家之间自动共享信息的需求。

CERN 的第一个网站——也是世界上的第一个网站——专门用于万维网项目本身，并托管在 Berners-Lee 的 NeXT 计算机上。 该网站描述了网络的基本特征； 如何交流处理其他人的文档以及如何设置自己的服务器。 NeXT 机器——最初的网络服务器——仍在 CERN。 作为恢复[第一个网站](http://info.cern.ch/) 项目的一部分，CERN 在 2013 年将世界上第一个网站恢复到其原始地址。

1993 年 4 月 30 日，CERN 将万维网软件置于公共领域。 CERN 以开放许可的方式发布了下一个版本，作为一种更可靠的方式来最大限度地传播它。 通过这些行动，免费提供运行网络服务器所需的软件以及[基本浏览器](http://line-mode.cern.ch/) 和代码库，网络得以蓬勃发展。

*更多阅读：*

[What really happens when you navigate to a URL](http://igoro.com/archive/what-really-happens-when-you-navigate-to-a-url/)

[How Browsers Work: Behind the scenes of modern web browsers](http://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)

[What exactly happens when you browse a website in your browser?](http://superuser.com/questions/31468/what-exactly-happens-when-you-browse-a-website-in-your-browser)

[What happens when](https://github.com/alex/what-happens-when)

[So how does the browser actually render a website](https://www.youtube.com/watch?v=SmE4OwHztCc)

[Constructing the Object Model](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/constructing-the-object-model)

[How the Web Works: A Primer for Newcomers to Web Development (or anyone, really)](https://medium.freecodecamp.com/how-the-web-works-a-primer-for-newcomers-to-web-development-or-anyone-really-b4584e63585c#.7l3tokoh1)

