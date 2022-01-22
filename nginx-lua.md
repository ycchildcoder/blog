---
title: nginx_lua 阶段
date: 2022-01-21 16:08:42
tags:
---

对刚接触Ngx_lua的读者来说，可能会存在下面两个困惑。

1、Lua在Nginx的哪些阶段可以执行代码？
2、Lua在Nginx的每个阶段可以执行哪些操作？

只有理解了这两个问题，才能在业务中巧妙地利用Ngx_Lua来完成各项需求。

Nginx的11个执行阶段，每个阶段都有自己能够执行的指令，并可以实现不同的功能。Ngx_Lua的功能大部分是基于Nginx这11个执行阶段开发和配置的，Lua代码在这些指令块中执行，并依赖于它们的执行顺序。本章将对Ngx_Lua的执行阶段进行一一讲解。

Nginx和Lua最基本的构建块脚本是指令集。指令集用于确定什么时候用户的Lua代码运行和结果将会被怎么使用。下面的图显示指令集的执行顺序。

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2699372-010f4989264186a1.png)

Lua Nginx 模块指令集



| 指令                                           | 所处处理阶段         | 使用范围                              | 解释                                                         |
| :--------------------------------------------- | :------------------- | :------------------------------------ | :----------------------------------------------------------- |
| init_by_lua init_by_lua_file                   | loading-config       | http                                  | nginx Master**进程加载配置**时执行； 通常用于初始化全局配置/预加载Lua模块 |
| init_worker_by_lua init_worker_by_lua_file     | starting-worker      | http                                  | 每个Nginx Worker**进程启动时调用的计时器**，如果Master进程不允许则只会在init_by_lua之后调用； 通常用于定时拉取配置/数据，或者后端服务的健康检查 |
| set_by_lua set_by_lua_file                     | rewrite              | server,server if,location,location if | **设置nginx变量**，可以实现复杂的赋值逻辑；**此处是阻塞的**，Lua代码要做到非常快； |
| rewrite_by_lua rewrite_by_lua_file             | rewrite tail         | http,server,location,location if      | rewrite阶段处理，可以实现**复杂的转发/重定向逻辑**；         |
| access_by_lua access_by_lua_file               | access tail          | http,server,location,location if      | 请求访问阶段处理，用于**访问控制**                           |
| content_by_lua content_by_lua_file             | content              | location，location if                 | 内容处理器，**接收请求处理**并输出响应                       |
| header_filter_by_lua header_filter_by_lua_file | output-header-filter | http，server，location，location if   | 设置header和cookie                                           |
| body_filter_by_lua body_filter_by_lua_file     | output-body-filter   | http，server，location，location if   | 对响应数据进行过滤，比如截断、替换。                         |
| log_by_lua log_by_lua_file                     | log                  | http，server，location，location if   | log阶段处理，比如记录访问量/统计平均响应时间                 |

更详细的解释请参考http://wiki.nginx.org/HttpLuaModule#Directives。如上指令很多并不常用，因此我们只拿其中的一部分做演示。

## init_by_lua

每次Nginx重新加载配置时执行，可以用它来完成一些耗时模块的加载，或者初始化一些全局配置；在Master进程创建Worker进程时，此指令中加载的全局变量会进行Copy-OnWrite，即会复制到所有全局变量到Worker进程。

#### 1. nginx.conf配置文件中的http部分添加如下代码

```nginx
#共享全局变量，在所有worker间共享
lua_shared_dict shared_data 1m;

init_by_lua_file /usr/example/lua/init.lua;
```

#### 2. init.lua

```nginx
-- 初始化耗时的模块
local redis = require("resty.redis")
local cjson = require("cjson")

-- 全局变量，不推荐
count = 1

-- 共享全局内存
local shared_data = ngx.shared.shared_data
shared_data:set("count", 1)
```

### 一、 init_by_lua_block

init_by_lua_block是init_by_lua的替代版本，在OpenResty 1.9.3.1或Lua-Nginx-Modulev 0.9.17之前使用的都是init_by_lua。init_by_lua_block比init_by_lua更灵活，所以建议优先选用init_by_lua_block。
本章中的执行阶段都采用*_block格式的指令，后续不再说明。

