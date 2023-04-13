---
title:       "Docker 搭建私有仓库（二）"
subtitle:    "带TLS的安全仓库的搭建方法"
date: 2023-04-10
tags:        ["Docker", "registry"]
categories:  ["Docker"]
---

搭建带TLS的registry仓库，首先需要CA证书。这个证书可以购买，也可以使用****Let’s encrypt**** 免费证书。如果个人使用的话，还是很推荐Let‘s encrypt签发的免费证书，获取方便，最重要的是免费获取的。

### 1. 获取Let‘s encrypt证书

首先，需要安装certbot客户端工具，certbot是Let‘s encrypt 发布的获取证书的工具。在Ubuntu下可以使用以下命令安装：

```
sudo apt-get update
sudo apt-get install certbot

```

安装完成后，使用以下命令获取证书：

```
sudo certbot certonly --standalone  -d example.com

```

如果运行无误，则会在/etc/letsencrypt/live/example.com/文件夹下产生若干证书文件，其中搭建带TLS的registry仓库需要其中的fullchain.pem和privkey.pem这2个文件。

**注意**： 

1. 需要将example.com替换为自己的域名。
2. 在执行该命令之前，需要确保域名的80端口没有被占用。例如Nignx，需要先停止Ningx服务，再执行命令获取证书
### 2. 配置registry镜像仓库

首先在用户家目录下（本例为/home/lell/）建立docker目录，在其中建立仓库目录和ssl目录

```bash
mkdir docker
cd docker
mkdir registry
mkdir ssl

```

进入docker/ssl目录，准备证书文件：

```bash
cd ssl 
sudo cp /etc/letsencrypt/live/example.com/fullchain.pem domain.crt
sudo cp /etc/letsencrypt/live/example.com/privkey.pem domin.key
```

注意，需要将example.com替换为自己的域名。

### 3. 运行registry镜像

在上一步完成配置文件的修改后，就可以运行registry镜像了，使用以下命令：

```
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /home/lell/docker/registry:/var/lib/registry \
  -v /home/lell/docker/ssl:/certs \
  -e REGISTRY_HTTP_HOST=https://example.com \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  registry:2

```

该命令解释如下：

```bash
-v /home/lell/docker/registry:/var/lib/registry 
```

将/home/lell/docker/registry目录挂载到容器内/var/lib/registry目录上，这样容器内外“共享空间”，可以比较方便的查看、删除仓库内容。

```bash
-v /home/lell/docker/ssl:/certs 
```

将ssl证书文件的存储目录挂载到容器内的/certs中，registry启动时可以加载证书文件

之后的4个环境变量的指定也一目了然，不做过多解释。

### 4. 测试

可以使用curl命令测试registry是否使用TLS。[假设registry的域名为example.com](http://xn--registryexample-q53x08wbvr58q3s8lff1d.com/)，那么可以使用以下命令：

```
curl -k <https://example.com:5000/v2/>

```

如果返回以下信息，则表示配置成功：

```
{}

```

同时也可以在其他机器上测试拉取镜像是否成功。

**注意：**

如果服务器之前搭建过非安全的registry, 需要将/etc/docker/daemon.json文件中的insecure-registries配置项删除或注释掉，并重启docker。

尝试推送镜像到远程仓库：

```bash
docker pull busybox:latest
docker tag busybox:latest example.com:5000/busybox:latest
docker push example.com:5000/busybox:latest
```

此时，可以成功推送镜像到远程仓库中

### 5. 去掉端口号

如果觉得5000端口号比较碍眼，每次打tag时都要带上这个端口号很不优雅。亦或着不想在远程主机上开放除了80、443之外的端口，那么可以在远程主机上安装nginx作为反向代理，通过443端口转发到服务器的5000端口上，由于此过程比较简单常见，故具体方法不再赘述。那么在打tag和推送的那一步则相应的变成如下形式：

```bash
docker tag busybox:latest example.com/busybox:latest
docker push example.com/busybox:latest
```

那么到这一步，安全的远程仓库已经搭建完毕可以使用了。

### 6. Let‘s encrypt证书 自动续期

由于Let‘s encrypt签发的免费证书只有90天的有效期，因此需要定期更新证书。可以使用以下命令完成证书的更新：

```
certbot renew

```

如果证书即将到期，则会自动进行更新。为了方便自动续期，可以将该命令添加到cronjob中，事实上当你使用certbot 成功申请证书后，certbot已经自动在系统的cronjob中添加了自动续期指令。我们可以查看/etc/cron.d/certbot文件：

```bash
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl
-e 'sleep int(rand(43200))' && certbot -q renew 

```

但是即使系统可以自动续期，但是证书续期后，/etc/letsencrypt/live/example.com/中的证书文件也会随之改变，但是挂在到registry容器中的证书文件却没发生变化，当3个月证书到期后，registry容器的访问肯定会发生错误，被禁止访问。

所以，我们需要修改一下这个自动续期的cornjob任务脚本，在续期完成后，同步更新挂载到容器内的证书文件，修改/etc/cron.d/certbot文件如下：

```bash

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl
-e 'sleep int(rand(43200))' && certbot -q renew &&  
cp /etc/letsencrypt/live/example.com/fullchain.pem /home/lell/docker/ssl/domain.crt &&  
cp /etc/letsencrypt/live/example.com/privkey.pem /home/lell/docker/ssl/domain.key 


```
