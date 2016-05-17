footer: _© 2016_ - Cassiano Aquino - caquino@zendesk.com
slidenumbers: true
autoscale: true
#[FIT]![inline](https://assets.wp.nginx.com/wp-content/uploads/2015/04/NGINX_logo_rgb-01.png)
#[FIT]REWIND AND FORWARD

---
##whoami
![filtered right 100%](https://scontent-syd1-1.xx.fbcdn.net/hphotos-xat1/v/t1.0-9/12107716_991986397489982_763074616118238289_n.jpg?oh=b397d7a3373102709d0185da3efde2f1&oe=57807FA5)

- Cassiano Aquino
- Brazilian
- Tech Lead DevOps Engineer @ Zendesk
- @syshero
- http://syshero.org/

---
#[FIT]![inline](https://assets.wp.nginx.com/wp-content/uploads/2015/04/NGINX_logo_rgb-01.png)
#[FIT]SWAG :+1:

---
#[FIT]tweet :bird:
#[FIT]#nginx #devopsdublin

^Send a tweet with both hashtags
and come talk to me in the end of the meetup

---
#[FIT]22%
#[FIT]Top 1 million websites

---
#[FIT]37%
#[FIT]Top 1000 websites

^Most used on top 1000 websites

---
#[FIT]![inline](https://assets.wp.nginx.com/wp-content/uploads/2015/04/NGINX_logo_rgb-01.png)
#[FIT]Just a webserver?

---
## NO!

_Nginx (pronounced "engine x") created by Igor Sysoev in 2002
Application accelerator_

- Reverse proxy
  - HTTP/HTTPS
  - SMTP/POP3/IMAP
  - TCP (> 1.9.0 --with-stream)/UDP ( > 1.9.13 )
- Load balancer
- Caching

^Question, use nginx?
Why instead of apache? Event based IO, non blocking.
Igor sosoievi - Moscow
Bandwidth Management, Content-based Routing, Resquest Manipulation
Response Rewriting, Application Acceleration, SSL and SPDY termination
Authentication, Video Delivery, Mail Proxy, GeoLocation, Performance Monitoring

---
#[FIT]location

---
##Basics?

```nginx
location /awesome {
  try_files $uri $uri/ index.html;
}

location ~ /awesome {
  try_files $uri $uri/ index.html;
}
```

---
#[FIT]upstream

---
## Request retry on error
### Open source and PLUS

```nginx
upstream mybackend {
  server 10.0.0.1:80;
  server 10.0.0.2:80;
}

server {
  proxy_next_upstream error timeout http_500 http_502 http_503 http_504 non_idempotent;
  proxy_pass http://mybackend;
}
```

* non_idempotent > 1.9.13

^POST, LOCK, and PATCH

---
## Advanced backend retry

```nginx
upstream mybackend {
  server 192.168.0.1;
  server 192.168.0.2;
}

server {
  location / {
    error_page 502 504 @fallback;
    proxy_pass http://mybackend;
    proxy_next_upstream off;
  }

  location @fallback {
    proxy_pass http://mybackend;
    proxy_next_upstream off;
  }
}
```

^If there is no need to change URI during internal redirection
it is possible to pass error processing into a named location:

---
## Debug server
### Open source and PLUS
![original fit right](https://assets.wp.nginx.com/wp-content/uploads/2015/11/debug-server-5xx-errors-e1448303642707.png)

```nginx
upstream app_server {
    server 172.16.0.1 max_fails=1 fail_timeout=10;
    server 172.16.0.2 max_fails=1 fail_timeout=10;
}
upstream debug_server {
    server 172.16.0.9 max_fails=1 fail_timeout=30 max_conns=20;
}
server {
    listen *:80;
    location / {
        proxy_pass http://app_server;
        proxy_intercept_errors on;
        error_page 500 503 504 @debug;
    }
    location @debug {
        proxy_pass http://debug_server;
        access_log /var/log/nginx/access_debug_server.log detailed;
        error_log  /var/log/nginx/error_debug_server.log;
    }
}
```

---
## Upstream keepalive
### Open source and PLUS

```nginx
upstream keepalive-upstream {
  server 127.0.0.1:8001;
  keepalive 32;
}

server {
  location / {
    proxy_pass http://keepalive-upstream;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
  }
}
```

---
## Service discovery
### PLUS only

- consul
- etcd
- zookeeper

```nginx
resolver 127.0.0.1;
upstream consul-service {
  server service.dc1.consul service=voice resolve;
}
```

---
# Dynamic upstream
### PLUS only

```nginx
upstream backend {
    zone backend 64k;
    state /etc/nginx/conf.d/backend.state;
}
```

```nginx
location /upstream_conf {
    upstream_conf;
}
```

---
# Node draining
### PLUS only

```bash
$ curl http://localhost/upstream_conf?upstream=backend
server 192.168.56.101:80; # id=0
server 192.168.56.102:80; # id=1
server 192.168.56.103:80; # id=2
$ curl http://localhost/upstream_conf?upstream=backend\&id=1\&drain=1
server 192.168.56.102:80; # id=1 draining
```



---
#[FIT]split_clients

---
## Ephemeral port limit
### Open source and PLUS

```nginx
upstream backend {
  server 10.0.0.100:1234;
  server 10.0.0.101:1234;
}

server {
  location / {
    proxy_pass http://backend;
    proxy_bind $split_ip;
    proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```
```nginx
split_clients "$remote_addr$remote_port" $split_ip {
  35%  10.0.0.210;
  35%  10.0.0.211;
  *    10.0.0.212;
}
```

^The first parameter to the split_clients directive is a string ("$remote_addr$remote_port")
which is hashed using a MurmurHash2 function during each request.
The statements inside the curly braces divide the hash table into “buckets”,
each of which contains a percentage of the hashes.
(The percentage for the last bucket is always represented by the asterisk [*] rather than a specific number,
because the number of hashes might not be evenly dividable into the specified percentages.)
The range of possible hash values is from 0 to 429.4967.295,
so in our case each bucket contains about 429496700 values (10% of the total):

---
## Multiple disk caching
### Open source and PLUS

```nginx
split_clients $request_uri $cache {
    50%     disk1;
    *       disk2;
}
```
```nginx
proxy_cache_path /mnt/disk1/cache keys_zone=disk1:100m levels=1:2 inactive=600s max_size=5G use_temp_path=off;
proxy_cache_path /mnt/disk2/cache keys_zone=disk2:100m levels=1:2 inactive=600s max_size=5G use_temp_path=off;
server {
    listen localhost:80;
    proxy_cache $cache;
    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_pass http://upstreams;
    }
}
```

---
#[FIT]map
#[FIT]proxy\_cache\_bypass

---
## Cache "purge"
### Open source and PLUS

```nginx
location / {
	add_header X-Cached $upstream_cache_status;
	proxy_cache_bypass $http_cache_purge;
	proxy_pass http://mybackend;
}
```

```bash
root@server:~# curl --header 'Cache-Purge: 1' http://myserver/url/topurge
```

---
## 0 byte caching
### Open source and PLUS

```nginx
map $upstream_http_content_length $flag_cache_empty {
	default         0;
	0               1;
}
```

```nginx
server {
	location / {
		proxy_no_cache $flag_cache_empty;
		proxy_cache_bypass $flag_cache_empty;
	}
}
```

---
## Cache disable based on cookies
### Open source and PLUS

```nginx
map $http_cookie $no_cache {
  default 0;
  ~ASPXAUTH 1;
}
```

```nginx
location / {
  proxy_cache_bypass $no_cache;
  proxy_pass http://mybackend;
}
```
---
#[FIT]lua

---
## Passive cache invalidation
### Open source and PLUS - requires Lua

![inline](http://33.media.tumblr.com/7b111df7e0d47773a4a50b9d2b8569b1/tumblr_inline_mx1gcieS6b1rlp39e.png)

---
## Dynamic SSL configuration
### Open source and PLUS - requires Lua module

- AWS KMS Integration
- Let's Encrypt integration

```nginx
ssl_certificate_by_lua '...';
ssl_certificate_by_lua_file lua/script.lua;
```

---
#[FIT]misc

---
## 0 Downtime deployment
![autoplay](0downtime.mov)

---
## 0 Downtime deployment
### Open source and PLUS

```nginx
upstream zerodowntime {
  server localhost:8001;
  server localhost:8002 backup;
}
```
- OSE

```bash
sed -i "s/\ backup//g" /etc/nginx/conf.d/upstream.conf
sed -i "s/:8001/:8001 backup/g" /etc/nginx/conf.d/upstream.conf
kill -HUP $(cat /var/run/nginx.pid)
```

- Plus API

---
## Executable upgrade on the Fly
### Open source and PLUS

```
root@myserver:~/usr/bin# cp new.nginx nginx
root@myserver:~/usr/bin# pkill -USR2 nginx
root@myserver:~/usr/bin# kill -s WINCH `cat /run/nginx.pid.oldbin`
COMMIT:
root@myserver:~/usr/bin# kill -s QUIT `cat /run/nginx.pid.oldbin`
ROLLBACK:
root@myserver:~/usr/bin# kill -s HUP `cat /run/nginx.pid.oldbin`
root@myserver:~/usr/bin# kill -s QUIT `cat /run/nginx.pid`
```

---
![](http://vignette1.wikia.nocookie.net/joke-battles/images/4/40/18360-doge-doge-simple.jpg/revision/latest?cb=20151209161638)
#[FIT]WOW
#[FIT]SUCH NGINX
#[FIT]MUCH INFORMATION

---
#[FIT]MOAR INFORMATION
#[FIT]http://syshero.org/
#[FIT]THANKS! QUESTIONS?

---
##Advantages/Why
###PLUS

- Support, have I said support? yes, support!
- Load balancing methods (e.g: least_time connect, first_byte, last_byte)
- Consistent hashing
- Health checking
- Backend connection limit
- UDP load balacing
- Cache performance improvement
- Retrying of non-idempotent requests

---
##Advantages/Why
###PLUS

- Dynamic upstream reconfiguration (http API/dns)
- Dynamic virtual host reconfiguration (http API)
- Service discovery (etcd/consul/zookeeper)
- Cache Purge
- Better monitoring and metrics
- Dashboard
- nginScript?

---
##Dashboard
###PLUS
![inline](https://assets.wp.nginx.com/wp-content/uploads/2015/09/Screen-Shot-2015-09-14-at-1.59.10-PM-1024x493.png)


---
![](http://vignette1.wikia.nocookie.net/joke-battles/images/4/40/18360-doge-doge-simple.jpg/revision/latest?cb=20151209161638)
#[FIT]SO FANCY
#[FIT]MUCH EXAMPLES
#[FIT]WOW

---
## Anomaly detection and rate limiting
### Open source and PLUS - requires Lua module, some scripting and external tools

https://github.com/openresty/lua-resty-limit-traffic

- Individual metric collection (URI, URI, Account, Type, Verb,...)
- Anomaly detection (Datadog, Morgoth)
- Cached/Uncached requests
- Response time
- Anonymous/Authenticated requests


