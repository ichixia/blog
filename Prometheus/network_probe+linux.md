本文描述在linux下，如果想要检测某些站点或者服务可用性的常用操作，包括
* Ping(ICMP协议)
* HTTP(HTTPS)
* DNS解析
* TCP，UDP  

#### Ping(ICMP协议)  
Ping是测试网络联接状况以及信息包发送和接收状况非常有用的工具，是网络测试最 常用的命令。Ping向目标主机(地址)发送一个回送请求数据包，要求目标主机收到请求后给予答复，从而判断网络的响应时间和本机是否与目标主机(地址)联通。  

如果执行Ping不成功，则可以预测故障出现在以下几个方面：网线故障，网络适配器配置不正确，IP地址不正确。如果执行Ping成功而网络仍无法使用，那么问题很可能出在网络系统的软件配置方面，Ping成功只能保证本机与目标主机间存在一条连通的物理路径。  

命令格式：  

```
ping IP地址或主机名 [-t] [-a] [-n count] [-l size]
```

参数含义：  
```
-t 不停地向目标主机发送数据；

-a 以IP地址格式来显示目标主机的网络地址 ；

-n count 指定要Ping多少次，具体次数由count来指定 ；

-l size 指定发送到目标主机的数据包的大小。
```

例如我们用下面的命令Ping一下某个网址或者ip  
```
ping www.baidu.com
ping 8.8.8.8
```

其测试结果如下所示：  
```
$ ping www.baidu.com
PING www.a.shifen.com (180.101.49.12) 56(84) bytes of data.
64 bytes from 180.101.49.12: icmp_seq=1 ttl=48 time=15.0 ms
64 bytes from 180.101.49.12: icmp_seq=2 ttl=48 time=15.0 ms
64 bytes from 180.101.49.12: icmp_seq=3 ttl=48 time=15.1 ms
64 bytes from 180.101.49.12: icmp_seq=4 ttl=48 time=15.0 ms
...

$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=48 time=33.3 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=48 time=33.3 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=48 time=33.3 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=48 time=33.4 ms
```

上面的结果都表示ping通了之后，目标主机给予的答复。

#### HTTP(HTTPS)  
在Linux下想要模拟发送http请求，常用的可以使用curl或者wget命令，这里我们使用curl来模拟发送http请求。  
curl是一种命令行工具，作用是发出网络请求，然后得到和提取数据，显示在"标准输出"（stdout）上面。  

##### 1. 查看网页源码  
直接在curl命令后加上网址，就可以看到网页源码。我们以网址www.sina.com为例  
```
curl www.sina.com
```

```
<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

##### 2. 自动跳转  
有的网址是自动跳转的。使用`-L`参数，curl就会跳转到新的网址。
```
curl -L www.sina.com
```
键入上面的命令，结果就自动跳转为www.sina.com.cn。

##### 3. 显示头信息
`-i`参数可以显示http response的头信息，连同网页代码一起。

```
curl -i www.sina.com
```

```
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Wed, 04 Dec 2019 09:29:32 GMT
Content-Type: text/html
Content-Length: 178
Connection: keep-alive
Location: http://www.sina.com.cn/
Expires: Wed, 04 Dec 2019 09:30:52 GMT
Cache-Control: max-age=120
X-Via-SSL: ssl.23.sinag1.qxg.lb.sinanode.com
Age: 40
Via: https/1.1 ctc.guangzhou.ha2ts4.182 (ApacheTrafficServer/6.2.1 [cRs f ]), https/1.1 ctc.jiangxi.ha2ts4.28 (ApacheTrafficServer/6.2.1 [cRs f ])
X-Via-Edge: 15754517722574caa85db50d81575479cf5bb
X-Cache: HIT.28
X-Via-CDN: f=edge,s=ctc.jiangxi.ha2ts4.28.nb.sinaedge.com,c=219.133.170.76;f=Edge,s=ctc.jiangxi.ha2ts4.28,c=127.0.0.1

<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

`-I`参数则是只显示http response的头信息，就不显示网页内容了。  

