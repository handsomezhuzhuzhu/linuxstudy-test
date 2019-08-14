## 五、Nginx Rewrite 相关功能

Nginx服务器利用ngx_http_rewrite_module模块解析和处理rewrite请求。rewrite用于实现URL的重写，类似于重定向功能，可以将用户的请求重写至别的目录，另外还可以在一定程度上提高网站的安全性。

### - 5.1：ngx_http_rewrite_module模块命令

https://nginx.org/en/docs/http/ngx_http_rewrite_module.html

#### - 5.1.1：if指令

用于条件匹配判断，根据条件判断结果选择不同的nginx配置，在server或location中配置。nginx的if语法只能支持单次判断，不支持多重判断，用法如下

```bash
if (条件匹配) {
    action
}

#示例1
location /main {
    index index.html;
    default_type text/html;
    if ( $scheme = http ){
        echo "if ---> $scheme";
    }
    if  ($scheme = https ){
        echo "if ---> $scheme";
    }
}
```

使用正则表达式对变量进行匹配，成功为true，否则false，变量与表达式之间可以用符号链接：

```bash
=  #比较变量与字符串是否相等
~  #表示在匹配过程中区分大小写字符
~* #表示在匹配过程中不区分大小写
-f 和 ！-f #判断请求的文件存在与不存在
-d 和 ！-d #判断请求的目录存在与不存在
-x 和 ！-x #判断文件是否可执行
-e 和 ！-e #判断请求的文件或目录是否存在（包括文件，目录，软链接）
```

注：若$变量的值为空或以0开头，则if指令认为此条件为false，其它条件为true

#### - 5.1.2 ：set指令

指定key并给其定义一个变量，定义格式为set $key value，只有将内置变量赋值给自定义变量时无论是key还是value都要加$符号

```bash
location /main {
    root /data/html;
    index index.html;
    default_type text/html;
    set $name javis;
    echo $name;
    set $my_port &server_port;
    echo $my_port;
}
```

#### - 5.1.3：break指令

用于中断当前相同作用域（location）中的其他配置，停止向下寻找，回到上一层作用域继续向下读取配置，该指令可在server块和location块以及ig块中使用，语法如下：

```bash
location /main {
    root /data/html;
    index index.html;
    default_type text/html;
    set $name javis;
    echo $name;
    break;
    set $my_port &server_port;
    echo $my_port;
}
#只输出$name的值javis，不再向下执行
```

#### - 5.1.4：return指令

return用于完成队请求的处理，并直接向客户端返回响应状态码处于此指令后的所有配置都将不被执行，return可以在server、if和location块进行配置，语法如下

```bash
return code #返回给客户端指定的http状态码
return code （text） #返回给客户端指定的状态码及响应内容，可以调用变量
return code URL #返回给客户端的URL地址
示例1
location /main {
    root /data/html;
    index index.html;
    default_type text/html;
    if ( $scheme = http ){
        return 500 "server error";
        echo "if --->" $scheme; #只输出server error，后面的将不执行
    }
    if ( $scheme = https){
        echo "if --->" $scheme;
    }
}
```

#### - 5.1.5：rewrite_log 指令

设置是否开启记录ngx_http_rewrite_module模块日志记录到error_log日志文件中，可以配置在http、server、location或if当中，需要日志级别为notice

```bash
location /main {
    root /data/html;
    index index.html;
    default_type text/html;
    set $name javis;
    echo $name;
    rewrite_log on;
    break;
    set $my_port &server_port;
    echo $my_port;
}
#重启nginx，访问error_log
```

### - 5.2：rewrite指令

通过正则表达式的匹配俩改变URI，可以同时存在一个或多个指令，按照次序对URI进行匹配，rewrite主要是针对用户请求的URL或URI作具体处理

```bash
URI：通用资源标识符，标识一个资源的路径，可以不带协议
URL：统一资源定位符，用于在internet中描述资源的字符串，时URI的子集，主要包括传输协议(scheme)、主机(IP、端口号或者域名)和资源具体地址(目录和文件名)等三部分，一般格式为 scheme://主机名[:端口号][/资源路径],如：http://www.a.com:8080/path/file/index.html就是一个URL路径，URL必须带访问协议。
区别：每一个URL都是URI，但URI不都是URL
示例：
http：//example.org/path/to/resource.txt #URI/URL
ftp://example.org/resource.txt  #URI/URL
/absolute/path/to/resource.txt  #URI
```

rewrite官方介绍：https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite，可以配置在server、location、if中，语法如下：

```bash
rewrite regex replacement [flag];
```

rewrite将用户请求的URI基于regex所描述的模式进行检查，匹配到时将其替换为表达式指定的新的URI。若在同一级配置模块中存在多个rewrite规则，就会自上而下逐个检查；被某规格替换后，会重新一轮新的替换检查，隐含有循环机制，但不超过10次；若超过则提示500响应吗，[flag]所标识的标志位用于控制此循环机制，，如果替换后的URL是以http：//或https：//开头，则替换结果会以重定向返回客户端，即永久重定向301

#### - 5.2.1：rewrite flag使用介绍

利用nginx的rewrite的指令，可以实现url的重新跳转，rewrtie有四种不同的flag，分别是redirect(临时重定向)、permanent(永久重定向)、break和last。其中前两种是跳转型的flag，后两种是代理型，跳转型是指有客户端浏览器重新对新地址进行请求，代理型是在WEB服务器内部实现跳转的。

```bash
redirect #临时重定向，重写完成后以临时重定向方式直接返回重写后生成的新URL给客户端，有客户端重新发起请求，使用相对路径，http：//或https：//开头，状态码：302
permanent #永久重定向，以永久重定向的方式直接返回重写后生成的新URL给客户端，由客户端重新发起新的请求，状态码：301
last #重写完成后停止对当前location中后续的其他重写操作，而后对新的URL启动新一轮重写检查，不建议在location中使用
break #重写完成后停止对当前URL在当前location中后续的其他重写操作，而后直接跳转至重写规则匹配块之后的其他配置；结束循环，建议在location中使用
```

