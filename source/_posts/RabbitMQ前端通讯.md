---
title: RabbitMQ前端通讯
date: 2025/10/22 12:00:00
---

# 简介

- `RabbitMQ` 默认通讯协议是 `AMQP` 只能与后端通讯
- `RabbitMQ` 与前端通讯需要使用自带的 `RabbitMQ Web STOMP 插件` 或 `RabbitMQ Web MQTT 插件`
- 官方文档 `https://rabbitmq.cn/docs/web-mqtt` (看英文原版把`.cn`改`.com`)
- 两个插件使用方法相似, 本篇教程使用更稳定的 `MQTT` 作为样例

# 启用脚本
执行以下命令:
``` shell
rabbitmq-plugins enable rabbitmq_web_mqtt
```

此命令等价于在 `/etc/rabbitmq/enabled_plugins` 中 在数组里新增 `rabbitmq_web_mqtt` 完成后文件内容看起来是这样:
```
[rabbitmq_management,rabbitmq_prometheus,rabbitmq_web_mqtt].

```

# 检测是否生效
执行以下命令查看所有插件:
``` shell
rabbitmq-plugins list
```

预期运行结果:
``` shell
root@my-rabbit:/# rabbitmq-plugins list
Listing plugins with pattern ".*" ...
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@my-rabbit
 |/
[  ] rabbitmq_amqp1_0                        4.1.4
[  ] rabbitmq_auth_backend_cache             4.1.4
[  ] rabbitmq_auth_backend_http              4.1.4
[  ] rabbitmq_auth_backend_internal_loopback 4.1.4
[  ] rabbitmq_auth_backend_ldap              4.1.4
[  ] rabbitmq_auth_backend_oauth2            4.1.4
[  ] rabbitmq_auth_mechanism_ssl             4.1.4
[  ] rabbitmq_consistent_hash_exchange       4.1.4
[  ] rabbitmq_event_exchange                 4.1.4
[  ] rabbitmq_federation                     4.1.4
[  ] rabbitmq_federation_management          4.1.4
[  ] rabbitmq_federation_prometheus          4.1.4
[  ] rabbitmq_jms_topic_exchange             4.1.4
[E*] rabbitmq_management                     4.1.4
[e*] rabbitmq_management_agent               4.1.4
[e*] rabbitmq_mqtt                           4.1.4
[  ] rabbitmq_peer_discovery_aws             4.1.4
[  ] rabbitmq_peer_discovery_common          4.1.4
[  ] rabbitmq_peer_discovery_consul          4.1.4
[  ] rabbitmq_peer_discovery_etcd            4.1.4
[  ] rabbitmq_peer_discovery_k8s             4.1.4
[E*] rabbitmq_prometheus                     4.1.4
[  ] rabbitmq_random_exchange                4.1.4
[  ] rabbitmq_recent_history_exchange        4.1.4
[  ] rabbitmq_sharding                       4.1.4
[  ] rabbitmq_shovel                         4.1.4
[  ] rabbitmq_shovel_management              4.1.4
[  ] rabbitmq_shovel_prometheus              4.1.4
[  ] rabbitmq_stomp                          4.1.4
[  ] rabbitmq_stream                         4.1.4
[  ] rabbitmq_stream_management              4.1.4
[  ] rabbitmq_top                            4.1.4
[  ] rabbitmq_tracing                        4.1.4
[  ] rabbitmq_trust_store                    4.1.4
[e*] rabbitmq_web_dispatch                   4.1.4
[E*] rabbitmq_web_mqtt                       4.1.4
[  ] rabbitmq_web_mqtt_examples              4.1.4
[  ] rabbitmq_web_stomp                      4.1.4
[  ] rabbitmq_web_stomp_examples             4.1.4
```

# 前端连接
前端需要引用 `mqtt` 包, 项目的 `package.json` 看起来像是这样:
``` json
{
  "name": "...",
  "version": "...",
  "dependencies": {
    "...": "...",
    "mqtt": "^5.14.1",
  },
}
```

