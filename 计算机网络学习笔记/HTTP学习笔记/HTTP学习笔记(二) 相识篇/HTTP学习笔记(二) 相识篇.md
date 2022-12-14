# HTTP学习笔记(二) 相识篇

[TOC]

## 前言

最初的HTTP协议是无状态的，也许在设计者Tim Berners-Lee就应该是无状态的，本身只是为分散在网络上的资料建立连接而已，在同一个连接中，两个执行成功的请求之间并没有什么联系。随着万维网的发展, 出现了各种各样的网站，慢慢的有些网站需要识别用户的身份，一个非常典型的场景就是网购，在大型超市中买东西，超市会为客户提供购物车，顾客在不同的商品区购物放在购物车，最终统一结账，但是由于HTTP的无状态，我们无法在线上实现购物车的效果，由此我们引出Cookie，Cookie的出现让HTTP有了状态，但是有了Cookie还不够，我们还希望对不同的用户实现不同的权限控制，由此我们引出HTTP的认证。随着网络的不断发展，服务器的压力在不断的增加，单台机器慢慢无法应对与日俱增的访问量，由此我们引出了代理服务器。

## Cookie概述

HTTP Cookie是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时并发送到服务器上，用来告知服务端两个请求来自同一个浏览器，保持用户状态等。

一般来说，Cookie主要用于以下三个方面:

- 会话状态管理(如用户登录状态、购物车等其他需要记录的信息)
- 浏览器行为跟踪(如跟踪分析用户行为等)

在过去由于没有其他合适的存储办法，Cookie一度用于客户端数据存储，但是随着现代浏览器的发展，浏览器开始支持各种各样的存储方式，新的浏览器API已经允许开发者直接将数据存储到本地，如使用 Web storage API 或 IndexedDB, 有了更多选择，Cooke渐渐的就被淘汰，浏览器支持更多的存储方式并不是Cookie唯一被淘汰的理由，还有一个就是性能问题，由于服务器指定Cookie之后，浏览器的每次请求都会携带Cookie数据，会带来额外的性能开销(尤其是在移动环境下)

当服务器收到HTTP的请求时，服务器可以在响应头里面添加一个Set-Cookie选项，一个简单的Cookie可能像下面这样:

```http
Set-Cookie: <cookie名>=<cookie值>
```

浏览器收到响应后通常会保存下Cookie，之后对该服务器每一个请求中都通过Cookie请求头部将Cookie信息发送给服务器。另外,Cookie的过期时间、域、路径、有效期、适用站点都可以根据需要来指定。

但是Cookie也不能永远保存，由此我们就引出了定义Cookie的生命周期, Cookie的生命周期有以下两种

- 会话期Cookie，这是最简单的Cookie，浏览器关闭之后会被自动删除，会话期Cookie不需要指定过期时间(Expires)或者有效期(Max-Age).但是值得注意的是，有些浏览器提供了会话恢复功能，这种情况下，即使关闭了浏览器，会话器Cookie也会被保留下来，就好像浏览器从来没有关闭一样，这就会导致Cookie的生命周期无限期延长。
- 持久性Cookie的生命周期取决于过期时间(Expires) 或有效期(Max-Age)指定的一段时间。

## 认证概述

作为一个后端仔，经常和postman打交道，如果你注意的话，会发现postman除了请求头，请求参数、请求体。还有一个Authorization。如下图所示:

![认证](http://tvax3.sinaimg.cn/large/006e5UvNgy1h2ny76oxh6j30y00h477n.jpg)



事实上认证也是HTTP协议标准的一部分，在RFC-7235引入，该草案定义了一个HTTP身份验证框架，服务器可以用来针对客户端的请求发送质询信息，客户端则可以用来提供身份验证凭证。质询与应答的工作流程如下: 服务端向客户端返回401(未授权)，并在WWW-Authenticate首部提供如何进行验证的信息，其中至少包含有一种质询方式。之后客户端可以在新的请求中添加Authorization首部字段进行验证，字段值为身份验证凭证信息。

上面我们描述的RFC-7235我们可以理解为一个标准，在这个标准之下，有多种方案。IANA维护了一系列验证方案，除此之外还有其他的验证方案由虚拟主机服务提供，例如Amazon AWS。常见的验证方案如下：

- Basic (RFC 7617 引入 base64编码凭证)
- Bearer(RFC 6750 引入 bearer领票通过Auth 2.0保护资源)
- Digest(RFC 7616 引入 只有md5散列在Firefox中支持, 有bug，bug编号为472823用于SHA加密支持)
- HOBA(由RFC 7486(目前还是草案)，HTTP Origin-Bound认证，基于数字签名)

## 代理服务器概述

上面我们提到了随着万维网的不断发展，单台计算机渐渐的无法满足与日俱增的访问量，简单而粗暴的解决这个问题的策略就是再加一台计算机，除此之外开发者们还发现服务器上的一些资源是静态的或者是很少发生改变的，将这类资源单独放到一台服务器上也能提升系统的响应能力，于是这里出现了代理服务器。存储静态资源，同时做负载均衡，由代理服务器将请求分发到集群中的服务器上。

![代理服务器](http://tva4.sinaimg.cn/large/006e5UvNgy1h2nz03widkj30ub0h741b.jpg)

说到这里，你一定想起了Nginx吧。





## 参考资料

- 火狐开发者文档 https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/DELETE
- bug编号472823 https://bugzilla.mozilla.org/show_bug.cgi?id=472823
