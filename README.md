[简体中文](README_ZH.md)

# hustlog - High performance network log service #

`hustlog` is a high performance network log service with high availability of http interface. It is implemented as nginx module based on `zlog`.  

## Features ##

`zlog` is a high-performance standalone log service, while `nginx` is an industrial-grade http server. Benefited much from them, `hustlog` has the following features:  
  
* High Throughput. The benchmark shows that `QPS` hits **12 thousand** and even more.  
* High Concurrency. Please refer to concurrency report of `nginx` for more details.  
* High Availability. Please refer to `master-worker` design of `nginx`.  

## Dependency ##
* [zlog](https://github.com/HardySimpson/zlog)

Installation of `zlog`:  

    $ git clone https://github.com/HardySimpson/zlog
    $ cd zlog
    $ make
    $ sudo make install
    $ sudo vi /etc/ld.so.conf
    $ /usr/local/lib
    $ sudo ldconfig

## Deployment ##
    $ cd nginx
    $ sh Config.sh
    $ make
    $ make install

## Structure ##

After deployment is done, you can see the directory structure as belows:  

`conf`  
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

Please pay attention to those who list the subdirectories.  

## Configuration ##
The following contents assume that the installation path of the `hustlog` is `/data/hustlog`.  

### Configuration of `zlog` ###
File path: `/data/hustlog/conf/hustlog.conf`

File contents sample:  

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

Please refer to [here](https://hardysimpson.github.io/zlog/UsersGuide-CN.html) for details on above fields. In addition, please pay attention to the notes as following:  

* `worker`  
In section `formats`, the value of field `default` contains a `MDC` named `worker` (refer to [zlog](https://hardysimpson.github.io/zlog/UsersGuide-CN.html) for details on `MDC`), which is usually used to identify the process of the client who delivers log. This is useful for log analysis and troubleshooting , making it easy to find the source of log.  
* `rules` 
The `rules` section is used to configure specific log output rules. In above example, `business` represents the business name, `/data/hustlog/logs/business/%d(%Y-%m-%d-%H).log` represents the output directory of log (**please make sure that `/data/hustlog/logs/business` already exists. Otherwise, you need to create it before starting the service**)，`default` represents the format of log.  

If you want to automate the creation of log directories, please modify the configuration of nginx project in file `hustlog/nginx/auto/install` as below:  

    install:	build $NGX_INSTALL_PERL_MODULES
        test -d '\$(DESTDIR)$NGX_PREFIX' || mkdir -p '\$(DESTDIR)$NGX_PREFIX'
        test -d '\$(DESTDIR)$NGX_PREFIX/client_body_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/client_body_temp'
        test -d '\$(DESTDIR)$NGX_PREFIX/proxy_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/proxy_temp'
        test -d '\$(DESTDIR)$NGX_PREFIX/fastcgi_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/fastcgi_temp'
        test -d '\$(DESTDIR)$NGX_PREFIX/uwsgi_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/uwsgi_temp'
        test -d '\$(DESTDIR)$NGX_PREFIX/scgi_temp' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/scgi_temp'
        
        test -d '\$(DESTDIR)$NGX_PREFIX/logs' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs'
        test -d '\$(DESTDIR)$NGX_PREFIX/logs/business' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business'

Configuration for log directory starts from this line:  

    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business'

If you want to create multiple log directories during installation, just follow this way, for example:  

    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business1' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business1'
    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business2' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business2'
    test -d '\$(DESTDIR)$NGX_PREFIX/logs/business3' || mkdir -p '\$(DESTDIR)$NGX_PREFIX/logs/business3'

Meanwhile, you need to update `hustlog.conf` as below:  

    ......
    [rules]
    business1.*             "/data/hustlog/logs/business1/%d(%Y-%m-%d-%H).log";         default
    business2.*             "/data/hustlog/logs/business2/%d(%Y-%m-%d-%H).log";         default
    business3.*             "/data/hustlog/logs/business3/%d(%Y-%m-%d-%H).log";         default

Then the log directories will be automatically created when the service is installed.  

### Configuration of `nginx` ###
File path: `/data/hustlog/conf/nginx.conf`

File contents sample:  

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

Please refer to [here](http://nginx.org/en/docs/) for details on above fields. You can see that the interface of `hustlog` is `/hustlog/post`. `http_basic_auth_file` is used to configure the path of key file for `http basic authentication`, the sample contents:  

    user:p@ssword

The username and password are asymmetrically encrypted and stored in key file, which is used by built-in `ngx_http_auth_basic_module` of nginx, results to **encryption on username and password during per request**. According to the test, the QPS will **decrease by nearly an order of magnitude** duing to the issue.  
`ngx_http_basic_auth_module` is used by `hustlog` just for such problem. Its key file uses **plain text** to store information to avoid performance problem on encryption. Please refer to `hustlog/nginx/src/http/modules/ngx_http_basic_auth_module.c` for details on implementation.  

**You need to evaluate the interest and risk by yourself, and carefully consider the situation before you decide to apply this module.**  

## API ##

### Post log ###

**Interface:** `/hustlog/post`

**Method:** `POST`

**Arguments:** 

*  **category** (required)  
log category, usually used to identify business type  
*  **level** (required)  
log level, optional values: `fatal|error|warn|notice|info|debug`, refer to `zlog`  
*  **worker** (required)  
Name of client process who posted log  

**Sample:**

    curl -i -X POST -d 'test_log' "localhost:8667/hustlog/post?category=business&level=debug&worker=testworker" -H "Content-Type:text/plain" --user user:p@ssword

**Result:**

	HTTP/1.1 200 OK
    Server: nginx/1.12.0
    Date: Tue, 18 Apr 2017 08:31:12 GMT
    Content-Type: text/plain
    Content-Length: 0
    Connection: keep-alive

## Performance of `hustlog` ##

* Configuration of machine  
24 core，64 gb，1 tb sata (7200rpm)    
* Configuration of hustlog  
4 worker  

**Press test script**

The script that generates random log content: `gen_data.sh`  

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

Run command:  
    
    sh gen_data.sh > /data/log_data

Press test command:  

    ab -A user:p@ssword -n 1000 -c 1000 -T "text/plain" -p "/data/log_data" "localhost:8667/hustlog/post?category=business&level=debug&worker=testworker"

Result:  

    Server Software:        nginx/1.12.0
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

You can see `QPS` hits **12 thousand** and even more.  

## LICENSE ##

hustlog is licensed under [New BSD License](https://opensource.org/licenses/BSD-3-Clause), a very flexible license to use.
