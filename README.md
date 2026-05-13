
# Home Assistant Mosquitto 桥接 EMQX 完整教程

## 一、最终实现效果

家庭局域网设备
        ↓
Home Assistant Mosquitto
        ↓ MQTT Bridge
云端 ECS EMQX
        ↓
远程 NodeRED / APP / 其他客户端

实现：

- 本地 MQTT 正常使用
- 云端统一 MQTT 接入
- Topic 双向同步
- Frigate / Traccar / Tasmota 等数据上云
- Home Assistant 本地设备不受影响

---

## 二、环境说明

### 本地环境

- Home Assistant OS
- Mosquitto Add-on
- 本地 MQTT Broker

例如：

192.168.1.111:1883

### 云端环境

- 阿里云 ECS
- EMQX Broker

例如：

59.100.111.222:1883

EMQX 用户：

用户名：mqtt
密码：123456

---

## 三、桥接原理

Mosquitto 主动连接 EMQX：

Mosquitto ---> EMQX

这是最推荐的方式。

优点：

- 不需要家庭公网 IP
- 不需要端口映射
- 更安全
- NAT 友好
- 更稳定

---

## 四、确认 Mosquitto Add-on 已运行

SSH 进入 HA：

docker ps | grep mosquitto

看到：

addon_core_mosquitto

表示运行正常。

---

## 五、找到正确的 share 目录（重要）

Home Assistant OS 中：

真正的共享目录是：

/mnt/data/supervisor/share

不是：

/share

---

## 六、创建桥接配置目录

执行：

mkdir -p /mnt/data/supervisor/share/mosquitto

---

## 七、创建 bridge.conf

创建：

vi /mnt/data/supervisor/share/mosquitto/bridge.conf

写入：

connection emqx

address 59.100.111.222:1883

remote_username mqtt

remote_password 123456

clientid ha_bridge

try_private false
start_type automatic
cleansession true
notifications false

topic # both 0

---

## 八、修改 Mosquitto 主配置

进入 Add-on：

docker exec -it addon_core_mosquitto sh

编辑：

vi /etc/mosquitto/mosquitto.conf

在最后增加：

include_dir /share/mosquitto

注意：

必须是：

include_dir /share/mosquitto

不要写：

include_dir /share

---

## 九、重启 Mosquitto

在 Home Assistant：

设置 → 插件 → Mosquitto broker → 重启

---

## 十、查看日志

执行：

docker logs -f addon_core_mosquitto

看到：

Loading config file /share/mosquitto/bridge.conf

Connecting bridge emqx (59.100.111.222:1883)

表示桥接配置已加载。

---

## 十一、验证 EMQX 是否已连接

进入 ECS：

emqx ctl clients list

看到：

Client(ha_bridge, username=mqtt ...)

表示桥接成功。

---

## 十二、测试 Topic 是否同步

例如：

EMQX 客户端发送：

USR-DR134/get

本地 Mosquitto 应该收到。

反之：

本地发送 Topic。

EMQX 也能收到。

---

## 十三、推荐的最终 bridge.conf

### 全量同步（简单）

connection emqx

address 59.100.111.222:1883

remote_username mqtt

remote_password 123456

clientid ha_bridge

try_private false
start_type automatic
cleansession true
notifications false

topic # both 0

---

## 十四、更推荐：仅同步指定 Topic

不要长期：

topic # both 0

推荐：

connection emqx

address 59.100.111.222:1883

remote_username mqtt

remote_password 123456

clientid ha_bridge

try_private false
start_type automatic
cleansession true
notifications false

topic frigate/# both 0
topic traccar/# both 0
topic office/# both 0

这样不会把：

- Home Assistant Discovery
- Zigbee
- ESPHome
- 内部 Topic

全部同步到云端。

---

## 十五、最终目录结构

宿主机：

/mnt/data/supervisor/share/
└── mosquitto/
    └── bridge.conf

容器内：

/share/mosquitto/bridge.conf

---

## 十六、最终网络结构（推荐）

局域网设备
    ↓
Mosquitto
    ↓
EMQX（云端）
    ↓
远程服务

而不是：

EMQX
    ↓
家庭 MQTT

因为：

- 家庭网络更安全
- 不需要暴露1883
- 不需要公网 IP

---

## 十七、安全建议（非常重要）

公网 MQTT 建议：

不要直接暴露：

1883

建议：

- 使用 TLS
- 使用 8883
- 配置 ACL
- 限制 Topic 权限

---

## 十八、常见坑（Q&A）

### Q1：为什么 include_dir /share/mosquitto 报错？

错误：

Unable to open include_dir '/share/mosquitto'

原因：

目录不存在。

解决：

mkdir -p /mnt/data/supervisor/share/mosquitto

因为：

宿主机：

/mnt/data/supervisor/share

会映射为：

/share

---

### Q2：为什么 topic 不同步？

最常见原因：

没有：

topic # both 0

或者：

topic xxx/# both 0

Mosquitto Bridge 必须明确声明同步哪些 Topic。

---

### Q3：为什么 EMQX 看不到连接？

检查：

emqx ctl clients list

如果没有：

ha_bridge

说明桥接未成功。

常见原因：

- 用户名密码错误
- 防火墙未开放1883
- bridge.conf 未加载
- include_dir 配置错误

---

### Q4：为什么 docker logs 一直重启？

一般是：

include_dir /share/mosquitto

目录不存在。

或者：

bridge.conf 配置语法错误。

---

### Q5：为什么 Telnet 通，但 MQTT 不通？

说明：

TCP 通 ≠ MQTT 认证成功

必须确认：

emqx ctl clients list

能看到客户端。

---

### Q6：为什么不要使用 topic # both 0？

因为会同步：

- Home Assistant 内部 Topic
- Discovery
- Zigbee2MQTT
- ESPHome
- 系统 Topic

可能：

- 占用大量带宽
- 暴露内部设备
- 形成 Topic 污染

生产环境建议只同步必要 Topic。

---

### Q7：EMQX 能不能反向连接 Mosquitto？

能。

但不推荐。

因为：

必须公网暴露家庭 MQTT：

1883

存在：

- 安全风险
- NAT 问题
- 动态 IP 问题

推荐：

Mosquitto 主动连接 EMQX

---

## 十九、最终成果

你现在已经实现：

✅ Home Assistant 本地 MQTT
✅ Mosquitto Bridge
✅ 云端 EMQX
✅ Topic 双向同步
✅ Frigate 上云
✅ Traccar 上云
✅ 本地设备不受影响
✅ 云端统一 MQTT 接入

已经是标准生产级 MQTT 架构。