这里应该了解nginx处理的十一个阶段

```bash
 NGX_HTTP_POST_READ_PHASE = 0,   #读取请求头
 NGX_HTTP_SERVER_REWRITE_PHASE,   #执行rewrite
 NGX_HTTP_FIND_CONFIG_PHASE,  #根据uri替换location
 NGX_HTTP_REWRITE_PHASE,      #根据替换结果继续执行rewrite
 NGX_HTTP_POST_REWRITE_PHASE, #执行rewrite后处理
 NGX_HTTP_PREACCESS_PHASE,    #认证预处理   请求限制，连接限制
 NGX_HTTP_ACCESS_PHASE,       #认证处理
 NGX_HTTP_POST_ACCESS_PHASE,  #认证后处理， 认证不通过， 丢包
 NGX_HTTP_TRY_FILES_PHASE,    #尝试try标签
 NGX_HTTP_CONTENT_PHASE,      #内容处理
 NGX_HTTP_LOG_PHASE           #日志处理
```

#### - 5.2.2：rewrite案例-域名永久与临时重定向

要求：因业务需要，将访问原域名www.linux.net的请求永久重定向到www.linux.com

临时重定向不会缓存域名解析记录（A记录），但是永久重定向会缓存

```bash
location / {
    root /data/html;
    index index.html;
    rewrite / http://www.linux.com permanent;
    #rewrite / http://www.linux.com redirect;
}
```

**（1）临时重定向：告诉浏览器域名不是固定重定向到当前目标域名，后期可能随时会更改，因此浏览器不会缓存当前域名的解析记录**

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559357749500.png)

**（2）永久重定向：永久重定向会缓存DNS解析记录**

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559357809697.png)

#### - 5.2.3：rewrite案例-URI重定向

要求：访问break的请求被转发至image，而访问last传递请求也被转发至image，以此测试last和break的区别

```bash
        location /break {
#       root /data/html;
        rewrite ^/break/(.*) /test/$1 break;#重写后直接直接找test目录，若没有为此处的test指定根目录，则访问不到，换言之，若要成功跳转，test需要完整路径
        return 666 "break";
        }
        location /last {
        rewrite ^/last/(.*) /test/$1 last;#完成重写后，在所有location里面重新开始一轮遍历，进到test，访问成功
        return 888 "last";
        }
        location /test {
        return 999 "test";
        }
```

重启nginx的访问测试

```bash
[root@centos ~]$curl -L -i  www.alijiujiu.com/break/ #break不会跳转至location /test中
HTTP/1.1 404 Not Found
Server: nginx/1.14.2
Date: Sat, 01 Jun 2019 06:14:27 GMT
Content-Type: text/html
Content-Length: 19
Connection: keep-alive
Keep-Alive: timeout=10
ETag: "5cee850d-13"

this is error page

[root@centos ~]$curl -L -i  www.alijiujiu.com/last/ #last会跳转至location /test中继续执行匹配操作
HTTP/1.1 999 
Server: nginx/1.14.2
Date: Sat, 01 Jun 2019 06:14:46 GMT
Content-Type: application/octet-stream
Content-Length: 4
Connection: keep-alive
Keep-Alive: timeout=10

test
```

#### - 5.2.4：rewrite案例-自动跳转https

要求：基于通信安全考虑公司网站要求全站https，所以要求将在不影响用户请求的情况下将http请求全部自动跳转至https，另外也可以实现部分location跳转

```bash
location / {
    root /data/html;
    index index.html;
    if ( $scheme = http ){ #未加条件判断，会陷入死循环
        rewrite / https://www.alijiujiu.com permanent;
    }
}
```

#### - 5.2.5：rewrite案例-判断文件是否存在

要求：当用户访问公司输入错误的URL，可以将用户重定向到公司首页

```bash
location / {
    root /data/html;
    index index.html;
    if ( !-f $request_filename ){
        #return 404 "input error";
        rewrite (.*) http://www.alijiujiu.com/index.html;
    }
}
```

### - 5.3:Nginx防盗链

防盗链基于客户端携带的referer实现，referer是记录打开一个页面之前记录是从哪个页面跳转过来的标记信息，如果别人只链接了自己网站图片或某个单独的资源，而不是打开了网站的整个页面，这就是盗链，referer就是之前的那个网站域名，正常的referer信息有以下几种

```bash
none：请求报文首部没有referer首部，比如用户直接在浏览器输入域名访问web网站，就没有referer信息。
blocked：请求报文有referer首部，但无有效值，比如为空。
server_names：referer首部中包含本主机名及即nginx 监听的server_name。
arbitrary_string：自定义指定字符串，但可使用*作通配符。
regular expression：被指定的正则表达式模式匹配到的字符串,要使用~开头，例如：~.*\.magedu\.com。
```

5.3.1：实现防盗链

基于访问安全考虑，nginx支持通过ungx_http_referer_module模块 https://nginx.org/en/docs/http/ngx_http_referer_module.html#valid_referers 检查访问请求的referer信息是否有效实现防盗链功能，定义方式如下：

```bash
[root@centos ~]$vim /apps/nginx/conf/conf.d/pc.conf
location /image {
    root /data/html/;
    index index.html;
    valid_referers none blocked server_names *.example.com example.* www.example.org/galleries/
    ~\.google\.;
    if ( $invalid_referer ){
        return 403;
    }
}
```

实现防盗链：

```bash
location ^~ /images {
 root /data/nginx;
 index index.html;
     valid_referers none blocked server_names *.alijiujiu.com www.alijiujiu.* 
    api.online.test/v1/hostlist  ~\.google\. ~\.baidu\.; #定义有效的referer
 if ($invalid_referer) { #假如是使用其他的无效的referer访问：
  return 403; #返回状态码403
 }
}
#重启Nginx并访问测试
```

### - 5.4：自定义serer信息，隐藏版本号

自定义Response Hearders中server信息