示例代码 (`react`) (官方js示例详见`https://rabbitmq.cn/docs/web-mqtt#usage`):
``` js
import { useEffect, useRef, useState } from 'react'
import mqtt from 'mqtt'

const wsbroker = 'xxx.xxx.xxx.xxx' // 这里改成自己 RabbitMQ 的部署地址
const wsport = 15675

export default function MqPage() {
  const [inputValue, setInputValue] = useState('测试消息')
  const [res, setRes] = useState('')
  const clientRef = useRef<mqtt.MqttClient | null>(null)

  useEffect(() => {
    const url = `ws://${wsbroker}:${wsport}/ws`

    const options: mqtt.IClientOptions = {
      username: 'guest',
      password: 'guest',
      keepalive: 30,
      connectTimeout: 3000,
      clean: true
    }

    const client = mqtt.connect(url, options)
    clientRef.current = client

    client.on('connect', () => {
      setRes(prev => prev + '\nCONNECTED SUCCESS')
      client.subscribe('test')
    })

    client.on('reconnect', () => {
      setRes(prev => prev + '\nRECONNECTING...')
    })

    client.on('error', err => {
      setRes(prev => prev + '\nERROR ' + err.message)
    })

    client.on('close', () => {
      setRes(prev => prev + '\nCONNECTION LOST')
    })

    client.on('message', (topic, message) => {
      const payload = message.toString()
      setRes(prev => prev + `\n${topic}: ${payload}`)
    })

    return () => {
      client.end()
    }
  }, [])

  const sendMessage = () => {
    const client = clientRef.current
    if (!client) {
      setRes(prev => prev + '\nCLIENT NOT READY')
      return
    }

    client.publish('test', inputValue, err => {
      if (err) {
        setRes(prev => prev + '\nPUBLISH ERROR: ' + err.message)
      } else {
        setRes(prev => prev + '\nPUBLISH SUCCESS')
      }
    })
  }

  return (
    <>
      <div>消息队列测试</div>
      <input
        type='text'
        value={inputValue}
        onChange={e => {
          setInputValue(e.target.value)
        }}
      ></input>
      <button onClick={sendMessage}>发送消息</button>
      <pre>{res}</pre>
    </>
  )
}
```

# 扩展内容
## 虚拟主机 (详见 `https://rabbitmq.cn/docs/mqtt#virtual-hosts`)
- 虚拟主机作为用户名的一部分
- 连接时指定 `vhost` 的另一种更具体的方法是将 `vhost` 预先添加到用户名，并用冒号分隔。

- 例如，使用以下方式连接
```
/:guest
```
- 等效于默认 vhost 和用户名，而
```
mqtt-vhost:mqtt-username
```
- 表示连接到 `vhost mqtt-host`，用户名为 `mqtt-username`。

- 在用户名中指定虚拟主机优先于使用 `mqtt_port_to_vhost_mapping` 全局参数指定的端口到 `vhost` 的映射。

## 主题订阅的差异 (详见 `https://rabbitmq.cn/docs/mqtt#topic-level-separator-and-wildcards`)
- 主题级别分隔符和通配符
- MQTT 协议规范定义斜杠 (“/”) 为 主题级别分隔符，而 AMQP 0-9-1 定义点 (“.”) 为主题级别分隔符。此插件在后台转换模式以桥接两者。

- 例如，MQTT 主题 cities/london 变为 AMQP 0.9.1 主题 cities.london，反之亦然。因此，当 MQTT 客户端发布主题为 cities/london 的消息时，如果 AMQP 0.9.1 客户端想要接收该消息，则应从其队列创建到主题交换机的绑定，绑定键为 cities.london。

- 反之亦然，当 AMQP 0.9.1 客户端发布主题为 cities.london 的消息时，如果 MQTT 客户端想要接收该消息，则应创建主题过滤器为 cities/london 的 MQTT 订阅。

- 这有一个重要的限制：主题中包含点的 MQTT 主题将无法按预期工作，应避免使用，同样适用于包含斜杠的 AMQP 0-9-1 路由和绑定键。

- 此外，MQTT 定义加号 (“+”) 为 单级别通配符，而 AMQP 0.9.1 定义星号 (“*”) 以匹配单个单词

| MQTT | AMQP 0.9.1 | 描述 |
|------|------------|------|
| /    | .          | 主题级别分隔符 |
| +    | *          | 单级别通配符（匹配单个单词） |
| #    | #          | 多级别通配符（匹配零个或多个单词） |

## 编码
- `mqtt` 传输 `UTF-8` 的二进制格式, 后端用 `AMQP` 接收之后要转码

# RabbitMQ部署参考
``` yml
services:
  rabbitmq:
    image: rabbitmq:4.1.4-management
    container_name: rabbitmq
    hostname: my-rabbit
    restart: unless-stopped
    ports:
      - "5672:5672" # AMQP
      - "15672:15672" # Management UI
      - "15675:15675" # web_mqtt
    volumes:
      - ./conf:/etc/rabbitmq # 配置挂载
      - ./data:/var/lib/rabbitmq # 数据挂载
```