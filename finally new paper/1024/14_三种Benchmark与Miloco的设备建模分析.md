# 14：三种 Benchmark 与 Miloco 的设备建模分析

核验日期：2026-10-29

本文分析三条论文路线和一套真实生态实现：

```text
PersonalHomeBench
SMH-Bench
SimuHome
Miloco / Xiaomi MIoT
```

不讨论如何迁移到其他系统。目标是回答：

```text
论文怎样表示设备
怎样表示能力和状态
怎样定义动作与参数
附录里有没有具体实例
最后有没有真实接入物理设备
```

## 0 结论

| 项目 | 建模核心 | 附录示例 | 是否实际接入设备 |
|---|---|---|---|
| PersonalHomeBench | Home/Appliance/Sensor + feature 工具 + plan | 附录 F/J：家电 feature 列表、工具调用、计划生成 JSON | 使用 `PersonalHomeTools` 控制沙箱；本地论文未见真实物理设备端到端实验 |
| SMH-Bench | HomeEnv 的 Room/Device/Attribute/Service/Component | 附录 A.7：立式空调和风扇灯的完整 JSON | 执行对象是 HomeEnv 仿真器；论文没有报告真实设备接入 |
| SimuHome | Matter Device/Endpoint/Cluster/Attribute/Command | 附录 A.2-A.4：13 个工具、Cluster 表、17 类设备与 Cluster 组合 | 是 Matter 对齐仿真；论文提出可较少适配迁移到真实设备，但本地材料未报告真机实验 |
| Miloco | MIoT Device/Service/Property/Action/Event；再压缩为 lite spec 和 catalog | `types_default.yaml` 的空调必选属性、`MIoTSpecDeviceLite`、catalog TSV | 代码路径面向真实小米 MIoT 设备；是否有当前账号设备取决于本地服务与登录状态 |

四个对象的“现实性”侧重点不同：

```text
PersonalHomeBench
  更接近真实厂商 feature 接口

SMH-Bench
  更接近一套完整的 typed device-state schema

SimuHome
  更接近真实 Matter 协议语义

Miloco
  以小米 MIoT-SPEC 为语义源
  并把同一模型压缩成模型可读的 lite spec 和 catalog
```

三个 benchmark 都不是实际家庭真机运行报告。Miloco 面向真实小米设备，但本文当前只核对本地源码中的建模和校验链路，是否已有可用的当前账号设备仍需看本地服务状态。

## 1 PersonalHomeBench

### 1.1 论文怎样建模

PersonalHomeBench 附录 G 的家庭 schema 是：

```text
Home
├─ household_id
├─ household_size
├─ occupancy_type
├─ layout
├─ pets[]
├─ members[]
├─ appliances[]
│   ├─ appliance: ApplianceObject
│   └─ location
├─ sensors[]
│   ├─ sensor: SensorObject
│   └─ location
├─ memories[]
└─ background
    ├─ events[]
    └─ weather
```

字段含义：

| 字段 | 作用 |
|---|---|
| `household_id` | 家庭 ID |
| `household_size` | 家庭人数 |
| `occupancy_type` | 居住类型 |
| `layout` | 家庭布局的文本描述 |
| `pets` | 宠物信息 |
| `members` | 家庭成员角色、年龄、职业、偏好和重大事件 |
| `appliances` | 家电对象及其位置 |
| `sensors` | 传感器对象及其位置 |
| `memories` | 家庭级或成员级长期记忆 |
| `background` | 当前事件和室内外天气等背景 |

设备部分的实际结构是：

```text
appliances: [
  {
    appliance: ApplianceObject,
    location: string
  }
]

sensors: [
  {
    sensor: SensorObject,
    location: string
  }
]
```

本地可读文本只明确写出 `ApplianceObject` 和 `SensorObject` 这两个引用，没有完整展开它们的所有字段。能确认的设备字段来自工具接口：

```text
设备名称
所在位置
当前状态
自然语言描述
可用 features
feature 参数
最近事件
```

因此 PersonalHomeBench 的核心不是先定义一套统一的 Device/Attribute/Service 树，而是：

