#Nginx
上有服务器配置：使用upstream server配置
负载均衡算法：配置多个上有服务器
失败重试机制；配置当超时或上游服务器不存货时，是否充实其他上有服务器
服务器心跳检查：上有服务器的健康检查/心跳检查

##一、upstream 配置
```
upstream backend{
    server 192.168.74.1:8080  weight=1;
    server 192.168.74.1:8090  weight=2;
}

#IP地址和端口：配置上有服务器的IP地址和端口。
#权重：weight用来配置权重，默认是1，权重越高分配给该服务的请求就越多

location /{
    proxy_pass http://backend;
}
#以上的负载均衡的算法是轮询，是nginx默认的负载均衡算法。
```
##二、负载均衡算法
负载均衡用来解决用户请求到来时如何选择upstream server进行处理，默认是采用round-robin（轮询），同时支持其他算法。
1. round-robin:轮询，默认的负载均衡算法，即以轮询的方式将请求转发至上有服务器，铜鼓哦配合weight配置来基于权重的轮询。
2. ip_hash:根据用户进行负载均衡，即相同的IP将负载到同一个upstream server
    ```
    upstream backend{
        ip_bash;
        server 192.168.74.1:8080  weight=1;
        server 192.168.74.1:8090  weight=2;
    }
    ```
3. hash key [consistent]:对于某一个key进行哈希或者使用一致性哈希算法进行负载均衡。使用Hash算法存在的问题时，放添加/删除一台服务器时，将导致很多key被重新负载均衡到不同的服务器；因此，建议考虑使用一致性哈希算法，这样没添加/删除服务器时，只有少数key被重新负载均衡到不同的服务器。
    ```
     upstream backend{
        hash $uri;
        server 192.168.74.1:8080  weight=1;
        server 192.168.74.1:8090  weight=2;
    }

    #根据请求uri进行负载均衡
    ```

    ```
     upstream backend{
        hash $consistent_key consistent;
        server 192.168.74.1:8080  weight=1;
        server 192.168.74.1:8090  weight=2;
    }
    #一致性哈希算法：consistent_key动态指定
    ```
    ```
    #通过Lua设置一致性哈希key
    set_by_lua_file $consistent_key "lua_balancing.lua"
    local consistent_key=args.cat
    if not consistent_key or consistent_key == '' then
        consistent_key = ngx_var.request_uri
    end

    local value = balancing_cache:get(consistent_key)
    if not value then
        success, err = balancing_cache:set(consistent_key, 1, 60)
    else
        newval,err = balancing_cache:incr(consistent_key, 1)
    #如果某一分类请求量太大，上游服务器可能处理不了这么多请求，此时可以在一致性哈希key后面加上递增的计数以实现类似轮询的算法
    if newval > 5000 then
        consistent_key = consistent_key .. '_' .. newval
    end
    ```
##三、失败重试
主要有两部分配置：upstream server和proxy_pass.
```
upstream backend{
    server 192.168.74.1:8080  weight=1 max_fials=2 fail_timeout=10s;
    server 192.168.74.1:8090  weight=2 max_fials=2 fail_timeout=10s;
}
#在fail_timeout时间内失败了max_fials次请求，则认为该上游服务器不可用，然后摘掉该服务器
#但是在fail_timeout时间之后会再次将该服务器添加到存活的上有服务器列表中进行重试.
```
```
location /test {
    proxy_connect_timeout 5s;
    proxy_read_timeout 5s;
    proxy_send_timeout 5s;

    proxy_next_upstream error timeout;
    proxy_next_upstream timeout 5s;
    proxy_next_upstream tries 5s;

    proxy_pass http://backend;
    addr_header upstream_addr $upstream_addr;
}
```
##四、健康检查
Nginx对上游服务器的健康检查默认采用的时惰性策略，Nginx商业版提供了health_check进行主动的健康检查。当然也可以继承nginx_upstream_check_module模块来进行主动的健康检查，其支持TCP心跳和HTTP心跳来进行健康检查。
1. TCP心跳检查
    ```
    upstream backend{
        server 192.168.74.1:8080  weight=1 max_fials=2 fail_timeout=10s;
        server 192.168.74.1:8090  weight=2 max_fials=2 fail_timeout=10s;
        check interval=3000 rise=1 fall=3 timout=2000 type=tcp;
    }
    #interval:检测时间间隔，单位为毫秒
    #rise：检测成功多少次后，上游服务器标识为存活，并可以接受处理请求
    #fall：检测失败多少次后，上游服务器标识为不存活
    #timout：检测请求超时时间，单位为毫秒
    ```
