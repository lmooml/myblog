---
title:       "Docker 搭建私有仓库（三）"
subtitle:    "为私有仓库增加用户验证"
date: 2023-04-11
tags:        ["Docker", "registry"]
categories:  ["Docker" ]
---

搭建了带TLS的registry仓库，解决了数据传输过程中的安全问题。但是在数据访问的源头上并没有什么限制措施，任何人知道了仓库地址，都可以去访问。所以，为了进一步提高安全性，我们可以给仓库加上用户验证措施。只有通过用户名及密码成功登录仓库，才有权限进行拉取、推送镜像的操作。

## 1. 在registry服务器上创建用户名及密码

在前一篇[《搭建带TLS的安全仓库》](/post/docker-registry-2)文章中的，我们在registry仓库所在的服务器中有创建过$HOME/docker目录，在这个目录中再创建一个auth目录，并生成一个基于htpasswd的用户名及密码对，保存在auth内的htpasswd文件中。

```bash
cd docker
mkdir auth
docker run \
  --entrypoint htpasswd \
  httpd:2 -Bbn testuser testpassword > auth/htpasswd
```

**注意**：需要把testuser替换成你的用户名，把testpassword替换成你的登录密码。

## 2. 用新指令重启registry容器

先停止registry容器，再用新的启动指令重新启动registry，将该密码文件挂载到registry容器内。这个过程比较简单，只需要在docker run命令中多加几个选项即可。例如：

```bash
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /home/lell/docker/registry:/var/lib/registry \
  -v /home/lell/docker/ssl:/certs \
  -v /home/lell/docker/auth:/auth \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_HOST=https://example.com \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/cert.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/privkey.key \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  registry:2

```

其中新增的三个-v选项分别为：

```bash
-v /home/lell/docker/auth:/auth   
# 将/auth目录挂载到容器内的/auth目录，以便容器内部可以访问到该密码文件；

```

```bash
-e REGISTRY_AUTH=htpasswd
-e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
# 指定认证方式；
```

```bash
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
# 指定密码文件的路径。
```

这样，我们就可以在客户机上使用刚刚创建的用户名和密码访问registry仓库了。例如：

```bash
docker login example.com   #登录仓库
docker logout example.com  #登出仓库
```

此时，不登录仓库，直接拉取镜像，会报用户验证没有通过的错误，拉取失败：

```bash
docker pull example.com/busyboy:latest
Error response from daemon: Head "https://example.com/v2/busyboy/manifests/latest": no basic auth credentials
```

那我们先登录，按提示输入刚刚创建的用户名和密码即可。

```bash
docker login example.com
```

结果现实登录成功，但是报出了一个警告：

```bash
WARNING! Your password will be stored unencrypted in /home/lell/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

我们先不管这个警告，继续尝试拉取镜像：

```bash
docker pull example.com/busybox:latest
```

如果不出意外的话，拉取成功。此时，带用户验证及TLS的registry仓库就搭建完成，可以安全使用了。

在网上，有很多的教学文章基本上都只介绍到这一步，这个警告几乎没人理会。但是那个警告每次登录时，都会出现，是否碍眼？没有强迫症的程序员不是合格的程序员，那么再进行几步设置，就可以去掉那个警告。

首先来看下这个警告是什么意思。意思是说，你的登录密码将会以非加密的形式存储在客户机上,需要你配置credential helper 来移除这个警告，具体方法看这个链接指向的文档说明。说白了，还是安全问题，就是要让你的客户机的登录密码以加密方式存储，使整个安全链尽量无死角。

## 3. 去掉客户机登录警告

其实，在警告中，docker已经给出了一个链接地址：[https://docs.docker.com/engine/reference/commandline/login/#credentials-store](https://docs.docker.com/engine/reference/commandline/login/#credentials-store)，这个链接指向了一个说明文档，用来解决这个警告。这个文档说明虽然比较清楚了，但对于新手来说可能不太友好，因为如果缺少了相关知识，可能看不懂或是根本完成不了这个安全配置。我也是查了不少相关资料，才成功配置成功。下面以ubuntu系统为例做一个说明，MacOs下也相差不多，可以举一反三进行设置。

### 第一步，在客户机上安装pass工具：

```bash
sudo apt install pass
```

### 第二步，下载安装docker-credential-helpers工具:
由于我们使用的是pass，所以要使用对应的docker-credential-pass这个工具，打开工具的发布页面 [https://github.com/docker/docker-credential-helpers/releases](https://github.com/docker/docker-credential-helpers/releases)，找到对应系统下的docker-credential-pass，我选择的是：[docker-credential-pass-v0.7.0.linux-amd64](https://github.com/docker/docker-credential-helpers/releases/download/v0.7.0/docker-credential-pass-v0.7.0.linux-amd64)

下载并安装到$PATH目录下：

```bash
wget https://github.com/docker/docker-credential-helpers/releases/download/v0.7.0/docker-credential-pass-v0.7.0.linux-amd64
mv docker-credential-pass-v0.7.0.linux-amd64 ~/.local/bin/docker-credential-pass
cd ~/.local/bin
chmod +x docker-credential-pass

```

解释下安装步骤：由于docker内部会调用名为docker-credential-pass的helper，所以我们需要把刚下载的helper改名为docker-credential-pass，并且放置在$PATH目录下，我选择放在$HOME/.local/bin目录下，你可以根据自己机器的实际情况选择$PATH目录。最后别忘了给程序增加使用权限。

编辑$HOME/.docker/config.json文件，没有就创建该文件，在文件中加入如下属性：

```bash
{
  "credsStore": "pass"
}
```

### 第三步，设置pass工具:

pass是linux下的密码管理工具，具体使用方法可自己google查询。之前我们已经安装好了pass,现在进行使用前的设置。

首先要初始化密码库，具体指令如下

```bash
pass init “Password Storage Key”
```

其中，“Password Storage Key”字符串为密码库的密钥，docker-credential-pass工具的文档指出，这个密钥必须由gpg2生成。

所以，我们需要先生成gpg2密钥。首先查看你的系统的gpg版本是不是2.0，如果不是需要升级或安装2.0版本以上的gpg程序。我自己的机器是gpg2.2.27版本，足够使用。

输入如下指令，并在之后按照提示信息输入姓名、email、保护密码，即可生成密钥对

```bash
gpg --generate-key
```

最后，初始化pass的密钥库,其中蓝色字串需要填写刚才生成gpg2 key 时输入的email信息

```bash
pass init "the ID of your GPG key"
```

此时，客户端的密钥设置完成。让我们来重新登录一下远程仓库,输入用户名、密码，显示登录成功，并且没有警告信息了

```bash
docker login example.com
Username: lell
Password: 
Login Succeeded
```

至此，用户验证功能配置完毕，打通了安全链的最后一环。我们可以安全的使用私有远程仓库了。