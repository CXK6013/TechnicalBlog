# 异步Servlet学习笔记(一)

> 两周没更新了，感觉不写点什么，有点不舒服的感觉。

## 前言

回忆一下学Java的历程，当时是看JavaSE(基本语法、线程、泛型)，然后是JavaEE，JavaEE也基本就是围绕着Servlet的使用、JSP、JDBC来学习，当时看的是B站up主颜群的教学视频:

- JavaWeb视频教程（JSP/Servlet/上传/下载/分页/MVC/三层架构/Ajax）https://www.bilibili.com/video/BV18s411u7EH?p=6&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799

现在一看这个播放量破百万了，当初我看的时候应该播放量很少，现在这么多倒是有点昨舌。学完了这个之后，开始学习框架：Spring、SpringMVC、MyBatis、SpringBoot。虽然Spring MVC本质上也是基于Servlet做封装，但后面基本就转型成Spring 工程师了，最近碰到一些问题，又看了一篇文章，觉得一些问题之前自己还是没考虑到，颇有种离了Spring家族，不会写后端一样。本来今天的行文最初是什么是异步Servlet，异步Servlet该如何使用。但是想想没有切入本质，所以将其换成了对话体。

## 正文

我们接着有请之前的实习生小陈，每当我们需要用到对话体、故事体这样的行文。实习生小陈就会出场。今天的小陈呢觉得行情有些不好，但是还是觉得想出去看看，毕竟金三银四，于是下午就向领导请假去面试了。进到面试的地方，一番自我介绍，面试官首先问了这样一个问题:

> 一个请求是怎么被Tomcat所处理的呢？

小陈回答到:

> 我目前用的都是Spring Boot工程，我看都是启动都是在main函数里面启动整个项目的，而main函数又被main线程执行，所以我想应该是请求过来之后，被main线程所处理，给出响应的。

面试官:

> ╮(╯▽╰)╭，main函数的确是被main线程执行，但都是被main线程处理的？ 这不合理吧，假设某个请求占用了main线程三秒，那这三秒内，系统都无法再回应请求了。你要不再想想？

小陈挠了挠头，接着答到:

> 确实是，浏览器和Tomcat通讯用的是HTTP协议，我也学过网络编程，所以我觉得应该是一个线程一个请求吧。像下面这样:

```java
public class ServerSocketDemo {

    private static final ExecutorService EXECUTOR_SERVICE = Executors.newFixedThreadPool(4);

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        while (true){
            // 一个socket对象代表一个链接
            // 等待TCP连接请求的建立,在TCP连接请求建立完成之前,会陷入阻塞
            Socket socket = serverSocket.accept();
            System.out.println("当前链接建立:"+ socket.getInetAddress().getHostName()+socket);
            EXECUTOR_SERVICE.submit(()->{
                try {
                    // 从输入流中读取客户端发送的内容
                    InputStream inputStream = socket.getInputStream();
                    // 从输出流里向客户端写入数据
                    OutputStream outPutStream = socket.getOutputStream();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }
    }
}
```

> serverSocket的

面试官点了点头, 接着问到:

> 你这个main线程负责检测连接是否建立，然后建立之后将后续的业务处理放入线程池，这个是NIO吧。

小陈笑了笑说道:

> 虽然我对NIO了解不多，但这应该也不是NIO，



Tomcat How To Work





## 参考资料

- Java NIO浅析 https://zhuanlan.zhihu.com/p/23488863