2. HTTP心跳检查
    ```
    upstream backend{
        server 192.168.74.1:8080  weight=1 max_fials=2 fail_timeout=10s;
        server 192.168.74.1:8090  weight=2 max_fials=2 fail_timeout=10s;
        check interval=3000 rise=1 fall=3 timout=2000 type=http;
        check_http_send "HEAD /status HTTP/1.0\r\n\r\n";
        check_http_send http_2xx http_3xx;
    }
    #check_http_send:即检查时发出的HTTP请求内容
    #check_http_send:当上游服务器返回匹配的响应状态码时，则认为上游服务器存活
    #注：检查时间间隔不能太短，否则检查包太多会冲垮服务器
    ```
##五、其他配置
1. 域名上游服务器
    ```
    upstream backend{
        server cc.com
        server aa.com
    }
    ```
    Nginx在解析配置文件的阶段将域名解析成IP地址并记录在upstream 中，当这两个域名对应的IP地址发生变化时，该upstream不会更新，商业版才支持动态更新。
2.备份上游服务器
    ```
    upstream backend{
        server 192.168.74.1:8080  weight=1 max_fials=2 fail_timeout=10s;
        server 192.168.74.1:8090  weight=2 max_fials=2 fail_timeout=10s backup;
    }
    ```
    将配有backup的服务器配置为备上游服务器，当所有主上游服务器都不存活时，请求会转发给备上游服务器。
3.不可用上游服务器
    ```
    upstream backend{
        server 192.168.74.1:8080  weight=1 max_fials=2 fail_timeout=10s;
        server 192.168.74.1:8090  weight=2 max_fials=2 fail_timeout=10s down;
    }
    ```
    down配置该服务器为永久不可用，当测试或者机器故障时，暂时通过配置临时摘掉该服务器。
##六、长连接
通过keepalive指令配置长连接数量：
通过该指令配置每个Worker进程与上游服务器可缓存的空闲连接的最大数量。当超出这个数量时最近最少使用的连接将被关闭。keepalive指令不限制Worker进程与上游服务器的总连接数。
```
upstream backend{
    server 192.168.74.1:8080  weight=1;
    server 192.168.74.1:8090  weight=2 backup;
    keepalive 100;
}
```
建立长连接：
如果时http/1.0，需要配置发送“Connectio:Keep-Alive”请求头
```
upstream backend{
    server 192.168.74.1:8080  weight=1;
    server 192.168.74.1:8090  weight=2 backup;
    keepalive 100;
}

location /{
    #支持keepalive
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_pass http://backend;
    
}
```
##七、HTTP反向代理
反向代理除了实现负载均衡之外，还提供了如缓存来减少上游服务器的压力。
1. 全局配置（proxy_cache)
    ```
    proxy_buffering                     on;
    proxy_buffering_size                4k;
    proxy_buffers                       512 4k;
    proxy_busy_buffers_size             64k;
    proxy_temp_file_write_size          256k;
    proxy_cache_lock                    on;
    proxy_cache_lock_timeout            200ms;
    proxy_temp_path                     /opt/nginx/proxy_temp;
    proxy_cache_path                    /opt/nginx/proxy_cache levels=1:2 key_zone=cache:512m 
                                        inactive-5m max_size=8g;

    proxy_connect_timeout               3s;
    proxy_read_timeout                  5s;
    proxy_send_timeout                  5s;
    ```
2. location配置
    ```
    location ~^/backend/(.*)${
        #设置一致性哈希负载均衡
        set_by_lua_file $consistent_key "/export/App/c.c3.cn/luna/luna_balancing_backedn.properties"

        #失败重试
        proxy_next_upstream error timeout http_500 http_502 http_504;
        proxy_next_upstream timeout 5s;
        proxy_next_upstream tries 5s;

        #请求上游服务器
        proxy_method  GET;
        #不给上游服务器传递请求体
        proxy_pass_request_body     off;
        #不给上游服务器传递请求头
        proxy_pass_request_headers  off;
        #设置上游服务器的那些响应头不发送给客户端
        proxy_hide_header           Vary;

        #支持keepalive
        proxy_http_version          1.1;
        proxy_set_header Connection "";

        #给上游服务器传递Reference Cookie和Host
        proxy_set_header Referer $http_referer;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header Host web.c.3.local;

        proxy_pass http://backend;
        addr_header upstream_addr $upstream_addr;
    }
    ```
    也可配置开启gzip支持，减少网络传输的数据包大小。

##八、HTTP动态负载均衡
1. Consul+Consul-template
    1.1 Consul-server
    1.2 Consul_template
    1.3 Java服务
2. Consul+OpenResty


##九、Nginx四层负载均衡

