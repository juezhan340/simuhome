# 09：Matter 协议与 HA 适配机制

调研日期：2026-10-27

先纠正一个名称：

```text
你说的应该是 Matter，不是 Meta。

Matter 是 Connectivity Standards Alliance（CSA）主导的智能家居应用层协议，
原名 Project CHIP。
Meta 不是这套智能家居协议的名字。
```

## 结论

本机同时有三层 Matter 相关内容：

```text
HA Matter 集成
  ha_integration_sources/01_matter_core
  ha_core_official/core/homeassistant/components/matter

HA 使用的 Matter 客户端
  matter-python-client 0.7.1
  Matter Server WebSocket 客户端

Matter 参考实现与数据模型
  D:\D_coderesource\morelearn\simuprocject\external_repos\connectedhomeip
```

本地 `connectedhomeip` 不是简单的 HA 插件，而是 Matter 的官方开源参考实现，包含：

```text
协议栈
Controller / Device 实现
Cluster 定义
Device Type 定义
Attribute / Command / Event 类型
代码生成链
测试工具 chip-tool
设备示例
```

当前克隆对应：

```text
repository  https://github.com/project-chip/connectedhomeip.git
commit      ace3ccce
Matter 1.6 数据模型
  92 个 Device Type XML
  127 个 Cluster XML
  28 个 Namespace XML
```

这里的“完整”指：

```text
可运行的参考实现
+ 机器可读的标准数据模型
+ 代码生成与测试工具
```

它不等于 CSA 发布的完整规范文档原文。协议规范、SDK 实现和数据模型 XML 是不同形态的资料。

## 1 Matter 的层级

```text
Node
  一个可独立寻址的 Matter 设备或桥接设备

Endpoint
  Node 内部的功能端点
  一个冰箱可以有主端点、冷藏室端点、冷冻室端点

Device Type
  Endpoint 的语义类型
  例如 Refrigerator、Temperature Controlled Cabinet、On/Off Light

Cluster
  Device Type 组合的标准功能块
  例如 OnOff、TemperatureControl、TemperatureMeasurement

Attribute
  Cluster 的可读/可写状态

Command
  Cluster 的动作

Event
  Cluster 产生的事件

Feature
  Cluster 内部可选能力位
```

```text
Device Type 规定“必须/可选包含哪些 Cluster”
Cluster 规定“有哪些 Attribute/Command/Event/Feature”
设备实例通过 Endpoint 选择并组合这些标准对象
```

## 2 为什么 Matter 的设计更整齐

Matter 把三类东西分开：

```text
实例身份
  Node ID、Endpoint ID、Fabric

设备语义
  Device Type

能力语义
  Cluster / Attribute / Command / Event / Feature
```

同类设备共享相同 Device Type 和 Cluster 契约，不会出现“某个厂商把属性叫这个名字，另一个厂商再发明一套”的情况。

规范还支持：

```text
mandatory
  必须实现

optional
  可选实现

conditional
  满足某条件时必须实现

disallow
  当前设备类型中禁止实现
```

因此 Device Type XML 能表达：

```text
某设备至少要实现哪些能力
哪些能力可选
哪些 feature 组合互斥
哪个 revision 后增加什么能力
```

## 3 冰箱在 Matter 中怎么表达

本地 Matter 1.6 数据模型直接包含：

```text
data_model/1.6/device_types/Refrigerator.xml
  Device Type 0x0070

data_model/1.6/device_types/TemperatureControlledCabinet.xml
  Device Type 0x0071

data_model/1.6/clusters/Mode_Refrigerator.xml
  Cluster 0x0052

data_model/1.6/clusters/RefrigeratorAlarm.xml
  Cluster 0x0057

examples/refrigerator-app/
  官方的冰箱与温控柜示例
```

### 3.1 冰箱由温控柜组成

`Refrigerator` Device Type：

```text
id: 0x0070
class: simple
scope: endpoint

条件要求
  Temperature Controlled Cabinet 0x0071
  condition = Cooler
  mandatory
```

它表达的是：

```text
一个 Refrigerator 端点包含具有制冷能力的温控柜
```

`Temperature Controlled Cabinet` 还可以用于冷藏、加热柜、烤箱腔体等不同场景，通过 Cooler / Heater 条件区分。

### 3.2 温控柜的 Cluster

`Temperature Controlled Cabinet` 的关键能力：

```text
Temperature Control 0x0056
  mandatory
  feature TN mandatory

Temperature Measurement 0x0402
  optional

Refrigerator And Temperature Controlled Cabinet Mode 0x0052
  Cooler 条件下 optional

Temperature Alarm 0x0064
  provisional / optional

Oven Cavity Operational State 0x0048
  Heater 条件下 optional

Oven Mode 0x0049
  Heater 条件下 optional
```

`Temperature Control` 的 TN feature 表示基于目标数值的控制，另外还有 TL 表示离散温度档位。Matter 1.6 的 TCC 当前强制使用 TN。

### 3.3 冰箱专用 Cluster

`Refrigerator` 端点还可以包含：

```text
Refrigerator And Temperature Controlled Cabinet Mode 0x0052
  CurrentMode
  SupportedModes

Refrigerator Alarm 0x0057
  State
  DoorOpen 等 bitmap 位
  DoorStateChange 等事件

Activated Carbon Filter Monitoring 0x0072
  滤芯寿命等能力
```

这比“编号属性 + 厂商自定义字段”稳定得多。

## 4 HA 如何把 Matter 映射成 entity

HA Matter 集成并不自己实现完整 Matter 协议栈。它通过 `matter-python-client` 连接 Matter Server。