##### 4. 指定请求方法
curl默认的请求方法是GET，使用`-X`参数可以支持其他请求方法。
```
curl -X POST www.example.com
```

```
curl -X DELETE www.example.com
```
##### 5. cookie
使用`--cookie`参数，可以让curl发送cookie。  
```
curl --cookie "name=xxx" www.example.com
```

至于具体的cookie的值，可以从http response头信息的`Set-Cookie`字段中得到。

`-c cookie-file`可以保存服务器返回的cookie到文件，`-b cookie-file`可以使用这个文件作为cookie信息，进行后续的请求。
```
curl -c cookies http://example.com

curl -b cookies http://example.com
```

##### 6. 增加头信息
有时需要在http request之中，自行增加一个头信息。`--header`参数就可以起到这个作用。

```
curl --header "Content-Type:application/json" http://example.com
```
##### 7. HTTP认证
有些网域需要HTTP认证，这时curl需要用到`--user`参数。
```
curl --user name:password example.com
```

当然，如果在windows或者其他有桌面的系统想要模拟浏览器发送http请求，其实用postman这个工具就可以了  
##### 参考  
[curl网站开发指南-阮一峰](http://www.ruanyifeng.com/blog/2011/09/curl.html)  

#### DNS解析  
##### 什么是DNS？  
DNS （Domain Name System 的缩写）的作用非常简单，就是根据域名查出IP地址。你可以把它想象成一本巨大的电话本。  
举例来说，如果你要访问域名www.baidu.com，首先要通过DNS查出它的IP地址是180.101.49.12。  

那么通过域名查询出IP是怎么查的呢，答案是分级查询。  
首先域名是有层级结构的，假设`www.baidu.com`这个域名来说，他的完整域名应该是`www.baidu.com.root`  
层级结构如下所示  
```
www.baidu.com.root  

主机名.次级域名.顶级域名.根域名  

# 即
host.sld.tld.root  
```

DNS服务器根据域名的层级，进行分级查询。

需要明确的是，每一级域名都有自己的NS记录，NS记录指向该级域名的域名服务器。这些服务器知道下一级域名的各种记录。  
以`www.baidu.com.root`这个例子来说的话，其按层查找的过程是这样的  
```
1. 从"根域名服务器(root)"查到"顶级域名服务器(com)"的NS记录和A记录（IP地址）,通过这些记录找到"顶级域名服务器(com)"进行下一层的查询
2. 从"顶级域名服务器(com)"查到"次级域名服务器(baidu)"的NS记录和A记录（IP地址），通过这些记录找到"次级域名服务器(baidu)"进行下一层的查询
3. 从"次级域名服务器(baidu)"查出"主机名(www)"的IP地址，找到IP地址，查询结束。
```

linux下dns解析的工具也有很多，常用的有以下几种:  
* ping
* nslookup  
* dig  
* host  

综合考虑下，dig工具的功能比较多和全，在解析的过程中显示的信息比较详细，这里我们介绍**dig**工具的常见使用  
##### 1. 安装dig命令  
一些linux系统下可能默认无此命令，根据不同的系统可以用一下命令进行安装，这个包中也包括nslookup命令。  
* Ubuntu
```
sudo apt-get install dnsutils
```
* Debian
```
apt-get install dnsutils
```
* Fedora / Centos
```
yum install bind-utils
```
##### 2. 查询域名的IP  
最常用的当然是根据域名查出服务器的IP地址了用下面的命令即可  
```
dig www.baidu.com
```
查询结果如下所示
```
; <<>> DiG 9.10.3-P4-Ubuntu <<>> www.baidu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5994
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.baidu.com.                 IN      A

;; ANSWER SECTION:
www.baidu.com.          164     IN      CNAME   www.a.shifen.com.
www.a.shifen.com.       293     IN      A       180.101.49.12
www.a.shifen.com.       293     IN      A       180.101.49.11

;; Query time: 0 msec
;; SERVER: 100.100.2.136#53(100.100.2.136)
;; WHEN: Fri Dec 06 00:01:20 CST 2019
;; MSG SIZE  rcvd: 90
```
第一段是查询参数和统计。如下所示：  
```
; <<>> DiG 9.10.3-P4-Ubuntu <<>> www.baidu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5994
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
```

第二段是查询内容。如下所示：  
```
;; QUESTION SECTION:
;www.baidu.com.                 IN      A
```
上面结果表示，查询域名www.baidu.com的A记录，A是address的缩写。  

第三段是DNS服务器的答复。如下所示：  
```
;; ANSWER SECTION:
www.baidu.com.          164     IN      CNAME   www.a.shifen.com.
www.a.shifen.com.       293     IN      A       180.101.49.12
www.a.shifen.com.       293     IN      A       180.101.49.11
```
上面结果显示，www.baidu.com有一个CNAME记录`www.a.shifen.com`，CNAME记录主要用于域名的内部跳转，为服务器配置提供灵活性，用户感知不到。在这个例子来说，`www.baidu.com`的CNAME记录指向`www.a.shifen.com`,实际返回的是`www.a.shifen.com`的ip地址。  

www.a.shifen.com有两个A记录，即两个IP地址。293（Time to live 的缩写），表示缓存时间，即293秒之内不用重新查询。

##### 3. DNS的记录类型  
域名与IP之间的对应关系，称为"记录"（record）。根据使用场景，"记录"可以分成不同的类型（type），前面已经看到了有A记录和CNAME记录。    
常见的DNS记录类型如下。  
```
（1） A：地址记录（Address），返回域名指向的IP地址。

（2） NS：域名服务器记录（Name Server），返回保存下一级域名信息的服务器地址。该记录只能设置为域名，不能设置为IP地址。

（3）MX：邮件记录（Mail eXchange），返回接收电子邮件的服务器地址。

（4）CNAME：规范名称记录（Canonical Name），返回另一个域名，即当前查询的域名是另一个域名的跳转。

（5）PTR：逆向查询记录（Pointer Record），只用于从IP地址查询域名。
```

##### 参考   
[域名DNS解析工具](https://www.cnblogs.com/EasonJim/p/10017100.html)  
[DNS 原理入门-阮一峰](http://www.ruanyifeng.com/blog/2016/06/dns.html)  

#### TCP和UDP检测  
##### 1. netcat  
如果仅仅是测试TCP端口的话，其实我们用telnet命令来测试就可以了，但是telnet不支持测试udp端口，所以我们使用另一个工具--nc。  

nc 是 Linux 下的一个强大的命令。nc 是简称，全名是 netcat。netcat透过使用TCP或UDP协议的网络连接去读写数据。  

##### 2. 使用nc进行TCP端口测试  
为了模拟两台服务器的tcp通讯，我们先在本机使用tcp协议监听一个端口，然后等待其他人在连接，可以使用下面的命令：  
```
nc -lp 2000
```
**-l [–listen]：** 绑定并监听传入的连接    
**-p [–source-port]：** 指定要使用的源端口  
然后使用tcp协议去连接这个目标ip和端口：  
```
nc localhost 2000
```
这样就能和目标建立连接， 并且在任意一方输入信息并按下回车发送后对方就能收到。  

##### 3. 使用nc进行UDP端口测试  
为了模拟两台服务器的udp通讯，我们先在本机使用udp协议监听一个端口，然后等待其他人在连接，可以使用下面的命令：  
```
nc -lup 2000
```
**-l [–listen]：** 绑定并监听传入的连接  
**-u ：** 使用udp模式  
**-p [–source-port]：** 指定要使用的源端口  
然后使用udp协议去连接这个目标ip和端口：  
```
nc -u localhost 2000
```
这样就能和目标建立连接， 并且在任意一方输入信息并按下回车发送后对方就能收到。 

##### 参考  
[Linux测试UDP 和 TCP 端口](https://blog.csdn.net/CLinuxF/article/details/84032571)  
[Linux 下 nc 命令的简单介绍](https://iacn.me/2017/08/16/linux-netcat/)  