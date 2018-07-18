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

