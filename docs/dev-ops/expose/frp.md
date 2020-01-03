# ~~frp~~

##### Repo

* [v7lin/frp](https://github.com/v7lin/frp)

##### Useage

相比于 Ngrok，frp 的亲和度会更好点。不过配置文件方式和 Docker 不是很搭。

服务器：打开 80、443、7000 端口 DNS解析：域名解析 test、frp、*.frp、frps 指向服务器

.env

```
# 顶级域名
SERVER_DOMAIN={your domain}

# Time Zone
TIME_ZONE=Asia/Shanghai

# Traefik - traefik.${SERVER_DOMAIN}
TRAEFIK_ACME_EMAIL={your email}
# generate basic auth password
# openssl passwd -apr1 {your password}
TRAEFIK_AUTH_BASIC={user}:{encrypt password}

# frp - frps.${SERVER_DOMAIN}
# 帐号密码: frps/frps

# Portainer - portainer.${SERVER_DOMAIN}
```

frp/frps.ini

```
[common]
bind_addr = 0.0.0.0
bind_port = 7000

# udp
# bind_udp_port = 7001

# http/https
vhost_http_port = 80
# vhost_https_port = 443

# dashboard
dashboard_addr = 0.0.0.0
dashboard_port = 7500
dashboard_user = frps
dashboard_pwd = frps

# trace, debug, info, warn, error
log_level = info

# auth token
# token = frps

# subdomain
subdomain_host = frp.{your domain}.com
```

frps

docker-compose.yml

```yaml
# 版本
version: "3"

# 服务
services:

  # 暂不支持设置时区
  traefik:
    container_name: traefik
    image: traefik:1.7.4
    restart: always
    hostname: traefik
    ports:
      - 80:80
      - 443:443
#      - 8080:8080
    volumes:
      - ../dev-ops-repo/traefik/acme:/etc/traefik/acme
      - /var/run/docker.sock:/var/run/docker.sock
#    environment:
#      - TZ=${TIME_ZONE}
    command:
      - "--loglevel=WARN"
      - "--api"
      - "--defaultentrypoints=http,https"
      - "--entrypoints=Name:http Address::80"
      - "--entrypoints=Name:https Address::443 TLS"
      - "--acme"
      - "--acme.storage=/etc/traefik/acme/acme.json"
      - "--acme.entryPoint=https"
      - "--acme.httpChallenge.entryPoint=http"
      - "--acme.onHostRule=true"
      - "--acme.onDemand=false"
      - "--acme.email=${TRAEFIK_ACME_EMAIL}"
      - "--docker"
    labels:
      - "traefik.backend=traefik"
      - "traefik.port=8080"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:traefik.${SERVER_DOMAIN}"
      - "traefik.frontend.auth.basic=${TRAEFIK_AUTH_BASIC}"

  # bind: frp.${SERVER_DOMAIN}
  frps:
    container_name: frps
    image: hyperapp/frp
    restart: always
    hostname: frps
    ports:
#      - 80 # vhost - http
#      - 443 # vhost - https
#      - 6000
      - 7000:7000 # bind
#      - 7500 # dashboard
    volumes:
      - ./frp/frps.ini:/frps.ini
    command: -c frps.ini
    entrypoint:
      - /frps
    labels:
      - "traefik.vhost.backend=vhost"
      - "traefik.vhost.port=80"
      - "traefik.vhost.frontend.entryPoints=http"
      - "traefik.vhost.frontend.rule=HostRegexp:test.${SERVER_DOMAIN},{subdomain:[a-z]+}.frp.${SERVER_DOMAIN}"
      - "traefik.dashboard.backend=dashboard"
      - "traefik.dashboard.port=7500"
      - "traefik.dashboard.frontend.entryPoints=http"
      - "traefik.dashboard.frontend.rule=Host:frps.${SERVER_DOMAIN}"

  # 运维中心
  # 管理员帐号：admin
  # 暂不支持设置时区
  portainer:
    container_name: portainer
    image: portainer/portainer:1.19.2
    restart: always
    hostname: portainer
#    ports:
#      - 9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
#    environment:
#      - TZ=${TIME_ZONE}
    labels:
      - "traefik.backend=portainer"
      - "traefik.port=9000"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:portainer.${SERVER_DOMAIN}"
```

frp/frpc.ini

```
[common]
server_addr = frp.{your domain}.com
server_port = 7000

# trace, debug, info, warn, error
log_level = trace

# auth token
# token = frps

[www]
type = http
local_ip = www
local_port = 80
subdomain = www
custom_domains = test.{your domain}.com
```

frpc

docker-compose-native.yml

```yaml
# 版本
version: "3"

# 服务
services:

  www:
    container_name: www
    image: nginx:1.15.6
    restart: always
    hostname: www
#    ports:
#      - 80
    volumes:
      - ../dev-ops-repo/www/html:/usr/share/nginx/html
    environment:
      - TZ=${TIME_ZONE}

  frpc:
    container_name: frpc
    image: hyperapp/frp
    restart: always
    hostname: frpc
#    ports:
#      - 80
#      - 443
#      - 6000
#      - 7000
#      - 7500
    volumes:
      - ./frp/frpc.ini:/frpc.ini
    command: -c frpc.ini
    entrypoint:
      - /frpc
    depends_on:
      - www
```