```text
家庭对象
  → 家电对象
     → 查询 feature
        → 调用 feature
```

论文没有把设备控制压成一个通用的 `turn_on()`。它使用 feature 级接口：

```text
get_owned_appliances()
  列出家中设备

get_appliances(appliance_names)
  查设备当前状态

get_appliance_features(appliance_names)
  查该设备有哪些 feature 和参数

run_appliance_feature(
  appliance_name,
  feature_name,
  feature_kwargs
)
  执行一个具体 feature
```

这意味着模型必须先知道“是哪台家电”，再找到“这个家电支持哪些 feature”，最后填写对应参数。

与 SMH-Bench 的结构对比：

```text
SMH-Bench
  Device 内直接展开 attributes / services / components

PersonalHomeBench
  ApplianceObject 的具体能力通过
  get_appliance_features()
  在运行时查询

PersonalHomeBench
  本地文本没有展示显式的 component namespace
  设备能力主要通过 appliance + feature 两级定位
```

### 1.2 附录里的 feature 示例

本地 PDF 的文字化附录列出了若干真实产品功能。以冰箱为例：

```text
冰箱内部摄像头远程查看
触屏家庭中心界面
食材管理与过期跟踪
食谱推荐
诊断警报
双冰制冰机
```

这些不是抽象的 `set_temperature`，而是具体产品功能。比如食材管理可以生成购物清单，诊断警报会在门未关时通知用户，双冰制冰机可以切换冰块、碎冰或球形冰。

烤箱的 feature 示例包括：

```text
远程预加热
内部 AI 摄像头
烹饪模式切换
肉探针温度读取与目标温度警报
```

电视的 feature 示例包括：

```text
跨应用内容搜索
免提语音控制
智能家居仪表盘覆盖
环境艺术模式
多视图/分屏
图像与声音校准
```

论文说明这些设备控制接口参考了 LG、Samsung 等主要智能家居供应商的公开文档，并为 40 多种智能家电实现了 feature 级接口。

### 1.3 附录里的计划 JSON

附录 J 给出的设备操作格式类似：

```json
[
  {
    "program_id": "secure_exit_and_music",
    "appliance": "garage door opener",
    "feature": "operate_door",
    "category": ["wellbeing", "exception"],
    "args": {
      "action": "close"
    }
  },
  {
    "program_id": "secure_exit_and_music",
    "appliance": "speaker",
    "feature": "complex_query",
    "category": ["routine"],
    "args": {
      "query_text": "play relaxing lo-fi music at volume 6",
      "context_history": true
    }
  }
]
```

这里能看出它的计划结构：

```text
program_id
  一个连续用户意图

appliance
  目标设备

feature
  设备能力

category
  任务性质

args
  能力参数
```

论文还限制：

```text
一个程序包含 1–5 个步骤
同一意图使用相同 program_id
只有互不冲突的动作才能放同一回合
视觉通知与媒体播放不能同时争用同一注意力
安全警报不能和娱乐动作同时执行
```

### 1.4 约束怎样检查

评测中的 `Validity` 定义：

```text
tool 名称有效
feature 名称有效
参数格式正确
```

因此它主要验证模型能否：

```text
发现正确设备
选择正确 feature
填写正确参数
形成结构化计划
```

### 1.5 是否实际接入设备

论文描述的是：

```text
基于厂商公开文档
构建 40+ 家电的 feature-level interface
放入 PersonalHomeTools 控制沙箱
```

本地材料可以确认：

```text
接口命名和 feature 含义有厂商文档依据
执行层是工具沙箱
```

本地材料没有报告：

```text
真实 LG / Samsung 家庭中的全量端到端执行
真实云端限流、掉线、鉴权和设备落定时间
同一 feature 在不同国家和固件中的兼容性
```

结论：

```text
接口参考性较强
物理设备执行证据不足
```

## 2 SMH-Bench

### 2.1 论文怎样建模

SMH-Bench 使用自定义 HomeEnv：

