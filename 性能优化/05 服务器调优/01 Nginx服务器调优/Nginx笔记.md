# Nginx

## 一、Nginx安装

1. 去官网nginx: download下载对应的nginx包，推荐使用稳定版本

   此处使用的是`nginx-1.20.2`版本

2. 上传到linux系统

3. 安装依赖环境

   ```powershell
   # 安装gcc环
   yum install gcc-c++ pcre pcre-devel zlib zlib-devle openssl openssl-devel
   
   # 安装PCRE库，用于解析正则表达式
   yum  install -y pcre pcre-devel
   
   # zlib压缩和解压缩依赖
   yum install -y zlib zlib-devle
   
   # SSL安全加密的套接字协议层，用于HTTP安全传输，也就是https
   yum install -y openssl openssl-devel
   ```

   

4. 解压

   ```shell
   tar -zxvf  nginx-1.20.2.tar.gz
   ```

   

5. 编译之前，先创建nginx临时目录，如果不创建，在启动nginx的过程中会报错

   ```power
   mkdir /var/temp/nginx -p
   ```

   

6. 在nginx的目录，输入如下命令进行配置，目的是为了创建Makefile文件

   ```powershell
   ./configure \
   --prefix=/usr/local/nginx \
   --pid-path=/var/run/nginx/nginx.pid \
   --lock-path=/var/lock/nginx.lock \
   --error-log-path=/var/log/nginx/error.log \
   --http-log-path=/var/log/nginx/access.log \
   --with-http_gzip_static_module \
   --http-client-body-temp-path=/var/temp/nginx/client \
   --http-proxy-temp-path=/var/temp/nginx/proxy \
   --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
   --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
   --http-scgi-temp-path=/var/temp/nginx/scgi \
   --with-http_stub_status_module \
   --with-http_ssl_module \
   --with-http_v2_module
   ```

   说明：

   \代表在命令中换行，用于提高可读性
   配置命令：
   命令	解释
   --prefix	指定nginx安装目录
   --pid-path	指向nginx的pid
   --lock-path	锁定安装文件，防止被恶意篡改或误操作
   --error-log-path	错误日志
   --http-log-path	http日志
   --with-http_gzip_static_module	启用gzip模块，在线实时压缩输出数据流
   --http-client-body-temp-path=	设定客户端请求的临时目录
   --http-proxy-temp-path	设定http代理临时目录
   --http-fastcgi-temp-path	设定fastcgi临时目录
   --http-uwsgi-temp-path	设定uwsgi临时目录
   --http-scgi-temp-path	设定scgi临时目录

   --with-http_stub_status_module  一个监视模块，可以查看目前的连接数等一些信息

   --with-http_ssl_module  启用ssl模块
   --with-http_v2_module  启用http2模块

7. make编译

   ```powershell
   make & make install
   ```

   

9. 进入sbin目录启动nginx

   ```powershell
   # 进入sbin目录
   cd /usr/local/nginx/sbin/
   
   # 启动
   ./nginx
   
   # 停止
   ./nginx -s stop
   
   # 重新加载
   ./nginx -s reload
   
   # 查看配置文件是否正确
   ./nginx -t
   
   #查看nginx安装的目录
   whereis nginx
   ```

   

10. 访问测试

    打开浏览器，访问nginx服务器地址ip即可打开nginx默认页面

    

## 二、Nginx基本配置

### 1. 基本结构

```makefile
// nginx全局块
...

// events块
events {
    ...
}

// http 块
http {
    // http全局块
    ...
    
    // upstream 块
    upstream {

    }
    
    // server块
    server {
        ...
    }
    
    // http全局块
    ...
}
```



### 2. 配置文件详解

