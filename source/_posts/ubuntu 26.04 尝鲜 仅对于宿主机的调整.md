---
title: ubuntu 26.04 尝鲜 仅对于宿主机的调整
date: 2026/5/8 20:40:00
---
## 前言
- 使用新一代长期支持版 ubuntu, 代替了原来的 unraid, 作为nas用

## 时区校准
```sh
timedatectl set-timezone Asia/Shanghai
```

## SSH 安全增强
- 禁止 Root 密码登录, 修改默认端口, 防止被暴力破解
- 详情看博客 `https://ssd.us.kg/2025/10/28/Ubuntu免密码root登录ssh的公私钥方案/`

## 安装 docker 看官网
`https://docs.docker.com/engine/install/ubuntu/`

## 安装 portainer
`https://docs.portainer.io/start/install-ce/server/docker/linux`
`/opt/containers/portainer/docker-compose.yml`
```yml
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:sts
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    ports:
      - 9443:9443
      - 9000:9000
      - 8000:8000  # Remove if you do not intend to use Edge Agents

volumes:
  portainer_data:
    name: portainer_data

networks:
  default:
    name: portainer_network
```

## 镜像源切换
`/etc/apt/sources.list.d/ubuntu.sources`
```
Types: deb
URIs: https://mirrors.aliyun.com/ubuntu/
Suites: resolute resolute-updates resolute-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: https://mirrors.aliyun.com/ubuntu/
Suites: resolute-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

```

`/etc/apt/sources.list.d/docker.sources`
```
Types: deb
URIs: https://mirrors.aliyun.com/docker-ce/linux/ubuntu
Suites: resolute
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc

```

## 电源管理(不间断电源)
```sh
apt install apcupsd
```

`/etc/apcupsd/apcupsd.conf`
```
UPSCABLE usb
UPSTYPE usb
DEVICE
```

`/etc/apcupsd/apcupsd.conf`
```
BATTERYLEVEL 20: 电量剩余 20% 时就关机(默认 5% 容易让电池过放)。

MINUTES 10: 预计续航还剩 10 分钟时就关机(默认 3 分钟对某些运行大量容器的 NAS 来说有点紧)。

TIMEOUT 300: 如果你想设置“断电 5 分钟后无论电量多少直接关机”, 就设置这个参数。设置为 0 则禁用。
```

```sh
# 重启服务
systemctl restart apcupsd
# 检查并开启自启
systemctl enable apcupsd
# 验证状态
systemctl status apcupsd
```

## 定期快照
- swap 要单独分子卷
``` sh
apt install snapper
snapper -c root create-config /
```

`/etc/snapper/configs/root`
```
# limit for number cleanup
NUMBER_MIN_AGE="1800"
# apt 更新备份数量
NUMBER_LIMIT="4"
# 重要升级(内核)备份数量
NUMBER_LIMIT_IMPORTANT="2"


# create hourly snapshots
TIMELINE_CREATE="yes"

# cleanup hourly snapshots after some time
TIMELINE_CLEANUP="yes"

# limits for timeline cleanup
TIMELINE_MIN_AGE="1800"
# 限制保留“每小时”快照的数量
TIMELINE_LIMIT_HOURLY="6"
# 限制保留“每天”快照的数量
TIMELINE_LIMIT_DAILY="5"
# 限制保留“每月”快照的数量
TIMELINE_LIMIT_WEEKLY="2"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"

# 同时也建议开启空间配额清理(防止磁盘满)
EMPTY_PRE_POST_CLEANUP="yes"
```

- 每小时清理快照
```sh
systemctl edit snapper-cleanup.timer
```

``` ini
### Editing /etc/systemd/system/snapper-cleanup.timer.d/override.conf
### Anything between here and the comment below will become the contents of the drop-in file

[Timer]
OnUnitActiveSec=1h

### Edits below this comment will be discarded


### /usr/lib/systemd/system/snapper-cleanup.timer
# [Unit]
# Description=Daily Cleanup of Snapper Snapshots
# Documentation=man:snapper(8) man:snapper-configs(5)
#
# [Timer]
# OnBootSec=10m
# OnUnitActiveSec=1d
#
# [Install]
# WantedBy=timers.target

```

## 其他
- docker 只用 compose 方便迁移
- 只用卷, 多用具名卷, 必要共享卷, 少用匿名卷
- 安装了 zsh, 但是体感作用不大
