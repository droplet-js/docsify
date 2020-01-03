# Docker-Compose VS Docker-Swarm

docker compose 和 docker swarm 各有千秋。

如果只是搭建开发环境, docker-compose 完全够用。喜欢折腾的除外，比如 -- 我！

##### 环境配置

.env 内容

```
# 顶级域名
SERVER_DOMAIN={your domain}

# Time Zone
TIME_ZONE=Asia/Shanghai

# Traefik - traefik.${SERVER_DOMAIN}
TRAEFIK_ACME_EMAIL={your email}
# generate basic auth password
# openssl passwd -apr1 {password}
TRAEFIK_AUTH_BASIC={user}:{encrypt password}

# WWW - www.${SERVER_DOMAIN}

# Blog - blog.${SERVER_DOMAIN}

# Filebrowser - filebrowser.${SERVER_DOMAIN}
# appleboy/drone-scp 部署到 ~/docker/dev-ops-repo/filebrowser/files
# http://filebrowser.${SERVER_DOMAIN}/files/{file_path}

# Nexus - nexus.${SERVER_DOMAIN}/nexus

# Gogs - git.${SERVER_DOMAIN}
# 初始化配置
# 1.应用 URL -> http://git.${SERVER_DOMAIN}/
# 2.应用域名 -> git.${SERVER_DOMAIN}
# 3.关闭注册功能

# Drone - drone.${SERVER_DOMAIN}
DRONE_RPC_SECRET={secret}
DRONE_USER_FILTER={user}

# frps - frps.${SERVER_DOMAIN}
# 帐号密码: frps/frps
# 客户端连接：frp.${SERVER_DOMAIN}:7000

# Registry - registry.${SERVER_DOMAIN} | registry-ui.${SERVER_DOMAIN}
# generate basic auth password
# openssl passwd -apr1 {password}
REGISTRY_AUTH_BASIC={user}:{encrypt password}

# Portainer - portainer.${SERVER_DOMAIN}
```

##### Docker-Compose

* 编辑 docker-compose.yml 内容

