########## NGINX GLOBAL CONFIGURATION ##########
# /etc/nginx/nginx.conf

user www-data;
worker_processes auto;
pid /run/nginx.pid;

# Max open file descriptors
worker_rlimit_nofile 100000;

events {
    worker_connections 16384;
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    client_max_body_size 512M;
    client_body_buffer_size 512k;

    fastcgi_buffers 64 16k;
    fastcgi_buffer_size 64k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 256k;

    fastcgi_read_timeout 300s;
    fastcgi_send_timeout 300s;

    gzip on;
    gzip_buffers 32 8k;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    limit_conn_zone $binary_remote_addr zone=addr:128m;
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:512m inactive=1h use_temp_path=off;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}


########## PHP-FPM POOL CONFIGURATION ##########
# /etc/php/7.4/fpm/pool.d/www.conf

[www]
user = www-data
group = www-data
listen = /run/php/php7.4-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic
pm.max_children = 300
pm.start_servers = 20
pm.min_spare_servers = 10
pm.max_spare_servers = 50
pm.max_requests = 500

request_terminate_timeout = 300
request_slowlog_timeout = 5
slowlog = /var/log/php7.4-fpm.slow.log

php_admin_value[error_log] = /var/log/php7.4-fpm.log
php_admin_flag[log_errors] = on


########## PHP-FPM GLOBAL CONFIGURATION ##########
# /etc/php/7.4/fpm/php-fpm.conf

[global]
pid = /run/php/php7.4-fpm.pid
error_log = /var/log/php7.4-fpm.log
include=/etc/php/7.4/fpm/pool.d/*.conf
