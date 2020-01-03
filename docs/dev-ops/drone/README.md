# Drone

相对于 Drone 0.8，Drone 1.0.0做了很大的改变。

* 不再采用 gRPC，直接使用 Http
* 多平台、多节点
* pipeline 也做了很大的修改
* 多平台编译应用的比较少，多节点应用会多点，特别是移动应用的自动化编译和发布

不过，目前 Drone 1.0.0-rc.1 的多节点不是很友好，节点匹配严格，不支持模糊匹配和通用匹配。

DRONE_RUNNER_LABELS & node

.env

```
# Drone - drone.${SERVER_DOMAIN}
DRONE_RPC_SECRET={your drone rpc secret}
DRONE_USER_FILTER={your gogs user a,your gogs user b}
```

docker-compose.yml

```yaml
  [...]
  # 只有 Repo 所有者/组织管理员 才有权限开启 CI/CD
  # 暂不支持设置时区
  drone-server:
    container_name: drone-server
    image: drone/drone:1.0.0-rc.1
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
      - "traefik.backend=drone"
      - "traefik.port=80"
      - "traefik.frontend.entryPoints=http"
      - "traefik.frontend.rule=Host:drone.${SERVER_DOMAIN}"

  # 暂不支持设置时区
  drone-agent:
    container_name: drone-agent
    image: drone/agent:1.0.0-rc.1
    restart: always
    hostname: drone-agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_RPC_SERVER=http://drone.${SERVER_DOMAIN}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_RUNNER_CAPACITY=2
    labels:
      - "traefik.enable=false"

  # 暂不支持设置时区
  drone-agent-linux:
    container_name: drone-agent-linux
    image: drone/agent:1.0.0-rc.1
    restart: always
    hostname: drone-agent-linux
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_RPC_SERVER=http://drone.${SERVER_DOMAIN}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_RUNNER_CAPACITY=2
      - DRONE_RUNNER_LABELS=location:remote,os:linux
    labels:
      - "traefik.enable=false"
  [...]
```

docker-compose-native.yml

```yaml
  [...]
  # 暂不支持设置时区
  drone-agent:
    container_name: drone-agent
    image: drone/agent:1.0.0-rc.1
    restart: always
    hostname: drone-agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
#      - TZ=${TIME_ZONE}
      - DRONE_RPC_SERVER=http://drone.${SERVER_DOMAIN}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_RUNNER_CAPACITY=2
      - DRONE_RUNNER_LABELS=location:native,os:mac
  [...]
```

.drone.yml

```yaml
kind: pipeline
name: default

clone:
  disable: true

steps:
- name: hello-world
  image: hello-world

---
kind: pipeline
name: remote

clone:
  disable: true

steps:
- name: hello-world
  image: hello-world

node:
  location: remote
  os: linux

---
kind: pipeline
name: native

clone:
  disable: true

steps:
- name: hello-world
  image: hello-world

# ssh连接宿主机
# https://discourse.drone.io/t/solved-app-automation-deployment/1599/6
# https://www.fwd.cloud/commit/post/drone-ios-deployments/
- name: ssh-docker-host
  image: appleboy/drone-ssh
  settings:
    host: host.docker.internal
    port: 22
    username: {your mac user}
    password: {your mac user password}
    script:
    - echo hello world
    - ls

node:
  location: native
  os: mac
```