```text
HA Matter Integration
  ↓  Matter Server WebSocket
Matter Server
  ↓  Matter 协议
Matter Node / Thread / Wi-Fi
```

HA 集成根据 Matter Cluster 和 Attribute 建 entity。

### 4.1 冰箱实例

```text
Matter Refrigerator / Temperature Controlled Cabinet
  ├─ TemperatureControl.TemperatureSetpoint
  │    → number.temperature_setpoint
  │
  ├─ TemperatureControl.SetTemperature
  │    → number 写操作执行 Matter command
  │
  ├─ TemperatureMeasurement.MeasuredValue
  │    → sensor.temperature
  │
  ├─ RefrigeratorAndTemperatureControlledCabinetMode
  │    → select.mode
  │
  ├─ RefrigeratorAlarm.State & DoorOpen
  │    → binary_sensor.alarm_door
  │
  └─ OnOff.OnOff
       → switch
```

HA 的实际映射代码：

```text
number.py
  TemperatureControlTemperatureSetpoint
  SetTemperature
  MinTemperature / MaxTemperature

select.py
  MatterRefrigeratorAndTemperatureControlledCabinetMode
  CurrentMode / SupportedModes

binary_sensor.py
  RefrigeratorAlarmDoorOpen
  State & AlarmBitmap.kDoorOpen

switch.py
  OnOff.OnOff
  Refrigerator Device Type
```

HA 把 Matter 的正式数据模型翻译成自己的 entity domain。

这一步仍然会丢失部分原始分组：

```text
Matter Endpoint / Device Type / Cluster
  ↓
HA number / sensor / select / switch / binary_sensor
```

设备语义清楚，但 HA entity 视图仍是扁平化的。

## 5 与 Xiaomi MIoT 的对比

| 维度 | Matter | Xiaomi MIoT |
|---|---|---|
| 设备类型 | 标准 Device Type | MIoT `urn` 设备类型 |
| 能力块 | 标准 Cluster | MIoT service/property/action/event |
| 能力键 | Cluster ID + Attribute/Command/Event ID | SIID / PIID / AIID |
| 设备组合 | Node → Endpoint → Device Type | Device → Service → Instance |
| 版本 | Device/Cluster revision + feature | MIoT SPEC 版本 |
| 设备差异处理 | 标准 cluster + vendor extensions | 云端 SPEC + 型号修补 |
| HA 映射 | 按 Matter cluster | 按 MIoT SPEC 属性形态 |
| 语义完整性 | 强，标准化 | 中，依赖 SPEC 与厂商命名 |

两边都采用类似思路：

```text
设备类型
  + 能力块
  + 状态字段
  + 动作
  + 事件
```

Matter 的标准化和约束更严格，MIoT 的覆盖更贴近小米现有产品，但后者更依赖云端 SPEC 和型号补丁。

## 6 本地完整协议的边界

当前本机具备：

```text
HA Matter 适配源码
matter-python-client
完整的 connectedhomeip 参考实现
Matter 1.6 的 Device Type / Cluster XML
Matter 设备示例与 chip-tool
```

当前本机没有运行：

```text
独立的 Matter Server 容器
```

HA 容器里可见 `matter_server` 的客户端和公共模型代码，但当前 Docker 只有 Home Assistant 容器，没有独立的 Matter Server 容器。HA 的 Matter 集成需要连接一个 Matter Server 才能实际控制 Matter 设备。

```text
完整实现代码存在
不表示 Matter Server 当前已经运行
```

## 7 对语义 MCP 的启示

Matter 是很好的标准语义层范例：

```text
Device Type
  固定设备语义

Cluster
  标准能力块

Attribute / Command / Event
  标准读写与事件接口

Feature
  同一 Cluster 内的能力细分

Revision
  向后兼容与能力演进
```

我们的智能家居语义 MCP 可以采用类似结构：

```text
Device Profile
  空调 / 冰箱 / 灯

Capability
  温度控制 / 开关 / 模式 / 门状态

Property
  当前值、目标值、单位、枚举

Action
  set_temperature / turn_on / set_mode

Event
  door_open / fault

Feature
  是否支持风速、摇摆、速冷等
```

模型可见视图仍然是压缩后的：

```text
冰箱
  冷藏当前温度
  冷藏目标温度
  冷冻当前温度
  冷冻目标温度
  模式
  门状态

完整 Matter/MIoT 能力留在后台
```

Matter 解决的是协议与设备能力标准化。给 LLM 的低 token profile 仍需在 Host 或语义 MCP 层做二次投影。

## 8 本地资料位置

```text
HA Matter 集成
  ha_integration_sources/01_matter_core/

HA Core Matter 集成
  ha_core_official/core/homeassistant/components/matter/

Matter 参考实现
  D:/D_coderesource/morelearn/simuprocject/external_repos/connectedhomeip/

已有对照文档
  D:/D_coderesource/morelearn/simuprocject/external_repos/connectedhomeip/
    SimuHome对照_Matter官方仓库实现梳理.md

冰箱 Device Type
  connectedhomeip/data_model/1.6/device_types/Refrigerator.xml

温控柜 Device Type
  connectedhomeip/data_model/1.6/device_types/TemperatureControlledCabinet.xml

冰箱模式 Cluster
  connectedhomeip/data_model/1.6/clusters/Mode_Refrigerator.xml

冰箱告警 Cluster
  connectedhomeip/data_model/1.6/clusters/RefrigeratorAlarm.xml
```

当前结论基于本地源码与数据模型，没有在当前环境启动 Matter Server 做真实设备配网与端到端控制。
