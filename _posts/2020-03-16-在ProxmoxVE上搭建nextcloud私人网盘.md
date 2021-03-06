---
layout:     post
title:      在Proxmox VE上搭建nextcloud私人网盘
subtitle:   ""
date:       2020-03-16
author:     Jet
header-img: img/post-bg-2015.jpg
catalog:    true
tags: 
- DOCKER
- NEXTCLOUD
- NGINX
---

# 在Proxmox VE上搭建nextcloud私人网盘

## OpenWrt 设置端口映射

/etc/config/firewall添加重定向：

```
config redirect
        option target 'DNAT'
        option src 'wan'
        option dest 'lan'
        option proto 'tcp udp'
        option dest_ip '192.168.8.254'
        option name 'PVE-NC-DK-WEB-SSL'
        option src_dport '44388'
        option dest_port '44388'
```

## Proxmox nginx 配置

/etc/nginx/sites-enabled/docker-nextcloud:

```
server {
  listen 8088;
  listen [::]:8088;
  server_name ${DOMAIN};
  return 301 https://$host$request_uri;
}

server {
    server_name ${DOMAIN} 0.0.0.0;
    listen 44388 ssl http2;
    ## comment this to avoid 400 error when visited through IP address
    #ssl on;
    ssl_certificate /etc/ssl/certs/${DOMAIN}.pem;
    ssl_certificate_key /etc/ssl/certs/${DOMAIN}.key;
    ##include /etc/nginx/conf/ssl_params.conf;##ERROR
    client_max_body_size 10G;
    location / {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:44378;
        proxy_set_header Host $http_host;
    }
    location = /.htaccess {
        return 404;
    }
}

server {
    server_name ${DOMAIN};
    listen [::]:44388 ssl http2;
    #ssl on;
    ssl_certificate /etc/ssl/certs/${DOMAIN2}.pem;
    ssl_certificate_key /etc/ssl/certs/${DOMAIN2}.key;
    ##include /etc/nginx/conf/ssl_params.conf;##ERROR
    client_max_body_size 10G;
    location / {
        proxy_redirect off;
        proxy_pass http://[::1]:44378;
        proxy_set_header Host $http_host;
    }
    location = /.htaccess {
        return 404;
    }
}

```

## docker 配置

docker-compose.yml:

```
version: '3'

networks:
    default:
        external:
            name: nas-bridge

services:
  nextcloud:
    container_name: nextcloud
    image: wonderfall/nextcloud
    restart: always
    depends_on:
      - nextcloud-db           # If using MySQL
    environment:
      - UID=1000
      - GID=1000
      - UPLOAD_MAX_SIZE=10G
      - APC_SHM_SIZE=128M
      - OPCACHE_MEM_SIZE=128
      - CRON_PERIOD=15m
#      - TZ=Aisa/Shanghai
      - ADMIN_USER=admin            # Don't set to configure through browser
      - ADMIN_PASSWORD=<>        # Don't set to configure through browser
      #- DOMAIN=localhost
      - DOMAIN=${DOMAIN}
      - DB_TYPE=mysql
      - DB_NAME=nextcloud
      - DB_USER=nextcloud
      - DB_PASSWORD=<>
      - DB_HOST=nextcloud-db
    volumes:
      - /etc/localtime:/etc/localtime
      - ./data:/data
      - ./config:/config
      - ./apps2:/apps2
      - ./themes:/nextcloud/themes
      - /mnt/wd500g/Downloads:/Downloads
      - /mnt/wd4000g/Photos:/Photos
      - /mnt/wd4000g/Study:/Study
      - /mnt/wd4000g/Work:/Work
#      - ./Study:/Study
#      - ./Work:/Work

    ports:
      - 44378:8888

  # If using MySQL
  nextcloud-db:
    container_name: mariadb
    image: mariadb:10
    restart: always
    stdin_open: true
    tty: true
    volumes:
      - /etc/localtime:/etc/localtime
      - ./db:/var/lib/mysql
    ports:
      - 3306:3306
    environment:
      - MYSQL_ROOT_PASSWORD=<MYSQL_ROOT_PASSWORD>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=<MYSQL_PASSWORD>
#      - UID=1000
#      - GID=1000
```

## 运行与排错

执行命令：

```
docker-compose up -d
```

输入域名访问，在页面里面出现错误：

>用户已存在

这是以前安装留下来的遗留数据的，先删除之前存储的数据,后启动就可以了：

```
sudo rm -Rf db/*
docker-compose up -d
```

输入浏览器地址栏输入域名，页面可能显示不在Trusted_Domain,按提示修改config/config.php文件即可：

```
 'trusted_domains' =>
  array (
    0 => '192.168.8.254:44388',
    1 => '${DOMAIN}:44388',
    ),
```

完了就可以登入了，此后可以在页面里增加一些用户。

扫描增加的文件：

```
docker exec -it nextcloud occ files:scan -v -p "/abc/files/" abc
```

## TO DO
待更新

```
sudo -u www-data php occ  maintenance:install \
--database "mysql" --database-name "nextcloud"  --database-user "root" --database-pass "pass" \
--admin-user "admin" --admin-pass "pass"
```

