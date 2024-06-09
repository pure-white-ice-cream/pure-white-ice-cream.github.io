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

## 二、测试是否可用(不安全, 生产环境建议跳过此测试直接看三)

### 防火墙开放端口 `:2375`

### 创建或修改`/etc/systemd/system/docker.service.d/override.conf`配置文件

```
##Add this to the file for the docker daemon to use different ExecStart parameters (more things can be added here)
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
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

### 防火墙开放端口 `:2375`

### 创建或修改`/etc/systemd/system/docker.service.d/override.conf`配置文件

```
##Add this to the file for the docker daemon to use different ExecStart parameters (more things can be added here)
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```

### 创建证书

这步有很多方法, 网上教程很多, 例如 linux 使用以下脚本生成

```shell
#!/bin/sh
ip=<你的ip公网地址>
password=<设置一个密码>
dir=<生成的证书放到这个目录>


if [ ! -d "$dir" ];then
echo ""
echo "$dir , not dir , will create"
echo ""
mkdir -p $dir
else
echo ""
echo "$dir , dir exist , will delete and create"
echo ""
rm -rf $dir
mkdir -p $dir
fi

cd $dir

openssl genrsa -aes256 -passout pass:$password  -out ca-key.pem 4096

openssl req -new -x509 -days 365 -key ca-key.pem -passin pass:$password -sha256 -out ca.pem -subj "/C=NL/ST=./L=./O=./CN=$ip"

openssl genrsa -out server-key.pem 4096

openssl req -subj "/CN=$ip" -sha256 -new -key server-key.pem -out server.csr

echo subjectAltName = IP:$ip,IP:0.0.0.0 >> extfile.cnf

echo extendedKeyUsage = serverAuth >> extfile.cnf

openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -passin "pass:$password" -CAcreateserial -out server-cert.pem -extfile extfile.cnf

openssl genrsa -out key.pem 4096

openssl req -subj '/CN=client' -new -key key.pem -out client.csr

echo extendedKeyUsage = clientAuth >> extfile.cnf

echo extendedKeyUsage = clientAuth > extfile-client.cnf

openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -passin "pass:$password" -CAcreateserial -out cert.pem -extfile extfile-client.cnf

rm -f -v client.csr server.csr extfile.cnf extfile-client.cnf

chmod -v 0400 ca-key.pem key.pem server-key.pem

chmod -v 0444 ca.pem server-cert.pem cert.pem

```

创建后可以得到以下五个文件
`ca.pem`
`cert.pem`
`key.pem`
`server-cert.pem`
`server-key.pem`
, 其中

`ca.pem`
`cert.pem`
`key.pem`
在本地端 IDEA 用(下载到本地),

`ca.pem`
`server-cert.pem`
`server-key.pem`
在服务器端 docker 用(放到服务器`/etc/docker/tls/`目录下)

### 修改`/etc/docker/daemon.json`配置文件

```json
{
  "hosts": ["fd://", "tcp://0.0.0.0:2375"],
  "tlsverify": true,
  "tlscacert": "/etc/docker/tls/ca.pem",
  "tlscert": "/etc/docker/tls/server-cert.pem",
  "tlskey": "/etc/docker/tls/server-key.pem"
}
```

### 重启 docker 服务

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
$ sudo systemctl status docker
```

### 使用 IDEA 添加 Docker 服务

`ca.pem`
`cert.pem`
`key.pem`
上面说的这三个文件放到同一个文件夹里, IDEA 证书选择这个文件夹

---

## 写在结尾

### 除了 Docker 以外, IDEA 还可以连接服务器的 ssh 和 sftp, 非常方便

### 参考文章

- [Docker 客户端连接远程 Docker 服务](https://zhuanlan.zhihu.com/p/94224305)
- [Docker 系列-docker 配置远程访问 - 狮子挽歌丿 - 博客园](https://www.cnblogs.com/mrxccc/p/16504718.html)
