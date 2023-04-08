# 反向代理学习笔记(一) Nginx与反向代理绪论

> 本来是只想学习Nginx的，但是想来只学Nginx学过来有些狭窄，因为现在反向代理服务器

[TOC]

## 什么是代理？

作为一个沪漂程序员，听到代理这个词，我下意识的想到了中介，现在在上海已经很少能找到房东看房子了，基本上都是从中介那里看房，从这个角度来说中介代理了房东的部分职责，带你看房。在汉语词典中也是这么阐释的，代他人处理事务。那我们给代理前面加上正向这个词呢，这也就成为了计算机领域内的专有名词，正向代理。

### 什么是正向代理？

那什么是正向代理呢，我想起大学的上机课，为了让我们好好学习，上机课的电脑是无法看到一些娱乐网站的, 比如知乎、腾讯游戏。因为有同学确实在课上打游戏，丝毫不听课。学校好像意识到了问题的不对，加上了限制，再一次上课的我们，就发现一些娱乐网站上不了了。是通过网络白名单来进行的限制，那有什么方法能够绕开这个限制呢，答案就是代理，上网的时候，流量先到达我们的代理服务器，获取到内容之后，再给我们。像下面这样:

![iDXFGH.jpeg](https://i.328888.xyz/2023/03/26/iDXFGH.jpeg)

这是代理服务器的第一个用处，绕开浏览限制，一些学校和其他组织使用防火墙来使用户访问受限制的互联网。 代理服务器可以用于绕开这些限制，因为它们使用户可以连接到代理服务器，而不是直接连接到它们正在访问的沾点。与之相对，我们也可以使用代理服务器来组织特定用户群访问某些站点，比如学校网络可能配置为通过启动内容筛选规则的代理连接到Web，不在白名单的，拒绝转发响应。某些场景下，互联网用户也希望保护自己的在线身份，在一些情况下，互联网用户希望增强其在线匿名性，比如不想显示自己的真实ip，如果我们用代理服务器来访问互联网，发表评论的时候就更难追溯到真实ip。这是正向代理，我们可以认为正向代理，代理的是人。

### 什么是反向代理?

那反向代理呢，反向代理代理的则是服务器，有的时候我们并不愿意将我们的服务端应用完整的暴露出去，这会增大服务端被攻击的风险，这也就是反向代理服务器，如下图所示：

![No alt text provided for this image](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fbucketeer-e05bbc84-baa3-437e-9518-adb32be77984.s3.amazonaws.com%2Fpublic%2Fimages%2F257642d6-9742-432b-9ca8-2a866dea04dd_1445x1536.jpeg)

图片来自ByteByteGo。反向代理服务器可以帮助我们保护服务端，同时也可以帮我们做负载均衡，缓存静态资源，加密和解密SSL请求。本篇我们的重头戏就是反向代理服务器。

## 反向代理服务器

想我刚实习的时候，因为会的比较少，觉得自己用的技术比较没有逼格，听到Nginx、Redis这些名词的时候，心里总是有几分敬畏，觉得很有感觉。那反向代理服务器有哪些呢？ 比较知名的是Nginx，网站用的什么服务器处理请求，有些网站可以在响应中看到，我们首先看下掘金有没有声明自己用的是什么服务器处理的请求: 

![i4x85F.jpeg](https://i.328888.xyz/2023/03/25/i4x85F.jpeg)



是Nginx，本来我以为反向代理服务器会使用一个，我扒了扒其他请求，发现不是:

![i4xgDJ.jpeg](https://i.328888.xyz/2023/03/25/i4xgDJ.jpeg)

这个Tengine是啥，我们搜索一下:

![i4xx3c.jpeg](https://i.328888.xyz/2023/03/25/i4xx3c.jpeg)

原来是基于Nginx的HTTP服务器，我们进他的官网看看:

![i4x66V.jpeg](https://i.328888.xyz/2023/03/25/i4x66V.jpeg)

在原来Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性。那可以理解为Nginx Plus嘛，那我们自然就会有一个问题，既然Tengine比Nginx更强大，为什么没有取代Nginx呢?  写到这里，突然想起一句话，物竞天择，适者生存。不是更强大的完全就能完全取代弱的，合适的，适合环境的，自然能够生存下来。 bing还贴心的搜出了下面这个问题:

>  既然 Tengine 比 Nginx 更强大，为什么没有取代 Nginx 呢？

这个问题来自于知乎，下面节选一下我认为比较不错的回答:

> **开源软件的模式从来不是谁取代谁，而是百花齐放，博采众长。若意见不合，也欢迎自立门户。**

写到这里，想起知乎之前看到的一个问题: 

> Cloudflare弃用NGINX，改用Rust编写的Pingora，你怎么看？

还真是白花齐放哈，又来了一个Pingora，对于Pingora，我们目前就姑且理解到另一个反向代理服务器吧，关于Cloudflare为什么用Pingora取代Rust，参看下面这篇文章:  https://blog.cloudflare.com/zh-cn/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet-zh-cn/。

在上面的知乎问答中，我又看到了其他方向代理服务器:

- BFE 

> BFE (Beyond Front End) 是百度开源的现代化、企业级的七层负载均衡系统。 使用Go编写。

对应的文章参看:  为什么BFE可以取代Nginx：十问十答 https://zhuanlan.zhihu.com/p/533272410

- Higress

> Higress 是基于阿里内部两年多的 Envoy Gateway 实践沉淀，以开源 Istio 与 Envoy 为核心构建的下一代云原生网关。Higress 实现了安全防护网关、流量网关、微服务网关三层网关合一，可以显著降低网关的部署和运维成本。

这些只做了解，我们的重头戏还在Nginx上。

## Nginx 入门

本篇的定位是入门，会简单使用即可。上面的的了解是从宏观上认识的Nginx，现在让我们走进Nginx官网，去看看Nginx。

> nginx [engine x] is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server, originally written by  Igor Sysoev. For a long time, it has been running on many heavily loaded Russian sites including Yandex  Mail.Ru , VK , and Rambler . According to Netcraft, nginx served or proxied  21.23% busiest sites in February 2023 . Here are some of the success stories: Dropbox , Netflix, Wordpress.com, FastMail.FM
>
> nginx(engine x的缩写)是一个HTTP代理服务器，邮箱代理服务器，通用的TCP/UDP代理服务器。原作者为Igor Sysoev。长期以来，工作在一些负载很重的俄罗斯网站，包括Yandex  、Mail.Ru、VK 、Rambler 。根据Netcraft的调研报告显示，2023年2月，最繁忙的网站中，21.23%都由Nginx提供服务或进行代理。

这里做到对Nginx有个基本印象，其他特性我们放到后面来讲，我们现在首先将Nginx装起来，我们依然在官网寻找，看看有没有写好的命令，让我们复制一下就可以让Nginx跑起来的命令。

![i4d8Hc.jpeg](https://i.328888.xyz/2023/03/25/i4d8Hc.jpeg)

![i4dIeA.jpeg](https://i.328888.xyz/2023/03/25/i4dIeA.jpeg)

很全面，我们就跟着文档走就行了:

```shell
# 安装Nginx
sudo yum install yum-utils
cd /etc/yum.repos.d
# 创建Nginx.repo
touch nginx.repo
# 然后在Nginx.repo里面添加配置
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
# 然后执行下面命令
yum install nginx
```

执行完上面的命令，我发现我把Nginx安装到了etc目录下，那如何确定安装文件的位置呢:

```shell
# 执行下面命令，会找到安装的文件名
rpm -qa|grep nginx
rpm -ql  上一步找到的文件名
```

一般在Linux上安装第三方软件，我们推荐安装在/opt下面，现在让我们给Nginx挪一挪窝:

```shell
mv /etc/nginx  /opt/nginx
```

挪过去之后发现，启动不行，我又挪回去了。这里先不深究了。

我们现在来简单的看下Nginx的目录结构：

![i4sifd.jpeg](https://i.328888.xyz/2023/03/25/i4sifd.jpeg)

貌似唯一认识的就是nginx.conf，反向代理和负载均衡应该就是在这里配置的，要不，我们先让Nginx启动下，给自己一点成就感: 

```shell
nginx 
```

默认配置中，Nginx会自动在80端口（HTTP）上提供一个欢迎页面，如果能够成功访问该页面，则表示Nginx已经成功启动。像下面这样：

![i4soEw.jpeg](https://i.328888.xyz/2023/03/25/i4soEw.jpeg)

接着我们来配置反向代理和负载均衡，首先我是一名Java程序员，我有一个服务端项目，我现在做了集群，一共是两个节点，我希望为这两个节点提供负载均衡和反向代理, 那应该怎么配置呢？这也就是反向代理。我们打开nginx下面的nginx.conf配置文件

```nginx
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
	# 日志格式 
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    # 日志所在位置
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    #gzip  on;
    #包含/etc/nginx/conf.d/路径下所有后缀为.conf的配置文件 
    include /etc/nginx/conf.d/*.conf;
}        
```

在/etc/nginx/conf.d有一个配置文件叫default.conf，里面是一个server{}格式的代码块，所以在Nginx的基本配置格式:

```nginx
http{
	server { 

	}
}
```

处理请求首先要声明你要处理什么请求，比如我们有个应用驻留在8080端口，根路径是test，所以所有请求根路径的最前缀是test，都转发到这个应用。

### 反向代理

那我们应该在这个配置文件的http语法块下写一个server语法块:

```nginx
server{
        listen  8080;
        location /test{
              proxy_pass  http://192.168.2.2:8080/test;
        }
}        
```

nginx暂时监听的是本机，那么所有本机ip，端口8080，根路径是test的请求都会被转发到http://192.168.2.2:8080/test。然后nginx -s reload 重新加载配置文件即可。反向代理好了，现在我们来做负载均衡。

### 负载均衡

Nginx支持的负载均衡策略：

- round-robin — requests to the application servers are distributed in a round-robin fashion,

> 请求被被应用循环接受

- least-connected — next request is assigned to the server with the least number of active connections,

> 当一个新的请求与需要被处理时，系统会查找所有的服务器，并且选择当前连接数量最少的服务器来处理这个请求。

- ip-hash — a hash-function is used to determine what server should be selected for the next request (based on the client’s IP address).

> 根据客户端的ip地址，通过hash算法来决定应该有哪个服务器处理请求。

基本语法:

```nginx
 upstream myapp1 {
       # ip_hash;这里声明负载均衡算法,候选值有:ip_hash、least_conn
       # 默认是轮询	
       server 192.168.2.2:8080;
       server 192.168.2.2:8081;
}
    server{
        listen  8080;
        location /test{            
              proxy_pass  http://myapp1/test;
        }
 }
```

之间的映射关系如下图所示:

![iDmzfo.jpeg](https://i.328888.xyz/2023/03/26/iDmzfo.jpeg)

nginx 支持热加载，语法如下:

```nginx
nginx -s signal
signal 的取值为: stop、quit、reload、reopen。 
stop — fast shutdown 快速停止
quit — graceful shutdown 优雅的停止
reload — reloading the configuration file 再次加载配置文件
reopen — reopening the log files 再次打开日志文件
```

## 总结一下

到现在为止，我们对正向代理、反向代理已经有了一个大致的印象，并且我们能够使用Nginx来搭建负载均衡和反向代理。本来还打算介绍一下Nginx作为静态资源的服务器，发现映射过去老是403，折腾良久，遂放弃。

## 参考资料

- Why is Nginx called a “reverse” proxy?  https://blog.bytebytego.com/p/ep25-proxy-vs-reverse-proxy#%C2%A7why-is-nginx-called-a-reverse-proxy
- 为什么BFE可以取代Nginx？ https://zhuanlan.zhihu.com/p/528485453
- 深入剖析Nginx负载均衡算法 https://www.taohui.tech/2021/02/08/nginx/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Nginx%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E7%AE%97%E6%B3%95/
