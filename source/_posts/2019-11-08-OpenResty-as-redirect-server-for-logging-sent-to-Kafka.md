---
title: OpenResty as redirect server for logging sent to Kafka
date: 2019-11-08 11:58:58
tags: openresty logging kafka
---

### Purpose

A redirect server for logging is used to trace various urls user clicked. The redirect server will log any query parameters in url and redirect to `url` parameter finally. The logging should be doing asynchronously to minimize redirect time.

OpenResty is a combination of Nginx and Lua. I won't explain that here, you could google it to know more.

![](/2019/11/08/OpenResty-as-redirect-server-for-logging-sent-to-Kafka/openresty-nginx-lua.png)

<!--more-->

### How it works

{% plantuml %}
hide footbox

actor "User" as User
participant "Rd Server" as Rd
participant "Kafka" as Kafka
participant "Destination Site" as Dest

User->Rd : Click url
Rd-> Kafka: Send message to kafka
Rd-> Dest: Redirect user to url
{% endplantuml %}

*click url: https://rd.example.com?show=true&t=32&tag=car&url=https%3A%2F%2Ftoday.line.me*
*query parameters should be url-encoded.*

### Environment

* MacOS
* Docker Desktop
* Download lua-resty-kafka for lua kafka operation
* Lua to get query paramters and send to kafka
* Encode query parameters using ***cjson*** as message
* Send message to kafka

### Download lua-resty-kafka

``` bash
mkdir openresty
cd openresty
wget https://github.com/doujiang24/lua-resty-kafka/archive/v0.07.zip
unzip v0.07.zip
mkdir logs
```

### Directory layout

We will mount docker host files(***nginx.conf***) and directory(***logs***, ***lua-resty-kafka-0.07***) to container

``` bash
(base) kevin.luo➜~/dev/openresty» ls -al
total 32
drwxr-xr-x   8 kevin.luo  staff   256 Nov  6 12:59 .
drwxr-xr-x@ 24 kevin.luo  staff   768 Nov  5 20:10 ..
drwxr-xr-x   5 kevin.luo  staff   160 Nov  6 10:47 logs
drwxr-xr-x@  9 kevin.luo  staff   288 Aug 31 08:48 lua-resty-kafka-0.07
-rw-r--r--   1 kevin.luo  staff  4490 Nov  6 12:59 nginx.conf
-rwxr-xr-x   1 kevin.luo  staff   874 Nov  6 11:19 run.sh
-rwxr-xr-x   1 kevin.luo  staff    58 Nov  5 20:29 stop.sh
```

### Files description and content

`run.sh` is to run this ***rd*** service

``` bash
#!/usr/bin/env bash

docker run -d --name="rd" -p 8089:8089 -v $PWD/nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf:ro -v $PWD/logs:/usr/local/openresty/nginx/logs -v $PWD/lua-resty-kafka-0.07/lib/resty/kafka:/usr/local/openresty/lualib/resty/kafka openresty/openresty:1.15.8.2-4-xenial-nosse42
```

`stop.sh` is to stop ***rd*** service and remove docker container

``` bash
#!/usr/bin/env bash

docker kill nginx && docker rm nginx
```

`nginx.conf` is the main nginx + lua configuration file.

``` bash
worker_processes  1;
events {
    use epoll;
    worker_connections 1024;
}

error_log logs/error.log;

http {
    include       mime.types;
    default_type  application/octet-stream;

    server_tokens off;
    more_clear_headers 'Server';

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    keepalive_timeout  65;

    lua_package_path "/usr/local/openresty/lualib/resty/kafka/?.lua;;";
    # without resolver setting, it will fail in DNS lookup
    # either 8.8.8.8 or using container's resolver
    # resolver 8.8.8.8
    resolver local=on ipv6=off;
    resolver_timeout 5s;

    server {
        listen       8089;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

        # Make sure Lua is working
        location /hello {
            default_type text/html;
            content_by_lua '
                ngx.say("Lua: hello world!")
            ';
        }

        # The default 1by1 gif
        location /empty_gif {
           empty_gif;
        }

        location /rd {
            default_type text/html;
		    content_by_lua '
                local cjson = require "cjson"
                local producer = require "resty.kafka.producer"
                -- local client = require "resty.kafka.client"

                local broker_list = {
                    { host = "192.168.0.16", port = 9092},
                    { host = "192.168.0.17", port = 9092},
                    { host = "192.168.0.18", port = 9092}
                }

                -- Parsing request args
                local params_json = {}
                local args, err = ngx.req.get_uri_args()

                if err == "truncated" then
                    -- one can choose to ignore or reject the current request here
                end

                for key, val in pairs(args) do
                    params_json[key] = val
                end
                ngx.log(ngx.ERR, "url: ", params_json["d"])

                local topic = "redirect-server"
                local params_message = cjson.encode(params_json)

                -- Set producer async
                local bp = producer:new(broker_list, { producer_type = "async" })

                -- The second parameter of send method is to control kafka routing.
                -- When it is nill, it will write message to same partition
                -- With designated key，it will write message to hash of key of partition
                local ok, err = bp:send(topic, nil, params_message)

                -- For debugging
                ngx.log(ngx.ERR, "Message in json: ", params_message)

                if not ok then
                    ngx.log(ngx.ERR, "kafka send err:", err)
                    return
                end

                if params_json["url"] ~= nil then
                    return ngx.redirect(params_json["url"], 301)
                end

                return ngx.exec("/empty_gif")
            ';

        }
    }
}
```

### Testing

Check you logs/error.log file, there should not have any `kafka send err:`

```
2019/11/06 04:59:44 [error] 6#6: *1 [lua] content_by_lua(nginx.conf:119):25: url: nil, client: 172.17.0.1, server: localhost, request: "GET /rd?a=b&open=false&run=success&yeild=ok&url=http://tw.news.yahoo.com HTTP/1.1", host: "127.0.0.1:8089"
2019/11/06 04:59:44 [error] 6#6: *1 [lua] content_by_lua(nginx.conf:119):56: Message in json: {"run":"success","yeild":"ok","url":"http:\/\/tw.news.yahoo.com","a":"b","open":"false"}, client: 172.17.0.1, server: localhost, request: "GET /rd?a=b&open=false&run=success&yeild=ok&url=http://tw.news.yahoo.com HTTP/1.1", host: "127.0.0.1:8089"
```

### Verify kafka message queue

``` bash
kafka-console-consumer --broker-list 192.168.0.16:9092 --topic redirect-server --from-beginning

kafka-console-producer --broker-list 192.168.0.16:9092 --topic redirect-server
```