```yaml
# 版本
version: "3"

# 服务
services:

  # 暂不支持设置时区
  traefik:
    container_name: traefik
    image: traefik:1.7.6
    restart: always
    hostname: traefik
    ports:
      - 80:80
      - 443:443
#      - 8080:8080
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ../dev-ops-repo/traefik/acme:/etc/traefik/acme
#    environment:
#      - TZ=${TIME_ZONE}
    command:
      - "--api"
      - "--entrypoints=Name:http Address::80"
      - "--entrypoints=Name:https Address::443 TLS"
      - "--defaultentrypoints=http,https"
      - "--acme"
      - "--acme.storage=/etc/traefik/acme/acme.json"
      - "--acme.entryPoint=https"
      - "--acme.httpChallenge.entryPoint=http"
      - "--acme.onHostRule=true"
      - "--acme.onDemand=false"
      - "--acme.email=${TRAEFIK_ACME_EMAIL:-foobar@example.com}"
      - "--docker"
      - "--docker.exposedbydefault=false"
    labels:
      - "traefik.enable=true"
      - "traefik.backend=traefik"
      - "traefik.port=8080"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:traefik.${SERVER_DOMAIN:-localhost}"
      - "traefik.frontend.auth.basic=${TRAEFIK_AUTH_BASIC}"

  # 运维中心
  # 管理员帐号：admin
  # 暂不支持设置时区
  portainer:
    container_name: portainer
    image: portainer/portainer:1.20.0
    restart: always
    hostname: portainer
#    ports:
#      - 9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ../dev-ops-repo/portainer:/data
#    environment:
#      - TZ=${TIME_ZONE}
    labels:
      - "traefik.enable=true"
      - "traefik.backend=portainer"
      - "traefik.port=9000"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:portainer.${SERVER_DOMAIN:-localhost}"

  filebrowser:
    container_name: filebrowser
    image: nginx:1.15.7
    restart: always
    hostname: filebrowser
#    ports:
#      - 80
    volumes:
      - ../dev-ops-repo/filebrowser/files:/usr/share/nginx/html/files
    environment:
      - TZ=${TIME_ZONE}
    labels:
      - "traefik.enable=true"
      - "traefik.backend=filebrowser"
      - "traefik.port=80"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:filebrowser.${SERVER_DOMAIN:-localhost}"

#  # maven 私库
#  # 账号/密码：admin/admin123
#  nexus:
#    container_name: nexus
#    image: sonatype/nexus:2.14.10
#    restart: always
#    hostname: nexus
##    ports:
##      - 8081
#    volumes:
#      - ../dev-ops-repo/nexus:/nexus-data
#    environment:
#      - TZ=${TIME_ZONE}
#    labels:
#      - "traefik.enable=true"
#      - "traefik.backend=nexus"
#      - "traefik.port=8081"
#      - "traefik.frontend.entryPoints=http"
#      - "traefik.frontend.rule=Host:nexus.${SERVER_DOMAIN:-localhost}"

  # git 私库
  gogs:
    container_name: gogs
    image: gogs/gogs:0.11.79
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
      - "traefik.enable=true"
      - "traefik.backend=gogs"
      - "traefik.port=3000"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:git.${SERVER_DOMAIN:-localhost}"

  # 只有 Repo 所有者/组织管理员 才有权限开启 CI/CD
  # traefik 下，不支持 localhost 模式
  # 暂不支持设置时区
  drone-server:
    container_name: drone-server
    image: drone/drone:1.0.0-rc.3
    restart: always
    hostname: drone-server
#    ports:
#      - 80
#      - 443
    volumes:
      - ../dev-ops-repo/drone-server:/data
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_TLS_AUTOCERT=false
      - DRONE_GIT_ALWAYS_AUTH=false
      - DRONE_GOGS_SERVER=http://git.${SERVER_DOMAIN}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_SERVER_HOST=drone.${SERVER_DOMAIN}
      - DRONE_SERVER_PROTO=http
      - DRONE_USER_FILTER=${DRONE_USER_FILTER}
      - DRONE_LOGS_DEBUG=true
      - DRONE_LOGS_PRETTY=true
      - DRONE_LOGS_COLOR=true
    labels:
      - "traefik.enable=true"
      - "traefik.backend=drone"
      - "traefik.port=80"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:drone.${SERVER_DOMAIN}"

  # traefik 下，不支持 localhost 模式
  # 暂不支持设置时区
  drone-agent:
    container_name: drone-agent
    image: drone/agent:1.0.0-rc.3
    restart: always
    hostname: drone-agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_RPC_SERVER=http://drone.${SERVER_DOMAIN}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
#      - DRONE_RUNNER_CAPACITY=2
#      - DRONE_RUNNER_LABELS=location:remote,os:linux
    labels:
      - "traefik.enable=false"

  # bind: frp.${SERVER_DOMAIN}
  # localhost 模式需修改 frpc.ini 和 frps.ini
  # 暂不支持设置时区
  frps:
    container_name: frps
    image: hyperapp/frp
    restart: always
    hostname: frps
    expose:
      - 7001
    ports:
#      - 80 # vhost - http
#      - 443 # vhost - https
#      - 6000
#      - 7001
      - 7000:7000 # bind
#      - 7500 # dashboard
    volumes:
      - ./frp/frps.ini:/frps.ini
#    environment:
#      - TZ=${TIME_ZONE}
    entrypoint:
      - /frps
    command: -c frps.ini
    labels:
      - "traefik.enable=true"
      - "traefik.vhost.backend=vhost"
      - "traefik.vhost.port=80"
      - "traefik.vhost.frontend.entryPoints=http"
      - "traefik.vhost.frontend.rule=HostRegexp:test.${SERVER_DOMAIN},{subdomain:[a-z]+}.frp.${SERVER_DOMAIN}"
      - "traefik.dashboard.backend=dashboard"
      - "traefik.dashboard.port=7500"
      - "traefik.dashboard.frontend.entryPoints=http"
      - "traefik.dashboard.frontend.rule=Host:frps.${SERVER_DOMAIN}"

#  # docker 私库
#  # 暂不支持设置时区
#  registry:
#    container_name: registry
#    image: registry:2.6.2
#    restart: always
#    hostname: registry
##    ports:
##      - 5000
#    volumes:
#      - ../dev-ops-repo/registry:/var/lib/registry
#    environment:
##      - TZ=${TIME_ZONE}
#      - REGISTRY_STORAGE_DELETE_ENABLED=true
#    labels:
#      - "traefik.enable=false"

#  # 坑爹
#  # 点完删除按钮后
#  # 到私库容器中执行'registry garbage-collect /etc/docker/registry/config.yml'
#  # 并删除相关仓库'var/lib/registry/docker/registry/v2/repositories'
#  registry-ui:
#    container_name: registry-ui
#    image: joxit/docker-registry-ui:0.6-static
#    restart: always
#    hostname: registry-ui
##    ports:
##      - 80
#    environment:
#      - TZ=${TIME_ZONE}
#      - REGISTRY_TITLE=registry.${SERVER_DOMAIN:-localhost}
#      - REGISTRY_URL=http://registry:5000
##      - DELETE_IMAGES=true # 功能不完善
#    depends_on:
#      - registry
#    labels:
#      - "traefik.enable=true"
#      - "traefik.backend=registry"
#      - "traefik.port=80"
#      - "traefik.frontend.redirect.entryPoint=https"
#      - "traefik.frontend.rule=Host:registry.${SERVER_DOMAIN:-localhost}"
#      - "traefik.frontend.auth.basic=${REGISTRY_AUTH_BASIC}"
```

