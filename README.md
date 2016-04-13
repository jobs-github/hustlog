# hustlog - 高性能的网络日志服务 #

`hustlog` 是一个高性能的网络日志服务，在 `zlog` 的基础上以 `nginx` 模块的方式提供了高可用的网络层以及 `http` 接口。

## 特性 ##

`zlog` 是一套高性能的单机版日志服务程序，`nginx` 是工业级的 `http server` 标准，得益于此，`hustlog` 具备以下特性：
  
* 高吞吐量。性能测试 `QPS` 在 **1.2w** 以上。  
* 高并发。参考 `nginx` 的并发能力。  
* 高可用性。参考 `nginx` 的 `master-worker` 架构。

## 依赖 ##
* [zlog](https://github.com/HardySimpson/zlog)

`zlog` 的安装方法：

    $ git clone https://github.com/HardySimpson/zlog
    $ cd zlog
    $ make
    $ sudo make install
    $ sudo vi /etc/ld.so.conf
    $ /usr/local/lib
    $ sudo ldconfig

## 部署 ##
    $ cd nginx
    $ ./configure --prefix=/data/hustlog --add-module=src/addon
    $ make
    $ make install

## 目录结构 ##

部署完毕之后，可以看到目录结构如下：

`conf`  
　　`htpasswd`  
　　`hustlog.conf`  
　　`nginx.conf`  
`sbin`  
　　`nginx`  
`logs`  
`client_body_temp`  
`fastcgi_temp`  
`html`  
`proxy_temp`  
`scgi_temp`  
`uwsgi_temp`  

其中列出了子目录的是 `hustlog` 需要重点关注的对象

## 配置 ##
以下内容均假定 `hustlog` 部署的路径为：`/data/hustlog`

### `zlog` 配置 ###
文件路径：`/data/hustlog/conf/hustlog.conf`

文件内容样例：

    [global]
    strict init = true
    buffer min = 2MB
    buffer max = 64MB
    rotate lock file = /tmp/zlog.lock
    file perms = 755
    
    [formats]
    default = "[%d] [%V] [%M(worker)] | %m%n"
    
    [rules]
    business.*             "/data/hustlog/logs/business/%d(%Y-%m-%d-%H).log";         default

以上字段的含义均可在 [这里](https://hardysimpson.github.io/zlog/UsersGuide-CN.html) 找到具体说明。除此之外需要注意的几点：

* `worker`  
`formats` 节下的 `default` 字段值里有一个 `MDC` 叫做 `worker` （MDC的含义参考 [zlog文档](https://hardysimpson.github.io/zlog/UsersGuide-CN.html)），通常用于标识投递日志的客户端进程，这在进行日志分析和故障排除的时候会很有用，可以方便定位到日志输入源。
* `rules`  
`rules` 节用于配置具体的日志输出规则，上述样例中，`business` 代表业务名称，`/data/hustlog/logs/business/%d(%Y-%m-%d-%H).log` 代表日志输出的路径（ **请确保 `/data/hustlog/logs/business` 已经存在，如果不存在，请在启动服务之前创建它** ），`default` 代表日志打印的格式。

如果想把日志目录的创建自动化，可以通过修改 `nginx` 工程配置文件实现，文件目录： `hustlog/nginx/auto/install` ，找到这一段：

    install:	$NGX_OBJS${ngx_dirsep}nginx${ngx_binext} \
    		$NGX_INSTALL_PERL_MODULES
    	test -d '\$(DESTDIR)$NGX_PREFIX' || mkdir -p '\$(DESTDIR)$NGX_PREFIX'
    	test -d '\$(DESTDIR)$NGX_PREFIX/client_body_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/client_body_temp'
    	test -d '\$(DESTDIR)$NGX_PREFIX/proxy_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/proxy_temp'
    	test -d '\$(DESTDIR)$NGX_PREFIX/fastcgi_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/fastcgi_temp'
    	test -d '\$(DESTDIR)$NGX_PREFIX/uwsgi_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/uwsgi_temp'
    	test -d '\$(DESTDIR)$NGX_PREFIX/scgi_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/scgi_temp'
    	
    	test -d '\$(DESTDIR)$NGX_PREFIX/logs' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs'
    	test -d '\$(DESTDIR)$NGX_PREFIX/logs/business' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business'
从这一行开始就是日志目录的配置：

    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business'

如果想在安装的时候就创建多个日志目录，按照类似的方法增加即可，例如：

    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business1' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business1'
    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business2' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business2'
    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business3' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business3'

同时要修改 `hustlog.conf` 如下：

    ......
    [rules]
    business1.*             "/data/hustlog/logs/business1/%d(%Y-%m-%d-%H).log";         default
    business2.*             "/data/hustlog/logs/business2/%d(%Y-%m-%d-%H).log";         default
    business3.*             "/data/hustlog/logs/business3/%d(%Y-%m-%d-%H).log";         default

这样在安装服务的时候就会自动创建日志目录。

### `nginx` 配置 ###
文件路径：`/data/hustlog/conf/nginx.conf`

文件内容样例：

    worker_processes  4;
    daemon on;
    master_process on;
    
    events {
        use epoll;
        multi_accept on;
        worker_connections  1048576;
    }
    
    http {
        include                      mime.types;
        default_type                 application/octet-stream;
    
        sendfile                     on;
        keepalive_timeout            180;
    
        client_body_timeout          10;
        client_header_timeout        10;
    
        client_header_buffer_size    1k;
        large_client_header_buffers  4  4k;
        output_buffers               1  32k;
        client_max_body_size         64m;
        client_body_buffer_size      2m;
    
        server {
            listen                    8667;
            #server_name              hostname;
            access_log                /dev/null;
            error_log                 /dev/null;
            chunked_transfer_encoding off;
            keepalive_requests        32768;
    
            location /hustlog/post {
                hustlog;
                #http_basic_auth_file /data/hustlog/conf/htpasswd;
            }
        }
    }

以上字段的含义均可在 [这里](http://nginx.org/en/docs/) 找到具体说明。可以看到，`hustlog` 提供的接口为 `/hustlog/post`，其中 `http_basic_auth_file` 用于配置 `http basic authentication` 的密钥文件路径，其内容样例如下：

    user:p@ssword

nginx 内置的 `ngx_http_auth_basic_module` 由于使用了非对称加密存储密钥文件，导致 **每一次请求都需要对用户信息做加密操作** 。经测试，以上问题会使得 `nginx` 服务的 `QPS` 下降 **接近一个数量级** 。  
为了解决这个问题， `hustlog` 使用了定制化的 `ngx_http_basic_auth_module` 来实现 `http basic authentication` 功能。
该模块对应的密钥文件使用 **明文** 存储，可以绕过加密操作所带来的性能问题。具体实现可参考 `hustlog/nginx/src/http/modules/ngx_http_basic_auth_module.c`。

实际部署时，可以根据生产环境 **酌情使用** 该配置选项。

## API ##

### 投递日志 ###

**接口:** `/hustlog/post`

**方法:** `POST`

**参数:** 

*  **category** （必选）  
日志类别，通常用于标识业务类型  
*  **level** （必选）  
日志等级，可选值包括 `fatal|error|warn|notice|info|debug`，参考 `zlog`
*  **worker** （必选）  
投递日志的客户端进程

**使用范例:**

    curl -i -X POST -d 'test_log' "localhost:8667/hustlog/post?category=business&level=debug&worker=testworker" -H "Content-Type:text/plain" --user user:p@ssword

**返回值范例:**

	HTTP/1.1 200 OK
    Server: nginx/1.9.4
    Date: Tue, 12 Apr 2016 08:31:12 GMT
    Content-Type: text/plain
    Content-Length: 0
    Connection: keep-alive

## `hustlog` 性能 ##

* 机器配置  
24 core，64 gb，1 tb sata (7200rpm)    
* hustlog 配置  
4 worker  

**压测脚本**

生成随机日志内容的脚本：`gen_data.sh`

    #!/bin/bash
    MATRIX="0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz~!@#$%^&*()_+="
    LENGTH="1024"
    while [ "${n:=1}" -le "$LENGTH" ]
    do
        PASS="$PASS${MATRIX:$(($RANDOM%${#MATRIX})):1}"
        let n+=1
    done
        echo "$PASS"
    exit 0

运行如下命令：
    
    sh gen_data.sh > /data/log_data

压测命令：

    ab -A user:p@ssword -n 1000 -c 1000 -T "text/plain" -p "/data/log_data" "localhost:8667/hustlog/post?category=business&level=debug&worker=testworker"

测试结果：

    Server Software:        nginx/1.9.4
    Server Hostname:        localhost
    Server Port:            8667
    
    Document Path:          /hustlog/post?category=business&level=debug&worker=testworker
    Document Length:        0 bytes
    
    Concurrency Level:      1000
    Time taken for tests:   0.080 seconds
    Complete requests:      1000
    Failed requests:        0
    Write errors:           0
    Total transferred:      141000 bytes
    Total POSTed:           1329328
    HTML transferred:       0 bytes
    Requests per second:    12479.41 [#/sec] (mean)
    Time per request:       80.132 [ms] (mean)
    Time per request:       0.080 [ms] (mean, across all concurrent requests)
    Transfer rate:          1718.36 [Kbytes/sec] received
                            16200.42 kb/s sent
                            17918.77 kb/s total
    
    Connection Times (ms)
                  min  mean[+/-sd] median   max
    Connect:        0   21   5.8     21      30
    Processing:    10   20   9.2     18      38
    Waiting:        1   20   9.5     18      38
    Total:         31   41   3.8     41      49
    
    Percentage of the requests served within a certain time (ms)
      50%     41
      66%     43
      75%     44
      80%     45
      90%     47
      95%     48
      98%     49
      99%     49
     100%     49 (longest request)
可以看到 `QPS` 超过 **1.2w**

## LICENSE ##

hustlog is licensed under [New BSD License](https://opensource.org/licenses/BSD-3-Clause), a very flexible license to use.