```text
Home
├─ rooms[]
└─ devices[]
     ├─ name
     ├─ description
     ├─ attributes[]
     ├─ services[]
     ├─ components[]
     └─ userdata
```

六个字段分别承担不同职责：

| 字段 | 作用 | 例子 | 容易混淆的点 |
|---|---|---|---|
| `name` | 给人看的设备实例名称 | `floor-standing AC`、`fan light` | 它不是稳定 ID，不能拿来代替 `did` |
| `description` | 自然语言描述，供模型理解和检索 | `current power state...` 或整台设备的用途说明 | 它可以有语义价值，但不能承担参数校验和状态迁移 |
| `attributes` | 当前状态变量 | `state=off`、`target_temperature=18`、`ac_mode=heat` | 它保存值，不直接执行动作；目标温度可以是可写属性的当前值 |
| `services` | 经过校验的状态转换 | `turn_on`、`set_target_temperature` | 它是动作入口，参数和状态修改逻辑都在这里 |
| `components` | 同一物理设备内部的功能命名空间 | `light`、`fan` | 它不是独立房间里的独立设备，也不单独拥有 `did` 和 `room` |
| `userdata` | 设备的静态元数据 | `did`、`spid`、`category`、`room` | 名字虽然叫 userdata，但这里不是用户画像或隐私资料 |

最重要的区分：

```text
attributes
  回答“设备现在是什么状态”

services
  回答“允许把状态改成什么”

components
  回答“这个动作作用于同一台设备的哪个功能”

userdata
  回答“这台设备是谁、属于哪类、安装在哪里”
```

例如一台单功能立式空调：

```text
name        floor-standing AC
description floor-standing AC
attributes  state / target_temperature / ac_mode
services    turn_on / set_target_temperature
components  []
userdata
  did       1693
  spid      100011197
  category  ac
  room      10
```

因为空调只有一个主功能面，所以 `components` 为空，属性和服务直接挂在 Device 上。

再看风扇灯：

```text
name        fan light
description fan light
attributes  []
services    []
components
  light
    attributes
      state
      brightness
    services
      turn_on
      set_brightness

  fan
    attributes
      state
      speed_level
    services
      turn_on
      set_speed_level

userdata
  did       3460
  spid      20000010
  category  fan_light
  room      25
```

这里是：

```text
1 个物理设备
1 个 did
1 个 room
2 个 components
   light.* 只能控制灯
   fan.*   只能控制风扇
```

如果动作只写 `turn_on`，无法知道用户要开灯还是开风扇。因此必须写成：

```text
light.turn_on
fan.turn_on
```

`userdata` 中的字段分别表示：

```text
did
  设备实例 ID

spid
  共享同一能力 schema 的 Product Specification ID

category / subcategory
  设备类别

tags
  检索标签

room
  所在房间
```

其中：

```text
did
  面向“这一台”

spid
  面向“这一类”

category / subcategory
  面向设备分类和检索

tags
  面向语义搜索

room
  面向空间定位
```

同一型号的两台空调可以有不同 `did` 和不同 `room`，但共享同一 `spid`。这也是 HomeEnv 把设备实例、设备类型和房间关联分开的关键。

`attributes` 的字段结构：

```text
name
description
type
value
range
options
items
precision
unit
```

`services` 的字段结构：

```text
name
description
arguments
code
```

### 2.2 附录 A.7：立式空调示例

附录给出的空调实例包含三个主要属性：

```json
{
  "name": "floor-standing AC",
  "attributes": [
    {
      "name": "state",
      "type": "str",
      "options": ["on", "off"],
      "value": "off"
    },
    {
      "name": "target_temperature",
      "type": "float",
      "range": [18, 30],
      "precision": 1,
      "value": 18,
      "unit": "celsius"
    },
    {
      "name": "ac_mode",
      "type": "str",
      "options": ["heat", "cool", "auto", "dry", "fan_only"],
      "value": "heat"
    }
  ]
}
```

服务包括：

```text
turn_on
  无参数
  code: self.state = "on"

set_target_temperature
  argument:
    target_temperature
    float
    range 18–30
    precision 1
    unit celsius
  code: self.target_temperature = target_temperature
```

