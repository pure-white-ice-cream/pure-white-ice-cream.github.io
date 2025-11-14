---
title: mihomo核心在docker部署教程
date: 2025/11/04 12:00:00
---

# 简介

- `mihomo` 核心 (原 `clash.meta`) 官方文档: `https://wiki.metacubex.one/startup/`
- 常用客户端里, 缺少 `linux` 服务器的 `docker` 部署的使用方案
- 本篇教程在 `ubuntu` 下的 `docker` 环境下实现 `mihomo` 核心代理和自动订阅更新

## 项目地址
`https://github.com/pure-white-ice-cream/mihomo-sub`

## 快速部署
``` yml
services:
  mihomo:
    image: purewhiteicecream/mihomo-sub:latest
    container_name: mihomo-sub
    volumes:
      - /opt/mihomo-sub:/root/.config/mihomo
    environment:
      - "TZ=Asia/Shanghai"
      - "sub_url=https://这里换成你的订阅地址"
    ports:
     - "7890:7890"
     - "9090:9090"
```

## 分步解决, 原理同上面的项目

### docker compose
``` yml
services:
  # mihomo 核心
  mihomo:
    image: metacubex/mihomo:v1.19.15
    container_name: mihomo
    restart: always
    environment:
      - "TZ=Asia/Shanghai"
    ports:
     - "7890:7890"
     - "9090:9090"
    # 挂载卷根据自己喜好位置选择
    volumes:
      - "/opt/mihomo:/root/.config/mihomo"

  # 订阅转换工具
  subconverter:
    image: tindy2013/subconverter:0.9.0
    container_name: subconverter
    restart: always
    environment:
      - "TZ=Asia/Shanghai"
    ports:
     - "25500:25500"
```

### WEB 面板
- 下载最新面板: `https://github.com/MetaCubeX/metacubexd/releases`
- 将下载到的文件解压复制到 `/root/.config/mihomo/ui` 路径下, 保证存在 `/root/.config/mihomo/ui/index.html` 文件

### 定期更新订阅脚本
``` shell
#!/bin/bash

# 这里写你的订阅连接, 转换成https://www.urlencoder.org/中的格式
url=""

function end {
  log+="\n"
  printf "%b" "$log" >> $LOG_FILE
  exit 0
}

CONFIG_DIR="/opt/mihomo"
CONFIG_FILE="$CONFIG_DIR/config.yaml"
LOG_FILE="$CONFIG_DIR/log.txt"

output=""     # 保存生成的 config 内容
log=""        # 保存日志内容

output+="mixed-port: 7890\n"
output+="external-ui: /root/.config/mihomo/ui\n"

# 订阅文件更新
log+="[$(date -R)] \n\t订阅文件更新...\n\t"
sub_response=$(curl -s --max-time 15 -w "%{http_code}" -o /tmp/mihomo_temp.yml "http://127.0.0.1:25500/sub?target=clash&url=$url")
sub_exit_code=$?

if [ "$sub_exit_code" -ne 0 ]; then
  log+="Error❌️: 网络错误，退出码: $sub_exit_code\n\t"
  end
elif [ "$sub_response" -ne 200 ]; then
  log+="Error❌️: 订阅文件更新失败，响应码: $sub_response\n\t"
  end
fi

output+="$(awk 'NR>=3' /tmp/mihomo_temp.yml)""\n"
printf "%b" "$output" > $CONFIG_FILE
log+="订阅文件更新成功 ✅\n\t"

# 配置重新加载
log+="配置重新加载...\n\t"
reload_response=$(curl -s --max-time 15 -w "%{http_code}" -X PUT "http://127.0.0.1:9090/configs?force=true" -H "Content-Type: application/json" -d '{"path":"","payload":""}')
reload_exit_code=$?

if [ "$reload_exit_code" -ne 0 ]; then
  log+="Error❌️: 网络错误，退出码: $reload_exit_code\n\t"
  end
elif [ "$reload_response" -ne 204 ]; then
  log+="Error❌️: 配置重新加载失败，响应码: $reload_response\n\t"
  end
fi

log+="配置重新加载完成 ✅\n\t"
end
```

### 将脚本加入定时服务
- 编辑定时任务: `crontab -e`
``` shell
# 参考写法, 文件路径改成自己脚本的路径
# m h  dom mon dow   command
5 * * * * bash /opt/mihomo/sub.sh
```
