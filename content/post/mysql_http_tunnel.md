---
title: "【mysql】通过http tunnel连接mysql"
date: 2020-12-12T18:05:54+08:00
draft: true
---

* 背景
>     (1) 公司为了降低工作效率, 连接mysql必须通过特定的代理才能连接
>     (2) 产线这个模式可以理解, 但是测试环境你也搞个这个,那我就要哭了
>     (3) 在公司用wifi可以连接,但是在家无法直接连接mysql,只能通过垃圾的网页版mysql进行数据库操作
>     (4) 可以通过网页访问docker的权限,并没有任何服务器权限
![image.png](https://image-static.segmentfault.com/403/980/403980092-5e8016c8c717a)
--------------------------------------------------
* 解决方案
>     (1) nginx的stream实现tcp流量转发
>     (2) nginx的http proxy connect实现http tunnel
>     (3) Navicat Premium的http tunnel实现
--------------------------------------------------
* 方案一
    * 在/etc/nginx/nginx.conf的第一行加上
        ```
        load_module /usr/lib64/nginx/modules/ngx_stream_module.so;
        ```
        这里modules的路径可以通过`nginx -V`的`--modules-path`看到
        ```
        stream {
            server {
                listen     443;
                proxy_pass stream_backend;
            }
        }
        ```

    到这这进行不下去了, 此方案行不通

    - 1、因为nginx的stream是基于tcp/ip的第4层,所以没有http中header的host参数,也就没有基于server_name的配置
    - 2、服务器主机也只会转发http和https流量到docker,stream在host的nginx没配置转发,所以tcp流量根本过不来
----------------------------------------------------
* 方案二
    * 在/etc/nginx/nginx.conf的第一行加上
        ```
        load_module /usr/lib64/nginx/modules/ngx_http_proxy_connect_module.so;
        ```

    * nginx -s reload   
        > ```nginx: [emerg] dlopen() "/usr/lib64/nginx/modules/ngx_http_proxy_connect_module.so" failed (/usr/lib64/nginx/modules/ngx_http_proxy_connect_module.so: cannot open shared object file: No such file or directory) in /etc/nginx/nginx.conf```

    *  又要开始折腾了, 没有ngx_http_proxy_connect_module, google了一把, [ngx_http_proxy_connect_module](https://github.com/chobits/ngx_http_proxy_connect_module)
    *  相当于重新编译nginx, 有几个坑, 不过意外发现有个go的开源库,专门编译nginx,刚好我是go开发,go的库[nginx-build](https://github.com/cubicdaiya/nginx-build), 但是下面说的跟这个库没关系, 只是打个go的广告^_^
     >     (1) 选定对应的nginx源码zip,更改nginx的version, nginx -v
     >     (2) ./configure的编译参数, 需要提前nginx -V复制之前的启动参数下来
     >     (3) 执行path的时候需要看`https://github.com/chobits/ngx_http_proxy_connect_module`的说明,哪个版本用哪个patch
     ```
     $ git clone https://github.com/chobits/ngx_http_proxy_connect_module.git /root/ngx_http_proxy_connect_module
     $ wget http://nginx.org/download/nginx-1.12.2.tar.gz
     $ tar -xzvf nginx-1.12.2.tar.gz
     $ cd nginx-1.12.2/
     $ patch -p1 < /root/ngx_http_proxy_connect_module/patch/proxy_connect.patch
     $ ./configure --add-module=/root/ngx_http_proxy_connect_module ·后面接上nginx -V里面的configure arguments·
     $ make && make install
     $ cp /root/ngx_http_proxy_connect_module
     ```

    *     执行过程
      ```
      $ ./configure --add-dynamic-module=/root/ngx_http_proxy_connect_module --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/run/nginx.pid --lock-path=/run/lock/subsys/nginx --user=nginx --group=nginx --with-file-aio --with-ipv6 --with-http_auth_request_module --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_addition_module --with-http_xslt_module=dynamic --with-http_image_filter_module=dynamic --with-http_geoip_module=dynamic --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_degradation_module --with-http_slice_module --with-http_stub_status_module --with-http_perl_module=dynamic --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_ssl_module --with-google_perftools_module --with-debug --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -m64 -mtune=generic' --with-ld-opt='-Wl,-z,relro -specs=/usr/lib/rpm/redhat/redhat-hardened-ld -Wl,-E'
      ```

    * 出现各种错误,各种google,最终
      ```
      yum -y install redhat-rpm-config.noarch
      yum -y install libxslt-devel 
      yum -y install perl-devel perl-ExtUtils-Embed 
      yum -y install gd-devel
      yum -y install geoip-devel
      yum -y install gperftools-devel
      ```

    * 最后编译只出现warning类的报错才算成功
```
    Configuration summary
          + using system PCRE library
          + using system OpenSSL library
          + using system zlib library
          nginx path prefix: "/usr/share/nginx"
          nginx binary file: "/usr/sbin/nginx"
          nginx modules path: "/usr/lib64/nginx/modules"
          nginx configuration prefix: "/etc/nginx"
          nginx configuration file: "/etc/nginx/nginx.conf"
          nginx pid file: "/run/nginx.pid"
          nginx error log file: "/var/log/nginx/error.log"
          nginx http access log file: "/var/log/nginx/access.log"
          nginx http client request body temporary files: "/var/lib/nginx/tmp/client_body"
          nginx http proxy temporary files: "/var/lib/nginx/tmp/proxy"
          nginx http fastcgi temporary files: "/var/lib/nginx/tmp/fastcgi"
          nginx http uwsgi temporary files: "/var/lib/nginx/tmp/uwsgi"
          nginx http scgi temporary files: "/var/lib/nginx/tmp/scgi"

        ./configure: warning: the "--with-ipv6" option is deprecated
```

* `ls /usr/lib64/nginx/modules` 就会发现ngx_http_proxy_connect_module.so文件已经生成了, /usr/lib64/nginx/modules是modules路径, nginx -V可以看到--modules-path
* 开始配置nginx
```
    stream {
        server {
             listen 443;
             server_name t1.test.example.com;

             proxy_connect;
             proxy_connect_allow            443;
             proxy_connect_connect_timeout  10s;
             proxy_connect_read_timeout     10s;
             proxy_connect_send_timeout     10s;

             proxy_pass http://$http_host;
       }
    }
```


* 尴尬的发现到这里又做下不去了,只能转发http的流量,无法利用http的tunnel改为tcp流量,到此结束
--------------------
* 方案三
    * 前面都死在半路上, 不甘心啊. 
    * 意外发现Navicat Premium的连接配置竟然有个http tunnel,惊喜啊, google了一下,原来是利用php访问mysql,然后再把数据转发回来
    * 对于一个php小白, 只会go和c++, 只能google下, 一通胡搞
    * 执行过程
  >     (1) 复制Navicat Premium目录下的`/Applications/Navicat Premium.app/Contents/Resources/ntunnel_mysql.php`到docker的nginx的静态文件目录www下, 这样nginx可以访问到
  >     (2) 安装php-fpm和php-mysql,yum install -y php-fpm php-mysql, 安装完成后运行php-fpm & .
  >     (3) 配置nginx
```
    server {
        listen      80;
        server_name t1.test.example.com;
        root        /etc/nginx/www;
        access_log  /var/log/nginx/access.log  main;
        error_log   /var/log/nginx/error.log;
        location /ntunnel_mysql.php {
            fastcgi_pass 127.0.0.1:9000;
            include fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME /etc/nginx/www/ntunnel_mysql.php;
        }
    }
```

* nginx -s reload, 访问https://t1.test.example.com/ntunnel_mysql.php
     ![image.png](/img/bVbFgcl)
* 在Navicat Premium的http tunnel输入上面这个链接就可以了
   

大功告成!!!!
    