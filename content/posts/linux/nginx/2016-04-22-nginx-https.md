---
title: Nginx实现部分页面https
date: 2016-04-22 20:33:37
tags:
    - nginx
    - https
---

由于业务需要，在checkout页面需要实现https加密访问，之前只做过nginx的全站https，对于部分页面https搜索方法有说在代码里直接写成https，有说服务器上直接配置的，做了对比觉得还是nginx上直接配置靠谱，开始动手.


需求: 全站除了checkout页面https，其他都采取http访问, 如果不符合这个规则的，按照规则强制跳转.

思路: http访问时，在nginx里判断路径包含/checkout，就强制将http转到https；https访问时，判断非checkout，就强制调到http。思路是没有问题，问题是我想用nginx的location实现，研究了半天没找到很好的方案实现[非checkout]. 不过最后还是实现了~_~,用到了nginx的proxy_pass，下面讲具体实现，最后会放出完整的配置,也许还有问题，欢迎提出.

直接把nginx配置放上来, server_name自行做修改，用了反向代理proxy_pass到负载均衡服务器.


```nginx

server {
    server_name heyuan110.com;
    rewrite ^/(.*) http://www.heyuan110.com/$1 permanent;
}

server {
    listen       80;
    server_name  www.heyuan110.com;

    gzip on;
    gzip_min_length 1k;
    gzip_buffers 16 64k;
    gzip_http_version 1.1;
    gzip_comp_level 4;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml;
    gzip_vary on;

    access_log  /var/log/nginx/access.log;
    error_log   /var/log/nginx/error.log;

    location ~* /news/show/* {
        return  301 https://$host$request_uri;
    }

    location / {
           proxy_pass        http://web;
           proxy_connect_timeout 600;
           proxy_read_timeout 600;
           proxy_send_timeout 600;
           proxy_buffer_size 64k;
           proxy_buffers  4 32k;
           proxy_busy_buffers_size 64k;
           proxy_temp_file_write_size 64k;
           proxy_set_header   Host             $host;
           proxy_set_header   X-Real-IP        $remote_addr;
           proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
           proxy_redirect     off;
    }

    location = /robots.txt {
        return 200 "User-agent: *\nDisallow:";
    }

    location ~ ^/nginx_status/ {
        stub_status on;
        access_log off;
    }

#    location ~* \.(ttf|woff|ttf|svg|eot)$ {
#       add_header 'Access-Control-Allow-Origin' '*';
#       add_header 'Access-Control-Allow-Credentials' 'true';
#       add_header 'Access-Control-Allow-Methods' 'GET,POST,HEAD';
#    }

#    location ~ \.(gif|jpg|jpeg|js|css|png|bmp|ico)$ {
#        expires 30d;
#    }

#     location ~* \.(jpg|jpeg|gif|css|png|js|ico|html|woff)$ {
#       expires max;
#     }

}

server {
    listen       443 ssl;
    server_name  www.heyuan110.com;

    access_log  /var/log/nginx/sslaccess.log;
    error_log   /var/log/nginx/sslerror.log;

    ssl on;
ssl_prefer_server_ciphers On;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
#ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS;
ssl_ciphers ALL:!ADH:!EXP:!LOW:!RC2:!3DES:!SEED:!RC4:+HIGH:+MEDIUM;
    ssl_certificate /usr/local/nginx/conf/ssl/website_ssl.crt;
    ssl_certificate_key /usr/local/nginx/conf/ssl/website_ssl.key;
#    ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
#    ssl_prefer_server_ciphers on;
#    keepalive_timeout   70;

    location ~* /news/show/* {
        proxy_pass http://web;
        proxy_read_timeout 300;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect     off;

        ### Most PHP, Python, Rails, Java App can use this header -> https ###
        proxy_set_header X-Forwarded-Proto  $scheme;
    }

    location ~ \.(css|js|gif|jpg|woff|woff2|png|ico)$ {
        proxy_pass  http://web;
    }

    location / {
      return  301 http://$server_name$request_uri;
    }

}

```