#### 1.1　阶段说明
语法：init_by_lua_block {lua-script-str}
配置环境：http
阶段：loading-config
含义：当Nginx的master进程加载Nginx配置文件（加载或重启Nginx进程）时，会在全局的Lua VM（Virtual Machine，虚拟机）层上运行<lua-script-str> 指定的代码，每次当Nginx获得HUP（即Hangup）重载信号加载进程时，代码都会被重新执行。

#### 1.2 初始化配置
在loading-config阶段一般会执行如下操作。
1．初始化Lua全局变量，特别适合用来处理在启动master进程时就要求存在的数据，对CPU消耗较多的功能也可以在此处处理。
2．预加载模块。
3．初始化lua_shared_dict共享内存的数据（关于共享内存详见第10章）。
示例如下：
```nginx
user  webuser webuser;
worker_processes  1;
worker_rlimit_nofile 10240;

events {
    use epoll;
    worker_connections  10240;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format main '$remote_addr-$remote_user[$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" ' 
                      '"$http_user_agent" "$http_x_forwarded_for" "$request_time" "$upstream_addr $upstream_status $upstream_response_time"  "upstream_time_sum:$upstream_time_sum"  "jk_uri:$jk_uri"';
    access_log  logs/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    

    lua_package_path "/usr/local/nginx_1.12.2/conf/lua_modules/?.lua;;";
    lua_package_cpath "/usr/local/nginx_1.12.2/conf/lua_modules/c_package/?.so;;";
    
    lua_shared_dict dict_a 100k;  --声明一个Lua共享内存，dict_a为100KB
    init_by_lua_block {
-- cjson.so文件需要手动存放在lua_package_cpath搜索路径下，如果是OpenResty，就不必了，因为它默认支持该操作
          cjson = require "cjson"; 
          local dict_a = ngx.shared.dict_a;
          dict_a:set("abc", 9)
    }

    server {
        listen       80;
        server_name  testnginx.com;
        location / {
           content_by_lua_block {
               ngx.say(cjson.encode({a = 1, b = 2}))
               local dict_a = ngx.shared.dict_a;
               ngx.say("abc=",dict_a:get("abc"))
           }
        }
    }
}
```
执行结果如下：

curl -I http://testnginx.com/
{"a":1,"b":2}
abc=9

#### 1.3　控制初始值
在init_by_lua_block阶段设置的初始值，即使在其他执行阶段被修改过，当Nginx重载配置时，这些值就又会恢复到初始状态。如果在重载Nginx配置时不希望再次改动这些初始值，可以在代码中做如下调整。
```nginx
init_by_lua_block {
      local cjson = require "cjson";
      local dict_a = ngx.shared.dict_a;
      local v = dict_a:get("abc");  --判断初始值是否已经被set
      if not v then                 --如果没有，就执行初始化操作
        dict_a:set("abc", 9)
      end
}
```
#### 1.4　init_by_lua_file
init_by_lua_file和init_by_lua_block的作用一样，主要用于将init_by_lua_block的内容转存到指定的文件中，这样Lua代码就不必全部写在Nginx配置里了，易读性更强。
init_by_lua_file支持配置绝对路径和相对路径。**相对路径是在启动Nginx时由-p PATH 决定的**，如果启动Nginx时没有配置-p PATH，就会使用编译时–prefix的值，该值一般存放在Nginx的prefix（也可以用prefix（也可以用{prefix}来表示）变量中。init_by_lua_file和Nginx的include指令的相对路径一致。
举例如下：

```nginx
init_by_lua_file conf/lua/init.lua;  --相对路径
init_by_lua_file /usr/local/nginx/conf/lua/init.lua; --绝对路径
init.lua文件的内容如下：
cjson = require "cjson"
local dict_a = ngx.shared.dict_a
local v = dict_a:get("abc")
if not v then
   dict_a:set("abc", 9)
end  
```
1.5　可使用的Lua API指令
init_by_lua是Nginx配置加载的阶段，很多Nginx API for Lua命令是不支持的。目前已知支持的Nginx API for Lua的命令有ngx.log、ngx.shared.DICT、print。
注意：init_by_lua中的表示通配符，init_by_lua即所有以init_by_lua开头的API。后续的通配符亦是如此，不再另行说明。

