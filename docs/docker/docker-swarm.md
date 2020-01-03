# Docker-Swarm

##### 问题记录

* 用服务编排方式集群部署 Docker，你会发现，特么的 .env 里面定义的环境变量没有被加载。

    [moby/issues#29133](https://github.com/moby/moby/issues/29133)
    
    解决办法
    ```shell
    docker stack deploy -c <(docker-compose config) traefik
    
    docker stack deploy -c <(docker-compose -f docker-swarm.yml config) traefik
    ```

* 相关挂载文件夹需提前创建

    解决办法
    ```shell
    # 手动创建相关文件夹
    mkdir -p xx/xx
    ```

* 私有镜像 pull 问题

    解决办法
    ```shell
    docker stack deploy -c <(docker-compose -f docker-swarm.yml config) services --with-registry-auth
    ```

* 镜像 tag 为 none，还引发了服务更新时有多个同名镜像问题

    目前无解
    
    ```shell
    [root@centos dev-ops]# docker images
    REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
    registry                                  <none>              9c1f09fe9a86        46 hours ago        33.3MB
    joxit/docker-registry-ui                  <none>              f59f8d09b3d1        2 days ago          18.5MB
    drone/agent                               <none>              120e85b52894        4 days ago          16.7MB
    drone/drone                               <none>              5728ad548bc8        4 days ago          63.5MB
    traefik                                   <none>              b0a0d7271403        8 days ago          69.3MB
    portainer/portainer                       <none>              a01958db7424        10 days ago         72.2MB
    gogs/gogs                                 <none>              0688786a8635        10 days ago         91.9MB
    nginx                                     <none>              568c4670fa80        3 weeks ago         109MB
    hyperapp/frp                              <none>              cc7a591c9c3a        2 months ago        20.7MB
    ```