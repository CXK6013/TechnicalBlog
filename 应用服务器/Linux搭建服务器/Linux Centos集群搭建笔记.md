# Linux Centos 搭建集群图文教程

> 本周写看线程池的源码、设计模式都卡文，但是想想自己还差几个中间件没学习，就打算先搭建一下这个Linux的服务环境。后面我们搭建集群、大数据， 都是用这个环境来搭建。

[TOC]



##   先装上Linux

我本机是windows，所以就需要一个Vm Ware，安装教程参看: https://zhuanlan.zhihu.com/p/272541376。这个教程非常详细。接下来我们需要操作系统的镜像，在搜索引擎中搜索Centos

![Centos地址](http://tvax3.sinaimg.cn/large/006e5UvNgy1h3chhfa7ksj313x0ipk3x.jpg)

![x86](http://tva2.sinaimg.cn/large/006e5UvNgy1h3chibsuwyj30vs0nvncc.jpg)

![下载镜像](http://tva3.sinaimg.cn/large/006e5UvNgy1h3chjccsxfj315v0b8wmc.jpg)

![Centos下载](http://tvax1.sinaimg.cn/large/006e5UvNgy1h3chlcl7afj30sb085jwb.jpg)

![新建虚拟机](http://tva3.sinaimg.cn/large/006e5UvNly1h3cn1ypfqnj313n0ksdn8.jpg)

![选择自定义](http://tvax4.sinaimg.cn/large/006e5UvNly1h3cn2ztznxj30vs0hwtdt.jpg)

![下一步](http://tva1.sinaimg.cn/large/006e5UvNly1h3cn417dz7j30kl0dumzs.jpg)

![稍后安装操作系统](http://tvax3.sinaimg.cn/large/006e5UvNly1h3cn5wer8oj30x60ic79i.jpg)

![选Linux](http://tvax1.sinaimg.cn/large/006e5UvNly1h3cn6otyutj30ub0geq71.jpg)

![选64位](http://tva3.sinaimg.cn/large/006e5UvNly1h3cn7lo311j30wl0h0n1m.jpg)

![搭建集群](http://tvax1.sinaimg.cn/large/006e5UvNly1h3cna16mqqj30wc0haaey.jpg)

![选择下一步](http://tvax2.sinaimg.cn/large/006e5UvNly1h3cnbyzwapj30n20f8jtm.jpg)

![给两个G](http://tva3.sinaimg.cn/large/006e5UvNly1h3cncydy8mj30qc0gxjvv.jpg)

![选NAT](http://tvax4.sinaimg.cn/large/006e5UvNgy1h3cnearoqwj30su0dwn0o.jpg)

![LSI Logic](http://tvax2.sinaimg.cn/large/006e5UvNgy1h3cnf6c74ej30ru0h7q6g.jpg)

![磁盘选择](http://tvax1.sinaimg.cn/large/006e5UvNgy1h3cng03ocaj30pm0f7q5a.jpg)

![创建虚拟磁盘](http://tva2.sinaimg.cn/large/006e5UvNgy1h3cnhit0o8j30o00eotbv.jpg)

![下一步 下一步](http://tva2.sinaimg.cn/large/006e5UvNgy1h3cnii34jjj30ol0e477m.jpg)

![下一步-下一步](http://tva3.sinaimg.cn/large/006e5UvNgy1h3cnj9f6t4j30np0foq5p.jpg)

![装好了](http://tvax3.sinaimg.cn/large/006e5UvNgy1h3cnk0z50nj30mz0e3adf.jpg)

![点这里](http://tvax4.sinaimg.cn/large/006e5UvNly1h3cnm71nc4j313h0kzahm.jpg)

![加载镜像](http://tvax1.sinaimg.cn/large/006e5UvNly1h3cnn3fo9xj30ro0ig0z2.jpg)

![加载ISO镜像](http://tva4.sinaimg.cn/large/006e5UvNly1h3cnnw1o7kj30px0ittcz.jpg)

![选中iso文件](http://tva2.sinaimg.cn/large/006e5UvNly1h3cnp6akt0j30qh0enwie.jpg)

![开启虚拟机](http://tvax4.sinaimg.cn/large/006e5UvNly1h3cnq729ujj313d0l4jyw.jpg)

进去之后选第一个就行。

![选中文](http://tva3.sinaimg.cn/large/006e5UvNgy1h3cnsm8ylij30oj0im790.jpg)

![选GNOME桌面](http://tva3.sinaimg.cn/large/006e5UvNgy1h3cpcomzfvj30ls0h9434.jpg)

![选择硬盘分区](http://tvax2.sinaimg.cn/large/006e5UvNgy1h3cpdzbd05j30rq0kk10z.jpg)

![选择手工分区](http://tvax2.sinaimg.cn/large/006e5UvNgy1h3cpfb1qa5j30md0jqagb.jpg)

![点完成](http://tva1.sinaimg.cn/large/006e5UvNgy1h3cpg707epj30m40j60yr.jpg)

![选择标准分区](http://tva4.sinaimg.cn/large/006e5UvNgy1h3cphhimfij30rk0lb10t.jpg)

![挂载分区](http://tva4.sinaimg.cn/large/006e5UvNgy1h3cpj0l0buj30rv0lfjzf.jpg)

![挂载10G](http://tvax4.sinaimg.cn/large/006e5UvNgy1h3cpkbg3dpj30r60ky7bo.jpg)

![启动盘给300M](http://tva4.sinaimg.cn/large/006e5UvNgy1h3cpm7nqf1j30mk0irjwh.jpg)

![swap=2048](http://tva2.sinaimg.cn/large/006e5UvNgy1h3cpp2fjglj30rk0latgk.jpg)

![内存配置完成](http://tvax3.sinaimg.cn/large/006e5UvNgy1h3cpqhk5t0j30rz0lowmy.jpg)

![分区完成](http://tva2.sinaimg.cn/large/006e5UvNgy1h3cprfj170j30m40hodkf.jpg)

![分区设计完成](http://tva1.sinaimg.cn/large/006e5UvNgy1h3cpt15t9ij30m70h3dkp.jpg)

![设置root密码](http://tva1.sinaimg.cn/large/006e5UvNgy1h3cpu78zvkj30m90gx418.jpg)

![等他装完](http://tva4.sinaimg.cn/large/006e5UvNly1h3cpxmlkr9j30mh0hptej.jpg)

![重启](http://tva2.sinaimg.cn/large/006e5UvNly1h3cq0opudlj30rp0lm10h.jpg)

![点击完成配置](http://tvax2.sinaimg.cn/large/006e5UvNgy1h3cq4gw5afj30zf0mhtdt.jpg)

![完成配置](http://tva1.sinaimg.cn/large/006e5UvNgy1h3cq5vmxqgj30zq0kegno.jpg)

接着点下一步，到时区那里选择上海。设置用户名，我们输入bigdata01。

![切换root用户登录](http://tva4.sinaimg.cn/large/006e5UvNly1h3cqdhizfyj30zu0kowq8.jpg)

![点这里切换用户](http://tvax1.sinaimg.cn/large/006e5UvNly1h3cqeuu3a9j30xh0ik49i.jpg)

![打开终端](http://tvax3.sinaimg.cn/large/006e5UvNly1h3cqfy7vcjj30q50es0yi.jpg)

## 布置网络

- 在终端中输入命令 hostname，查看主机名字

- 设置hostname: hostnamectl set-hostname bigdata01

- 网关统一设置为 192.168.2.1，后面搭建集群, 分配的IP为
  - 192.168.2.128 bigdata01
  - 192.168.2.129 bigdata02
  - 192.168.2.130 bigdata03

- 终端中输入halt，关机

- ![虚拟机网络编辑器](http://tva1.sinaimg.cn/large/006e5UvNgy1h3cqvoa1jkj30yv0lm0yf.jpg)![更改配置](http://tva1.sinaimg.cn/large/006e5UvNgy1h3dau00v21j30q80ivtct.jpg)

- ![NAT模式](http://tvax1.sinaimg.cn/large/006e5UvNly1h3dawifxe2j30so0jaaet.jpg)![设置子网](http://tva2.sinaimg.cn/large/006e5UvNly1h3db02g3mdj30ge0eowj3.jpg)

  ![点NAT设置](http://tva3.sinaimg.cn/large/006e5UvNly1h3db12no9kj30gi0emadp.jpg)![网关设置为](http://tvax2.sinaimg.cn/large/006e5UvNly1h3db235sllj30dw0ecmzz.jpg)![DHCP设置](http://tva4.sinaimg.cn/large/006e5UvNly1h3db334g3hj30g60ehn0m.jpg)

![128](http://tvax3.sinaimg.cn/large/006e5UvNly1h3db8htlbyj30d40fz0wa.jpg)

然后进入Linux终端, 我们配置一下网络.

```shell
cd  /etc/sysconfig/network-scripts/ 
vi ifcfg-ens33
将BOOTPROTO改为static
ONBOOT由no改为yes
新加配置
IPADDR=192.168.2.128
GATEWAY=192.168.2.1
BROADCAST=192.168.2.255
DNS1=114.114.114.114
DNS2=8.8.8.8
vi /etc/hosts 改下主机名 追加配置 192.168.2.128 bigdata01
service NetworkManager stop
/etc/init.d/network restart
chkconfig NetworkManager off
vi /etc/resolv.conf 追加 nameserver=192.168.2.1
systemctl restart network
然后ping www.baidu.com 这台虚拟机就能连互联网了
然后就可以通过XShell 连接虚拟机了
```

## 安装服务器上必要的软件

```shell
# 时间同步
yum -y install npt  ntpdate
ntpdate cn.pool.ntp.org
hwclock --systoch
yum install lrzsz
wget https://repo.huaweicloud.com/java/jdk/8u151-b12/jdk-8u151-linux-x64.rpm
rpm -ivh jdk-8u151-linux-x64.rpm
# 配置环境变量
vi /etc/profile
# 追加下面这样
export JAVA_HOME=/usr/java/jdk1.8.0_151
export CLASSPATH=$JAVA_HOMT\lib:$CLASSPATH
export PATH=$JAVA_HOME\bin:$PATH
source /etc/profile
```

那说好的集群呢？ 每一台都要这样来一次，太麻烦了吧？ VM Ware 有个复制功能，我们复制就行。复制之前先关机一下。

![克隆](http://tvax1.sinaimg.cn/large/006e5UvNly1h3dd4vmeatj30j30jq0xa.jpg)

![完整克隆](http://tvax4.sinaimg.cn/large/006e5UvNly1h3dd7adas6j316z0o3ag7.jpg)



然后选择文件存储位置就行了。克隆出来的还需要再改一下网络配置、hostname、域名映射。命令如下:

```shell
# 设置hostname: 
hostnamectl set-hostname bigdata01
cd  /etc/sysconfig/network-scripts/ 
vi ifcfg-ens33
将BOOTPROTO改为static
ONBOOT由no改为yes
新加配置
IPADDR=192.168.2.128
GATEWAY=192.168.2.1
BROADCAST=192.168.2.255
DNS1=114.114.114.114
DNS2=8.8.8.8
vi /etc/hosts 改下主机名 追加配置 192.168.2.128 bigdata01  192.168.2.129 bigdata02
# 记得bigdata01的主机也加下映射
service NetworkManager stop
/etc/init.d/network restart
chkconfig NetworkManager off
vi /etc/resolv.conf 追加 nameserver=192.168.2.1
systemctl restart network
然后这台虚拟机就能连互联网了
然后就可以通过XShell 连接虚拟机了
```

到此我们的集群基本搭建完毕了。