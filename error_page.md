---
title: nginx错误页面error_page
date: 2022-01-21 10:50:31
tags:

---

最近网上看到这几篇文章，这里记录一下，分享给大家
nginx要自定义404和401的页面，但是error_page 配置没有生效，没有正常跳转。

```nginx
error_page 404  /404.html;
error_page 404 = http://www.test.com/error.html;
http://tengine.taobao.org/nginx_docs/cn/docs/http/ngx_http_core_module.html#error_page
```
这是因为我们的404静态资源在上游服务器上，而不是当前nginx直接提供

**nginx proxy 启用自定义错误页面：**

语法:proxy_intercept_errors on | off;
默认值:
proxy_intercept_errors off;
上下文:http, server, location
**<u>当被代理的后端服务器的响应状态码大于等于300时，决定是否直接将响应发送给客户端，亦或将响应转发给nginx由error_page指令来处理。</u>**
**<u>proxy_intercept_errors 为on 表示 nginx按照原response code 输出，后端是404就是404。这个变量开启后，我们才能自定义错误页面。</u>**

http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_intercept_errors

#### 一、Nginx在Linux上设置404错误页面

Linux版本：Centos 7.4
Nginx版本：nginx-1.14.0.tar.gz
nginx安装目录参考： /usr/local/nginx则是我的安装目录
说明：我Linux服务器上已经在tomcat上部署了一个项目，使用Nginx进行的代理， 访问项目不存在的页面时，出现的是Nginx默认的404页面，现在我配置我自己写的404页面进行提示

注意：网上大多数博客写的都只有一种情况，要么就是使用 proxy_intercept_errors on;， 要么就是使用fastcgi_intercept_errors on; 没有说明这两种的区别， 还有也没有说明404.html文件应该放在服务器的什么位置。

我在此处优先进行说明：
1、如果你本地有部署项目，优先使用proxy_intercept_errors on;这个配置进行尝试，
2、如果没有部署项目，则使用fastcgi_intercept_errors on; 这个进行尝试，
3、也可以两个全加上， 其次404.html文件放在nginx安装目录的html文件夹下

##### 1.1 第一种配置情况（跳转网络地址）

error_page配置的是http这种的网络地址

在http下配置 proxy_intercept_errors on;
```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;


    proxy_intercept_errors on;
... 以下省略
```
在server下配置 error_page 以下两种情况都可以起作用：
1、error_page 可以配置在server第一层的任何位置， 不受影响
2、error_page 也可以配置在location里面

我下面代码注释的地方都是可以配置的
```nginx
server {
    listen       80;
    server_name  www.xxxxxxx.com;
    #error_page  404  http://www.baidu.com;
    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        proxy_pass http://search-masteryee;
        proxy_set_header   REMOTE-HOST $remote_addr;
        proxy_set_header   Host $host; 
        proxy_set_header   X-Real-IP $remote_addr; 
        proxy_set_header   X-Forwarded-For 	$proxy_add_x_forwarded_for;
        client_max_body_size    20m; 
        #error_page  404  http://www.baidu.com;
    }

    location /upload {
        root   /usr/;
        index  index.html index.htm;
    }

    error_page  404  http://www.baidu.com;
}
```
##### 1.2 第二种配置情况（跳转本地地址）

error_page配置的是本地服务器的页面地址,

说明：
我的404.html页面文件放在nginx安装目录下的html文件夹内
如果编写的404.html页面中有图片等外部文件，使用相对地址是不行的
在http下配置 proxy_intercept_errors on;
```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    proxy_intercept_errors on;
... 以下省略
```
在server中配置error_page 说明：我的nginx安装在/usr/local/下

```nginx
server {
    listen       80;
    server_name  www.xxxxxxx.com;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        proxy_pass http://search-masteryee;
        proxy_set_header   REMOTE-HOST $remote_addr;
        proxy_set_header   Host $host; 
        proxy_set_header   X-Real-IP $remote_addr; 
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size    20m; 
    }

    location /upload {
        root   /usr/;
        index  index.html index.htm;
    }


    error_page   404  /404.html;
    location = /404.html {
        #使用绝对地址， 跳转服务器/usr/local/nginx/html/404.html
        root   /usr/local/nginx/html;
    }

    # 这种方式和上面的方式均可起作用，只需要选择一种即可，本文中没有进行进一步注释
    error_page   404  /404.html;
    location = /404.html {
        # 使用相对地址, 跳转nginx安装目录下的html/404.html
        root   html;
        # 下面这种多了一个/ 反而不起作用
        #root   /html;
    }

    # 以下这几种网上比较多的方式，均试过，无法跳转正确页面或不起跳转作用
    #error_page   404  404.html;
    #error_page   404  /404.html;
    #error_page   404  html/404.html;
    #error_page   404  /html/404.html;
    #error_page   404  /usr/local/nginx/html/404.html;
    #error_page   404  usr/local/nginx/html/404.html;

}
```
可以配置多种返回码的多个错误页面，也可以同时配置多个错误码跳转一个页面，可以同时存在 如下所示
```nginx
server {
    listen       80;
    server_name  www.xxxxxxx.com;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        proxy_pass http://search-masteryee;
        proxy_set_header   REMOTE-HOST $remote_addr;
        proxy_set_header   Host $host; 
        proxy_set_header   X-Real-IP $remote_addr; 
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size    20m; 
    }

    location /upload {
        root   /usr/;
        index  index.html index.htm;
    }

    #error_page  404	/404.html;
    # 错误页面的种类也可以是多个

    # 这里的错误码可以是多个
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }

    # 这里是错误吗也可以是一个
    error_page   404  /404.html;
    location = /404.html {
        root   html;
    }
}
```
##### 1.3 第三种情况（tomcat未启动时）

