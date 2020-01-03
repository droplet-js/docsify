# Drone Secret

有时候我们想用 .drone.yml 登录服务器自动部署服务，但是又不想将相关帐号密码等铭感信息暴露出来，这时候就需要[Secrets](https://docs.drone.io/configure/secrets/)

.drone.yml

```yaml
kind: pipeline
name: default

clone:
  disable: true

steps:
- name: hello-world
  image: hello-world

- name: alpine
  image: alpine
  environment:
    MAC_USER:
      from_secret: MAC_USER
  commands:
  - echo $MAC_USER
  - echo $${MAC_USER}

- name: ssh-remote
  image: appleboy/drone-ssh
  settings:
    host:
      from_secret: ALIYUN_ECS_HOST
    port: 22
    username:
      from_secret: ALIYUN_ECS_USER
    password:
      from_secret: ALIYUN_ECS_PASSWORD
    script:
    - date -R
    - uname -a
    - date -R

- name: notify-webhook
  image: plugins/webhook
  settings:
    urls: {dingtalk webhook url} # 替换为钉钉自定义机器人 webhook url
    debug: true
    content_type: application/json
    template: |
      {
        "msgtype": "actionCard",
        "actionCard": {
          "title": "Drone Notification",
          {{#success build.status}}
          "text": "Repo Build Event\n- Repo: **{{repo.owner}}/{{repo.name}}**\n- Pusher: **{{build.author}}**\n- Build Status: **Successfully**",
          {{else}}
          "text": "Repo Build Event\n- Repo: **{{repo.owner}}/{{repo.name}}**\n- Pusher: **{{build.author}}**\n- Build Status: **Failure**",
          {{/success}}
          "singleTitle": "View Build Result",
          "singleURL": "{{build.link}}"
        }
      }

- name: notify-email
  image: drillster/drone-email
  settings:
    host: smtp.qq.com
#    port: 465 # 587
    skip_verify: true
    username:
      from_secret: EMAIL_USER
    password:
      from_secret: EMAIL_PASSWORD
    from:
      from_secret: EMAIL_USER
    recipients: [ v7lin@qq.com ]
#    recipients_only: true
  when:
    event:
    - tag

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
  environment:
    MAC_USER:
      from_secret: MAC_USER
    MAC_PASSWORD:
      from_secret: MAC_PASSWORD
  settings:
    host: host.docker.internal
    port: 22
    username:
      from_secret: MAC_USER
    password:
      from_secret: MAC_PASSWORD
    envs: [ MAC_USER,MAC_PASSWORD ]
    script:
    - date -R
    - system_profiler SPSoftwareDataType
    - bash -lc 'flutter --version'
    - echo $MAC_USER/$MAC_PASSWORD
    - echo $${MAC_USER}/$${MAC_PASSWORD}
    - bash -lc 'security unlock-keychain -p '$${MAC_PASSWORD}' login.keychain'
    - date -R

node:
  location: native
  os: mac
```