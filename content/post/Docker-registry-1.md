---
title: "Docker 搭建私有仓库（一）"
subtitle: 非安全仓库的搭建方法
date: 2023-04-06
draft: false
tags: ["Docker", "registry"]
categories: ["Docker"]
---

一般我们构建Docker image 的时候，是从Docker 官方维护的镜像仓库Docker Hub上拉取的镜像。DockerHub 相当于Docker世界的Github, 上面有非常丰富的镜像资源。当然，你也可以在DockerHub上注册帐号，将自己的镜像存储在DockHub上。不过，如果自己或企业开发的程序，并没有开源的打算，只会自己内部使用，那就需要搭建自己的私有仓库，存放这些私有镜像。

一般来说私有仓库有2种应用场景：

- 搭建在局域网内使用，一般用在开发环境。仓库搭建在某台局域网中的服务器上，或是开发机上的虚拟机中，只有在此局域网中的机器才可以访问这个仓库。这种情况下，默认使用环境是安全的，可以使用http协议，也不需要进行用户验证。搭建这种私有仓库相对来说比较简单;
- 搭建在云服务器上，任何连接了Internet的机器，都可以访问到这个仓库，一般用在生产环境。这种情况下为了保证安全，必须使用带TLS的通信协议，也最好加上用户验证。

此篇blog先介绍非安全的私有仓库搭建过程，之后会介绍带TLS的安全仓库搭建过程。

### 1. 拉取、运行registry镜像

registry镜像是Docker官方维护的镜像仓库程序。首先我们要从DockerHub上拉取该镜像，并运行。

```bash
docker pull registry:2    #拉取registry镜像
docker run -d -p 5000:5000 --restart=always --name registry registry:2  #运行registry
```

此时registry容器运行起来。由于registry使用5000端口，所以需要将容器内5000端口映射到主机的5000端口上，所以加上了-p 5000:5000 参数。

### 2. 进行本地仓库的推送、拉取测试

下面演示本机推送镜像到刚刚搭建的registry仓库中：

1. 从DockerHub拉取nginx:alpine 这个镜像：

```bash
docker pull nginx:alpine
```

1. 给刚刚拉取的镜像打上tag

```bash
docker tag nginx:alpine localhost:5000/nginx:alpine
```

该tag中，localhost:5000/表明了仓库的url地址，在之后的拉取、推送过程中，docker会在这个url地址中寻找registry仓库，如果该仓库存在并正常运行，就会对这个仓库进行相应的操作。也就是说，这个tag指定了仓库地址。而如果没有这个tag指定仓库地址, docker会则会尝试连接DockerHub。

1. 推送镜像到本地仓库中

```bash
docker push localhost:5000/nginx:alpine
```

1. 删除本地的nginx镜像

```bash
docker rmi nginx:alpine
docker rmi localhost:5000/nginx:alpine
```

此时，本地已经没有了刚才拉取，以及打上tag的nginx镜像了，可以用docker images指令查看是否删除干净

```bash
docker images
$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED        SIZE
registry                      2         8db46f9d7550   29 hours ago   24.2MB
```

可以看到本地只有一个registry镜像，已经没有nginx镜像了。

1. 拉取本地registry仓库中的nginx镜像到本地

```bash
docker pull localhost:5000/nginx:alpine
docker images
$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED        SIZE
registry                      2         8db46f9d7550   29 hours ago   24.2MB
localhost:5000/nginx          alpine    8e75cbc5b25c   33 hours ago   41MB
```

可以看到，已经将仓库中的nginx拉取到了本地。

### 3. 局域网访问registry仓库

下面测试局域网内的另一台机器访问该registry仓库，以下假设regisitry仓库所在机器的ip地址为：192.168.56.101

我们试着拉取该仓库内的nginx镜像:

```bash
docker pull 192.168.56.101:5000/nginx:alpine
```

结果并不成功，报错了：

```bash
Error response from daemon: Get "<https://192.168.56.101:5000/v2/>": http: server gave HTTP response to HTTPS client
```

原因是registry默认会使用https协议，也就是使用安全模式。所以非安全模式需要进行一定的设置才行。方法如下：

在linux环境下，编辑/etc/docker/daemon.json文件，如果没有这个文件，就创建该文件。加入如下设置：

```bash
{
  "insecure-registries" : ["http://192.168.56.101:5000"]
}
```

保存文件，重启docker:

```
sudo systemctl restart docker
```

**注意**：

1. 你需要将192.168.56.101替换为你自己的registry仓库所在机器的ip地址；
2. 该daemon.json文件，必须要在所有相关机器上创建并且重启docker。即包括registry仓库所在机器，以及客户端访问机器；

设置完成后，再次测试：

```bash
docker pull 192.168.56.101:5000/nginx:alpine
```

结果还是报错：

```bash
Error response from daemon: manifest for 192.168.56.101:5000/nginx:alpine not found: manifest unknown: manifest unknown
```

原因仓库中目前保存的nginx 是localhost:5000/nginx:alpine,而不是192.168.56.101:5000/nginx:alpine，相当于没有这个镜像存在。所以我们需要重新给nginx重新tag一下，并推送进仓库。

回到仓库所在机器，重新tag并推送：

```bash
docker tag localhost:5000/nginx:alpine 192.168.56.101:5000/nginx:alpine
docker push 192.168.56.101:5000/nginx
```

再次回到客户机拉取镜像

```bash
docker pull 192.168.56.101:5000/nginx:alpine
alpine: Pulling from nginx
f56be85fc22e: Pull complete 
2ce963c369bc: Pull complete 
59b9d2200e63: Pull complete 
3e1e579c95fe: Pull complete 
547a97583f72: Pull complete 
1f21f983520d: Pull complete 
c23b4f8cf279: Pull complete 
Digest: sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
Status: Downloaded newer image for 192.168.56.101:5000/nginx:alpine
192.168.56.101:5000/nginx:alpine
```

此时拉取成功。这样局域网内的非安全仓库就搭建完毕了。其实也可以用该方法在公网服务器上搭建非安全的仓库，采用http协议进行镜像的拉取和推送。事实上网上也有不少blog文章，教授的是在公网上进行非安全的registry搭建，但我极其不建议这么做。如果仓库建立在公网上，那么最好还是做好安全措施，搭建采用带TLS的registry仓库，下一篇我会介绍怎样搭建安全的registry仓库。