二、init_worker_by_lua_block
2.1　阶段说明
**语法：**init_worker_by_lua_block {lua-script-str}
**配置环境：**http
**阶段：**starting-worker
**含义：**当master进程被启动后，每个worker进程都会执行Lua代码。如果Nginx禁用了master进程，init_by_lua*将会直接运行。

2.2　启动Nginx的定时任务
在init_worker_by_lua_block执行阶段最常见的功能是执行定时任务。示例如下：
```nginx
user  webuser webuser;
worker_processes  3;
worker_rlimit_nofile 10240;

events {
    use epoll;
    worker_connections  10240;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    upstream  test_12 {
        server 127.0.0.1:81  weight=20  max_fails=300000 fail_timeout=5s;
        server 127.0.0.1:82  weight=20  max_fails=300000 fail_timeout=5s;
    }
    
    lua_package_path "${prefix}conf/lua_modules/?.lua;;";
    lua_package_cpath "${prefix}conf/lua_modules/c_package/?.so;;";
    
    init_worker_by_lua_block  {
        local delay = 3  --3秒
        local cron_a   
    --定时任务的函数
        cron_a = function (premature)   
            if not  premature then  --如果执行函数时没有传参，则该任务会一直被触发执行
                ngx.log(ngx.ERR, "Just do it !")
            end
    
        end
    --每隔delay参数值的时间，就执行一次cron_a函数
        local ok, err = ngx.timer.every(delay, cron_a)   
        if not ok then
            ngx.log(ngx.ERR, "failed to create the timer: ", err)
            return
        end
    }
}
```
2.3　动态进行后端健康检查
在init_worker_by_lua_block阶段，也可以实现后端健康检查的功能，用于检查后端HTTP服务是否正常，类似于Nginx商业版中的health_check功能。
如果使用OpenResty 1.9.3.2及以上的版本，默认已支持此模块；如果使用Nginx，则首先需要安装此模块
实现动态进行后端健康检查的功能，配置如下：
```nginx
user  webuser webuser;
worker_processes  3;
worker_rlimit_nofile 10240;

events {
    use epoll;
    worker_connections  10240;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    upstream  test_12 {
        server 127.0.0.1:81  weight=20  max_fails=10 fail_timeout=5s;
        server 127.0.0.1:82  weight=20  max_fails=10 fail_timeout=5s;
        server 127.0.0.1:8231  weight=20  max_fails=10 fail_timeout=5s;
    }
    lua_shared_dict healthcheck 1m;  # 存放upstream servers的共享内存，后端服务器组越多，配置就越大
    lua_socket_log_errors off;  # 当TCP发送失败时，会发送error日志到error.log中，该过程会增加性能开销，建议关闭，以避免在健康检查过程中出现多台服务器宕机的情况，异常情况请使用ngx.log来记录
    lua_package_path "${prefix}conf/lua_modules/?.lua;;";
    lua_package_cpath "${prefix}conf/lua_modules/c_package/?.so;;";
    
    init_worker_by_lua_block  {
        local hc = require "resty.upstream.healthcheck"
        local ok, err = hc.spawn_checker{
            shm = "healthcheck",  -- 使用共享内存 
            upstream = "test_12", -- 进行健康检查的upstream名字
            type = "http",        -- 检查类型是http
            http_req = "GET /status HTTP/1.0\r\nHost: testnginx.com\r\n\r\n",
                    -- 用来发送HTTP请求的格式和数据，核实服务是否正常 
            interval = 3000,  -- 设置检查的频率为每3s一次
            timeout = 1000,   -- 设置请求的超时时间为1s
            fall = 3,  --设置连续失败3次后就把后端服务改为down 
            rise = 2,  --设置连续成功2次后就把后端服务改为up 
        valid_statuses = {200, 302},  --设置请求成功的响应状态码是200和302
            concurrency = 10,  --设置发送请求的并发等级
        }
        if not ok then
            ngx.log(ngx.ERR, "failed to spawn health checker: ", err)
            return
        end
    }
    server {
        listen       80;
        server_name  testnginx.com;
        location / {
           proxy_pass http://test_12;
        }
        # /status 定义了后端健康检查结果的输出页面
        location = /status {
            default_type text/plain;
            content_by_lua_block {
                local hc = require "resty.upstream.healthcheck"
                --输出当前检查结果是哪个worker进程的
                ngx.say("Nginx Worker PID: ", ngx.worker.pid())
--status_page()输出后端服务器的详细情况
                ngx.print(hc.status_page())
            }
        }
    }
}
```