* 部署/更新 docker-compose.yml

```shell
docker-compose up -d
# 或
docker-compose -f docker-compose.yml up -d
```

##### Docker-Swarm

相对于 docker-compose，docker-swarm 比较麻烦，也存在若干问题，均在[Docker-Swarm](/docker/docker-swarm.md)中提及。

* 编辑 docker-swarm-traefik.yml 内容

```yaml
# 版本
version: "3.7"

# 服务
services:

  # 暂不支持设置时区
  traefik:
    image: traefik:1.7.6
    ports:
      - 80:80
      - 443:443
#      - 8080:8080
    networks:
      - net # traefik_net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ../dev-ops-repo/traefik/acme:/etc/traefik/acme
#    environment:
#      - TZ=${TIME_ZONE}
    command:
      - "--api"
      - "--entrypoints=Name:http Address::80"
      - "--entrypoints=Name:https Address::443 TLS"
      - "--defaultentrypoints=http,https"
      - "--acme"
      - "--acme.storage=/etc/traefik/acme/acme.json"
      - "--acme.entryPoint=https"
      - "--acme.httpChallenge.entryPoint=http"
      - "--acme.onHostRule=true"
      - "--acme.onDemand=false"
      - "--acme.email=${TRAEFIK_ACME_EMAIL:-foobar@example.com}"
      - "--docker"
      - "--docker.exposedbydefault=false"
      - "--docker.swarmMode"
      - "--docker.domain=${SERVER_DOMAIN:-localhost}"
      - "--docker.watch"
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik_net"
        - "traefik.port=8080"
        - "traefik.frontend.entryPoints=http"
        - "traefik.frontend.rule=Host:traefik.${SERVER_DOMAIN:-localhost}"
        - "traefik.frontend.auth.basic=${TRAEFIK_AUTH_BASIC}"

networks:
  # docker stack deploy -c <(docker-compose -f docker-compose-swarm-traefik.yml config) traefik
  # Creating network traefik_net
  # 外部引用 "traefik_net"
  net:
    driver: overlay
    attachable: true
```

* 部署/更新 docker-swarm-traefik.yml

```shell
# 部署/更新
docker stack deploy -c <(docker-compose -f docker-swarm-traefik.yml config) traefik
```

* 编辑 docker-swarm-devops.yml 内容