```makefile
# 配置nginx的用户组 默认为nobody
#user  nobody;

#指定工作进程的个数，默认是1个，具体可以根据服务器cpu数量进行设置，如果不知道cpu的数量，可以设置为auto
worker_processes  1;

# 配置nginx的错误日志 格式为 log路径 log级别
# error_log 的日志级别为： debug info notice warn error crit alert emerg 紧急由低到高
# error_log的默认日志级别为error，那么就只有紧急程度大于等于error的才会记录在日志
# error_log 的作用域为 main http mail stream server location

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

# 指定nginx进程运行文件存放地址
#pid        logs/nginx.pid;

#工作模式及连接数上限
events {
    #参考事件模型，use [ select | poll | kqueue | epoll | rtsig | /dev/poll | eventport ]
    # poll是多路复用IO中的一种方式,但是仅用于linux2.6以上内核,可以大大提高nginx的性能
    # use epoll
    
    # 设置网络的连接序列化 防止惊群现象发生 默认为 on
    # accept_mutex on;
    
    # 设置一个进程是否同时接受多个网络连接 默认为 off
    # multi_accept off
    
    # 最大连接数 默认为 512
    worker_connections  1024;
    #######################################################################
    # 并发总数是 worker_processes 和 worker_connections 的乘积
    # 即 max_clients = worker_processes * worker_connections
    # 在设置了反向代理的情况下，max_clients = worker_processes * worker_connections / 4
    # 为什么上面反向代理要除以4，应该说是一个经验值
    # 根据以上条件，正常情况下的Nginx Server可以应付的最大连接数为：4 * 8000 = 32000
    # worker_connections 值的设置跟物理内存大小有关
    # 因为并发受IO约束，max_clients的值须小于系统可以打开的最大文件数
    # 而系统可以打开的最大文件数和内存大小成正比，一般1GB内存的机器上可以打开的文件数大约是10万左右
    # 我们来看看360M内存的VPS可以打开的文件句柄数是多少：
    # $ cat /proc/sys/fs/file-max
    # 输出 395374
    # 32000 < 34336，即并发连接总数小于系统可以打开的文件句柄总数，这样就在操作系统可以承受的范围之内
    # 所以，worker_connections 的值需根据 worker_processes 进程数目和系统可以打开的最大文件总数进行适当地进行设置
    # 使得并发总数小于操作系统可以打开的最大文件数目
    # 其实质也就是根据主机的物理CPU和内存进行配置
    # 当然，理论上的并发总数可能会和实际有所偏差，因为主机还有其他的工作进程需要消耗系统资源。
}


http {
    # 文件扩展名和文件类型映射表 text/css,image/jpeg,text/html
    include       mime.types;
    
    # 默认文件类型
    default_type  application/octet-stream;
    
    # 日志格式 后面会介绍
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    
    // 允许通过日志配置
    #access_log  logs/access.log  main;
    
    # sendfile 指定使用 sendfile 系统调用来传输文件。优点在于在两个文件描述符之间传递数据（完全在内核中操作），从而避免了数据在内核缓冲区和用户缓冲区之间的拷贝，效率高，称之为零拷贝
    # sendfile 作用域 location server http
    sendfile        on;
    
    #减少网络报文段的数量
    #tcp_nopush     on;
    
    # 链接超时时间 默认 75s 作用域 http server location
    #keepalive_timeout  0;
    keepalive_timeout  65;
    
    # 开始gzip压缩，降低带宽使用和加快传输速度，但增加了CPU的使用
    # gzip  on;

    server {
        # 端口号
        listen       8080;
        # 域名或ip
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
        
        # 对请求的路由进行过滤 正则匹配
        location / {
            root   html;
            index  index.html index.htm;
        }

        ...
    }
    
    # 建议保留原有配置，追加自定义配置到servers，并引入配置
    include servers/*;
}
```



### 3. 日志配置

nginx的日志大致分为 `access_log` 和 `error_log`。error_log 记录的是nginx的错误日志。（以下对日志的理解不是很全面，还只是基础的）

#### 3.1 error_log

- 记录nginx错误日志
- 作用域为 `main http mail stream server location`
- 日志级别 `debug info notice warn error crit alert emerg`
- 日志级别默认为 error 当级别高于或等于指定级别时才会记录

#### 3.2 access_log

- 记录请求通过的日志
- 作用域为 `http server location limit_except`
- 日志格式默认为 `combined`
- 日志格式是可以自定义的

```shell
# 定义一个为 main 的日志格式
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
                      
access_log  logs/access.log  main;
```



上方的 `log_format` 后面类似 `$remote_addr` 是nginx的内置变量，取值如下