```bash
[root@centos ~]$vim /data/nginx-1.14.2/src/core/nginx.h 
#define NGINX_VERSION      "9527"
#define NGINX_VER          "linux/" NGINX_VERSION #开启server_tokens显示此信息
[root@centos ~]$vim /data/nginx-1.14.2/src/http/ngx_http_header_filter_module.c
static u_char ngx_http_server_string[] = "Server: javis" CRLF; #关闭server_tokens显示此信息
#配置完成后需要重新编译nginx
[root@centos ~]$vim /apps/nginx/conf/nginx.conf
server_tokens off; #放在http配置项中
```

#### - 5.4.1：server_tokens off

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559481062037.png)

#### - 5.4.2：server-tokens on

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559481138693.png)

## 六、Nginx反向代理功能

反向代理：reverse proxy，指的是代理外网用户的请求到内部的指定web服务器，并将数据返回给用户的一种方式，运用比较广泛

Nginx除了可以在企业提供高性能的web服务之外，另外还可以将本身不具备的请求通过某种预定义的协议转发至其它服务器处理，不同的协议就是Nginx服务器与其他服务器进行通信的一种规范，主要在不同的场景使用以下模块实现不同的功能：

```bash
ngx_http_proxy_module:将客户端的请求以http协议转发至指定服务器进行处理。
ngx_stream_proxy_module：将客户端的请求以tcp协议转发至指定服务器处理。
ngx_http_fastcgi_module：将客户端对php的请求以fastcgi协议转发至指定服务器助理。
ngx_http_uwsgi_module：将客户端对Python的请求以uwsgi协议转发至指定服务器处理。
```

生产环境部署架构：

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559373425972.png)

### - 6.1：实现http反向代理

要求：将用户对www.linux.com的请求转发至后端服务器处理，官方文档：https://nginx.org/en/docs/http/ngx_http_proxy_module.html

环境准备：

```bash
192.168.30.16 #Nginx代理服务器
192.168.30.6  #后端web A，apache部署
192.168.30.26 #后端web B，apache部署
```

#### - 6.1.1：部署后端apache服务器

```bash
yum install httpd -y
echo "this is web1" > /var/www/html/index.html
echo "this is web2" > /var/www/html/index.html
systemctl start httpd
systemctl enable httpd
#访问测试
[root@centos ~]$curl http://192.168.30.6
this is web1
[root@centos ~]$curl http://192.168.30.26
this is web2
```

#### - 6.1.2：Nginx http反向代理入门

官方文档：https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass

6.2：反向代理配置参数

```bash
proxy_pass
#用来设置客户端请求转发给后端服务器的主机，可以是主机名、IP地址：端口的方式。也可以代理到预先设置的主机群组，需要模块gx_http_upstream_module支持。

location /web {
    index index.html;
    proxy_pass http://192.168.30.6:80;
    #若不带斜线访问/web,等于访问后端服务器 http://192.168.7.103:80/web/index.html，即后端服务器配置的站点根目录要有web目录才可以被访问，这是一个追加/web到后端服务器http://servername:port/WEB/INDEX.HTML的操作
    proxy_pass http://192.168.30.6:80/;
    #带斜线，等于访问后端服务器的http://192.168.30.6:80/index.html,内容返回给客户端
}
proxy_hide_header
#用于nginx作为反向代理的时候，在返回给客户端http相应的时候，隐藏后端服务版本相关头部信息，可以设置在http/server或location块：
location /web {
    index index.html;
    proxy_pass http://192.168.30.6:80/;
    proxy_hide_header ETag;
}
proxy_pass_request_body on | off；
#是否向后端服务器发送HTTP包体部分,可以设置在http/server或location块，默认即为开启
proxy_pass_request_headers on | off；
#是否将客户端的请求头部转发给后端服务器，可以设置在http/server或location块，默认即为开启
proxy_set_header；
#可以更改或添加客户端的请求头部信息内容并转发至后端服务器，比如在后端服务器想要获取客户端的真实IP的时
候，就要更改每一个报文的头部，如下：
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#proxy_set_header HOST $remote_addr; 
#添加HOST到报文头部，如果客户端为NAT上网那么其值为客户端的共用的公网IP地址。
proxy_hide_header field;
#用于隐藏后端服务器特定的响应首部，默认nginx在响应报文中不传递后端服务器的首部字段Date, Server, X-Pad, X-Accel等
proxy_connect_timeout time；
#配置nginx服务器与后端服务器尝试建立连接的超时时间，默认为60秒，用法如下：
proxy_connect_timeout 60s；
#60s为自定义nginx与后端服务器建立连接的超时时间
proxy_read_time time；
#配置nginx服务器向后端服务器或服务器组发起read请求后，等待的超时时间，默认60s
proxy_send_time time；
#配置nginx项后端服务器或服务器组发起write请求后，等待的超时时间，默认60s
proxy_http_version 1.0；
#用于设置nginx提供代理服务的HTTP协议的版本，默认http 1.0
proxy_ignore_client_abort off；
#当客户端网络中断请求时，nginx服务器中断其对后端服务器的请求。即如果此项设置为on开启，则服务器会忽略客户端中断并一直等着代理服务执行返回，如果设置为off，则客户端中断后Nginx也会中断客户端请求并立即记录499日志，默认为off。
proxy_headers_hash_bucket_size 64；
#当配置了 proxy_hide_header和proxy_set_header的时候，用于设置nginx保存HTTP报文头的hash表的上
限。
proxy_headers_hash_max_size 512；
#设置proxy_headers_hash_bucket_size的最大可用空间
server_namse_hash_bucket_size 512;
#server_name hash表申请空间大小
server_names_hash_max_szie  512;
#设置服务器名称hash表的上限大小
```

#### - 6.1.3：反向代理示例-单台web服务器

```bash
[root@centos ~]$vim /apps/nginx/conf/conf.d/proxy.conf
server {
    listen 80;
    server_name www.study.com;
    location / {
        proxy_pass http://192.168.30.6:80/;
    }
}
#访问测试
[root@centos ~]$curl -L http://www.study.com
this is web1
```