设备元数据：

```text
did     1693
spid    100011197
category ac
subcategory floor_ac
room     10
```

这个例子里：

```text
state
  只读或可写取决于 service 是否修改它

target_temperature
  由一个带参数的 service 修改

ac_mode
  有一个明确的枚举集合

room
  通过 userdata.room 与房间关联
```

### 2.3 附录 A.7：风扇灯示例

同一个附录还给出多组件设备：

```text
fan_light
├─ light
│   ├─ state
│   ├─ brightness [1, 100]
│   ├─ turn_on
│   └─ set_brightness
└─ fan
    ├─ state
    ├─ speed_level [low, medium, high]
    ├─ turn_on
    └─ set_speed_level
```

这解决了物理设备与功能命名冲突：

```text
device did = 3460
  light.turn_on
  fan.turn_on
```

同一个 `turn_on` 必须通过 component namespace 区分目标。

### 2.4 约束怎样执行

HomeEnv 在执行 Service 前验证：

```text
服务是否存在
参数名是否存在
参数类型是否正确
数值是否在范围内
值是否属于 options
精度和单位是否符合
```

失败时：

```text
调用被拒绝
家庭状态不变
```

任务执行后还要检查：

```text
目标设备是否达到目标状态
未被任务涉及的设备是否保持不变
```

### 2.5 是否为美的真实系统

本地 SMH-Bench 论文作者单位包括：

```text
Midea Group
Beijing University of Posts and Telecommunications
Donghua University
The University of Sydney
Peking University
```

但本地论文没有说明 HomeEnv 的设备规格来自：

```text
美的真实产品目录
米的 MIoT 定义
美的生产 API
真实用户家庭设备
```

现有一致证据只支持：

```text
论文由美的研究人员参与
HomeEnv 是论文自行设计的可执行 schema
```

### 2.6 是否实际接入设备

没有。

论文中的动作由 HomeEnv 服务规则执行，状态也在 HomeEnv 内更新。它不是 HA、Matter Server 或美的真实设备的端到端控制实验。

## 3 SimuHome

### 3.1 论文怎样建模

SimuHome 明确以 Matter 为基础：

```text
Device
  → Endpoint
     → Cluster
        → Attribute
        → Command
        → Event
```

环境状态写成：

```text
s_t = <Matter device state, environment state>
```

其中：

```text
Matter device state
  endpoints + clusters + attributes

environment state
  temperature / illuminance / humidity / air quality
```

动作是 Matter Command：

```text
device_id
endpoint_id
cluster_id
command_id
arguments
```

这五层分别是什么：

| 层级 | 含义 | 例子 |
|---|---|---|
| Device | 可独立寻址的物理设备或逻辑设备集合 | 一台空调、一台冰箱 |
| Endpoint | 设备内部的功能端点或分区 | 空调控制端点；冰箱的冷藏、冷冻温控柜 |
| Cluster | 挂在 Endpoint 上的一组标准能力 | On/Off、Thermostat、Fan Control、Temperature Control |
| Attribute | Cluster 内的状态字段 | `OnOff`、`LocalTemperature`、`PercentCurrent` |
| Command | Cluster 内的动作入口 | `On`、`Off`、`SetTemperature` |
| Event | Cluster 产生的状态事件 | 运行完成、异常告警等 |

Device 的元数据来自 Matter Basic Information 和 Descriptor：

```text
Basic Information
  vendor name
  vendor id
  product name
  product id

Descriptor
  device type list
  server cluster list
  client cluster list
  parts list
  tags
```

这里的 Descriptor 与 SMH-Bench 的 `userdata` 有相似作用，但 Matter 使用标准字段：

```text
vendor / product
  厂商和产品身份

device type list
  该端点是什么设备

server / client cluster list
  端点提供和依赖哪些能力

parts list
  端点之间的组合关系

tags
  语义标签
```

因此 SimuHome 的设备元数据不是自造的 `did/spid/category/room`，而是 Matter 的标准信息加仿真器自己的 ID/房间查询。