```yaml
# 版本
version: "3.7"

# 服务
services:

  # 管理员帐号：admin
  # 暂不支持设置时区
  portainer:
    image: portainer/portainer:1.20.0
#    ports:
#      - 9000
    networks:
      - traefik_net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ../dev-ops-repo/portainer:/data
#    environment:
#      - TZ=${TIME_ZONE}
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik_net"
        - "traefik.port=9000"
        - "traefik.frontend.entryPoints=http"
        - "traefik.frontend.rule=Host:portainer.${SERVER_DOMAIN:-localhost}"

  filebrowser:
    image: nginx:1.15.7
#    ports:
#      - 80
    networks:
      - traefik_net
    volumes:
      - ../dev-ops-repo/filebrowser/files:/usr/share/nginx/html/files
    environment:
      - TZ=${TIME_ZONE}
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik_net"
        - "traefik.port=80"
        - "traefik.frontend.entryPoints=http"
        - "traefik.frontend.rule=Host:filebrowser.${SERVER_DOMAIN:-localhost}"

  # 管理员帐号：admin
  # 暂不支持设置时区
  gogs:
    image: gogs/gogs:0.11.79
#    ports:
#      - 3000
#      - 22
    networks:
      - traefik_net
    volumes:
      - ../dev-ops-repo/gogs:/data
    environment:
      - TZ=${TIME_ZONE}
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik_net"
        - "traefik.port=3000"
        - "traefik.frontend.entryPoints=http"
        - "traefik.frontend.rule=Host:git.${SERVER_DOMAIN:-localhost}"

#  # maven 私库
#  # 账号/密码：admin/admin123
#  nexus:
#    image: sonatype/nexus:2.14.10
##    ports:
##      - 8081
#    networks:
#      - traefik_net
#    volumes:
#      - ../dev-ops-repo/nexus:/nexus-data
#    environment:
#      - TZ=${TIME_ZONE}
#    deploy:
#      replicas: 1
#      placement:
#        constraints:
#          - node.role == manager
#      update_config:
#        parallelism: 1
#        delay: 10s
#      restart_policy:
#        condition: on-failure
#      labels:
#        - "traefik.enable=true"
#        - "traefik.docker.network=traefik_net"
#        - "traefik.port=8081"
#        - "traefik.frontend.entryPoints=http"
#        - "traefik.frontend.rule=Host:nexus.${SERVER_DOMAIN:-localhost}"

#  # docker 私库
#  # 暂不支持设置时区
#  registry:
#    image: registry:2.6.2
##    ports:
##      - 5000
#    networks:
#      - registry
#    volumes:
#      - ../dev-ops-repo/registry:/var/lib/registry
#    environment:
##      - TZ=${TIME_ZONE}
#      - REGISTRY_STORAGE_DELETE_ENABLED=true
#    deploy:
#      replicas: 1
#      placement:
#        constraints:
#          - node.role == manager
#      update_config:
#        parallelism: 1
#        delay: 10s
#      restart_policy:
#        condition: on-failure
#      labels:
#        - "traefik.enable=false"

#  # 坑爹
#  # 点完删除按钮后
#  # 到私库容器中执行'registry garbage-collect /etc/docker/registry/config.yml'
#  # 并手动删除相关仓库'var/lib/registry/docker/registry/v2/repositories'
#  registry-ui:
#    image: joxit/docker-registry-ui:0.6-static
##    ports:
##      - 80
#    networks:
#      - registry
#      - traefik_net
#    environment:
#      - TZ=${TIME_ZONE}
#      - REGISTRY_TITLE=registry.${SERVER_DOMAIN:-localhost}
#      - REGISTRY_URL=http://registry:5000
##      - DELETE_IMAGES=true # 功能不完善
#    deploy:
#      replicas: 1
#      placement:
#        constraints:
#          - node.role == manager
#      update_config:
#        parallelism: 1
#        delay: 10s
#      restart_policy:
#        condition: on-failure
#      labels:
#        - "traefik.enable=true"
#        - "traefik.docker.network=traefik_net"
#        - "traefik.port=80"
#        - "traefik.frontend.redirect.entryPoint=https"
#        - "traefik.frontend.rule=Host:registry.${SERVER_DOMAIN:-localhost}"
#        - "traefik.frontend.auth.basic=${REGISTRY_AUTH_BASIC}"

  # 只有 Repo 所有者/组织管理员 才有权限开启 CI/CD
  # traefik 下，不支持 localhost 模式
  # 暂不支持设置时区
  drone-server:
    image: drone/drone:1.0.0-rc.3
#    ports:
#      - 80
#      - 443
    networks:
      - traefik_net
    volumes:
      - ../dev-ops-repo/drone-server:/data
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_TLS_AUTOCERT=false
      - DRONE_GIT_ALWAYS_AUTH=false
      - DRONE_GOGS_SERVER=http://git.${SERVER_DOMAIN}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_SERVER_HOST=drone.${SERVER_DOMAIN}
      - DRONE_SERVER_PROTO=http
      - DRONE_USER_FILTER=${DRONE_USER_FILTER}
      - DRONE_LOGS_DEBUG=true
      - DRONE_LOGS_PRETTY=true
      - DRONE_LOGS_COLOR=true
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik_net"
        - "traefik.port=80"
        - "traefik.frontend.entryPoints=http"
        - "traefik.frontend.rule=Host:drone.${SERVER_DOMAIN}"

  # traefik 下，不支持 localhost 模式
  # 暂不支持设置时区
  drone-agent:
    image: drone/agent:1.0.0-rc.3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_RPC_SERVER=http://drone.${SERVER_DOMAIN}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
#      - DRONE_RUNNER_CAPACITY=2
#      - DRONE_RUNNER_LABELS=location:remote,os:linux
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.enable=false"

  # bind: frp.${SERVER_DOMAIN}
  # localhost 模式需修改 frpc.ini 和 frps.ini
  # 暂不支持设置时区
  frps:
    image: hyperapp/frp
    expose:
      - 7001
    ports:
#      - 80 # vhost - http
#      - 443 # vhost - https
#      - 6000
#      - 7001
      - 7000:7000 # bind
#      - 7500 # dashboard
    networks:
      - traefik_net
    volumes:
      - ./frp/frps.ini:/frps.ini
#    environment:
#      - TZ=${TIME_ZONE}
    entrypoint:
      - /frps
    command: -c frps.ini
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik_net"
        - "traefik.vhost.port=80"
        - "traefik.vhost.frontend.entryPoints=http"
        - "traefik.vhost.frontend.rule=HostRegexp:test.${SERVER_DOMAIN},{subdomain:[a-z]+}.frp.${SERVER_DOMAIN}"
        - "traefik.dashboard.port=7500"
        - "traefik.dashboard.frontend.entryPoints=http"
        - "traefik.dashboard.frontend.rule=Host:frps.${SERVER_DOMAIN}"

networks:
  registry:
  traefik_net:
    external: true
```

* 部署/更新 docker-swarm-devops.yml

```shell
# 部署/更新
docker stack deploy -c <(docker-compose -f docker-swarm-devops.yml config) devops
```