#### - 6.1.4：反向代理示例-指定location

```bash
server {
    listen 80;
    server_name www.study.com;
    location / {
        index index.html;
        root /data/html;
    }
    location /web {
        proxy_pass http://192.168.30.6:80/;
       # proxy_pass http://192.168.30.26:80/;
    }
}
#访问测试
[root@centos ~]$curl -L http://www.study.com
this is master
[root@centos ~]$curl -L http://www.study.com/web
this is web1
#查看apache访问日志
tail /etc/httpd/logs/access_log
192.168.30.16 - - [01/Jun/2019:16:47:46 +0800] "GET // HTTP/1.0" 200 13 "-" "curl/7.29.0"
192.168.30.16 - - [01/Jun/2019:16:49:54 +0800] "GET / HTTP/1.0" 200 13 "-" "curl/7.29.0"
```

#### - 6.1.5：反向代理示例-缓存功能

缓存功能默认关闭状态

```bash
proxy_cache zone | off; 默认off
#指明调用的缓存，或关闭缓存机制；Context:http, server, location
proxy_cache_key string;
#缓存中用于“键”的内容，默认值：proxy_cache_key $scheme$proxy_host$request_uri;
proxy_cache_valid [code ...] time;
#定义对特定响应码的响应内容的缓存时长，定义在http{...}中
示例:
proxy_cache_valid 200 302 10m;
proxy_cache_valid 404 1m;
proxy_cache_path;
定义可用于proxy功能的缓存；Context:http
proxy_cache_path path [levels=levels] [use_temp_path=on|off]
keys_zone=name:size [inactive=time] [max_size=size] [manager_files=number]
[manager_sleep=time] [manager_threshold=time] [loader_files=number]
[loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number]
[purger_sleep=time] [purger_threshold=time];
示例：在http配置定义缓存信息
 proxy_cache_path /var/cache/nginx/proxy_cache #定义缓存保存路径，proxy_cache会自动创
建
 levels=1:2:2 #定义缓存目录结构层次，1:2:2可以生成2^4x2^8x2^8=1048576个目录
 keys_zone=proxycache:20m #指内存中缓存的大小，主要用于存放key和metadata（如：使用次数） 
 inactive=120s； #缓存有效时间 
 max_size=1g; #最大磁盘占用空间，磁盘存入文件内容的缓存空间最大值
#调用缓存功能，需要定义在相应的配置段，如server{...}；或者location等
   proxy_cache proxycache;
   proxy_cache_key $request_uri;
   proxy_cache_valid 200 302 301 1h;
   proxy_cache_valid any 1m;
proxy_cache_use_stale;
#在被代理的后端服务器出现哪种情况下，可直接使用过期的缓存响应客户端，
proxy_cache_use_stale error | timeout | invalid_header | updating | http_500 |
http_502 | http_503 | http_504 | http_403 | http_404 | off ; #默认是off
proxy_cache_methods GET | HEAD | POST ...;
#对哪些客户端请求方法对应的响应进行缓存，GET和HEAD方法总是被缓存
proxy_set_header field value;
#设定发往后端主机的请求报文的请求首部的值
	Context: http, server, location
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For 			$proxy_add_x_forwarded_for;
		请求报文的标准格式如下：X-Forwarded-For: client1, proxy1, proxy2
```

##### - 6.1.5.1：非缓存场景压测

```bash
[root@centos ~]$ab -n 2000 -c200 http://www.study.com/web/
Total transferred:      526000 bytes
HTML transferred:       26000 bytes
Requests per second:    1113.92 [#/sec] (mean)
Time per request:       179.547 [ms] (mean)
Time per request:       0.898 [ms] (mean, across all concurrent requests)
Transfer rate:          286.09 [Kbytes/sec] received
```

##### - 6.1.5.2：准备缓存配置

```bash
[root@centos ~]$vim /apps/nginx/conf/nginx.conf
proxy_cache_path /data/nginx/proxycache  levels=1:1:1 keys_zone=proxycache:20m inactive=120s max_size=1g; #配置在nginx.conf http配置段
[root@centos ~]$vim /apps/nginx/conf/conf.d/proxy.conf 
location /web { #要缓存的URL 或者放在server配置项对所有URL都进行缓存
 proxy_pass http://192.168.30.6:80/;
 proxy_set_header clientip $remote_addr;
 proxy_cache proxycache;
 proxy_cache_key $request_uri;
 proxy_cache_valid 200 302 301 1h;
 proxy_cache_valid any 1m;
}
[root@centos ~]$ab -n2000 -c200 http://www.study.com/web/
Total transferred:      526000 bytes
HTML transferred:       26000 bytes
Requests per second:    4517.52 [#/sec] (mean)
Time per request:       44.272 [ms] (mean)
Time per request:       0.221 [ms] (mean, across all concurrent requests)
Transfer rate:          1160.26 [Kbytes/sec] received
#验证缓存目录结构
[root@centos ~]$tree /data/nginx/proxycache/
/data/nginx/proxycache/
└── 1
    └── 8
        └── 2
            └── 50d3e5509ac52e81e4ecbc30ed112281

3 directories, 1 file
```

##### - 6.1.5.3：添加头部报文信息

nginx基于模块ngx_http_headers_module可以实现对头部报文添加指定的key与值， https://nginx.org/en/docs/http/ngx_http_headers_module.html

```bash
Syntax: add_header name value [always];
Default: —
Context: http, server, location, if in location
#添加自定义首部，如下：
add_header name value [always];
	add_header X-Via  $server_addr;
	add_header X-Cache $upstream_cache_status;
	add_header X-Accel $server_name;
	add_trailer name value [always];
	添加自定义响应信息的尾部， 1.13.2版后支持
```

6.1.5.4:验证头部信息

第一次访问

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559382546849.png)

第二次访问

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559382573280.png)

