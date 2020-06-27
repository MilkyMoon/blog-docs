### 使用Nginx的反向代理做负载均衡

<br desc/>

通过修改nginx.conf配置文件，可以实现服务器的负载均衡。

<br desc/>

负载均衡方式有如下几种：

##### 轮训

Nginx默认的方式，按照请求的顺序逐个分配到不同的服务器上。

```
upstream  api-server {
    server    192.168.56.101:8181;
    server    192.168.56.102:8181;
    server    192.168.56.103:8181;
}
```



##### 加权

可以根据服务器的负载能力不同，给不同的服务器添加不同的访问几率。

```
upstream  api-server {
    server    192.168.56.101:8181 weight=1;
    server    192.168.56.102:8181 weight=1;
    server    192.168.56.103:8181 weight=2;
}
```



##### iphash

根据用户请求的ip的hash值，来固定用户访问的服务器。iphash可以与加权结合使用。

```
upstream  api-server {
		ip_hash;
    server    192.168.56.101:8181;
    server    192.168.56.102:8181;
    server    192.168.56.103:8181;
}
```



##### leastconn

可以根据服务器的实际访问数来分配访问主机。会将最少访问的主机优先访问。

```
upstream  api-server {
	  leash_conn;
    server    192.168.56.101:8181;
    server    192.168.56.102:8181;
    server    192.168.56.103:8181;
}
```



##### fair

按照服务器的响应时间来分配请求，响应时间短的优先访问。

```
upstream  api-server {
		fair;
    server    192.168.56.101:8181;
    server    192.168.56.102:8181;
    server    192.168.56.103:8181;
}
```



#### 示例

```
http {
		upstream  api-server {
       server    192.168.56.101:8181;
       server    192.168.56.102:8181;
       server    192.168.56.103:8181;
    }
    
    server {
       listen       80;
       server_name  localhost;

       location / {
        proxy_pass http://api-server;
        proxy_redirect default;
      }
    }
}
```

