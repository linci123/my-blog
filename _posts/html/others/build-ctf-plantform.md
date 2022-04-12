---
title: 使用ctfd搭建一个可以启动docker的ctf平台
date: 2020-07-01 12:27:49
tags: 
  - docker
  - ctf
categories: [ctf]
---

帮学校搞一个CTF平台，还要能自己开启docker环境，找到一个ctfd平台的插件 [CTFd-Whale](https://github.com/glzjin/CTFd-Whale)

这个插件提供的环境有一段时间没更新，有些配置已经不能用了，于是clone了下来改了几个地方

下面是完整Centos7搭建步骤，最好在爱国上网的环境下进行

## 安装Docker

### 一键安装docker-ce

```bash
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

### 安装docker-compose

```bash
pip3 install docker-compose
```

### 修改docker镜像源

```bash
# 首先启动一下docker
systemctl enable docker
systemctl start docker

vim /etc/docker/daemon.json #没有则新建一个文件
```

```json
{
	"registry-mirrors": ["http://hub-mirror.c.163.com"]
}
```

## 安装CTFD到docker

### 初始化swarm

```bash
docker swarm init --advertise-addr <你的ip> #这里就用一台机子，直接填自己的ip就行
docker node ls #查看节点ID
docker node update --label-add name=linux-1 <节点ID>
```

### 获取修改后的ctfd

```bash
git clone https://github.com/Kevin0z0/CTFd.git
cd CTFd/
```

### 修改token值

```bash
# 直接随机用md5sum生成一串字符
echo -n "jkdsarh8w9erhdkjfbasjkd" | md5sum
# 下面两个文件的token值用上面的md5就行
vim frp/frps.ini
vim frp/frpc.ini
```

### 开启服务

```bash
git submodule update --init # 下载插件
docker-compose up -d # 开启docker
```

参考链接: [CTFd-Whale](https://www.zhaoj.in/read-6333.html)

## 配置平台

![](/images/QQ截图20200701134043.png)

![](/images/QQ截图20200701134148.png)

直接跳到最后即可，进入插件

![](/images/QQ截图20200701134313.png)

剩下的配置可参考此处[CTFd-Whale](https://www.zhaoj.in/read-6333.html)

## 开启题目

他这个插件没有加进度条，所以点击按钮的一瞬间就创建好了，实际上后台还需要拉取镜像，所以一时还访问不了端口，可以事先将镜像拉取到本地

以他文档中给的例子为例，在服务器上先拉取镜像

```bash
docker pull ctftraining/qwb_2019_supersqli
```

然后再开启题目，大概等个10秒钟就可以正常访问了

![](/images/QQ截图20200701141827.png)
