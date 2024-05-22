---
title: IDEA连接远程Docker服务
date: 2024/5/22 21:40:43
---

# 环境

- CentOS 7.9.2009
- docker 26.1.3
- IDEA 2023.3.2

# 步骤

## 一、安装

### 通过 yum 安装 Docker

```shell
#通过 yum 安装 Docker
sudo yum install docker-ce docker-ce-cli containerd.io
```

### 启动和自启动

```shell
#启动 Docker
sudo systemctl start docker
#设置 Docker 开机自启
sudo systemctl enable docker
#查看 Docker 版本
sudo docker version
```

## 二、测试是否可用

### 防火墙开放端口 `:2375`

### 修改`/lib/systemd/system/docker.service`配置文件

源文件

```
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

修改后(去掉了` -H fd://`)

```
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```

### 添加`/etc/docker/daemon.json`配置文件

```json
{
  "hosts": ["fd://", "tcp://0.0.0.0:2375"]
}
```

### 重启 docker 服务

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
$ sudo systemctl status docker
```

### 使用 IDEA 添加 Docker 服务测试

## 三、安全地远程连接

### 创建 CA 和证书

这步有很多方法, 网上教程很多, 创建后可以得到以下五个文件
`ca.pem`
`cert.pem`
`key.pem`
`server-cert.pem`
`server-key.pem`
, 其中
`ca.pem`
`cert.pem`
`key.pem`
在本地端,
`ca.pem`
`server-cert.pem`
`server-key.pem`
在服务器端(放到`/etc/docker/tls/`目录下)

### 修改`/etc/docker/daemon.json`配置文件

```json
{
  "hosts": ["fd://", "tcp://0.0.0.0:2376"],
  "tlsverify": true,
  "tlscacert": "/etc/docker/tls/ca.pem",
  "tlscert": "/etc/docker/tls/server-cert.pem",
  "tlskey": "/etc/docker/tls/server-key.pem"
}
```

### 使用 IDEA 添加 Docker 服务

`ca.pem`
`cert.pem`
`key.pem`
这三个文件放到同一个文件夹里, IDEA 证书选择这个文件夹

---

## 写在结尾

### 除了 Docker 以外, 还可以连接服务器的 ssh 和 sftp, 非常方便

### 参考文章: [Docker 客户端连接远程 Docker 服务](https://zhuanlan.zhihu.com/p/94224305)
