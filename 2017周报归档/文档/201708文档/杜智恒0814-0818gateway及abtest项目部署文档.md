## abtest及gateway项目部署

### 步骤
#### 1. 复制abtest.zip，gateway.zip到 /usr/local/ 目录下并解压

```
<!--解压zip-->
[root@test113 local]# unzip -o -d /usr/local/gateway  gateway.zip
```

#### 2. 编译so文件

```
<!--如果已经有usertime.so，signcheck.so文件  删除后再执行下面代码-->
[root@test113 local]# cd /usr/local/gateway/sign/solib/
[root@test113 local]# cc -g -O2 -Wall -fPIC --shared usertime.c -o usertime.so
[root@test113 local]# cc -g -O2 -Wall -fPIC --shared signcheck.c -o signcheck.so 
```

#### 3. 检查redis配置


```
<!--unixsocket、unixsocketperm去掉注释-->
[root@test113 local]# vim ../redis/redis.conf

# Unix socket.
#
# Specify the path for the Unix socket that will be used to listen for
# incoming connections. There is no default, so Redis will not listen
# on a unix socket when not specified.
#
 unixsocket /tmp/redis.sock
 unixsocketperm 700

```

#### 4. 检查nginx配置

```
[root@test113 local]# cd /usr/local/abtest/utils/conf/
[root@test113 local]# vim vhost.conf
```
    a. redis依赖(server下) 注：HTTP和HTTPS 是两个不同的server，需检查两个是否都有配置
        set $redis_host '127.0.0.1';
        set $redis_port '6379';
        set $redis_uds '/var/run/redis6379.sock';
        set $redis_connect_timeout 10000;
        set $redis_dbid 0;
        set $redis_pool_size 1000;
        set $redis_keepalive_timeout 90000;
        
    b. 对消息体的解析的依赖配置(server下)
        lua_need_request_body on;
        
    c. 共享字典内存的依赖配置(http下)
        lua_shared_dict sign_url 2m;
        
    d. 返回头的时间戳(location下)
        header_filter_by_lua_block {
            local utime = require "usertime"
            local millisecond = utime.getmillisecond()
            ngx.header["timestamp"] = millisecond
        }

    e. 路径配置(该配置在 vhost同级目录的 nginx.conf 文件下)
        lua_package_path "$gateway的根目录/?lua"
        lua_package_cpath "$gateway的根目录/gateway/sign/solib/?.so;;";  #c模
        
    f. 共享字典内存的依赖配置(http下)
        在该文档查看：http://218.205.244.254:8080/miguwiki/pages/viewpage.action?pageId=3148858




#### 5. 启动nginx
```
<!--重启-->
[root@test113 local]# cd /usr/local/abtest/utils/
[root@test113 local]# sh abtesting.sh

<!--reload 保持原来的进程号-->
[root@test113 local]# ps -ef | grep nginx
[root@test113 local]# kill -HUP 主进程号

<!--启动后访问,如返回 下面内容 说明启动成功-->
[root@test113 local]# curl "http://IP:18089/MIGUM2.0/v1.0/tone/subscribe_info.do?ua=Ios_migu&version=4.2"
{"summary":"\"简介：彩铃使用基础功能，包含购买、使用和赠送彩铃。全国统一资费：5元/月，各省本地资费详询 10086","prompt":"您正在开通彩铃功能，5元/月，开通成功后可以设置彩铃，确定开通吗？","code":"000000","info":"success"}
```
