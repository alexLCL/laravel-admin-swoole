后台框架使用的是[laravel-admin](https://laravel-admin.org/docs/zh),使用了swoole的http替代了php-fpm:

环境选用的是[laradock](http://laradock.io/).
##laradock的配置
- 先把env里面的 `WORKSPACE_INSTALL_SWOOLE=true`，以及 `PHP_FPM_INSTALL_SWOOLE=true`
之后运行 docker-compose up -d nginx mysql

- 然后修改nginx的配置文件 ./nginx/sites/default.conf，**着重看下upstream**的配置


    `
    
    map $http_upgrade $connection_upgrade {
    
             default upgrade;
             ''      close;
         }
         
         upstream laravels {
         
             # Connect IP:Port
             server workspace:1215 weight=5 max_fails=3 fail_timeout=30s;
             keepalive 16;
         }
         
         server {
    
             listen 80;
         #    listen [::]:80 ipv6only=on;
    
             server_name localhost;
             root /var/www/swoole/public;
             index index.php index.html index.htm;
             error_log /var/www/swoole_error.log;
    
             location = /index.php {
                 # Ensure that there is no such file named "not_exists"
                 # in your "public" directory.
                 try_files /not_exists @swoole;
             }
    
             location / {
                  try_files $uri $uri/ @swoole;
             }
    
             location @swoole {
                 set $suffix "";
    
                 if ($uri = /index.php) {
                     set $suffix ?$query_string;
                 }
    
                 proxy_set_header Host $http_host;
                 proxy_set_header Scheme $scheme;
                 proxy_set_header SERVER_PORT $server_port;
                 proxy_set_header REMOTE_ADDR $remote_addr;
                 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                 proxy_set_header Upgrade $http_upgrade;
                 proxy_set_header Connection $connection_upgrade;
    
                 # IF https
                 # proxy_set_header HTTPS "on";
    
                 proxy_pass http://laravels$suffix;
             }
    
             location ~ /\.ht {
                 deny all;
             }
    
             location /.well-known/acme-challenge/ {
                 root /var/www/letsencrypt/;
                 log_not_found off;
             }
         }
         
         `

         
         
修改完之后，记得`docker-compose build nginx`

然后在重启`nginx docker-compos up -d nginx`

##laravel-admin的配置
 安装好laravel-admin以后，安装laravel-swoole
 
`$ composer require swooletw/laravel-swoole`

laravel5.5以上版本可以使用命令`php artisan vendor:publish`,选择发布刚才安装的laravel swoole的扩展包

发布完成后，项目的config下面会多出两个文件 swoole_http.php 和swoole_websocket.php
打开swoole_http.php,workspace对应的是刚才laradock里nginx的配置文件中**upstream**做的配置

` 'host' => env('SWOOLE_HTTP_HOST', 'workspace'),`

`  'access_log' => env('SWOOLE_HTTP_ACCESS_LOG', false),
`

最后，在laradock目录下，使用`docker-compose exec workspace bash`命令，进入到workspace镜像中
window系统下，此命令前面需要加**winpty**

然后进入你的项目的跟目录，例如我的是/var/www/swoole/，运行命令
`php artisan swoole:http start` 来启动服务