```shell
$remote_addr, $http_x_forwarded_for（反向） 记录客户端IP地址
$remote_user 记录客户端用户名称
$request 记录请求的URL和HTTP协议
$status 记录请求状态
$body_bytes_sent 发送给客户端的字节数，不包括响应头的大小； 该变量与Apache模块mod_log_config里的“%B”参数兼容。
$bytes_sent 发送给客户端的总字节数。
$connection 连接的序列号。
$connection_requests 当前通过一个连接获得的请求数量。
$msec 日志写入时间。单位为秒，精度是毫秒。
$pipe 如果请求是通过HTTP流水线(pipelined)发送，pipe值为“p”，否则为“.”。
$http_referer 记录从哪个页面链接访问过来的
$http_user_agent 记录客户端浏览器相关信息
$request_length 请求的长度（包括请求行，请求头和请求正文）。
$request_time 请求处理时间，单位为秒，精度毫秒； 从读入客户端的第一个字节开始，直到把最后一个字符发送给客户端后进行日志写入为止。
$time_iso8601 ISO8601标准格式下的本地时间。
$time_local 通用日志格式下的本地时间。
```

### 4. 访问设置

#### 4.1 错误页面

```shell
        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        # error_page用于设置错误页面，如下所示
        # 500、502、503、504这些是常见的HTTP的错误代码，
        # /50x.html用于当发生上述指定的任意一个错误的时候，使用网站根目录下的/50x.html进行处理
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
```



#### 4.2 权限指令

（1）Deny

 设置禁止访问的IP

```shell
#禁止IP：192.168.6.101访问
location / {
    deny   192.168.6.101;
}

#禁止所有IP访问
location / {
    deny   all;
}
```

（2） allow

设置运行访问的IP

```shell
#只允许IP：192.168.6.101访问
location / {
    allow  192.168.6.101;
}

#允许所有IP访问
location / {
    allow   all;
}
```

（3）deny和allow的优先级

nginx的权限指令是从上往下执行的，在同一个块下deny和allow指令同时存在时，谁先匹配谁就会起作用，后面的权限指令就不会执行了。如下图，如果 “deny 192.168.6.101” 触发了，那它后面的两条都不会触发。

```shell
location / {
    deny 192.168.6.101;
    allow 192.168.6.220;
    deny 192.168.6.221;
}
```

其实他们的关系就像 “if…else if…else if”，谁先触发，谁起作用

```C#
if (deny 192.168.6.101) {...}
else if (allow 192.168.6.220) {...}
else if (deny 192.168.6.221) {...}
```

### 5. 虚拟主机配置

（1）修改宿主机的hosts文件(系统盘/windows/system32/driver/etc/HOSTS)

 linux ： vim /etc/hosts

格式： ip地址 域名

eg: 192.168.3.193 www.gerry.com

（2）在nginx.conf文件中配置server段

```shell
server {
    listen 80; # 端口区分
    # server_name 192.168.3.203; # ip区分
    server_name www.gerry.com; # 域名区分
    
    location / {
        root html;
        index index.html;
    }
}
```

### 6. Location匹配

#### 6.1 语法规则

>= 开头表示精准匹配
>~ 大小写敏感
>~* 忽略大小写
>^~ 只需匹配uri开头
>@ 定义一个命名的 location，在内部定向时使用，例如 error_page
>location  [ = | ~ | ~* | ^~ ] /uri/ { ... }
>location @name { ... }

（1）任意匹配

```shell
# 匹配所有请求，但是正则和最长字符串会优先匹配
location / {
    #规则
}
```



（2）“=” 精准匹配