当我把我的tomcat服务器关掉时，我服务器就没有运行项目了，这时在访问页面，则上述配置没有产生效果，此时则需要添加一个配置 fastcgi_intercept_errors on;

在http下配置 fastcgi_intercept_errors on;
```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    fastcgi_intercept_errors on;
    proxy_intercept_errors on;
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #   
```
server中配置如下，将500 502 503 504等状态码一致性跳转404页面
```nginx
server {
    listen       80;
    server_name  www.xxxxxxx.com;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        proxy_pass http://search-masteryee;
        proxy_set_header   REMOTE-HOST $remote_addr;
        proxy_set_header   Host $host; 
        proxy_set_header   X-Real-IP $remote_addr; 
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size    20m; 
    }

    location /upload {
        root   /usr/;
        index  index.html index.htm;
    }

    #error_page  404	/404.html;

    error_page   500 502 503 504  /404.html;
    error_page   404  /404.html;
    location = /404.html {
        root   html;
    }
}
```
##### 1.4 第四种情况（proxy_intercept_errors的配置地址可多样）

proxy_intercept_errors on;这个配置不一定需要放在http下面，也可以是server下，也可以是server的location下
```nginx
server {
    listen       80;
    server_name  www.masteryee.com;
    proxy_intercept_errors on;
    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        proxy_pass http://search-masteryee;
        proxy_set_header   REMOTE-HOST $remote_addr;
        proxy_set_header   Host $host; 
        proxy_set_header   X-Real-IP $remote_addr; 
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size    20m; 
        #可以是这里
        #proxy_intercept_errors on;
    }
    location /upload {
        root   /usr/;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /404.html;
    error_page   404  /404.html;
    location = /404.html {
        root   html;
    }

}
```
###### 1.5 proxy_intercept_errors和fastcgi_intercept_errors的理解

配置proxy_intercept_errors on; 时配置的错误页面表示的是当服务器返回的状态码为配置的状态码时，才进行页面跳转。如：服务器中没有xxxx.do接口时，访问了这个接口，配置了 proxy_intercept_errors on;则也会进行页面跳转

如果服务器中没有开启服务，则配置proxy_intercept_errors on; 无用，则需要再添加fastcgi_intercept_errors on; 配置， 这样的话，出现页面错误时也会进行跳转

#### 2 Nginx自定义404页面3种方法

一个网站项目，肯定是避免不了404页面的，通常使用Nginx作为Web服务器时，有以下集中配置方式，一起来看看。

##### 第一种：Nginx自己的错误页面

Nginx访问一个静态的html 页面，当这个页面没有的时候，Nginx抛出404，那么如何返回给客户端404呢？

看下面的配置，这种情况下不需要修改任何参数，就能实现这个功能。

```nginx
server {
    listen      80;
    server_name  www.test.com;
    root   /var/www/test;
    index  index.html index.htm;

    location / {
    }

    # 定义错误页面码，如果出现相应的错误页面码，转发到那里。
    error_page  404 403 500 502 503 504  /404.html;

    # 承接上面的location。
    location = /404.html {
        # 放错误页面的目录路径。
        root   /usr/share/nginx/html;
    }
}
```
##### 第二种：反向代理的错误页面

如果后台Tomcat处理报错抛出404，想把这个状态叫Nginx反馈给客户端或者重定向到某个连接，配置如下：

```nginx

upstream www {
    server 192.168.1.201:7777  weight=20 max_fails=2 fail_timeout=30s;
    ip_hash;
}

server {
    listen       80;
    server_name www.test.com;
    root   /var/www/test;
    index  index.html index.htm;

    location / {
        if ($request_uri ~* ‘^/$’) {
            rewrite .*   http://www.test.com/index.html redirect;
        }

        # 关键参数：这个变量开启后，我们才能自定义错误页面，当后端返回404，nginx拦截错误定义错误页面
        proxy_intercept_errors on;

        proxy_pass      http://www;
        proxy_set_header HOST   $host;
        proxy_set_header X-Real-IP      $remote_addr;
        proxy_set_header X-Forwarded-FOR $proxy_add_x_forwarded_for;
    }

    error_page    404  /404.html;

    location = /404.html {
        root   /usr/share/nginx/html;
    }
}
```
##### 第三种：Nginx解析php代码的错误页面

如果后端是php解析的，需要加一个变量

在http段中加一个变量 fastcgi_intercept_errors on 就可以了。

指定一个错误页面：

```nginx

error_page    404  /404.html;

location = /404.html {
    root   /usr/share/nginx/html;
}

```
指定一个url地址：

```nginx
error_page 404  /404.html;
error_page 404 = http://www.test.com/error.html;
```
参考：
https://cloud.tencent.com/developer/article/1488582

https://www.centos.bz/2017/08/nginx-custom-404-page/
————————————————