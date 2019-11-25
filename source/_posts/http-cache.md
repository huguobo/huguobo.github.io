---
title: 不懂就问系列-浏览器里到底有多少种缓存机制
date: 2019-11-25 17:07:44
categories: 
- HTTP
tags:
- cache
- expires
- cache-control
- last-modified
- etag
- 不懂就问
---

> 对于Web应用来说，对于网络的的依赖性很公安，性能瓶颈一般都在于如果减少IO消耗，这时候一般都会想起使用缓存，但是听说过 cache-control，expires 还有 Etag等名词的缓存策略，他们又分别代表什么呢，抓紧时间总结一下

可以为强缓存和协商缓存，优先级较高的是强缓存，在命中强缓存失败的情况下，才会走协商缓存。

## 强缓存机制
关键词： `Cache-Control` 和 `Expires`

### Expires
当服务器返回响应时，在 Response Headers 中将过期时间写入 expires 字段。
```
expires: Mon, 11 Nov 2019 16:13:18 GMT
```
可以看到expires的值为一个时间戳，如果接下来的请求早于这个时间，浏览器不会再次跟服务器请求而是直接从缓存中返回，返回状态码是 200。

但是由于expires涉及本地时间设置不同问题，可能造成机制的不准确。

### Cache-Control
因此 cache-control 就出现了
```
cache-control: max-age=31536000
```
cache-control的max-age使用一个相对时间（单位：秒）控制资源的有效期，它表示在收到该资源 31536000 秒内，会使用本地的强缓策略。

### no-store与no-cache
no-cache 忽视了浏览器：为资源设置了 no-cache 后，每一次发起请求都不会再去询问浏览器的缓存情况，而是直接向服务端去确认该资源是否过期，走到协商缓存的阶段。

no-store 全部忽视，不适用任何缓存策略，每次都是重新请求。


## 协商缓存
协商缓存依赖于服务端与浏览器之间的通信。
关键词： `Last-Modified` 和 `Etag`

### Last-Modified
Last-Modified的值为一个时间戳，会随着 Response Header 返回：
```
Last-Modified: Mon, 11 Nov 2019 16:13:18 GMT
```
之后的每次请求，Request Header  会带上一个 `If-Modified-Since` 的Header，他的值是上一次 response 返回的 last-modified 的值。
```
If-Modified-Since: Mon, 11 Nov 2019 16:13:18 GMT
```

这样服务端可以对比时间和服务器上的最后修改时间是否一致，来决定是否需要重新返回资源，如果未更新，直接返回304，此时的返回头中也不会有 last-modified 字段。如果更新正常返回并更新返回last-modified 的值。

这样一来就实现了协商缓存，但是使用时间戳还是会有一些问题，因为如果编辑了文件但是没有实际内容的更改，last-modified会更新，但是其实文件酶标幺和从新请求，所以我们需要一个能表明文件是否真实更新的字段，类似与内容摘要的东西。

### Etag

Etag 和 Last-Modified 类似，当首次请求时，我们会在响应头里获取到一个最初的标识符字符串，它可以是这样的：
```
ETag: W/"2s3s-2602480f239"
```

那么下一次请求时，请求头里就会带上一个值相同的、名为 if-None-Match 的字符串供服务端比对了：
```
If-None-Match: W/"2s3s-2602480f239"
```
但是 Etag 的生成需要额外的开销，会在一定程度上影响服务端的性能。所以Etag不能完全代替Last-Modified。

## Memory Cache 和 Disk Cache

从 network调试中发现命中浏览器缓存的资源有的是 from memory cache，有的是 from disk cache，甚至还有 from service worker.

### memory cache
指的是在内存中的缓存，读取速度应该是最快的，优先级也是最高的。

### disk cache
磁盘内存

### service worker
我们借助 Service worker 实现的离线缓存就称为 Service Worker Cache。需要注意的是 Server Worker 对协议是有要求的，必须以 https 协议为前提。