### - 6.2：Nginx http反向代理高级应用

Nginx可以基于ngx_http_upstream_module模块提供服务器分组转发、权重分配、状态监测、调度算法等高级功能，官方文档： https://nginx.org/en/docs/http/ngx_http_upstream_module.html

#### - 6.2.1：http upstream配置参数

```bash
upstream name {
    
}
#自定义一组服务器，配置在http内
```

```bash
server address [parameters];
#配置一个后端web服务器，配置在upstream内，至少要有一个server服务器配置。
#server支持的parameters如下：
weight=number #设置权重，默认为1。
max_conns=number #给当前server设置最大活动链接数，默认为0表示没有限制。
max_fails=number #对后端服务器连续监测失败多少次就标记为不可用。
fail_timeout=time #对后端服务器的单次监测超时时间，默认为10秒。
backup #设置为备份服务器，当所有服务器不可用时将重新启用次服务器。
down  #标记为down状态。
resolve #当server定义的是主机名的时候，当A记录发生变化会自动应用新IP而不用重启Nginx。
```

```bash
hash KEY consistent；
#基于指定key做hash计算，使用consistent参数，将使用ketama一致性hash算法，适用于后端是Cache服务器（如varnish）时使用，consistent定义使用一致性hash运算，一致性hash基于取模运算。
所谓取模运算，就是计算两个数相除之后的余数，比如10%7=3, 7%4=3
hash $request_uri consistent; #基于用户请求的uri做hash
```

```bash
ip_hash
#源地址hash调度算法，基于客户端的remote_addr（源地址）做hash运算，以实现会话保持
```

```bash
least_conn
#最少连接调度算法，优先将客户端请求调度到当前连接最少的后端服务器
```

6.2.2：反向代理示例-多台web服务器

```bash
upstream webserver{
	#hash $request_uri consistent;
	#ip_hash；
	#least_conn;
    server 192.168.30.6:80 wgight=1 fail_timeout=5s max_fails=3 #后端服务器状态监测
    server 192.168.30.26:80 weight=1 fail_timeout=5s
    max_fails=3;
}
server {
    listen 80;
    server_name www.study.com;
    location / {
        index index.html;
        root /data/html;
    }
    location /web {
        proxy_pass http://webserver/;
}
}
#访问测试
[root@centos ~]$curl http://www.study.com/web
this is web1
[root@centos ~]$curl http://www.study.com/web
this is web2
```

模拟一台主机宕机，测试nginx服务的可用性

```bash
[root@centos ~]$while true;do curl http://www.study.com/web; sleep 0.5; done
```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559388960937.png)

尝试使用不同算法，查看访问结果

#### - 6.2.2：反向代理示例-客户端IP透传

```bash
location /web {
    index index.html;
 	proxy_pass http://webserver/;
 	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; #添加客户端IP到报文头部
}
#后端web服务器配置
[root@centos ~]$vim /etc/httpd/conf/httpd.conf
LogFormat "%{X-Forwarded-For}i %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
#重启后验证apache日志
[root@centos ~]$vim /var/log/httpd/access_log
192.168.30.6 192.168.30.16 - - [01/Jun/2019:20:15:15 +0800] "GET / HTTP/1.0" 200 13 "-" "curl/7.29.0"
#查看nginx端日志
{"@timestamp":"2019-06-01T20:15:15+08:00",   '"host":"192.168.30.16",'   '"clientip":"192.168.30.6",'   '"size":13,'   '"responsetime":0.001,'   '"upstreamtime":"0.001",'   '"upstreamhost":"192.168.30.26:80",'   '"http_host":"www.study.com",'   '"uri":"/web",'   '"domain":"www.study.com",'   '"xff":"-",'   '"referer":"-",'   '"tcp_xff":"",'   '"http_user_agent":"curl/7.29.0",'   '"status":"200"}'
```

#### - 6.2.3：实现动静分离

要求：将客户端对图片的访问转接到后端nfs服务器，图片文件存放在后端的存储服务器，然后通过nfs共享挂载至nginx上，这样客户端访问图片就会由nginx直接回应

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559391793968.png)

```bash
#共享服务器配置
[root@centos ~]$yum install nfs-utils -y
[root@centos ~]$systemctl start nfs
[root@centos ~]$mkdir /data/share #创建共享目录并上传图片
[root@centos ~]$ll /data/share/
total 80
-rw-r--r-- 1 root root 30762 May 29 19:52 1.jpg
-rw-r--r-- 1 root root 46691 May 29 19:51 2.jpg
[root@centos ~]$vim /etc/exports
/data/share 192.168.30.16(rw,no_root_squash)
[root@centos ~]$exportfs -r
[root@centos ~]$exportfs -v
/data/share   	192.168.30.16(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
#nginx服务器配置
[root@centos images]$showmount -e 192.168.30.36
Export list for 192.168.30.36:
/data/share 192.168.30.16
[root@centos images]$mount 192.168.30.36:/data/share /data/html/images
[root@centos ~]$vim /apps/nginx/conf/conf.d/pc.conf
        location ~* \.(gif|jpg|jpeg|bmp|png|tiff|tif|ico|wmf|js)$ {
           root /data/html/images;
           index index.html;
        }#当访问以上结尾的字符就跳转至/data/html/images下寻找
```

访问测试：

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559393912509.png)

### - 6.3：实现Nginx tcp负载均衡

udp主要用于DNS的域名解析，其配置方式和指令和http 代理类似，其基于ngx_stream_proxy_module模块实现tcp负载，另外基于模块ngx_stream_upstream_module实现后端服务器分组转发、权重分配、状态监测、调度算法等高级功能

#### - 6.3.1：tcp负载均衡配置参数

#### - 6.3.2：负载均衡实例-MySQL

