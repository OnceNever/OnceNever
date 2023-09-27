---
title: HTTP缓存
date: 2023/9/25 22:01:39
categories:
- [网络, HTTP]
tags:
- HTTP
- 缓存
---

### 简介

#### 为什么要使用缓存？

HTTP(超文本传输协议)是一种无状态的应用程序级请求/响应协议，使用可扩展语义和自描述消息与基于网络的超文本信息系统进行灵活的交互。对于现代互联网来说存在着大量的静态文件例如HTML、CSS和图片等资源，这些资源在后续的请求中不会发生变更。我们可以缓存存储可缓存的响应，以减少未来等效请求的响应时间和网络带宽消耗。



HTTP **缓存的目标**是通过重用先前的响应消息来满足当前请求，每次缓存重用新响应时，新响应都可以减少延迟和网络开销，从而显着提高性能。



### 缓存操作概述

#### 缓存需要解决的主要问题是什么？

虽然缓存是HTTP的可选功能，但我们可以假设重用缓存的响应是有效的，并且当没有要求或本地配置阻止对响应进行缓存时，这种重用是**默认行为**。也就是说HTTP的缓存默认是开启的。因此，**HTTP 缓存实现的重点是防止缓存存储不可重用的响应或不适当地重用已存储的响应**，而不是强制缓存存储和重用特定响应。



#### 本地缓存存储格式

缓存在本地是以 【请求-响应】 的key/value键值对的形式存储的。

缓存键（key）是缓存用来选择响应的信息，至少由请求方法和用于检索存储的响应的目标 URL 组成，该方法确定在什么情况下（GET/POST等）可以使用该响应来满足后续请求。然而，目前许多常用的 HTTP 缓存**仅缓存 GET** 响应，因此仅使用 URL 作为缓存键。



##### 使用Vary标头字段计算缓存键

**注意！！！**

虽然区分响应的方式本质上是基于它们的 URL。

