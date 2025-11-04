---
title: mihomo核心在docker部署教程
date: 2025/11/04 12:00:00
---

# 简介

- `mihomo` 核心 (原 `clash.meta`) 官方文档: `https://wiki.metacubex.one/startup/`
- 常用客户端里, 缺少 `linux` 服务器的 `docker` 部署的使用方案
- 本篇教程在 `ubuntu` 下的 `docker` 环境下实现 `mihomo` 核心代理和自动订阅更新

## docker compose
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

## WEB 面板
- 下载最新面板: `https://github.com/MetaCubeX/metacubexd/releases`
- 将下载到的文件解压复制到 `/root/.config/mihomo/ui` 路径下, 保证存在 `/root/.config/mihomo/ui/index.html` 文件

## 定期更新订阅脚本
``` shell
# 输出当前时间
date -R

{
  # 配置端口和面板位置
  echo "mixed-port: 7890"
  echo "external-ui: /root/.config/mihomo/ui"
  
  # 执行curl请求并检查是否成功
  response=$(curl -s -w "%{http_code}" -o /tmp/curl_output "http://127.0.0.1:25500/sub?target=clash&url=这里写你的订阅连接, 转换成https://www.urlencoder.org/中的格式")
  
  # 如果请求失败，退出并报错
  if [[ "$response" -ne 200 ]]; then
    echo "Error: Request failed with status code $response"
    exit 1
  fi

  # 如果请求成功，继续处理数据, 这里截断了前三行, 根据自己订阅结果写规则
  awk 'NR>=3' /tmp/curl_output
} > /opt/mihomo/config.yaml

# 让核心重载订阅文件
curl -X PUT "http://127.0.0.1:9090/configs?force=true" \
     -H "Content-Type: application/json" \
     -d '{"path":"","payload":""}'

echo "订阅更新成功"
```

## 将脚本加入定时服务
- 编辑定时任务: `crontab -e`
``` shell
# 参考写法, 文件路径改成自己脚本的路径
# m h  dom mon dow   command
5 * * * * bash /opt/mihomo/sub.sh >> /opt/mihomo/log.txt
```
