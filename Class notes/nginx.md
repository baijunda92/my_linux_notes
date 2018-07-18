#### nginx主进程功能：

1. 读取并验证配置信息；
2. 创建、绑定及关闭套接字；
3. 启动、终止及维护worker进程的个数；
4. 无须终止服务而重新配置工作的特性；
5. 重新打开日志文件，实现日志滚动
6. 编译嵌入式perl脚本

##### worker进程主要完成的任务：

1. 接收、传入并处理来自客户端的连接；
2. 提供反向代理及过滤功能；

##### cache loader进程主要完成的任务：

1. 检查缓存存储中的缓存对象
2. 使用缓存元数据建立内存数据库

##### cache manager进程主要任务：

1. 缓存的失效及过期检验

##### nginx配置文件不同的上下文：

1. main段：核心配置部分，对任何功能都生效的配置

   - user

   - group

   - worker

   - http {

      	server {

     ​		location {

     ​		}

     ​	}

     }

2. http段：当前指令配置仅对web服务器有效

   - server段：位于http段中

3. upstream段

4. location段

5. mail段：

#### 架构和扩展性

- 一个主进程和多个工作进程，工作进程以非特权用户运行
- 支持的事件驱动IO框架：epoll
- 支持sendfile :实现将数据报文直接在内核中封装响应给客户端

#### CentOS7 nginx源码编译

```shell
yum -y install pcre-devel
yum -y install openssl
yum -y install gd
groupadd -r -g 108 nginx
useradd -r -g 108 -u 108 nginx
./configure --prefix=/usr/local/nginx \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/run/nginx.lock \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_dav_module \ 
--with-http_stub_status_module \
--with-threads \
--with-file-aio \
make && make install
```

httpd:

<DocumentRoot "">

</DocumentRoot> #根据文件系统来定义网页文件访问路径

<Location "/bbs">

</Location> #根据URI定义网页文件访问路径

nginx:将上述两种功能结合在一起了

location /URI/ {

​	root /web/app; #root定义了/URI/下的所有网页资源文件都位于哪个具体的文件系统路径下  

}

location / {

​	root	html; #这里的相对路径，是相对于NGINX的安装路径

​	index	index.html;  #index表示主页面

}

location URI {}:

- 对当前路径及子路径下的所有对象都生效

location = URI {}:

- 精确匹配指定的路径，不包括子路径，只对当前资源生效；

location ~ URI{}:

location ~* URI {}:

- 模式匹配URI，此处的URI可使用正则表达式；~ 区分字符大小写；~*不区分字符大小写；

location ^~ URI {}:

- 不使用正则表达式

htpasswd:

​	第二次不能使用-c选项

stub_status:

`accepts`: 已经接收的客户端连接个数

​	The total number of accepted client connections. 

`handled`：已经处理的客户端连接个数

​	 The total number of handled connections. Generally, the parameter value is the same as `accepts` 	unless some resource limits have been reached (for example, the [worker_connections](http://nginx.org/en/docs/ngx_core_module.html#worker_connections) limit). 

`requests`：客户端的请求总数

​	 The total number of client requests. 

`Reading`：nginx正在读取其首部请求的个数

 	The current number of connections where nginx is reading the request header. 

`Writing`：nginx正在读取其主体的请求的个数、或正处理其请求内容的请求的个数或正在向客户端发送响应的个数

​	 The current number of connections where nginx is writing the response back to the client. 

`Waiting`：长连接模式中保持的连接个数；

​	 The current number of idle client connections waiting for a request. 

配置基于https的nginx

- nginx https配置

```shell
vim /etc/nginx/nginx.conf
:.,$s/\([^[:space:]]+#\)*/\1/
server {
    ssl_certificate      /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key  /etc/nginx/ssl/nginx.key; #当前server段中其余配置项默认
	
	location / {
        root	/web/ssl;
        index	index.html	index.htm;
	}
}

```



- 如何创建CA证书？

```shell
(umask 077; openssl genrsa 2048 > private/cakey.pem)
openssl req -new -x509 0key private/cakey.pem -out cacert.pem 
openssl req -new -x509 -key private/cakey.pem -out cacert.pem 
touch serial
echo 01 > serial
touch index.txt
cd /etc/nginx/
mkdir ssl
cd ssl/
(umask 077; openssl genrsa 1024 > nginx.key)
openssl req -new -key nginx.key -out nginx.csr
openssl ca -in nginx.csr -out nginx.cr-days 3650
openssl ca -in nginx.csr -out nginx.crt -days 3650
```

nginx虚拟主机配置

- 基于主机名的虚拟主机

```shell
vim /etc/nginx/nginx.conf
server {
        listen       80;
        server_name  www.magedu.com;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   /web/htdocs;
            index  home.html;
        }
}

  server {
        listen 80;
        server_name www.a.org;
        location / {
            root    /web/a.org;
            index   index.html;
        }
}

client端测试：
vim /etc/hosts
192.168.1.104 www.magedu.com
192.168.1.104 www.a.org
curl http://www.magedu.com/index.html
curl http://www.a.org/index.html
```

lnmp实现：

PHP+MySQL

FastCGI: nginx只能通过FastCGI协议连接PHP

php-fpm: fastcgi进程管理器

​	127.0.0.1:9000



