## nginx模块文档

### nginx模块划分

**从结构上分：**   
核心模块：HTTP模块，EVENT模块，MAIL模块。   
基础模块：HTTPAccess模块，HTTP FastCGI模块，HTTP Proxy模块，HTTP Rewrite模块。   
第三方模块：HTTP Upstream Hash模块，Notice模块，HTTP Access Key模块


**从功能上分：**   
Handles（处理器模块）：直接处理请求，并进行输出内容和修改headers信息，Handles处理器模块一般只能有一个。   
Filters（过滤器模块）：对其他处理器模块输出内容进行修改操作。最后由Nginx输出。   
Proxies（代理类模块）：与后端一些服务，比如FastCGI进行交互。实现服务代理和负载均衡等功能。   

**工作模式：**  
1.单工作进程模式：除主进程外，还有一个工作进程，且工作进程是单线程的。   
2.多工作进程模式：每个工作进程包含多工作线程。   


### nginx常用模块

#### ngx_http_gzip_static_module 
这个模块主要用来对请求的文件进行压缩。然后传送至客户端可以节省网络带宽，加快传输速度，但是经过服务器处理压缩需要消耗一定的cpu资源，常用的配置如下：
```nginx
gzip on;     #开启 gzip 压缩。
gzip_buffers 4 16k;		#分配多少内存用来存储压缩过的数据。
gzip_min_length 1K;		#最小压缩文件、当文件小于1K时不进行压缩。
gzip_comp_level 2;		#进行压缩的比例级别 1-9 分别是从低到高的压缩。
gzip_types 类型;			#进行压缩的文件类型，默认情况为 text/html 都进行压缩。
gzip_http_version 1.1;	#指定哪个协议版本进行压缩。
```

#### upstream 
这个模块主要用来做负载均衡，常用配置如下：
```nginx
upstream text{	#指定负载均衡物理池的名称。
	ip_hash;	#设置服务器用客户端的 IP 地址进行 hash 计算将来自同一 IP 地址的请求分配到同一后端真实服务器，预防 session 的丢失。
	server 172.16.1.1:80 weight=1 max_fails=1 fail_timeout=20s；	#设置后端真实服务器的（服务器IP:端口，权重，最大连接失效数，到达最大连接数失效数后服务停止时间）
	server 172.16.1.2:80 weight=1 max_fails=1 fail_timeout=20s;
}
```

#### rewrite
这个模块主要用来做url重写，使用 nginx 提供的变量，结合正则表达式和标志位实现重定向或url重写。常用配置如下：
```nginx
location / {
    # 注意这里要用‘’单引号引起来，避免{}, 对形如/images/ef/uh7b3/test.png的请求，重写到/data?file=test.png。
    rewrite '^/images/([a-z]{2})/([a-z0-9]{5})/(.*)\.(png|jpg|gif)$' /data?file=$3.$4;
}
```
更多 rewrite 写法请参考 [rewirte写法](http://seanlook.com/2015/05/17/nginx-location-rewrite/)

#### access_log
这个模块用于控制 nginx 访问日志的行为，常见配置如下：
```nginx
access_log path format gzip[=level] [buffer=size] [flush=time]; #设置日志文件的(压缩等级，缓存区大小，内存缓存区最长时间)。
access_log off;			#不记录日志。
```
#### error_log
这个模块用来记录 nginx 错误日志。常见配置如下：
```nginx
error_log /var/log/nginx/error.log error
```

#### log_format
这个模块用来设置日志格式，常见的配置如下：
```nginx
log_format common '$remote_addr - $remote_user [$time_local] "$request" '
    						'$status $body_bytes_sent "$http_referer" '
    						'"$http_user_agent" "$http_x_forwarded_for" "$http_cookie" "$host"';   
```
日志格式允许包含的变量注释如下：
```bash
$remote_addr, $http_x_forwarded_for 记录客户端IP地址
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

#### open_log_file_cache
这个模块用来设置日志文件缓存，默认为 off。常见配置如下：
```nginx
open_log_file_cache max=1000 inactive=20s valid=1m min_uses=2;  #设置日志文件中（缓存中最大文件描述符数量，存活时间，检查频率，在inactive时间段中日志最少使用多少次后将文件描述符计入缓存中）。
```

#### auth_basic
这个模块是用来做用户认证的模块，常用配置如下：
```nginx
location /admin/ {
	auth_basic "input you user name and  password"；	#可以设置为 off,或设置文本信息，通常简称为"认证请求"。
	auth_basic_user_file access/password_file；		#定义密码文件的相对路径。创建密码格式为：htpasswd -b -c site_pass username password。
}
```

#### Access
这个模块用来指令你通过指定的 ip 或 ip范围 来允许或拒绝被访问的资源。包含两个重要指令（allow 和 deny）。常见配置如下：
```nginx
location {
	allow 127.0.0.1; #允许本地IP访问。
}
```

#### real_ip
这个模块是通过在X-Real-IP HTTP头中指定的IP来代替客户端的IP地址。常见配置如下：
```nginx
real_ip_header X-forwarded-For;		#用来定义被利用的HTTP头（X-Real-IP 或 X-Forwarded-For）。
set_real_ip_from 192.168.0.0/16;	#定义被信任的IP地址。
set_real_ip_from 127.0.0.1;			#定义能够接受IP地址和CIDR范围。
```

#### proxy_pass
这个模块用来做代理模块，指定转发到后端服务器的请求。常见配置如下:
```nginx
proxy_pass http://hostname:port;	#指定后端服务器的 ip 和端口号。
```

#### proxy_redirect
这个模块的主要功能是对发送给客户端的URL进行修改。常见配置如下：
```nginx
proxy_redirect http://localhost:8000/two/ http://frontend/one/;		#将http://location:8000/two/some/uri/ 重写为 http://frontend/one/some/uri/
```

#### proxy_buffering
这个模块使用了定义是否缓冲后端服务器的响应。常见配置如下:
```nginx
proxy_buffering on/off; #如果设置为on，那么nginx将把响应数据存储在内存中，使用内存空间来提供缓冲。如果缓冲用完，响应数据就存储到临时文件；如果设置为off，响应就直接转发到客户端。
```

#### proxy_buffers
这个模块用来设置缓冲数量和大小，用于存放从后端服务器读取的响应。常见配置如下：
```nginx
proxy_buffers 8 4k;		#8个缓冲区，每个缓冲区4K。
```

#### proxy_connect_timeout
这个模块用来定义后端服务器连接超时。这和读取、发送超时不一样，如果nginx已经连接到后端服务器，那么该指令是不合适的。
```nginx
proxy_connect_timeout 15;	#设置后端连接超时时间为15秒。
```

#### proxy_set_header
这个模块允许你重新定义代理header值再转到后端服务器。常见配置如下：
```nginx
proxy_set_header Host $host;
```
