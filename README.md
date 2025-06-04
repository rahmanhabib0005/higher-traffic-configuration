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

# 📦 Fixing "413 Request Entity Too Large" in Nginx on Ubuntu

This guide explains how to resolve the **413 Request Entity Too Large** error when uploading large files via a PHP application served with **Nginx** on **Ubuntu**. The fix involves adjusting both Nginx and PHP configuration limits.

---

## 🚨 Problem

When attempting to upload a file larger than the allowed size, the following error appears:

```
413 Request Entity Too Large
```

---

## ✅ Solution Overview

1. Increase upload limits in **Nginx**.
2. Increase file size limits in **PHP**.
3. Restart services to apply changes.

---

## 🔧 Step-by-Step Instructions

### 1. Increase File Upload Limit in Nginx

#### Option A: Per-site Configuration

Edit your Nginx site file (example: `/etc/nginx/sites-enabled/your-site.conf`):

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    client_max_body_size 200M;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

#### Option B: Global Configuration

Edit the main nginx config:

```bash
sudo nano /etc/nginx/nginx.conf
```

Inside the `http` block, add or update:

```nginx
http {
    client_max_body_size 200M;

    # Other settings...
}
```

Save and exit.

---

### 2. Update PHP File Size Limits

Edit the PHP FPM config:

```bash
sudo nano /etc/php/7.4/fpm/php.ini
```

Change these values:

```ini
upload_max_filesize = 200M
post_max_size = 200M
```

> 💡 Tip: You can also check `/etc/php/7.4/cli/php.ini` if you're running scripts from the CLI.

---

### 3. Restart Services

Run the following commands to apply changes:

```bash
sudo systemctl restart php7.4-fpm
sudo systemctl reload nginx
```

---

## ✅ Confirm Success

1. Re-upload your large file via the web interface.
2. Check if the error is gone.
3. If issues persist, double-check logs:

```bash
tail -f /var/log/nginx/error.log
```

---

## 📌 Additional Nginx Performance Tips

To optimize further for high-load systems (e.g. 1k employees, 10k member requests):

```nginx
# in nginx.conf inside http block
worker_rlimit_nofile 100000;
worker_processes auto;

events {
    worker_connections 10240;
    use epoll;
    multi_accept on;
}
```

---

## 📂 Summary

| Component | Setting                | Value |
| --------: | ---------------------- | ----- |
|     Nginx | `client_max_body_size` | 200M  |
|       PHP | `upload_max_filesize`  | 200M  |
|       PHP | `post_max_size`        | 200M  |

---

## 🔪 Tested On

* **Ubuntu 22.04 LTS**
* **Nginx 1.18+**
* **PHP 7.4 FPM**
* **32 GB RAM VPS**

---

> 🛡️ Remember to always test after changes and monitor logs for remaining issues.