```shell
# 精确匹配 / ，可以匹配类似 `http://www.example.com/` 这种请求，'/'后不能带有任何内容
location = / {
   #规则
}
```



（3）“~” 大小写敏感

```shell
location ~ /Test/ {
    #规则
}
#可以匹配    http://www.test.com/Test/
#不可以匹配    http://www.test.com/test/
```



（4） “~*” 忽略大小写

```shell
# ~* 会忽略uri部分的大小写
location ~* /Test/ {
    #规则
} 
#可以匹配    http://www.test.com/Test/
#可以匹配    http://www.test.com/test/
```



（5） “^~” 只匹配uri开头

```shell
#以 /test/ 开头的请求，都可以匹配上
location ^~ /test/ {    #规则
}
#可以匹配    http://www.test.com/test/
```



（6） “@” 命名匹配

```shell
#以 /img/ 开头的请求，如果链接的状态为 404。则会匹配到 @img_err 这条规则上。
location /img/ {
    error_page 404 @img_err;
}
location @img_err {
    # 规则
}
```



（7）以某个内容结尾

```shell
# 匹配所有以 gif,jpg或jpeg 结尾的请求
location ~* \.(gif|jpg|jpeg)$ {
  #规则
}
```



#### 6.2 优先级

在配置中需要注意的一点就是location的匹配规则和优先级

- = 开头表示精确匹配
- ^~ 开头表示uri以某个常规字符串开头，不是正则匹配;
- ~ 开头表示区分大小写的正则匹配;
- ~* 开头表示不区分大小写的正则匹配;
- / 通用匹配, 如果没有其它匹配,任何请求都会匹配到;



#### 6.3 匹配流程

- 判断是否精准匹配，如果匹配，直接返回结果并结束搜索匹配过程。
- 判断是否普通匹配，如果匹配，看是否包含^~前缀，包含则返回，否则记录匹配结果，(如果匹配到多个location时返回或记录最长匹配的那个)
- 判断是否正则匹配，按配置文件里的正则表达式的顺序，由上到下开始匹配，一旦匹配成功，直接返回结果，并结束搜索匹配过程。
- 如果正则匹配没有匹配到结果，则返回普通匹配的匹配结果。

> 注意：
>
> A、多个普通匹配的location时，和location的顺序无关，总是匹配所有的location，然后取匹配最长的location作为结果
>
> B、多个正则匹配的location时，和顺序有关，从上到下依次匹配，一旦匹配成功便结束，直接返回结果。

### 7. 配置指令

#### 7.1 root

语法：root path
默认值：root html
配置段：http、server、location、if

#### 7.2 alias

语法：alias path
配置段：只能处于location

#### 7.3 root与alias区别

- root 在请求的时候，会把请求的路径包含到请求地址中

​	#若按照这种配置的话，则访问/i/目录下的文件时，nginx会去/usr/local/nginx/html/admin/home下找文件。

```shell
location /home/{
	root  /usr/local/nginx/html/admin/；
}
```



- alias 会把 location 后面配置的路径丢弃掉,把当前匹配到的目录指向到指定的目录。

  #若按照以下配置的话，则访问/homt/目录里面的文件时，ningx会自动去/usr/local/nginx/html/admin目录找文件。

```shell
location /home/{
	alias  /usr/local/nginx/html/admin/；
}
```

>（1）、alias是一个目录别名的定义，root则是最上层目录的定义。
>
>（2）、还有一个重要的区别是alias后面必须要用“/”结束，否则会找不到文件的。而root则可有可无。



### 8. ReWriter

### 9. Ngnx代理

### 10. 负载均衡

> *upstream*是Nginx的HTTP Upstream模块，这个模块通过一个简单的调度算法来实现客户端IP到后端服务器的负载均衡

#### 10.1 参数介绍

**语法：**server address [parameters]

其中关键字server必选。
address也必选，可以是主机名、域名、ip或unix socket，也可以指定端口号。
parameters是可选参数，可以是如下参数：

 **down**：表示当前server已停用

 **backup**：表示当前server是备用服务器，只有其它非backup后端服务器都挂掉了或者很忙才会分配到请求

 **weight**：表示当前server负载权重，权重越大被请求几率越大。默认是1

**max_fails**和**fail_timeout**一般会关联使用，如果某台server在fail_timeout时间内出现了max_fails次连接失败，那么Nginx会认为其已经挂掉了，从而在fail_timeout时间内不再去请求它，fail_timeout默认是10s，max_fails默认是1，即默认情况是只要发生错误就认为服务器挂掉了，如果将max_fails设置为0，则表示取消这项检查。



例如：

```shell
upstream tomcatcluster { ## 方向代理名称不能有下划线
	server 192.168.3.199:6666 weight=1;
    server 192.168.3.199:7777 weight=1;
	server 192.168.3.199:8888 weight=1;
}