```bash
#服务器安装MySQL
[root@centos ~]$yum install mariadb-server -y
[root@centos ~]$systemctl start mariadb
[root@centos ~]$ mysql_secure_installation #安全初始化
[root@centos ~]$ systemctl enable mariadb
[root@centos ~]$mysql -uroot -pzhuzhuzhu
MariaDB [(none)]> GRANT ALL PRIVILEGES ON *.* TO nguser@'192.168.30.%' IDENTIFIED BY 'zhuzhuzhu' with grant option;
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> exit
#nginx配置
[root@centos ~]$vim /apps/nginx/conf/nginx.conf
include /apps/nginx/conf/tcp/*.conf; #加与http上面
[root@centos ~]$vim /apps/nginx/conf/tcp/tcp.conf
stream {
    upstream mysql_server {
        server 192.168.30.46:3306 max_fails=3 fail_timeout=30s;#mysql服务器IP
    }
    server {
        listen 192.168.30.16:3306;
        proxy_connect_timeout 6s;
        proxy_timeout 15s;
        proxy_pass mysql_server;
    }
}
#重启nginx并执行访问测试
[root@centos ~]$systemctl restart nginx
#测试通过nginx负载连接mysql服务器
[root@centos ~]$mysql -unguser -pzhuzhuzhu -h192.168.30.16
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 15
Server version: 5.5.60-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)

MariaDB [(none)]>
```

### - 6.4：实现FastCGI

**CGI**

Nginx通过某种特定协议将客户端请求转发给第三方服务处理，第三方服务器会新建新的进程处理用户的请求，处理完成后返回数据给Nginx并回收进程，最后nginx在返回给客户端，那这个约定就是通用网关接口(common gateway interface，简称CGI)，CGI（协议）是web服务器和外部应用程序之间的接口标准，是cgi程序和web服务器之间传递信息的标准化接口

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559454589523.png)

**FastCGI**

FastCGI用来提高CGI性能，FastCGI每次处理完请求之后不会关闭掉进程，而是保留这个进程，使这个进程可以处理多个请求。这样的话每个请求都不用再重新创建一个进程了，大大提升了处理效率

**PHP-FPM**

PHP-FPM(FastCGI Process Manager：FastCGI进程管理器)是一个实现了Fastcgi的程序，并且提供进程管理的功能。进程包括master进程和worker进程。master进程只有一个，负责监听端口，接受来自webserver的请求。worker进程一般会有多个，每个进程中会嵌入一个PHP解析器，进行PHP代码的处理

#### - 6.4.1:FastCGI配置指令

Nginx基于模块ngx_http_fastcgi_module实现通过fastcgi协议将指定的客户端请求转发至php-fpm处理，其配置指令如下

```bash
fastcgi_pass address;
#转发请求到后端服务器，address为后端的fastcgi server的地址，可用位置：location
fastcgi_index name;
#fastcgi默认的主页资源，示例：fastcgi_index index.php;
fastcgi_param parameter value [if_not_empty];
#设置传递给FastCGI服务器的参数值，可以是文本，变量或组合，可用于将Nginx的内置变量赋值给自定义key
fastcgi_param REMOTE_ADDR    $remote_addr; #客户端源IP
fastcgi_param REMOTE_PORT    $remote_port; #客户端源端口
fastcgi_param SERVER_ADDR    $server_addr; #请求的服务器IP地址
fastcgi_param SERVER_PORT    $server_port; #请求的服务器端口
fastcgi_param SERVER_NAME    $server_name; #请求的server name
Nginx默认配置示例：
 location ~ \.php$ {
  root      html;
  fastcgi_pass  127.0.0.1:9000
  fastcgi_index index.php;
  fastcgi_param SCRIPT_FILENAME /scripts$fastcgi_script_name; #默认脚本路径
  include    fastcgi_params;
 }
```

缓存定义指令：

```bash
fastcgi_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size
[inactive=time] [max_size=size] [manager_files=number] [manager_sleep=time]
[manager_threshold=time] [loader_files=number] [loader_sleep=time]
[loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time]
[purger_threshold=time];
定义fastcgi的缓存；
path  #缓存位置为磁盘上的文件系统路径
max_size=size #磁盘path路径中用于缓存数据的缓存空间上限
levels=levels#缓存目录的层级数量，以及每一级的目录数量，levels=ONE:TWO:THREE，示例：leves=1:2:2
keys_zone=name:size #设置缓存名称及k/v映射的内存空间的名称及大小
inactive=time #缓存有效时间，默认10分钟，需要在指定时间满足fastcgi_cache_min_uses 次数被视为活动缓存
```

缓存调用指令

```bash
fastcgi_cache zone | off;
#调用指定的缓存空间来缓存数据，可用位置：http, server, location
fastcgi_cache_key string;
#定义用作缓存项的key的字符串，示例：fastcgi_cache_key $request_uri;
fastcgi_cache_methods GET | HEAD | POST ...;
#为哪些请求方法使用缓存
fastcgi_cache_min_uses number;
#缓存空间中的缓存项在inactive定义的非活动时间内至少要被访问到此处所指定的次数方可被认作活动项
fastcgi_keep_conn on | off; 
#收到后端服务器响应后，fastcgi服务器是否关闭连接，建议启用长连接
fastcgi_cache_valid [code ...] time;
#不同的响应码各自的缓存时长
fastcgi_hide_header field; #隐藏响应头指定信息
fastcgi_pass_header field; #返回响应头指定信息，默认不会将Status、X-Accel-...返回
```

#### - 6.4.2:FastCGI示例-nginx与php-fpm在同一服务器

##### - 6.4.2.1：php环境准备

```bash
#yum安装php
[root@centos ~]$yum install php-fpm php-mysql -y
[root@centos ~]$systemctl start php-fpm
[root@centos ~]$systemctl enable php-fpm
Created symlink from /etc/systemd/system/multi-user.target.wants/php-fpm.service to /usr/lib/systemd/system/php-fpm.service.
[root@centos ~]$ps -ef | grep php-fpm
root      10577      1  0 14:20 ?        00:00:00 php-fpm: master process (/etc/php-fpm.conf)
apache    10579  10577  0 14:20 ?        00:00:00 php-fpm: pool www
apache    10580  10577  0 14:20 ?        00:00:00 php-fpm: pool www
apache    10581  10577  0 14:20 ?        00:00:00 php-fpm: pool www
apache    10582  10577  0 14:20 ?        00:00:00 php-fpm: pool www
apache    10583  10577  0 14:20 ?        00:00:00 php-fpm: pool www
root      10651   9837  0 14:35 pts/0    00:00:00 grep --color=auto php-fpm
```