### 3.2 附录 A.2：工具如何暴露模型

SimuHome 的 13 个工具包括：

```text
get_device_structure
  查看 Endpoint / Cluster / Attribute / Command

get_attribute
  读取一个 Attribute

get_all_attributes
  读取设备的全部 Attribute

write_attribute
  写 Attribute

execute_command
  执行 Command

get_cluster_docs
  语义检索 Matter Cluster 文档

get_rooms
get_room_devices
get_room_states
schedule_workflow
get_current_time
get_workflows
finish
```

执行工具的必填参数：

```text
execute_command
  device_id
  endpoint_id
  cluster_id
  command_id
  arguments

write_attribute
  device_id
  endpoint_id
  cluster_id
  attribute_id
  value
```

按论文中的参数形式，一次空调操作可以写成：

```text
get_device_structure(
  device_id="ac_01"
)

execute_command(
  device_id="ac_01",
  endpoint_id=1,
  cluster_id="OnOff",
  command_id="On",
  arguments={}
)

write_attribute(
  device_id="ac_01",
  endpoint_id=1,
  cluster_id="Thermostat",
  attribute_id="OccupiedCoolingSetpoint",
  value=2500
)
```

最后一项中的 `2500` 对应 Matter 温度编码的 `25.00°C`。这里展示的是层级关系；实际 unit、min/max 和可写性仍要读取 Attribute 与 Cluster 约束。

### 3.3 附录 A.4：空调设备类型示例

SimuHome 的空调由以下 Matter Cluster 组成：

```text
Basic Information
Fan Control
On/Off
Thermostat
```

不是：

```text
AirConditioner
  set_temperature()
  set_fan_speed()
```

而是：

```text
Air Conditioner Device Type
├─ On/Off Cluster
├─ Thermostat Cluster
│   ├─ LocalTemperature
│   ├─ OccupiedCoolingSetpoint
│   ├─ OccupiedHeatingSetpoint
│   └─ SystemMode
└─ Fan Control Cluster
    ├─ FanMode
    ├─ PercentSetting
    ├─ PercentCurrent
    └─ RockSetting
```

附录 A.3 还列出每个 Cluster 的 Attribute 和 Command。

### 3.4 约束怎样执行

约束来自三层：

```text
Matter 协议
  Device Type 的 required / optional Cluster
  Cluster 的 Attribute / Command / Feature

SimuHome 设备规则
  例如电源未开启时不能调风速

时间与环境规约
  动作持续时间、环境变化、任务截止时间
```

不可行任务同样被纳入 benchmark：

```text
设备存在但操作无法达到目标
时间窗口无法满足
模式或前置条件冲突
```

### 3.5 是否实际接入设备

SimuHome 是仿真器。

论文确实提出：

```text
与 Matter 对齐
训练和验证过的 agent 可以用较少适配运行在真实 Matter 设备上
```

但本地论文和附录没有报告：

```text
研究团队用多少种真实 Matter 物理设备做了端到端实验
真实设备与仿真 Cluster/Attribute 的差异
真实配网、权限和网络故障的结果
```

因此应区分：

```text
协议模型真实度较高
实际硬件执行未被本地材料证明
```

## 4 Miloco：从 MIoT-SPEC 到模型目录

Miloco 不能只用“一个 MCP”概括。它的设备语义至少经过三层：

```text
L1 小米 MIoT-SPEC
  设备、服务、属性、动作、事件、访问权限、范围、枚举

L2 MIoT lite spec
  把原始 SPEC 扁平化为 prop.0.siid.piid / action.0.siid.aiid

L3 device catalog
  再把多台设备和高频能力压缩成模型可读的 TSV 上下文
```

三者的职责：

| 层 | 是否语义来源 | 作用 |
|---|---|---|
| MIoT-SPEC | 是 | 定义小米设备的真实能力结构 |
| MIoT lite spec | 派生 | 把 SPEC 整理成模型和 CLI 容易消费的平表 |
| device catalog | 派生 | 做设备数、能力数、白名单和 token 预算压缩 |

