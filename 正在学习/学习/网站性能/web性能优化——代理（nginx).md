# 简介
一个很好的原则是调优时每次只个性一个配置。如果对配置的个性不能提高性能的话，改回默认值
优化必须要通过性能测试。不能意淫，需要前后对比，真实说明问题。

# 场景
1. 优化nginx。
2. 确保每次请求控制一定资源。
3. 减少访问web容器


# 解决方案
## nginx优化
### 全局优化
```
# nginx进程数，建议按照cpu数目来指定，一般为它的倍数。
worker_processes 8;
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
# 这个指令是指当一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与nginx进程数相除，但是nginx分配请求并不是那么均匀，所以最好与ulimit -n的值保持一致
worker_rlimit_nofile 65535;
events {
	# 使用epoll的I/O模型。
    use epoll;
	# 每个进程允许的最多连接数，理论上每台nginx服务器的最大连接数为worker_processes*worker_connections
    worker_connections  50000;
}

#sendfile 指令指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，对于普通应用，
#必须设为 on,如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，以平衡磁盘与网络I/O处理速度，降低系统的uptime.
sendfile        on; 

#连接超时时间，单位时间是秒
keepalive_timeout  180;

#FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。下面参数看字面意思都能理解。
fastcgi_connect_timeout 300;
fastcgi_send_timeout 300;
fastcgi_read_timeout 300;
fastcgi_buffer_size 64k;
fastcgi_buffers 4 64k;
fastcgi_busy_buffers_size 128k;
fastcgi_temp_file_write_size 128k;

#防止网络阻塞
tcp_nopush      on; 
tcp_nodelay        on; 
#开启gzip压缩
gzip  on; 
gzip_min_length 512;
gzip_buffers 4 16k;
gzip_http_version 1.0;   #压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
gzip_comp_level 5;
#压缩类型，默认就已经包含text/html，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
gzip_types text/plain application/json application/x-javascript text/css application/xml;
gzip_vary on; 
gzip_disable "MSIE [1-6]\.";



# 这个是指多长时间检查一次缓存的有效信息。
open_file_cache_valid 30s;
# open_file_cache指令中的inactive参数时间内文件的最少使用次数，如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive时间内一次没被使用，它将被移除。
open_file_cache_min_uses 1;
# 客户端请求头部的缓冲区大小，这个可以根据你的系统分页大小来设置，一般一个请求的头部大小不会超过1k，不过由于一般系统分页都要大于1k，所以这里设置为分页大小。分页大小可以用命令getconf PAGESIZE取得
open_file_cache max=102400 inactive=20s;
```

### 日志
日志是要读写文件的，I/O消耗特别严重。日志是否开启可以根据自己具体的架构需求。
建议的是如下：

1. 静态资源不记录。
```
# 静态资源通过nginx来管理
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|js|css|ico)$ {
    # 关闭日志
    access_log off;
    include deny/agent.conf;
    if (-f $request_filename) {
        expires 1d;
        break;
    }
}
```
2. 动态资源要记，但是需要添加缓存。
```
access_log  logs/access.log  main buffer=12k;
```

## 限制资源使用
这个其实和安全非常相似。本质是限制资源使用，这样就能让有限的资源为更多人提供服务。
所以可以参见：**[web安全——代理（nginx)](http://www.cnblogs.com/ansn001/p/5643711.html)**

## 减少访问web容器
一般web容器的性能都是比较差了，所以尽量阻止访问web容器。
### 动静分离
一般的web容器的长链接性能都比较弱，而nginx在这方面又特别优秀。
```
# 动态的服务
server {
        server_name  sso.xxx.com;
        #监听
		listen       80; 
		location / { 
		    #反向代理到指定的服务
		    proxy_pass http://xxx-server;
			#定义服务器的默认网站根目录位置
			root    /; 
			proxy_set_header Host $host;
			#后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
			proxy_set_header  X-Real-IP  $remote_addr;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			client_max_body_size  100m;
		}
}
# 静态的服务
server {
        server_name  static.xxx.com;
        location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|js|css|ico)$ {
		# 文件根目录，这个目录根据文件的位置变更即可。
        root /home/static;
        if (-f $request_filename) {
           expires 1d;
           break;
        }
}
}
```

### 缓存化
把一些热点的数据放在缓存服务器，这样能提升性能。
srcache_nginx+redis构建缓存系统。在web应用中通过设置http的缓存特性（最好是基于注释或者配置，对开发者尽量是透明的，不要增加业务复杂度），来判断是否需要缓存。
```
#设置key,根据域名和uri
set $key $host$request_uri;
#来一个md5，要不然key太长。而且key特别长的时候，貌似在srache_store的时候执行不了
set_md5 $md5key $key;
#调用HttpSRCacheModule的srcache_fetch 在http进入时，执行该方法，如果取到值，则不执行代理（即不执行tomcat），直接返回
srcache_fetch GET /redis2_get $md5key;
#如果没有设置缓存时间，则默认时间为这个
srcache_default_expire 3600s;
#调用HttpSRCacheModule的srcache_store在执行代理（即执行tomcat）后，执行该方法，而且必需为“cache-control” 不等于"no-cahce"才执行。
#srcache_expire的取值优先级
#1、如果有cache-control;max-age=N，则取N
#2、如果没有，有expire，则取expire
#3、如果都没有取srcache_default_expire
srcache_store PUT /redis2_set key=$md5key&expire=$srcache_expire;
```
### 静态化
把基本没有变化的请求转为静态文件资源。直接通过文件提供服务。
实现思路见下面的参考资料。
不过可能会有个问题，需要注意并且以后去解决
```
# 问题
平常访问是没有问题，但在高并发下，你想死的心都有。打开文件不对，一看是前面还没写完，另外一个用户就访问了，又写，导致文件格式不对。
# 建议思路
如果最终还是要写html，还是通过程序去实现。例如：新闻类更新时，由编辑人员点击发布，此时可能就写了一个静态的html，而不是由用户去访问出发写html，避免高并发会出现的问题 
```

需要进一步思考的地方：
`缓存化`和`静态化`使用的场景。以及在nginx使用`静态化`是否合适。

# 验证方法
1. pagespeed.webkaka.com。验证是否开启gzip。
2. ab|webbench，来测试接口。

# 参考资料
1. [srcache_nginx+redis构建缓存系统](http://www.ttlsa.com/nginx/construction-of-srcache_nginx_redis-caching-system/)
2. [nginx - 性能优化，突破十万并发](http://361324767.blog.163.com/blog/static/114902525201281224328427/)
3. [通过Nginx架设灵活的网站静态化方案](http://www.oschina.net/question/54100_9105)