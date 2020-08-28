---
title: 一篇文章带你理解HTTP缓存
date: 2019-05-18 22:44:01
tags: HTTP
categories:
  - HTTP
---
## 为什么会有HTTP缓存？

HTTP缓存的存在是因为web前端的性能瓶颈大部分的原因在于HTTP传输的时间耗费过长。如果能够减少这种HTTP请求的时间，对网页的性能来说是非常大的提升，对于用户的体验也能得到极大的改善。

## HTTP缓存标识

HTTP缓存可分为<font color=blue>强缓存</font>（Cache-Control和Expires）以及<font color=blue>协商缓存</font>（Etag和Last-Modified）。

强缓存和协商缓存的区别在于：如果命中强缓存，会直接从缓存中读取资源，不向服务器请求。协商缓存则会向服务器请求确认资源是否过期。这也是缓存验证的顺序，先使用强缓存，后使用协商缓存。

下面则简单介绍一下这两种缓存类型的标识

### Cache-Control

Cache-Control设置有效期max-age的值是时间的相对值。

下面则简单介绍Cache-Control常用的使用值：

| value               | description                                                  |
| ------------------- | ------------------------------------------------------------ |
| public              | 表明响应可以被任何对象（包括：发送请求的客户端，代理服务器，等等）缓存。 |
| private             | 表明响应只能被单个用户缓存，不能作为共享缓存（即代理服务器不能缓存它）,可以缓存响应内容。 |
| no-cache            | 在发布缓存副本之前，强制高速缓存将请求提交给原始服务器进行验证。 |
| no-store            | 缓存不应存储有关客户端请求或服务器响应的任何内容。           |
| max-age=\<seconds\> | 设置缓存存储的最大周期，超过这个时间缓存被认为过期(单位秒)。与Expires相反，时间是相对于请求的时间。 |
| must-revalidate     | 缓存必须在使用之前验证旧资源的状态，并且不可使用过期资源。   |
| proxy-revalidate    | 与must-revalidate作用相同，但它仅适用于共享缓存（例如代理），并被私有缓存忽略。 |

若想了解更详细的说明，可参考此链接：[https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control)

### Expires

Expires 响应头包含日期/时间， 即在此时候之后，响应过期。

Expires设置的值是一个绝对值。

```
Expires: Wed, 21 Oct 2015 07:28:00 GMT
```

### Etag

Etag HTTP响应头是资源的特定版本的标识符。这可以让缓存更高效，并节省带宽，因为如果内容没有改变，Web服务器不需要发送完整的响应。而如果内容发生了变化，使用ETag有助于防止资源的同时更新相互覆盖（“空中碰撞”）。

Etag的值是根据资源文件内容生成的一个hash值。

```
Etag: 33a64df551425fcc55e4d42a148795d9f25f89d4
```

### Last-Modified

Last-Modified包含源头服务器认定的资源做出修改的日期及时间。 它通常被用作一个验证器来判断接收到的或者存储的资源是否彼此一致。由于精确度比  ETag 要低，所以这是一个备用机制。

Last-Modified的值是一个时间的绝对值。

```
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

## HTTP缓存验证规则

### HTTP缓存的验证顺序

1. 先使用强缓存（Cache-Control > Expires） 判断资源是否过期；
2. 若资源未过期则直接从缓存读取，若资源过期则使用协商缓存；
3. 使用协商缓存（Etag > Last-Modified）请求服务器确认资源有无修改（请求头会带上 If-None-Match 或 If-Modified-Since）；
4. 若资源无修改则返回 <code>304</code> 状态码告诉客户端可读取缓存；若资源已修改则返回 <code>200</code> 状态码重新获取资源；

图示：

![](https://user-gold-cdn.xitu.io/2019/5/18/16ac9fe6feb72d69?w=1398&h=1298&f=png&s=344446)
图片来源：[https://www.cnblogs.com/xiaohuochai/p/6178810.html](https://www.cnblogs.com/xiaohuochai/p/6178810.html)

### HTTP缓存验证说明

以下的HTTP缓存验证说明是基于假设请求存在4个缓存头标记；

1. HTTP缓存是否过期，先判断Cache-Control是否设置<code>max-age</code>或<code>s-max-age</code>，如果已设置则忽略Expires并且验证是否过期，否则验证Expires是否过期；其实按照MDN文档的说明，Last-Modified也是可以计算出一个缓存时间；

下面是MDN文档的说明：

> 对于含有特定头信息的请求，会去计算缓存寿命。比如Cache-control: max-age=N的头，相应的缓存的寿命就是N。通常情况下，对于不含这个属性的请求则会去查看是否包含Expires属性，通过比较Expires的值和头里面Date属性的值来判断是否缓存还有效。如果max-age和expires属性都没有，找找头里的Last-Modified信息。如果有，缓存的寿命就等于头里面Date的值减去Last-Modified的值除以10（注：根据rfc2626其实也就是乘以10%）。

缓存时间计算公式：

> expirationTime = responseTime + freshnessLifetime - currentAge

2. 如果该缓存未过期，是不会向服务器发送请求的，而是直接从缓存中读取，也就是我们可以从HTTP请求信息里得到的状态码 <code> 200 (from cache memory || from disk memory) </code>，如果缓存过期，则使用协商缓存；

- 先尝试从内存中读取，再尝试从硬盘中读取。
- 200 form memory cache 不访问服务器，一般已经加载过该资源且缓存在了内存当中，直接从内存中读取缓存。浏览器关闭后，数据将不存在（资源被释放掉了），再次打开相同的页面时，不会出现from memory cache。
- 200 from disk cache 不访问服务器，已经在之前的某个时间加载过该资源，直接从硬盘中读取缓存，关闭浏览器后，数据依然存在，此资源不会随着该页面的关闭而释放掉下次打开仍然会是from disk cache。

3. 使用协商缓存验证缓存资源是否修改；<font color=red>Etag > Last-Modified</font>，当存在Etag则使用Etag，否则使用Last-Modified；并且请求时会带上特定的标识；

- Etag 请求时会带上 If-None-Match
- Last-Modified 请求时会带上 If-Modified-Since

4. 当确认资源未修改时则返回 <code> 304 (Not Modified)</code>告诉客户端可以继续使用缓存，否则返回<code>200</code>重新获取新的资源。

HTTP缓存早期的时候只有Expires和Last-Modified，为什么后面又会出现Cache-Control和Etag呢？

先说说Expires和Cache-Control，Expires的值是一个准确的时间，比较的时候先根据返回头的Date值比较判断，但是若无Date头信息返回则是根据客户端的本地时间进行比较；本地时间因为各种因素影响，会存在各种不同的值，导致Expires缓存的时间并不是我们想要的效果。而Cache-Control设置max-age使用的相对值则相对来说控制粒度更精确了；并且Cache-Control的其他值则让我们对于缓存的控制更加灵活。

再说说Last-Modified和Etag，Last-Modified的值也是一个准确的时间，精确到秒；使用时间来判断资源是否修改则可能存在以下问题：

1. 1秒内的修改可能不被检查到，导致缓存无法更新；
2. 资源可能只是多了几个空格或无变化，但是Last-Modified的时间已经变化；

针对以上的问题所以有了Etag的存在，Etag是根据资源内容生成的hash值对比判断资源是否更新，控制粒度比Last-Modified更加精确。

## 总结

HTTP缓存的本质上是以空间换时间，缓存的存在是为了尽可能的减少HTTP请求次数和HTTP传输的内容大小，这也是前端性能优化中重要的一环。合理的设置页面资源的缓存，有助你提升页面的体验。