![image-20230925230408064](https://blog.seeyourface.cn/blog/image-20230925230408064.png)

但是响应的内容并不总是相同的，即使它们具有相同的 URL。例如在执行内容协商时，来自服务器的响应可能取决于 `Accept`、`Accept-Language` 和 `Accept-Encoding` 请求标头的值。



打个比方：对于带有 `Accept-Language: en` 标头并已缓存的英语内容，不希望再对具有 `Accept-Language: ja 请求标头的请求重用该缓存响应。在这种情况下，可以通过在 `Vary` 标头的值中添加“`Accept-Language`”，根据语言单独缓存响应。



```http
Vary: Accept-Language
```



当缓存收到可由存储响应满足的请求并且该本地存储的响应包含 Vary 标头字段时，缓存**不允许**在没有重新验证的情况下使用该存储的响应，除非该 Vary 字段值指定的所有呈现的请求标头字段都与原始请求中的字段匹配。这会导致缓存基于响应 URL 和 `Accept-Language`请求标头的组合进行键控——而不是仅仅基于响应 URL。

![image-20230925230553189](https://blog.seeyourface.cn/blog/image-20230925230553189.png)

同时，Vary标头字段值包含 “*” 的存储响应将**始终无法匹配**。



#### 哪些响应能被缓存？

并不是所有响应都能被缓存存储，响应中存在 no-store 缓存指令将不会被缓存。

最常见的是缓存请求成功的结果：即对 GET 请求200（OK）的响应，其中包含目标资源的表示。

然而，也可以存储重定向、否定结果（404 未找到）、不完整的结果（206 部分内容）、以及对 GET 以外的方法的响应如果该方法的定义允许此类缓存并定义了适合用作缓存键的内容。



### 缓存类型

[RFC 9111 HTTP Caching](https://httpwg.org/specs/rfc9111.html#calculating.freshness.lifetime) 标准定义了两种不同的缓存：**私有缓存和共享缓存**。



#### 私有缓存

私有缓存是与特定客户端相绑定的缓存——通常是指浏览器缓存。由于存储的响应不与其他客户端共享，因此私有缓存可以存储该用户的个性化响应。

如果响应包含个性化内容并且你只想将响应存储在私有缓存中，则必须指定 `private` 指令。

```http
Cache-Control: private
```



但是如果响应具有 `Authorization` 标头，则不能将其存储在私有缓存（或共享缓存，除非 Cache-Control 指定的是 `public`）中。



> A cache *MUST NOT* store a response to a request unless if the cache is shared: the Authorization header field is not present in the request or a response directive is present that explicitly allows shared caching;



#### 共享缓存

共享缓存位于客户端和服务器之间，可以存储能在用户之间**共享**的响应。共享缓存可以进一步细分为代理缓存和托管缓存。



### 启发式缓存

[RFC 9111 HTTP Caching](https://httpwg.org/specs/rfc9111.html#calculating.freshness.lifetime) 规范中提到，由于源服务器并不总是提供显式过期时间，因此当未指定显式时间时，缓存*可以*分配启发式过期时间，并采用使用其他字段值（例如上次修改时间）的算法来估计合理的过期时间。该规范没有提供具体的算法，但它确实对其结果施加了最坏情况的限制。



当存储的响应中存在显式过期时间(例如`Cache-Control`)时，缓存**不得**使用启发式方法来确定新鲜度。也就是说**启发式缓存只适用于存储响应中没有显式给出过期时间的情况**。



例如，我们通过请求获取到以下响应，内容最后一次更新是在 1 年前。



```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
Date: Tue, 26 Sep 2022 22:22:22 GMT
Last-Modified: Tue, 26 Sep 2021 22:22:22 GMT

<!doctype html>
…
```



通过响应我们可以推测整整一年没有更新的内容在那之后的一段时间内也将不会更新。因此，客户端决定存储此响应（尽管缺少 `max-age`）并重用它一段时间。复用多长时间取决于实现，但规范建议存储**不超过自该修改时间以来至今的一部分**。这个值在规范中推荐为10%，所以上面的例子中重用的时间不超过0.1年。

因此，如果源服务器希望阻止缓存，则鼓励它们发送显式指令（例如，`Cache-Control：no-cache`）



> Since origin servers do not always provide explicit expiration times, a cache *MAY* assign a heuristic expiration time when an explicit time is not specified, employing algorithms that use other field values (such as the Last-Modified time) to estimate a plausible expiration time. This specification does not provide specific algorithms, but it does impose worst-case constraints on their results.
>
> A cache *MUST NOT* use heuristics to determine freshness when an explicit expiration time is present in the stored response. Because of the requirements in Section 3), heuristics can only be used on responses without explicit freshness whose status codes are defined as heuristically cacheable (e.g., see Section 15.1 of HTTP) and on responses without explicit freshness that have been marked as explicitly cacheable (e.g., with a public response directive).
>
> Note that in previous specifications, heuristically cacheable response status codes were called "cacheable by default".
>
> If the response has a Last-Modified header field of HTTP, caches are encouraged to use a heuristic expiration value that is no more than some fraction of the interval since that time. A typical setting of this fraction might be 10%.



### 强制缓存

命中强制缓存时，客户端不会再请求服务器，直接从重用缓存的响应，并返回HTTP状态码为200。



![image-20230927225454971](https://blog.seeyourface.cn/blog/image-20230927225454971.png)



强制缓存由响应标头中的`Expries`、`Cache-Control` 和 `Pragma`来控制：

- Expires：在 HTTP/1.0 中，缓存的有效期是通过 `Expires` 标头来指定的。`Expires` 标头使用明确的时间而不是通过指定经过的时间来指定缓存的生命周期。

```http
Expires: Tue, 28 Feb 2022 22:22:22 GMT
```

这种方式存在一些问题，例如时间格式难以解析，并且客户端可能通过故意偏移系统时钟来诱发问题。

- Cache-Control：由于Expires可能导致的问题，在 HTTP/1.1 中引入了`Cache-Control` 标头，它采用了 `max-age`——即用于指定经过的时间，这是一个相对时间，是自响应生成以来经过的时间。如果 `Expires` 和 `Cache-Control: max-age` 都可用，则将 `max-age` 定义为首选。因此，由于 HTTP/1.1 已被广泛使用，无需特地提供 `Expires`。
  - no-store： 禁用缓存，表明服务器希望**永远都不要在客户端存储资源**，总是去原始服务器去获取最新资源。
  - no-cache：可以在客户端存储资源，但**每次都必须去服务端做新鲜度校验**。
  - private/public：private指的单个用户，public可以被任何中间人、CDN等缓存
  - max-age=：max-age是距离请求发起的时间的秒数
  - must-revalidate：在缓存过期前可以使用，过期后必须向服务器验证
- Pragma：`Pragma`请求标头字段是为 HTTP/1.0 缓存定义的，以便客户端可以指定“无缓存”的请求（因为Cache-Control直到HTTP/1.1才被定义）。然而，对 `Cache-Control` 的支持现在已经很广泛了，因此，*RFC 9111 - HTTP Caching*规范不赞成使用 Pragma。



#### 强制缓存存储的位置



| 状态 | Network - Size    | 含义                                                         |
| ---- | ----------------- | ------------------------------------------------------------ |
| 200  | from memory cache | 不请求网络资源，资源在内存，一般是脚本、字体、图片，浏览器关闭，数据将被释放 |
| 200  | from disk cache   | 请求网络资源，资源在磁盘，一般是CSS等，关闭数据还在          |
| 200  | 资源大小          | 从服务器下载最新资源                                         |
| 304  | 报文大小          | 请求服务端发现资源未更新，使用本地资源，不从服务器携带完整响应 |



比如下图返回的响应Size字段中的内容：



![image-20230927233159423](https://blog.seeyourface.cn/blog/image-20230927233159423.png)



### 协商缓存

客户端向服务器发送请求，服务器会根据这个请求的请求头的一些参数来判断是否命中协商缓存，如果命中，则返回304状态码并带上新的响应头通知浏览器从缓存中读取资源，由于此响应仅表示“没有变化”，因此没有响应主体——只有一个状态码——因此传输大小非常小。

协商缓存，响应头中有两对字段配合使用标记规则。



#### Last-Modified / If-Modified-Since

以下响应在 22:22:22 生成，`max-age` 为 1 小时，因此你知道它在 23:22:22 之前是有效的。



```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
Date: Tue, 22 Feb 2022 22:22:22 GMT
Last-Modified: Tue, 22 Feb 2022 22:00:00 GMT
Cache-Control: max-age=3600

<!doctype html>
…
```



到 23:22:22 时，响应会过时并且不能重用缓存。因此，下面的请求显示客户端发送带有 `If-Modified-Since` 请求标头的请求，以询问服务器自指定时间以来是否有任何的改变。



```http
GET /index.html HTTP/1.1
Host: example.com
Accept: text/html
If-Modified-Since: Tue, 22 Feb 2022 22:00:00 GMT
```



如果内容自指定时间以来没有更改，服务器将响应 `304 Not Modified`。



```http
HTTP/1.1 304 Not Modified
Content-Type: text/html
Date: Tue, 22 Feb 2022 23:22:22 GMT
Last-Modified: Tue, 22 Feb 2022 22:00:00 GMT
Cache-Control: max-age=3600
```



收到该响应后，客户端将存储的过期响应恢复为有效的，并可以在剩余的 1 小时内重复使用它。



这种方法同样存在一些问题：例如，时间格式复杂且难以解析，分布式服务器难以同步文件更新时间。



所以为了解决这些问题，`ETag` 响应标头被标准化作为替代方案。



#### ETag/If-None-Match

`ETag` 响应标头的值是服务器生成的任意值。服务器对于生成值没有任何限制，因此服务器可以根据他们选择的任何方式自由设置值——例如主体内容的哈希或版本号，只需要能唯一标识这个资源即可。



举个例子，如果 `ETag` 标头使用了 hash 值，`index.html` 资源的 hash 值是 `deadbeef`，响应如下：



```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
Date: Tue, 22 Feb 2022 22:22:22 GMT
ETag: "deadbeef"
Cache-Control: max-age=3600

<!doctype html>
…
```



如果该响应是陈旧的，则客户端获取缓存响应的 `ETag` 响应标头的值，并将其放入 `If-None-Match` 请求标头中，以询问服务器资源是否已被修改：



```http
GET /index.html HTTP/1.1
Host: example.com
Accept: text/html
If-None-Match: "deadbeef"
```



如果服务器为请求的资源确定的 `ETag` 标头的值与请求中的 `If-None-Match` 值相同，则服务器将返回 `304 Not Modified`。



但是，如果服务器确定请求的资源现在应该具有不同的 `ETag` 值，则服务器将其改为 `200 OK` 和资源的最新版本进行响应。



那是不是有了`ETag`之后就没必要使用`Last-Modified`呢？答案是否定的。



引用MDN开发规范中的内容：



> **备注：** 在评估如何使用 `ETag` 和 `Last-Modified` 时，请考虑以下几点：在缓存重新验证期间，如果 `ETag` 和 `Last-Modified` 都存在，则 `ETag` 优先。因此，如果你只考虑缓存，你可能会认为 `Last-Modified` 是不必要的。然而，`Last-Modified` 不仅仅对缓存有用；相反，它是一个标准的 HTTP 标头，内容管理 (CMS) 系统也使用它来显示上次修改时间，由爬虫调整爬取频率，以及用于其他各种目的。所以考虑到整个 HTTP 生态系统，最好同时提供 `ETag` 和 `Last-Modified`。	



### 强制重新验证

如果你是个喜新厌旧的人，不希望重复使用响应，而是希望始终从服务器获取最新内容，HTTP提供了`no-cache`指令强制验证。



通过在响应中添加 `Cache-Control: no-cache` 以及 `Last-Modified` 和 `ETag`——如下所示——如果请求的资源已更新，客户端将收到 `200 OK` 响应，否则，如果请求的资源尚未更新，则会收到 `304 Not Modified` 响应。



```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
Date: Tue, 22 Feb 2022 22:22:22 GMT
Last-Modified: Tue, 22 Feb 2022 22:00:00 GMT
ETag: deadbeef
Cache-Control: no-cache

<!doctype html>
…
```



`max-age=0` 和 `must-revalidate` 的组合与 `no-cache` 具有相同的含义。



```http
Cache-Control: max-age=0, must-revalidate
```



`max-age=0` 意味着响应立即过时，而 `must-revalidate` 意味着一旦过时就不得在没有重新验证的情况下重用它——因此，结合起来，语义似乎与 `no-cache` 相同。



然而，`max-age=0` 的使用是解决 HTTP/1.1 之前的许多实现无法处理 `no-cache` 这一指令——因此为了解决这个限制，`max-age=0` 被用作解决方法。



但是现在符合 HTTP/1.1 的服务器已经广泛部署，没有理由使用 `max-age=0` 和 `must-revalidate` 组合——你应该只使用 `no-cache`。



### 不使用缓存

`no-cache` 指令不会阻止响应的存储，而是阻止在没有重新验证的情况下重用响应。



如果你不希望将响应存储在任何缓存中，应使用 `no-store`。



```http
Cache-Control: no-store
```



### 重新加载

为了从页面错误中恢复或更新到最新版本的资源，浏览器为用户提供了重新加载功能。



在浏览器重新加载期间发送的 HTTP 请求的简化视图如下所示：



```http
GET / HTTP/1.1
Host: example.com
Cache-Control: max-age=0
If-None-Match: "deadbeef"
If-Modified-Since: Tue, 22 Feb 2022 20:20:20 GMT
```



请求中的 `max-age=0` 指令指定“重用 age 为 0 或更少的响应”——因此，中间存储的响应不会被重用。



请求通过 `If-None-Match` 和 `If-Modified-Since` 进行验证。



### 强制重新加载

出于向后兼容的原因，浏览器在重新加载期间使用 `max-age=0`——因为在 HTTP/1.1 之前的许多过时的实现中不理解 `no-cache`。但是在这个用例中，`no-cache` 已被支持，并且**强制重新加载**是绕过缓存响应的另一种方法。



浏览器**强制重新加载**期间的 HTTP 请求如下所示：



```http
GET / HTTP/1.1
Host: example.com
Pragma: no-cache
Cache-Control: no-cache
```



### 避免重新验证

永远不会改变的内容应该被赋予一个较长的 `max-age`，方法是使用缓存破坏——也就是说，在请求 URL 中包含版本号、哈希值等。



但是，当用户重新加载时，即使服务器知道内容是不可变的，也会发送重新验证请求。



为了防止这种情况，`immutable` 指令可用于明确指示不需要重新验证，因为内容永远不会改变。



```http
Cache-Control: max-age=31536000, immutable
```



这可以防止在重新加载期间进行不必要的重新验证。



### 举个栗子

如下图请求网站https://seeyourface.cn，第一次请求我们注意服务器返回的响应内容：

HTTP Code : 200 —— 没有使用强制缓存，同时也没有进行协商缓存。

Etag —— 协商缓存用到的标头，缓存失效时下一个请求在`If-None-Match`中携带缓存中Etag内容，服务器经过验证后，如果没有对资源进行修改，返回304 Not Modified。

![image-20230928000111033](https://blog.seeyourface.cn/blog/image-20230928000111033.png)





我们刷新浏览器再次请求，状态码确实是304，我这个请求是在600秒之内刷新的，那为什么第一次的请求标头中的 `Cache-Control : max-age=600` 字段没有生效呢，不应该是通过强制缓存直接重用响应吗？



这是因为请求中携带了 `Cache-Control : max-age=0` ，请求中的 `max-age=0` 指令指定“重用 age 为 0 或更少的响应”——因此，我们存储的响应不会被重用。而是通过 `If-None-Match` 和 `If-Modified-Since` 进行验证。所以请求标头`If-None-Match`中携带了上一次响应的`Etag`字段内容，服务器经过验证，认为该资源自上次请求以来没有被修改，所以返回304状态码，客户端可重用已存储的响应。



![image-20230928000248303](https://blog.seeyourface.cn/blog/image-20230928000248303.png)