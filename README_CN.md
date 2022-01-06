## 使用nginx+image_filter_module作为S3的图片处理服务器配置

### 一、安装依赖库

要编译image_filter模块与https模块，需要安装 zlib libpng libjpeg libvpx libtiff libfreetype gd-devel pcre-devel libcurl-devel openssl libwebp等模块

```bash
yum -y install gcc glibc zlib libpng libjpeg libvpx libtiff libfreetype gd-devel pcre-devel libcurl-devel openssl libwebp pcre
```

### 二、下载nginx源码

```bash
wget http://nginx.org/download/nginx-1.21.5.tar.gz
```

### 三、编译

编译时需要把ssl与image_filter相关的模块带上

```bash
tar zxvf nginx-1.21.5.tar.gz
cd nginx-1.21.5
./configure --prefix=/usr/local/nginx--with-http_stub_status_module --with-http_ssl_module --with-file-aio --with-http_realip_module --with-pcre --with-http_image_filter_module --with-debug
make
make install
```

### 四、使用说明

在正常S3网关生成的图片分享地址uri路径之前加上/<命令>/<参数>/即可

如原公开地址: http://192.168.1.5:7480/bucket1/img/test1.jpg

原私有地址:  http://192.168.1.5:7480/bucket2/img/test2.jpg?AWSAccessKeyId=8CJC7G2JIBLKIYKJ589O&Expires=1642068269&Signature=p%2FfHYSnHOh70xViGoO5gcA%2F5rEE%3D

#### 1. 缩放

命令: resize

参数: 宽x高

说明: 缩放是按比例自动适应的，以缩放后最大边为准。

如: 

缩放后公有地址: http://192.168.1.3:8000/resize/360x240/bucket1/img/test1.jpg

缩放后私有地址:  http://192.168.1.3:8000/resize/360x240/bucket2/img/test2.jpg?AWSAccessKeyId=8CJC7G2JIBLKIYKJ589O&Expires=1642068269&Signature=p%2FfHYSnHOh70xViGoO5gcA%2F5rEE%3D

#### 2. 剪切

命令: crop

参数: 宽x高

说明：剪切是先缩放到相应尺寸后再居中剪切。

如: 

缩放后公有地址: http://192.168.1.3:8000/crop/240x240/bucket1/img/test1.jpg

缩放后私有地址:  http://192.168.1.3:8000/crop/240x240/bucket2/img/test2.jpg?AWSAccessKeyId=8CJC7G2JIBLKIYKJ589O&Expires=1642068269&Signature=p%2FfHYSnHOh70xViGoO5gcA%2F5rEE%3D

#### 3. 旋转

命令: rotate

参数: 旋转角度，只有 90|180|270这三种角度有效

如: 

缩放后公有地址: http://192.168.1.3:8000/rotate/90/bucket1/img/test1.jpg

缩放后私有地址:  http://192.168.1.3:8000/rotate/180/bucket2/img/test2.jpg?AWSAccessKeyId=8CJC7G2JIBLKIYKJ589O&Expires=1642068269&Signature=p%2FfHYSnHOh70xViGoO5gcA%2F5rEE%3D



### 五、修改nginx.conf配置，实现图片处理与代理

假设S3 API接口地址是 http://192.168.1.5:7480  http://192.168.1.6:7480  (Ceph + rgw)，nginx侦听端口为8000

*说明：本nginx只能做图片缩放、剪切、旋转功能代理，**不可作为S3 API代理(**S3API的header中签名算法会导致代理后验签失败)*

```

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"' '$upstream_addr';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    upstream rgw.ceph {
        server 192.168.1.5:7480;
        server 192.168.1.6:7480;
    }

    proxy_cache_path /tmp/nginx-thumbnails levels=1:2 keys_zone=thumbnail_cache:256M inactive=60d max_size=2048M;



    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    server {
        listen       8000;
        server_name  localhost;
        access_log  logs/access_8000.log  main;
        error_log  logs/error_8000.log error;

        location  / {
            root   html;
            index  index.html index.htm;
            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_connect_timeout 300s;
            proxy_send_timeout 300s;
            proxy_read_timeout 300s;
            proxy_pass http://rgw.ceph/;
            error_page   415 = /empty;
        }

        location = /empty {
            empty_gif;
        }

        location ~ ^/resize/(\d+)x(\d+)/(.*)$ {
            image_filter test;
            #root html;
            set $ur $3;
            if ($args != "") {
               set $ur $3?$args;
            }
            set $width $1;
            set $height $2;

            #add_header debug_nginx "$1,$2,$3,$4";
            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_pass http://rgw.ceph/$ur;
            if ($width = "0") {
               set $width "-";
            }
            if ($height = "0") {
               set $height "-";
            }
            image_filter_buffer 200M;
            image_filter_interlace on;
            image_filter resize $width $height;
            error_page   415 = /empty;
       }

        location ~ ^/crop/(\d+)x(\d+)/(.*)$ {
            image_filter test;
            set $ur $3;
            if ($args != "") {
               set $ur $3?$args;
            }
            set $width $1;
            set $height $2;

            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_pass http://rgw.ceph/$ur;
            if ($width = "0") {
               set $width "-";
            }
            if ($height = "0") {
               set $height "-";
            }
            image_filter_buffer 200M;
            image_filter_interlace on;
            image_filter crop $width $height;
            error_page   415 = /empty;
       }

        location ~ ^/rotate/(\d+)/(.*)$ {
            image_filter test;
            set $ur $2;
            if ($args != "") {
               set $ur $2?$args;
            }
            set $degree $1;

            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_pass http://rgw.ceph/$ur;
            image_filter_buffer 200M;
            image_filter_interlace on;
            image_filter rotate $degree;
            error_page   415 = /empty;
       }


    }


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

```


