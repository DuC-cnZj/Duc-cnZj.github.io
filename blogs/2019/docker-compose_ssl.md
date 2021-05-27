---
title: docker-compose 服务安装 ssl
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - ssl
tags:
 - docker-compose
 - OpenID connect
publish: true
---




[Nginx-proxy](https://github.com/jwilder/nginx-proxy)

[docker-letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)

[docker-compose v3 解决 volumes_from 废弃的问题](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/issues/102)



```yaml
version: '3.5'

services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam
      - vhost:/etc/nginx/vhost.d
      - certs:/etc/nginx/certs:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
    labels:
      - "com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy"

  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: nginx-proxy-lets-encrypt
    depends_on:
      - "nginx-proxy"
    volumes:
      - certs:/etc/nginx/certs:rw
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - /var/run/docker.sock:/var/run/docker.sock:ro
  whoami:
    image: jwilder/whoami
    environment:
      - VIRTUAL_HOST=whoami.local
      - LETSENCRYPT_HOST=whoami.local
      - VIRTUAL_PORT=8080

volumes:
  certs:
  html:
  vhost:
  dhparam:
```