### 4.1 第一层：小米 MIoT-SPEC

源码：

```text
backend/miot/src/miot/spec.py
backend/miot/types_default.yaml
```

核心对象：

```text
MIoTSpecDevice
  urn
  name
  description
  services[]

MIoTSpecService
  iid
  type
  description
  type_level
  proprietary
  need_filter
  properties[]
  actions[]
  events[]

MIoTSpecProperty
  format
  access
  unit
  value_range
  value_list

MIoTSpecAction
  in[]
  out[]

MIoTSpecEvent
  arguments[]
```

其中 `type_level` 明确分为：

```text
UNKNOWN
OPTIONAL
REQUIRED
CUSTOM
```

属性访问权限来自：

```text
read
write
notify
```

参数可以有：

```text
format
value_range
value_list
unit
```

以 `types_default.yaml` 的空调标准服务为例：

```text
air-conditioner service
  required properties:
    on
    mode
    target-temperature

  optional properties:
    target-humidity
    eco
    anion
    heater
    dryer
    sleep-mode
    soft-wind
    uv
    humidity-switch
    ...
```

在设备级还可以看到：

```text
air-conditioner device
  optional services:
    air-conditioner
    fan-control
    environment
    alarm
    countdown
    filter
    auto-clean
    air-fresh
    power-consumption
    ...
```

因此小米的约束不是只有一组扁平字段，而是：

```text
Device Type
  → required / optional Services
     → required / optional Properties、Actions、Events
```

这和 Matter 的“Device Type → Cluster → Feature/Attribute/Command”思路非常接近。

实际设备会按自己的 URN 从 MIoT-SPEC 获取规格，并缓存解析结果。标准库和解析器可以在源码中看到；具体某台设备的完整实例参数仍取决于当前小米账号返回的设备 URN。

### 4.2 第二层：MIoT lite spec

`MIoTSpecParser.parse_lite_async()` 把层级 SPEC 压平为：

```text
prop.0.<siid>.<piid>
action.0.<siid>.<aiid>
```

每个条目保留：

```text
description
format
writeable
readable
unit
value_range
value_list
type_name
service_type_name
service_description
in_params
prop_description
```

一个空调属性最终会类似：

```text
{
  "iid": "prop.0.<siid>.<piid>",
  "type_name": "target-temperature",
  "service_type_name": "air-conditioner",
  "format": "float",
  "writeable": true,
  "readable": true,
  "value_range": {"min": 16, "max": 30, "step": 1},
  "unit": "celsius"
}
```

这一层适合模型直接阅读，但它已经发生了信息损失：

```text
补入了 readable / writeable
展开了 value_range / value_list
供 MCP 使用

没有保留 type_level
```

也就是说，源码和标准库能区分 required / optional / custom，但模型拿到的 `MIoTSpecDeviceLite` 默认不再显示这个标记，required 与 optional 项会混在同一张 lite spec 里。

### 4.3 第三层：device catalog

当前 OpenClaw/CLI 路径又对 lite spec 做了一次面向上下文的压缩。

目录项：

```text
did|device_name|room|category|online-or-offline
```

能力项：

```text
属性
  spec_name|access|format|constraint|unit

动作
  spec_name|x|in_params
```

测试中的空调能力实际渲染为：

```text
on|wr|bool
target-temperature|wr|float|[16,30;1]|℃
mode|wr|uint8|Cool=2,Heat=5
```

这里的：

```text
wr
  读写

float / uint8 / bool
  参数类型

[16,30;1]
  16–30，步长 1

Cool=2,Heat=5
  枚举名称到协议值
```

当前 catalog 还有这些压缩策略：

```text
最多选 50 台设备
每台默认最多 7 个 spec key
按最近使用和冷启动骨架选择
whitelist 过滤无关能力
同能力骨架设备共享 spec 块
设备间差异旁挂
整体约 5000 token 预算
目录明确声明自己是子集，不是全量
```

这一层不是设备语义本体，只是在决定“模型第一眼看到哪些能力”。