server {
	listen 80;
	server_name www.zhaoxi.com;
	
	location / {
		proxy_pass http://tomcat_cluster;
	}
}
```

#### 10.2 负载均衡算法

当没有指定任何信息时， NGINX 默认使用了 Round Robin(轮询)算法来重定向流量。其实 NGINX 提供了多种算法来做负载均衡，下面我们来介绍一下：

##### 10.2.1 Round Robin (轮询)

在没有指定 weight(权重) 的情况下，Round Robin 会将所有请求均匀地分发给所有后台服务实例：

```shell
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
}
```

这里我们没有指定权重，所以两个后台服务会收到等量的请求。但是，当指定了权重之后，NGINX 就会将权重考虑在内：

```shell
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com;
}
```

在 NGINX 中，weight 默认被设置为 1。这里我们用一开始的配置举例， [backend1.example.com](https://link.juejin.im?target=http%3A%2F%2Fbackend1.example.com) 的权重被设置为 5，另一个的权重没设置，所以是默认值 1。我们也没有设置轮询算法，所以这时候 NGINX 会以 5：1 的比例转发请求，即 6 个请求中， 5 个被放到了 [backend1.example.com](https://link.juejin.im?target=http%3A%2F%2Fbackend1.example.com) 上， 有一个被发到了 [backend2.example.com](https://link.juejin.im?target=http%3A%2F%2Fbackend2.example.com) 上。

##### 10.2.2 Least Connections（最少连接算法）

在这个模式下，一个请求会被 NGINX 转发到当前活跃请求数量最少的服务实例上:

```shell
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}
```

##### 10.2.3 IP Hash（IP哈希）

在 IP Hash 模式下，NGINX 会根据发送请求的 IP 地址的 hash 值来决定将这个请求转发给哪个后端服务实例。被 hash 的 IP 地址要么是 IPv4 地址的前三个 16 进制数或者是整个 IPv6 地址。使用这个模式的负载均衡模式可以保证来自同一个 IP 的请求被转发到同一个服务实例上。当然，这种方法在某一个后端实例发生故障时候会导致一些节点的访问出现问题。

```shell
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}
```

如果某一台服务器出现故障或者无法进行服务，我们可以给它标记上 down，这样之前被转发到这台服务器上的请求就会重新进行 hash 计算并转发到新的服务实例上:

```shell
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
}
```



### 11. Nginx缓存

### 12. 动静分离

### 13. Nginx跨域

### 13. GZip压缩

> gzip是网页的一种压缩技术，经过gzip压缩后，页面大小可以变为原来的30%甚至更小。gzip网页压缩的实现需要浏览器和服务器的支持，实际上就是服务器端压缩，传到浏览器后浏览器解压并解析。



配置说明

```shell
#是否开启gzip模块，on表示开启，off表示关闭
gzip on;

#设置允许压缩的页面最小字节(从header头的Content-Length中获取) 
gzip_min_length 1k;

#设置gzip申请内存的大小，其作用是按块大小的倍数申请内存空间，param2:int(k) 后面单位是k。
#这里设置以16k为单位，按照原始数据大小以16k为单位的4倍申请内存
#如果没有设置，默认值是申请跟原始数据相同大小的内存空间去存储gzip压缩结果
gzip_buffers 4 16k;

#识别http协议的版本
gzip_http_version 1.1;

#设置gzip压缩等级，等级越底压缩速度越快文件压缩比越小，反之速度越慢文件压缩比越大
#等级1-9，9 压缩比最大但处理最慢（传输快但比较消耗cpu）
gzip_comp_level 2;

#设置需要压缩的MIME类型,非设置值不进行压缩，即匹配压缩类型
gzip_types text/plain application/x-javascript text/css application/xml;

#用于在响应消息头中添加Vary：Accept-Encoding
#使代理服务器根据请求头中的Accept-Encoding识别是否启用gzip压缩
gzip_vary on;
```



### 14. UDP/TCP代理

（1）安装模块

```shell
./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-stream
```

（2）配置文件

```shell
#nginx.conf部分配置
# upd/tcp
stream {
    upstream backend {
        server 192.168.3.199:3306;
    }
    
    ## TCP
    server {
        listen 8848;
        proxy_connect_timeout 8s;
        proxy_timeout 24h;
        proxy_pass backend111;
    }
    
    ## UDP
    server {
        listen 3000 udp;
        proxy_pass 192.168.6.101:3001;

    }
}
```



### 15. Https配置

参考《Nginx服务器优化》文档





1、Nginx复习

2、Nginx安装

3、Nginx配置

nginx-proxy.conf

nginx-upstream.conf

4、负载均衡：轮询、权重、IP Hash

5、参数优化

work_process auto

worker_connnections 65535;最大

use poll; 内核>Linux2.4

multi_accept on;一个worker可以处理多个连接



/usr/local/sbin/nginx -s load 热重载