##### - 6.4.2.1：php相关配置优化

```bash
[root@centos ~]$grep "^[a-z]" /etc/php-fpm.conf
include=/etc/php-fpm.d/*.conf
pid = /run/php-fpm/php-fpm.pid
error_log = /var/log/php-fpm/error.log
daemonize = no #是否后台启动
[root@centos ~]$grep "^[a-z]" /etc/php-fpm.d/www.conf
listen = 127.0.0.1:9000 #监听地址及IP
listen.allowed_clients = 127.0.0.1 #允许客户端从哪个源IP访问，如要允许所有只需注释掉即可
user = apache #php-fpm启动用户和组，会涉及到后期文件的权限问题
group = apache
pm = dynamic #动态模式进程管理
pm.max_children = 500 #静态方式下开启的php-fpm进程数量，在动态方式下他限定php-fpm的最大进程数
pm.start_servers = 100 #动态模式下初始进程数，必须大于等于pm.min_spare_servers和小于等于pm.max_children的值
pm.min_spare_servers = 100 #最小空闲进程数
pm.max_spare_servers = 200 #最大空闲进程数
pm.max_requests = 500000 #进程累计请求回收值，会重启
pm.status_path = /pm_status #状态访问URL
ping.path = /ping #ping访问动地址
ping.response = ping-pong #ping返回值
slowlog = /var/log/php-fpm/www-slow.log  #慢日志路径
php_admin_value[error_log] = /var/log/php-fpm/www-error.log #错误日志
php_admin_flag[log_errors] = on #
php_value[session.save_handler] = files #phpsession保存方式及路径
php_value[session.save_path] = /var/lib/php/session #当前使用file保存session的文件路径
[root@centos ~]$systemctl restart php-fpm
```

6.4.2.3：准备php测试页面

```bash
[root@centos ~]$mkdir /data/nginx/php
[root@centos ~]$vim /data/nginx/php/index.php
<?php
phpinfo();
?>
```

##### - 6.4.2.4：nginx配置转发

Nginx安装完成之后默认生成了与fastcgi的相关配置文件，一般保存在nginx的安装路径的conf目录当中，比
如/apps/nginx/conf/fastcgi.conf、/apps/nginx/conf/fastcgi_params

```bash
[root@centos ~]$vim /apps/nginx/conf/conf.d/pc.conf
location ~ \.php$ {
    root /data/nginx/php; #$document_root调用root目录
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME 	$document_root$fastcgi_script_name; #如果SCRIPT_FILENAME是绝对路径则可以省略root /data/nginx/php
    include fastcgi_params;
}
#或者这样写
location ~ \.php$ {
   # root /data/nginx/php; 
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME 	/data/nginx/php/$fastcgi_script_name; 
    include fastcgi_params;
}
[root@centos ~]$nginx -t
nginx: the configuration file /apps/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /apps/nginx/conf/nginx.conf test is successful
[root@centos ~]$nginx -s reload

```

##### - 6.4.2.5：访问验证php测试页面

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559459671302.png)

```bash
常见的错误：
File not found. #路径不对
502： php-fpm处理超时、服务停止运行等原因导致的无法连接或请求超时
```

6.4.2.6：php-fpm的运行状态界面

访问配置文件中指定的路径，会返回php-fpm的当前运行状态

```bash
location ~ ^/(pm_status|ping)$ {
    #access_log off;
 	#allow 127.0.0.1;
	 #deny all;
 	include fastcgi_params;
 	fastcgi_pass 127.0.0.1:9000;
	 fastcgi_param PATH_TRANSLATED $document_root$fastcgi_script_name;
}
[root@centos ~]$systemctl restart nginx
```

访问测试

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559461185987.png)

#### - 6.4.3：FastCGI示例-nginx与php不在同一服务器

nginx会处理静态请求，但是会转发动态请求到后端指定的php-fpm服务器，因此代码也需要放在后端的php-fpm服务器，即静态页面放在Nginx上而动态页面放在后端php-fpm服务器，正常情况下，一般都是采用6.3.2的部署方式

##### - 6.4.3.1：yum安装新版本php-fpm

```bash
[root@centos ~]$cd /usr/local/src/
[root@centos src]$wget 
https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/remi-release-7.rpm
[root@centos src]$yum install remi-release-7.rpm -y
[root@centos src]$rpm -ql remi-release
/etc/pki/rpm-gpg/RPM-GPG-KEY-remi
/etc/pki/rpm-gpg/RPM-GPG-KEY-remi2017
/etc/pki/rpm-gpg/RPM-GPG-KEY-remi2018
/etc/yum.repos.d/remi-glpi91.repo
/etc/yum.repos.d/remi-glpi92.repo
/etc/yum.repos.d/remi-glpi93.repo
/etc/yum.repos.d/remi-glpi94.repo
/etc/yum.repos.d/remi-modular.repo
/etc/yum.repos.d/remi-php54.repo
/etc/yum.repos.d/remi-php70.repo
/etc/yum.repos.d/remi-php71.repo
/etc/yum.repos.d/remi-php72.repo
/etc/yum.repos.d/remi-php73.repo
/etc/yum.repos.d/remi-safe.repo
/etc/yum.repos.d/remi.repo
[root@centos src]$yum install php70-php-fpm php70-php-mysql -y
[root@centos src]$rpm -ql php70-php-fpm
/etc/logrotate.d/php70-php-fpm
/etc/opt/remi/php70/php-fpm.conf
/etc/opt/remi/php70/php-fpm.d
/etc/opt/remi/php70/php-fpm.d/www.conf
/etc/opt/remi/php70/sysconfig/php-fpm
...
```