### 4.4 三层分别对应什么约束

```text
MIoT-SPEC
  required / optional / custom
  property / action / event
  read / write / notify
  format
  range
  enum
  unit

MIoT lite spec
  扁平 iid
  read/write 标记
  参数类型、范围和枚举
  适合 MCP / CLI 调用

device catalog
  设备数量
  每台能力数量
  设备-房间检索
  LRU
  白名单
  token 预算
```

### 4.5 参数约束在哪里真正执行

原生 MIoT MCP 的 `send_ctrl_rpc` 会：

```text
检查 iid 是 prop 还是 action
从 lite spec 查对应格式
转换 int / float / string / bool
要求 action 入参是数组
```

CLI 层还会在发送前执行：

```text
did 必须存在
iid 必须属于该设备 spec
枚举值必须命中 value_list
数值必须落在 value_range 和 step 内
control 不能调用 action
```

这比仅把 SPEC 放进 Prompt 多了一层运行时强制。

### 4.6 Miloco 自己增加的流程约束

当前 `miloco-devices` Skill 还定义了不属于 MIoT 协议的业务规则：

```text
不能猜 did / spec_name
同类设备不能直接套同一份 spec
相对调温先查当前值
控制非 on 属性时补该设备的实际主开关
厨房电器要先设参数再调用 start-cook
门锁 / 摄像头 / 燃气阀等需要二次确认
单次控制不超过 10 台
catalog 是子集，复数查询必须重新 device list
```

这些约束属于 Miloco 的设备操作策略，不属于小米 MIoT 本体。

### 4.7 是否实际接入设备

Miloco 与前三篇 benchmark 最大的区别是：

```text
前三篇
  受控仿真或沙箱

Miloco
  代码链路面向真实小米账号、真实设备列表和真实 MIoT 设备控制
```

当前产品路径通过后端 `/api/miot/home` 拉取真实家庭与设备数据，通过 CLI 执行 `control`、`props`、`action`。仓库里同时保留了原生 MIoT MCP 类，但主线产品接入是 OpenClaw + `miloco-cli`。

能否在本文中直接展示某台真实设备的完整 SPEC，取决于本地 Miloco 服务、小米账号登录和当前家庭数据是否可用。代码层面的建模与约束已经可以完整阅读。

### 4.8 哪一层最适合类比前面三篇论文

```text
第一层 MIoT-SPEC
  最适合作为语义约束层的类比
  与 SimuHome 使用的 Matter 设备模型最接近
  与 SMH-Bench 的 Device/Attribute/Service 也有相似结构

第二层 MIoT lite spec
  适合类比 PersonalHomeBench 的 feature 接口
  也适合类比 SMH-Bench 的模型侧紧凑能力视图

第三层 device catalog
  主要类比 benchmark 把状态和方法注入 Prompt 的方式
  它解决上下文预算，不是设备本体建模
```

因此如果当前关注点是“设备语义层怎样约束”，应以第一层为主：

```text
MIoT-SPEC
  ↔ Matter 的 Device Type / Cluster / Feature
  ↔ SMH 的 Device / Attribute / Service
```

第二层只在讨论“模型实际看到什么”时加入：

```text
MIoT lite spec
  ↔ PersonalHomeBench 的 feature 接口
  ↔ 经过压缩的 SMH capability view
```

第三层不承担语义本体，只承担 MCP Host 的上下文裁剪。

## 5 四种对象的建模对照