访问http://testnginx.com/status查看检查的结果，图8-1所示为健康检查数据结果。

图8-1　健康检查数据结果
如果要检查多个upstream，则配置如下（只有黑色加粗位置的配置有变化）：
```nginx
local ok, err = hc.spawn_checker{
    shm = "healthcheck",
    upstream = "test_12",
    type = "http",

    http_req = "GET /status HTTP/1.0\r\nHost: testnginx.com\r\n\r\n",
    
    interval = 3000,
    timeout = 1000,
    fall = 3,
    rise = 2,
    valid_statuses = {200, 302},  
    concurrency = 10, 
}
local ok, err = hc.spawn_checker{
    shm = "healthcheck",
    upstream = "test_34",
    type = "http",

    http_req = "GET /test HTTP/1.0\r\nHost: testnginx.com\r\n\r\n",
    interval = 3000,
    timeout = 1000,
    fall = 3,
    rise = 2,
    valid_statuses = {200, 302}, 
    concurrency = 10,  
}
```

如果把lua_socket_log_errors设置为on，那么当有异常出现时，例如出现了超时，就会往error.log里写日志，日志记录如图8-2所示。

图8-2　日志记录
经过lua-resty-upstream-healthcheck的健康检查发现异常的服务器后，Nginx会动态地将异常服务器在upstream中禁用，以实现更精准的故障转移。

三、set_by_lua_block
3.1　阶段说明
**语法：**set_by_lua_block res {lua-script-str} 
**配置环境：**server，server if，location，location if 
**阶段：**rewrite 
**含义：**执行<lua-script-str>代码，并将返回的字符串赋值给res

3.2　变量赋值
本指令一次只能返回一个值，并赋值给变量res（即只有一个res（即只有一个res被赋值），示例如下：
```nginx
server {
    listen       80;
    server_name  testnginx.com;
    location / {
    set $a '';
    set_by_lua_block $a {
         local t = 'tes'
         return t
    }
    return 200 $a;
}
```

执行结果如下：

  curl http://testnginx.com/
test

那如果希望返回多个变量，该怎么办呢？可以使用ngx.var.VARIABLE，示例如下：
```nginx
server {
    listen       80;
    server_name  testnginx.com;
    location / {
    #使用ngx.var.VARIABLE前需先定义变量
    set $b '';
    set_by_lua_block $a {
        local t = 'test'
        ngx.var.b = 'test_b'
        return t
    }
    return 200 $a,$b;
    }
}
```

执行结果如下：

  curl http://testnginx.com/test
test,test_b

3.2　Rewrite阶段的混用模式
因为set_by_lua_block处在rewrite阶段，所以它可以和ngx_http_rewrite_module、set-misc-nginx-module，以及array-var-nginx-module一起使用，在代码执行时，按照配置文件的顺序从上到下执行，示例如下：
```nginx
server {
    listen       80;
    server_name  testnginx.com;
    location / {

    set $a '123';
    set_by_lua_block $b {
        local t = 'bbb'
        return t
    }
    set_by_lua_block $c {
        local t = 'ccc'  .. ngx.var.b
        return t
    }
    set $d "456$c"; 
    return 200 $a,$b,$c,$d;
    }

}
```

从执行结果可以看出数据是从上到下执行的，如下所示：

 curl  http://testnginx.com/test
123,bbb,cccbbb,456cccbbb
1.
2.
3.3　阻塞事件
set_by_lua_block指令块在Nginx中执行的指令是阻塞型操作，因此应尽量在这个阶段执行轻、快、短、少的代码，以避免耗时过多。set_by_lua_block不支持非阻塞I/O，所以不支持yield当前Lua的轻线程。
3.4　被禁用的Lua API指令
在set_by_lua_block阶段的上下文中，下面的Lua API是被禁止的（只罗列部分）。
输出类型的API函数（如ngx.say和ngx.send_headers）。
控制类型的API函数（如ngx.exit）。
子请求的API函数（如ngx.location.capture和ngx.location.capture_multi）。
Cosocket API函数（如ngx.socket.tcp和ngx.req.socket）。
休眠API函数ngx.sleep。