##### - 6.4.3.2：修改php-fpm配置

php-fpm默认监听在127.0.0.1的9000端口，也就是无法远程连接，因此要做相应的修改

```bash
#修改监听配置
[root@centos src]$vim /etc/opt/remi/php70/php-fpm.d/www.conf
39 listen = 192.168.30.46:9000 #指定监听IP
65 #listen.allowed_clients = 127.0.0.1 #注释运行的客户端
#准备php测试页面
[root@centos src]$mkdir /data/php 
[root@centos src]$vim /data/php/index.php
<?php
	phpinfo();
?>
#启动php-fpm
[root@centos src]$systemctl start php70-php-fpm
[root@centos src]$systemctl enable php70-php-fpm
Created symlink from /etc/systemd/system/multi-user.target.wants/php70-php-fpm.service to /usr/lib/systemd/system/php70-php-fpm.service.
#验证php-fpm进程及端口
[root@centos src]$ps -ef | grep php-fpm
root      12657      1  0 16:22 ?        00:00:00 php-fpm: master process (/etc/opt/remi/php70/php-fpm.conf)
apache    12658  12657  0 16:22 ?        00:00:00 php-fpm: pool www
apache    12659  12657  0 16:22 ?        00:00:00 php-fpm: pool www
apache    12660  12657  0 16:22 ?        00:00:00 php-fpm: pool www
apache    12661  12657  0 16:22 ?        00:00:00 php-fpm: pool www
apache    12662  12657  0 16:22 ?        00:00:00 php-fpm: pool www
root      12710   8639  0 16:27 pts/0    00:00:00 grep --color=auto php-fpm
[root@centos src]$ss -tnl | grep 9000
LISTEN     0      128    192.168.30.46:9000                     *:*     
[root@centos src]$netstat -tuanlp | grep 9000
tcp        0      0 192.168.30.46:9000      0.0.0.0:*               LISTEN      12657/php-fpm: mast 
```

##### - 6.4.3.3：nginx配置转发

```bash
[root@centos ~]$vim /apps/nginx/conf/conf.d/pc.conf
location ~ \.php$ {
    root /data/php; 
    fastcgi_pass 192.168.30.46:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME 	$document_root$fastcgi_script_name; 
    include fastcgi_params;
}
[root@centos ~]$nginx -t
nginx: the configuration file /apps/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /apps/nginx/conf/nginx.conf test is successful
[root@centos ~]$nginx -s reload
```

##### - 6.4.3.4：访问验证php测试页面

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559464565644.png)

## 七、系统参数优化

默认的Linux内核参数考虑的是最通用场景，不符合用于支持高并发访问的Web服务器的定义，根据业务特点来进行调整，当Nginx作为静态web内容服务器、反向代理或者提供压缩服务器的服务器时，内核参数的调整都是不同的，此处针对最通用的、使Nginx支持更多并发请求的TCP网络参数做简单的配置,修改/etc/sysctl.conf来更改内核参数

```bash
fs.file-max = 1000000
#表示单个进程较大可以打net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_reuse = 1
#参数设置为 1 ，表示允许将TIME_WAIT状态的socket重新用于新的TCP链接，这对于服务器来说意义重大，因为总有大量TIME_WAIT状态的链接存在
net.ipv4.tcp_keepalive_time = 600
#当keepalive启动时，TCP发送keepalive消息的频度；默认是2小时，将其设置为10分钟，可更快的清理无效链接
net.ipv4.tcp_fin_timeout = 30
#当服务器主动关闭链接时，socket保持在FIN_WAIT_2状态的较大时间
net.ipv4.tcp_max_tw_buckets = 5000
#表示操作系统允许TIME_WAIT套接字数量的较大值，如超过此值，TIME_WAIT套接字将立刻被清除并打印警告信息,默认为8000，过多的TIME_WAIT套接字会使Web服务器变慢
net.ipv4.ip_local_port_range = 1024 65000
#定义UDP和TCP链接的本地端口的取值范围
net.ipv4.tcp_rmem = 10240 87380 12582912
#定义了TCP接受缓存的最小值、默认值、较大值
net.ipv4.tcp_wmem = 10240 87380 12582912
#定义TCP发送缓存的最小值、默认值、较大值
net.core.netdev_max_backlog = 8096
#当网卡接收数据包的速度大于内核处理速度时，会有一个列队保存这些数据包。这个参数表示该列队的较大值
net.core.rmem_default = 6291456
#表示内核套接字接受缓存区默认大小
net.core.wmem_default = 6291456
#表示内核套接字发送缓存区默认大小
net.core.rmem_max = 12582912
#表示内核套接字接受缓存区较大大小
net.core.wmem_max = 12582912
#表示内核套接字发送缓存区较大大小
注意：以上的四个参数，需要根据业务逻辑和实际的硬件成本来综合考虑
net.ipv4.tcp_syncookies = 1
#与性能无关。用于解决TCP的SYN攻击
net.ipv4.tcp_max_syn_backlog = 8192
#这个参数表示TCP三次握手建立阶段接受SYN请求列队的较大长度，默认1024，将其设置的大一些可使出现Nginx繁忙来不及accept新连接时，Linux不至于丢失客户端发起的链接请求
net.ipv4.tcp_tw_recycle = 1
#这个参数用于设置启用timewait快速回收
net.core.somaxconn=262114
#选项默认值是128，这个参数用于调节系统同时发起的TCP连接数，在高并发的请求中，默认的值可能会导致链接
超时或者重传，因此需要结合高并发请求数来调节此值。
net.ipv4.tcp_max_orphans=262114
#选项用于设定系统中最多有多少个TCP套接字不被关联到任何一个用户文件句柄上。如果超过这个数字，孤立链接将立即被复位并输出警告信息。这个限制指示为了防止简单的DOS攻击，不用过分依靠这个限制甚至认为的减小这个值，更多的情况是增加这个值
```