| 维度 | PersonalHomeBench | SMH-Bench | SimuHome | Miloco |
|---|---|---|---|---|
| 顶层环境 | Home + context + memory | HomeEnv rooms + devices | Matter devices + environment | Xiaomi home + rooms + devices |
| 设备模型 | ApplianceObject / SensorObject | Device + attributes/services/components | Device + endpoints/clusters | MIoT Device + Services + Properties / Actions / Events |
| 能力表示 | feature | Service + Argument | Matter Command | MIoT Property / Action + type level |
| 状态表示 | appliance state / context | typed Attribute | Matter Attribute + environment | MIoT property value |
| 参数约束 | tool/feature 名称与 kwargs | type/range/options/precision/unit | Matter schema + SimuHome rules | access / format / range / value-list / unit |
| 组合设备 | 多 appliance，组件关系较弱 | 显式 components namespace | Endpoint/Cluster 层级 | Service namespace + `type_name@service_description` |
| 动作校验 | feature 有效性与计划有效性 | service 参数校验 + state transition | command 参数校验 + 状态迁移 | iid + 参数类型；CLI 还校验范围和枚举 |
| 执行成功条件 | 计划有效性和结果 | 目标状态 + 无关设备不变 | 设备/环境状态 + 时间条件 | 主要依赖设备返回 code，缺少统一回读验证 |
| 厂商文档依据 | 有，LG/Samsung 等公开文档 | 未在本地稿中证明 | Matter 标准 | 小米 MIoT-SPEC |
| 真实设备执行 | 未见 | 未见 | 未见 | 面向真实 Xiaomi 设备；取决于当前账号与服务状态 |

## 6 从附录能看出四类不同的“例子”

```text
PersonalHomeBench
  例子偏“产品 feature”
  冰箱内部摄像头、食材管理、烤箱预热、电视内容搜索

SMH-Bench
  例子偏“可执行状态与转换”
  空调的 state/target_temperature/ac_mode
  以及 turn_on/set_target_temperature 的参数和赋值代码

SimuHome
  例子偏“协议对象”
  空调用 On/Off、Thermostat、Fan Control 集群组合
  动作通过 Cluster Command 发出

Miloco
  例子偏“厂商能力契约”
  空调服务的必选属性为 on / mode / target-temperature
  控件的读写、范围、枚举和单位来自 MIoT-SPEC
```

这四类例子分别对应：

```text
产品功能层
设备状态机层
协议对象层
厂商能力契约层
```

## 7 证据边界

PersonalHomeBench：

```text
可以确认
  40+ 家电 feature 级接口
  LG/Samsung 公开文档依据
  控制沙箱和工具化执行

不能确认
  所有 benchmark 动作在真实物理设备上执行
```

SMH-Bench：

```text
可以确认
  HomeEnv typed schema
  attribute/service/component
  参数验证、状态迁移、最终状态检查
  美的研究人员参与

不能确认
  设备规格直接来自美的生产系统或真实用户家庭
```

SimuHome：

```text
可以确认
  Matter Endpoint/Cluster/Attribute/Command
  17 类设备
  13 个工具
  设备前置条件、时间与环境约束

不能确认
  已用真实 Matter 硬件完成论文实验
```

Miloco：

```text
可以确认
  MIoT Device / Service / Property / Action / Event 建模
  required / optional / custom 类型层级
  access / range / value-list / unit
  lite spec 与 catalog 压缩
  CLI 运行时范围和枚举校验

不能确认
  当前本地服务是否已登录且返回了具体家庭设备
  所有真实设备都能按标准库获得完整 SPEC
```

## 8 最终判断

```text
PersonalHomeBench
  最适合参考“厂商 feature 如何暴露给 agent”

SMH-Bench
  最适合参考“设备状态、服务、参数和组件怎样写成可执行 schema”

SimuHome
  最适合参考“跨厂商协议如何形成分层设备和能力结构”

Miloco
  最适合参考“同一厂商协议如何从完整 SPEC 投影到模型接口”
```

三篇论文都提供了有价值的建模实例，但论文中的执行环境分别是：

```text
PersonalHomeBench
  厂商文档驱动的控制沙箱

SMH-Bench
  HomeEnv 自定义仿真器

SimuHome
  Matter 对齐仿真器
```

Miloco 的路径不同：

```text
Miloco
  面向真实小米账号与 MIoT 设备
  但在模型接口侧增加了 lite spec 和 catalog 压缩
```

所以前三条路线可以分别支撑接口设计、状态约束和协议语义研究，Miloco 则提供了真实生态中“原始协议模型 → 模型接口 → 上下文压缩”的完整案例。
