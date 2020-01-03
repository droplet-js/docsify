# ~~Gogs~~

工欲善其事，必先利其器 ...

对于我们开发者而言，代码托管和CI/CD是必不可少的，这能节约很多时间。

对于代码托管和CI/CD做得最好的，毋庸置疑是 Gitlab。然而 Gitlab 对服务器的要求，对于个人开发者或初创团队是一块心病。那么有没有其他方案呢？Gogs & Drone CI/CD，内存消耗小，运行速度快，你值得拥有！

[Docker](/docker/README.md)

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

# Gogs - git.${SERVER_DOMAIN}
# 初始化配置
# 1.应用 URL -> http://git.${SERVER_DOMAIN}/
# 2.应用域名 -> git.${SERVER_DOMAIN}
# 3.关闭注册功能

# Drone - drone.${SERVER_DOMAIN}
DRONE_SECRET={your drone secret}
DRONE_ADMIN={your drone admin}
DRONE_GOGS_GIT_USERNAME={your gogs user}
DRONE_GOGS_GIT_PASSWORD={your gogs user password}

# Portainer - portainer.${SERVER_DOMAIN}
```

docker-compose.yml

```yaml
# 版本
version: "3"

# 服务
services:

  # 暂不支持设置时区
  traefik:
    container_name: traefik
    image: traefik
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

  # git 私库
  gogs:
    container_name: gogs
    image: gogs/gogs
    restart: always
    hostname: gogs
#    ports:
#      - 3000
#      - 22
    volumes:
      - ../dev-ops-repo/gogs:/data
    environment:
      - TZ=${TIME_ZONE}
    labels:
      - "traefik.backend=gogs"
      - "traefik.port=3000"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:git.${SERVER_DOMAIN}"

  # 暂不支持设置时区
  drone-server:
    container_name: drone-server
    image: drone/drone:0.8
    restart: always
    hostname: drone-server
#    ports:
#      - 8000
#      - 9000
    volumes:
      - ../dev-ops-repo/drone-server:/var/lib/drone/
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_OPEN=false
      - DRONE_HOST=http://drone.${SERVER_DOMAIN}
      - DRONE_SECRET=${DRONE_SECRET}
      - DRONE_ADMIN=${DRONE_ADMIN}
      - DRONE_GOGS=true
      - DRONE_GOGS_URL=http://git.${SERVER_DOMAIN}
      - DRONE_GOGS_GIT_USERNAME=${DRONE_GOGS_GIT_USERNAME}
      - DRONE_GOGS_GIT_PASSWORD=${DRONE_GOGS_GIT_PASSWORD}
      - DRONE_GOGS_PRIVATE_MODE=true
    labels:
      - "traefik.drone.backend=drone"
      - "traefik.drone.port=8000"
      - "traefik.drone.frontend.entryPoints=http"
      - "traefik.drone.frontend.rule=Host:drone.${SERVER_DOMAIN}"
      - "traefik.drone-server.backend=drone-server"
      - "traefik.drone-server.port=9000"
      - "traefik.drone-server.protocol=h2c"
      - "traefik.drone-server.frontend.entryPoints=http"
      - "traefik.drone-server.frontend.rule=Host:drone-server.${SERVER_DOMAIN}"

  # 暂不支持设置时区
  drone-agent:
    container_name: drone-agent
    image: drone/agent:0.8
    restart: always
    hostname: drone-agent
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_SERVER=drone-server:9000
      - DRONE_SECRET=${DRONE_SECRET}
    command: agent
    labels:
      - "traefik.enable=false"

  # 运维中心
  # 管理员帐号：admin
  # 暂不支持设置时区
  portainer:
    container_name: portainer
    image: portainer/portainer
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
