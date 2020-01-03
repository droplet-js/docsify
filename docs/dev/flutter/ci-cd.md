# CI/CD

##### Drone

.drone.yml

```yaml
kind: pipeline
name: default

steps:
- name: clone-mac
  image: appleboy/drone-scp
  settings:
    host:
      from_secret: MAC_HOST
    port:
      from_secret: MAC_PORT
    username:
      from_secret: MAC_USER
    password:
      from_secret: MAC_PASSWORD
    source: ./*
    target: ~/.drone-build/${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}/${DRONE_BUILD_NUMBER}
    rm: true
  when:
    event:
    - tag

- name: prepare
  image: nathansamson/flutter-builder-docker:v1.0.0
  volumes:
  - name: pub-cache
    path: /opt/flutter/.pub-cache
  commands:
  - flutter packages get

- name: prepare-mac
  image: appleboy/drone-ssh
  settings:
    host:
      from_secret: MAC_HOST
    port:
      from_secret: MAC_PORT
    username:
      from_secret: MAC_USER
    password:
      from_secret: MAC_PASSWORD
    command_timeout: 600
    script:
    - date -R
    - BUILD_DIR=~/.drone-build/${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}/${DRONE_BUILD_NUMBER}
    - cd $BUILD_DIR
    - bash -lc 'flutter packages get'
    - if [ $? -ne 0 ]; then exit 1; fi; # 断言
    - date -R
  when:
    event:
    - tag

- name: analyze
  image: nathansamson/flutter-builder-docker:v1.0.0
  volumes:
  - name: pub-cache
    path: /opt/flutter/.pub-cache
  commands:
  - flutter analyze

- name: test
  image: nathansamson/flutter-builder-docker:v1.0.0
  volumes:
  - name: pub-cache
    path: /opt/flutter/.pub-cache
  commands:
  - flutter test
  - pushd example/
  - flutter test

- name: build-apk
  image: nathansamson/flutter-builder-docker:v1.0.0
  volumes:
  - name: pub-cache
    path: /opt/flutter/.pub-cache
  commands:
  - pushd example/
  - flutter -v build apk
  when:
    event:
    - tag

- name: build-ipa-mac
  image: appleboy/drone-ssh
  settings:
    host:
      from_secret: MAC_HOST # 私有化 drone-server，将 drone-agent 部署到 mac 机器上效果更好。 mac 支持 host.docker.internal
    port:
      from_secret: MAC_PORT
    username:
      from_secret: MAC_USER
    password:
      from_secret: MAC_PASSWORD
    command_timeout: 600
    script:
    - date -R
    - BUILD_DIR=~/.drone-build/${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}/${DRONE_BUILD_NUMBER}
    - cd $BUILD_DIR/example/
    - bash -lc 'flutter -v build ios --no-codesign'
    - if [ $? -ne 0 ]; then exit 1; fi; # 断言
    - date -R
  when:
    event:
    - tag

- name: clear-mac
  image: appleboy/drone-ssh
  settings:
    host:
      from_secret: MAC_HOST
    port:
      from_secret: MAC_PORT
    username:
      from_secret: MAC_USER
    password:
      from_secret: MAC_PASSWORD
    command_timeout: 600
    script:
    - date -R
    - BUILD_DIR=~/.drone-build/${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}/${DRONE_BUILD_NUMBER}
    - rm -rf $BUILD_DIR
    - date -R
  when:
    event:
    - tag
    status:
    - success
    - failure

volumes:
- name: pub-cache
  temp: {}
```