# 13：论文中的设备能力约束与 Matter 组件分析

核验日期：2026-10-29

本文分析两个本地论文目录：

```text
finally new paper\0520_方法相关的论文
finally new paper\0524_4个bench
```

重点不是比较 benchmark 分数，而是看它们怎样定义：

```text
设备
状态
能力
动作
参数约束
依赖关系
任务成功条件
```

并回答一个问题：我们的设备 Profile、common/custom 能力组件，以及“必选/可选”设计，能否参照这些论文和 Matter 的做法。

## 0 结论

可以，而且论文里已经出现了接近你想法的多种实现。

```text
HomeBench
  每类设备分配基础功能，再随机加入额外方法

SimuHome
  使用 Matter Endpoint / Cluster / Attribute / Command
  并且增加时间、环境传播和操作前置条件

SMH-Bench
  使用自定义 HomeEnv Schema
  Device → Attributes + Services
  支持 type / range / options / precision / unit

PersonalHomeBench
  设备按 feature 级接口控制
  校验 tool/feature 名称和参数格式

HomeFlow
  用 YAML 声明属性和 service
  用 Blueprint、沙箱和边界检查过滤不可执行任务

SAGE
  不预装全部设备文档
  先定位设备，再按需读取 API 文档和参数

TaskSense
  把工具当词汇，把数据和控制依赖当语法
  生成计划后用依赖图校验

PEC-Home
  使用 room.device.method(args) 形式的动作 DSL
  方法参数可以 required 或 optional，并带 range/enum
```

有一个需要先纠正的点：

```text
SimuHome
  明确基于 Matter 建模

HomeBench
  自定义 Python 虚拟设备和方法

SMH-Bench
  自定义 HomeEnv Schema

HomeFlow
  自定义 YAML 设备规格和 HomeEnv

PersonalHomeBench
  自定义 ApplianceObject + feature 工具
```

因此不能概括成“这些 Home 系列项目全都是 Matter”。真正直接拿 Matter 设备状态作为仿真状态的是 SimuHome。其他论文借鉴了相同的“设备类型 + 状态 + 动作”思想，但协议模型是各自定义的。

## 1 四条代表性建模路线

```text
HomeBench
  扁平方法表
  device.method(params)

SimuHome
  Matter 分层模型
  Endpoint → Cluster → Attribute / Command

SMH-Bench / HomeFlow
  声明式 HomeEnv
  Device → Attributes + Services
  多组件设备再用 Component namespace

PersonalHomeBench / SAGE
  工具化设备接口
  get_features(device)
  execute_feature(device, feature, arguments)
```

## 2 四篇 benchmark 怎样约束

| 论文 | 设备模型 | 状态与动作 | 主要约束 | 必选/可选证据 |
|---|---|---|---|---|
| HomeBench | 15 类设备，每类有一组方法 | 方法直接修改设备属性 | 设备不存在、方法不存在、参数错误、房间内设备互斥 | 每台设备一套基础功能，再随机添加额外功能 |
| SimuHome | 17 类设备，Matter 分层模型 | Attribute 表示状态，Command 表示动作 | 协议规则、操作前置条件、时间约束、环境传播、不可行任务 | Matter Device Type / Cluster / Feature 自带 mandatory 和 optional |
| SMH-Bench | HomeEnv rooms + devices + components | `attributes` 保存状态，`services` 修改状态 | 类型、范围、选项、精度、单位、操作存在性、无关状态不变 | 参数可 required 或 optional；组件可存在或为空 |
| PersonalHomeBench | ApplianceObject、SensorObject、40+ 家电 | `get_appliance_features` / `run_appliance_feature` | 工具名、feature 名、参数格式、部分可观测、计划有效性 | 设备特性通过 feature 接口按需发现 |

### 2.1 HomeBench

HomeBench 的核心模型很直接：

```text
Room
  └─ Device
       └─ Methods
```

机器指令形式：

```text
room.device.method(argument)

guest_bedroom.air_conditioner.set_temperature(26)
study_room.air_conditioner.turn_on()
```

源码与论文中的约束：

```text
设备必须存在
方法必须存在
参数必须符合方法要求
房间与设备组合必须成立
```

它还专门构造了无效指令：

```text
设备不存在
设备存在但没有该方法
多设备中只有部分指令可执行
```

设备能力方面有一处和你的想法很接近：

```text
每个设备类先有一组基础功能
再随机加入额外功能
用于模拟不同品牌和型号
```

例如空调的基础能力包括：

```text
turn_on
turn_off
set_temperature
set_mode
set_fan_speed
set_swing
```

这接近：

```text
基础/常见能力
  +
额外/可选能力
```

不过 HomeBench 没有进一步规范“每项能力属于 common 还是 custom”。

HomeBench 还加入了房间级互斥约束：

```text
空调 / 加热器 / 风扇不能同时放在同一个房间
加湿器 / 除湿器不能同时放在同一个房间
```

这说明约束分两层：

```text
单设备能力约束
家庭或房间级组合约束
```

我们的设计如果只定义单个设备的能力卡，还要另留“跨设备约束”位置，不能把所有逻辑都塞进空调 Profile。

### 2.2 SMH-Bench

SMH-Bench 的 HomeEnv 给出了最接近可实现规格书的 schema。

```text
Home
├─ rooms[]
└─ devices[]
     ├─ attributes[]
     ├─ services[]
     ├─ components[]
     └─ userdata
          ├─ did
          ├─ spid
          ├─ category
          ├─ subcategory
          ├─ tags
          └─ room
```

属性字段：

```text
name
description
type
value
range optional
options optional
items optional
precision optional
unit optional
```

服务字段：

```text
name
description
arguments
code
```

服务参数复用属性的类型系统，因此可以验证：

```text
参数名
类型
范围
精度
枚举选项
```

调用无效服务时：

```text
动作被拒绝
家庭状态不发生修改
```

多组件设备使用 namespace：

```text
fan_light
  light.turn_on
  light.set_brightness
  fan.turn_on
  fan.set_speed_level
```

这对我们非常有用。一个物理设备内部可以有多个能力组件，组件需要自己的命名空间，避免 `turn_on` 同时指灯、风扇或整机。

SMH-Bench 还要求：

```text
目标设备达到目标状态
未被任务涉及的设备状态保持不变
```

这比“最后一个工具调用成功”更严格。我们执行 Capability 后也应做：

```text
目标状态验证
无关状态未变化
```

### 2.3 SimuHome

SimuHome 是最直接采用 Matter 的 benchmark。

Matter 状态模型：

```text
Device
  → Endpoint
     → Cluster
        → Attribute
        → Command
```

动作：

```text
device_id
endpoint_id
cluster_id
command_id
arguments
```

SimuHome 的 13 个工具直接围绕这个模型：

```text
get_device_structure
get_attribute
get_all_attributes
write_attribute
execute_command
get_cluster_docs
get_rooms
get_room_devices
get_room_states
schedule_workflow
get_current_time
get_workflows
finish
```

多个参数是必填：

```text
execute_command
  device_id                 必填
  endpoint_id               必填
  cluster_id                必填
  command_id                必填
  arguments                 必填

write_attribute
  device_id                 必填
  endpoint_id               必填
  cluster_id                必填
  attribute_id              必填
  value                     必填
```

这里比简单方法表多出三个重要层级：

```text
一个 Device 可以有多个 Endpoint
一个 Endpoint 可以有多个 Cluster
一个 Cluster 有 Attribute、Command、Event、Feature
```

SimuHome 还加入插件或项目层的约束。例如：

```text
空调必须先开启电源
之后才能修改风扇转速
```

它把不可行任务也纳入 benchmark，因此约束不仅用于执行校验，还用于判断任务可解性。

### 2.4 PersonalHomeBench

PersonalHomeBench 使用 `PersonalHomeTools`。

设备接口分成查询和执行：

```text
get_appliance_features(device_name)
  → 返回支持的 feature 与参数

run_appliance_feature(device_name, feature, arguments)
  → 执行指定 feature
```

它基于 LG、Samsung 的公开文档实现 40 多种家电的 feature 级接口。

这与 MCP 的分层很接近：

```text
先发现设备
  ↓
查询设备 feature
  ↓
执行 feature
```

评测中会计算 Validity：

```text
工具名有效
feature 名有效
参数格式正确
```

PersonalHomeBench 的重点还包括部分可观测。模型一开始不知道完整家庭状态，必须主动查：

```text
设备清单
设备状态
设备 feature
用户记忆
事件时间线
```

这支持我们的设计：不应把完整设备能力卡一次性全部发给模型。

## 3 方法论文怎样约束设备与环境

| 论文族 | 主要建模对象 | 对设备能力约束的价值 |
|---|---|---|
| Sasha、SAGE、VCU-LLM、MiCU、AdaHome | 意图、设备指代、工具调用、执行计划 | 依赖现成设备方法或 API，没有提出完整设备能力本体 |
| HomeFlow | 房间、设备、状态、函数、用户目标和成功条件 | 声明式 YAML 设备规格 + 环境验证，最接近可执行规格 |
| TaskSense | 传感器工具、输入输出和控制依赖 | 用词汇表与 DAG 语法约束工具组合 |
| PEC-Home | 房间、设备、方法、参数和对话历史 | 方法 DSL 已明确参数类型、范围、枚举和 optional |
| MvTAP、HomeGenii | 触发器、动作、自动化规则 | 主要约束规则组合，不提供通用的单设备能力模型 |
| HomeLLaMA、MemAura、PromptShield-Home 等 | 隐私、记忆、上下文和安全 | 约束推理或执行策略，不定义设备能力 schema |

这说明方法文件夹里并不是每篇论文都设计了设备模型。真正能直接借鉴到 Capability 层的是 HomeFlow、SAGE、TaskSense 和 PEC-Home；其余论文更多处理“意图如何落到已有 API”或“执行前后如何保证安全”。

### 3.1 HomeFlow：声明式设备规格 + 可验证条件

HomeFlow 使用自己的 HomeEnv，不是 Matter。

状态定义：

```text
s_t = <L, D, ψ, X_t, F>

L      房间与空间结构
D      智能设备集合
ψ      Device → Room
X_t    Device 的动态状态
F      设备暴露的可执行 Function
```

设备由 YAML 定义：

```yaml
name: light
userdata:
  category: light
attributes:
  - name: state
    type: str
    options: [on, off]
  - name: brightness
    type: int
    range: [1, 100]
    unit: percent
services:
  - name: turn_on
  - name: set_brightness
    args:
      - name: brightness
        type: int
        range: [1, 100]
```

这种结构与 SMH-Bench 很接近：

```text
Device
  ├─ attributes
  └─ services
```

HomeFlow 的 Blueprint 条件还要经过三重过滤：

```text
1 parse / undefined device check
2 device schema / parameter boundary check
3 instruction-condition-profile logical consistency check
```

它说明“必选/可选”之外还需要：

```text
合法性
可执行性
语义一致性
```

### 3.2 SAGE：运行时发现 API 文档

SAGE 不为所有设备预写控制代码，而是：

```text
用户请求
  ↓
设备交互规划器
  ↓
API 文档检索
  ↓
读取属性 / 执行命令
  ↓
API 错误回传
  ↓
修正参数后重试
```

它能看到的信息：

```text
设备 ID
属性名
命令名
参数名
参数类型
组件
错误信息
```

这套方法适合长尾设备，但代价是每次可能多做一轮工具调用。

对我们的启示：

```text
common
  可以直接放在设备能力卡里

custom
  可以延迟到 inspect_capabilities 再取
```

### 3.3 TaskSense：能力之外还有依赖语法

TaskSense 把工具系统建模成一种语言：

```text
词汇
  Tool name
  category
  modality
  device
  description
  input
  output
  label options

语法
  工具依赖有向无环图
```

依赖分两类：

```text
Data dependency
  一个工具的输出必须是另一个工具的输入

Control dependency
  一个工具的输出决定另一个工具是否允许执行
```

它给出的例子是：

```text
Face Detection
  → Face Recognition
```

或者：

```text
RGB 人体检测发现至少一人
  → 才允许说话人日志化
```

对于设备，我们对应需要：

```text
Device capability dependency
service input/output dependency
precondition / postcondition
```

例如：

```text
空调 target_temperature
  可能要求 power = on

烘干机 start
  可能要求 door = closed

信封门锁
  unlock
  可能需要二次确认
```

这说明只定义“有哪些 Capability”还不够，Capability 之间还需要依赖边。

### 3.4 PEC-Home：动作 DSL 和可选参数

PEC-Home 使用：

```text
room.device.method(argument)
```

方法表中的参数明确带有约束：

```text
turn_on(time: optional string = None, format="%H:%M")
turn_off(time: optional string = None, format="%H:%M")
set_brightness(brightness: int, range: 0-4)
set_mode(mode: string, options=[...])
```

这里已经出现了：

```text
required argument
optional argument
type
range
options
format
```

论文列出的错误类型也很有代表性：

```text
房间缺失
房间错误
参数缺失
参数错误
历史指代错误
```

这些正是 MCP Schema 和 Host 校验应该拦截的内容。

## 4 论文与 Matter 的约束层次对比

| 约束层次 | HomeBench | SimuHome | SMH-Bench | HomeFlow | PEC-Home | Matter |
|---|---|---|---|---|---|---|
| 设备存在 | 有 | 有 | 有 | 有 | 有 | Endpoint / Device Type |
| 能力存在 | 方法存在 | Cluster / Command 存在 | Service 存在 | Service 存在 | Method 存在 | 由 Cluster / FeatureMap 决定 |
| 参数类型 | Python API | Matter Schema | 显式 typed schema | YAML typed schema | 方法签名 | Cluster 定义 |
| 数值范围 | 方法内部 | Matter/项目规则 | range | range | range | Min/Max 属性 |
| 枚举 | 弱 | Matter enum | options | options | options | Enum / constraint |
| 设备到房间 | 有 | 有 | `userdata.room` | ψ: D→L | 命令前缀 | 非 Matter 核心范围 |
| 多组件命名空间 | 少 | Endpoint 分层 | components | components | 房间+设备+方法 | Endpoint / Cluster 分层 |
| 操作前置条件 | 部分 | 有 | service code | service code | 部分 | Cluster 约束与协议规则 |
| 时态/调度 | 弱 | 强 | 状态驱动调度 | Blueprint 条件 | 可选时间参数 | 协议本身弱约束 |
| 跨设备组合 | 有互斥 | 有环境传播 | 有组合任务 | Blueprint/MCTS | 有 | 主要由上层控制 |
| 成功条件 | 方法调用正确 | 最终状态 | 最终状态 + 无关状态不变 | 布尔条件 | 最终状态 | 上层实现 |

## 5 Matter 自己怎样做“必选和可选组件”

Matter 的约束分布在不同层级：

```text
Device Type
  定义一类端点必须和可选包含哪些 Cluster

Cluster
  定义 Attribute / Command / Event / Feature

Conformance
  Mandatory
  Optional
  Conditional
  Disallowed

FeatureMap
  表示当前实例实现了哪些 Feature

Attribute
  可以有 range、quality、default、feature dependency

Command
  定义参数、数据类型、必选参数和响应
```

### 5.1 Room Air Conditioner

本地 Matter 1.6 中：

```text
Room Air Conditioner
  Device Type 0x0072

Mandatory clusters
  Identify
  On/Off
  Thermostat

Optional clusters
  Fan Control
  Temperature Measurement
  Relative Humidity Measurement
  HEPA Filter Monitoring
  Activated Carbon Filter Monitoring
  Thermostat User Interface Configuration
```

它表达的约束是：

```text
要成为 Room Air Conditioner
  必须有 On/Off 与 Thermostat

不一定有风速
  Fan Control 是 optional

不一定能读当前温度
  Temperature Measurement 是 optional

不一定有摆风
  Fan Control 的 rocking/wind/direction 又受 Feature 控制
```

这和“空调最小核心是开关 + 设定温度”很接近。

### 5.2 Refrigerator

Matter 冰箱采用组合 Device Type：

```text
Refrigerator 0x0070
  └─ Temperature Controlled Cabinet 0x0071
       └─ Temperature Control 0x0056
            TemperatureSetpoint
            MinTemperature
            MaxTemperature
```

Temperature Controlled Cabinet 的关键约束：

```text
Temperature Control
  mandatory
  TN feature mandatory
  TL feature disallowed

Temperature Measurement
  optional

Mode
  optional

Temperature Alarm
  optional
```

父级 Refrigerator：

```text
Refrigerator Alarm
  optional

carbon filter monitoring
  optional
```

这证明“一个冰箱”本身可以是由多个 endpoint 组成的复合设备。Matter 在协议层已经完成了：

```text
Device Type
  +
Cluster composition
  +
Feature conformance
  +
Endpoint hierarchy
```

我们的 Profile 可以借用同样的结构，只需把底层 Cluster 换成 HA Capability。

## 6 适合我们项目的结构

论文和 Matter 共同支持以下设计：

```text
Profile Contract
  required_all
  required_any
  optional capabilities

Capability Card
  summary
  observe
    common
    custom
  control
    common
    custom

Constraints
  parameter type
  range
  options
  unit
  format
  precondition
  dependency
  cross-device conflict
  time condition

Runtime
  current values
  availability
  execution result
  post-action verification
```

能力项建议采用：

```yaml
id: air_conditioner.set_target_temperature
kind: control
presence: optional
tier: common
zone: main
argument:
  name: value
  type: number
  unit: celsius
  min: 16
  max: 30
  step: 0.5
source:
  adapter: home_assistant
  entity_id: climate.living_room
  service: climate.set_temperature
verify:
  readback: target_temperature
  tolerance: 0.5
```

其中：

```text
presence
  required / optional

tier
  common / custom
```

这里有两个独立轴，每个轴只有两档。这样既保留 MCP 的必选/可选校验，又能控制合成数据只使用 common。

约束不应全写进文字摘要。文字摘要只说明：

```text
这个设备能做什么
哪些是常用能力
```

数值范围、依赖、成功条件仍由结构字段负责。

## 7 对空调和冰箱的最终判断

### 空调

```text
identity
  Device Type / integration evidence

required_all
  set_power
  set_target_temperature

common
  current_temperature
  set_hvac_mode
  set_fan_speed

custom
  swing_vertical
  swing_horizontal
  timer
  sleep
  eco
  boost
  self_clean

summary
  从上述实际能力和温区生成
```

这不是唯一答案，但证据最稳：

```text
HomeBench / PEC-Home
  提供空调方法表

SMH-Bench
  明确 target_temperature、ac_mode、set_target_temperature、set_ac_mode

SimuHome
  用 Matter On/Off + Thermostat + optional Fan Control

Matter
  On/Off + Thermostat mandatory，Fan Control optional
```

### 冰箱

```text
identity
  Device Type / integration evidence

observe.common
  各温区 current_temperature
  door_state
  fault_state

control.common
  各温区 set_target_temperature

control.custom
  mode
  super_cool
  super_freeze
  holiday_mode
  ice_maker

constraints
  temperature range 由温区实例决定
  部分冰箱只有温度等级 select，没有精确 setpoint
```

Matter 还提示我们要用：

```text
Device → Endpoint/Cabinet → Capability
```

而不是：

```text
Device → 所有温度字段平铺
```

## 8 最终判断

这批论文和 Matter 已经支持你的两个判断：

```text
第一，设备能力可以拆成必选组件、常见组件和可选组件
第二，同一个设备内部可以按只读状态、可执行动作和文字摘要组织
```

需要补的一点是：

```text
必选/可选
  属于 Profile 合法性

常见/不常见
  属于模型可见性与合成数据采样

约束
  属于 Capability 自身和 Capability 之间的关系
```

把这三类信息分开，既能吸收 HomeBench、SMH-Bench、HomeFlow、SAGE、TaskSense 和 Matter 的优点，也不会让 Schema 变成一套混在一起的复杂枚举。

## 9 证据边界

```text
HomeBench
  设备方法表是作者为评测设计的简化模型

SimuHome
  采用 Matter，但仍增加了时间、环境与业务约束

SMH-Bench
  HomeEnv 是论文自定义 schema，不是 Matter

HomeFlow
  HomeEnv/YAML 是论文自定义 schema，不是 Matter

PersonalHomeBench
  feature 接口来自厂商公开文档，覆盖范围按其实现边界

SAGE
  API 文档动态发现，能力面取决于真实服务和设备

TaskSense
  依赖语法用于传感器工具协作，可借鉴但不等同于设备类型系统
```

因此我们可以借鉴这些结构，但要区分：

```text
标准协议约束
论文仿真器约束
插件映射约束
实际设备实例约束
MCP 端侧投影约束
```

这五层不能混写成同一张静态表。

## 10 现实迁移与外部效度

讨论 benchmark 分数是否能迁移到真实设备，需要把“真实”拆成四层：

```text
语义真实
  设备类别、能力名、参数语义是否正确

接口真实
  设备能力是否来自真实厂商 SDK / API / 协议

执行真实
  操作是否真的发到物理设备，并产生可观测副作用

分布真实
  测试家庭、设备型号、固件、网络故障和用户行为是否接近现实
```

四个 benchmark 在这四层上的证据强度不同。

| Benchmark | 设备来源 | 协议/接口真实性 | 执行真实性 | 主要能证明什么 |
|---|---|---|---|---|
| HomeBench | 15 类自定义 Python 虚拟设备 | 自定义简化方法 | 虚拟状态修改 | 意图到设备方法的映射、无效设备/方法判断、多设备组合 |
| PersonalHomeBench | LG / Samsung 公开文档中的 40+ 家电 feature 级接口 | 比 HomeBench 更接近真实厂商接口 | 自定义控制沙箱 | feature 发现、工具选择、参数有效性、部分可观测和个性化 |
| SimuHome | 17 类设备，Matter Endpoint / Cluster / Attribute / Command | 与 Matter 数据结构对齐 | 自定义仿真器 | 协议语义、设备前置条件、时间依赖、不可行任务 |
| SMH-Bench | HomeEnv 自定义房间、设备、属性、服务、组件 | 自定义 typed schema | 可执行但仍是自定义仿真器 | 状态依赖、组件命名、参数校验、任务成功与无关状态保持 |

### 10.1 HomeBench

HomeBench 能支持的现实结论：

```text
较大模型是否理解 room.device.method(arg) 这种控制形式
是否能识别不存在的设备或方法
是否能组合多设备动作
```

不能直接支持的结论：

```text
模型能否操作某个真实品牌设备
模型能否处理真实协议的 Endpoint / Cluster / DPCode
模型能否处理网络超时、设备离线、模式互斥和固件差异
```

HomeBench 的作者自己把虚拟设备方法做了简化，只保留核心智能家居功能，再随机添加额外方法。因此它更像“控制语义和无效指令测试”，缺少厂商协议和物理执行的真实性。

对大模型而言，HomeBench 分数可以作为：

```text
语义规划、工具调用、无效动作识别的大致能力信号
```

对小模型而言，相关性更弱：

```text
模型可能记住 benchmark 的设备名和模板
换到真实 Entity ID、别名、协议错误后性能可能明显下降
低分也可能来自格式或工具接口不熟悉，而不是设备操作能力本身不足
```

所以：

```text
HomeBench 高分 ≠ 真实设备可操作
HomeBench 低分 ≠ 真实场景一定不可用
```

### 10.2 PersonalHomeBench

PersonalHomeBench 比 HomeBench 更接近真实接口。

它的设备控制沙箱基于 LG 和三星的公开文档，为 40 多种智能家电实现了 feature 级接口。测试中还要检查：

```text
工具名是否有效
feature 名称是否有效
参数是否格式良好
```

因此它比 HomeBench更能证明：

```text
模型是否能发现设备能力
是否能从多个 feature 中选择正确接口
是否能在部分可观测条件下主动查询
是否能处理家庭记忆、人物和个性化需求
```

但它仍然不是物理设备执行环境：

```text
设备状态由沙箱维护
家庭和事件由生成流程构造
工具接口是 PersonalHomeTools 的统一封装
论文没有证明所有打分设备都在真实家庭中完成了端到端控制
```

它较好的迁移方向是：

```text
工具发现
能力选择
参数格式
个性化推理
部分可观测交互
```

它仍然不能直接证明：

```text
真实设备写入是否成功
厂商云是否限流
设备状态多久落定
不同固件是否暴露相同 capability
```

### 10.3 SimuHome

SimuHome 与 Matter 对齐，因此在“语义真实”和“接口真实”上明显强于 HomeBench。

设备状态使用：

```text
Endpoint
  → Cluster
     → Attribute
```

动作使用：

```text
Endpoint
  + Cluster
  + Command
  + arguments
```

它还能执行项目规则，例如：

```text
空调电源未开启时，不能直接修改风速
```

因此 SimuHome 的分数更能反映：

```text
模型能否理解分层设备结构
能否查询 Attribute
能否选择 Command
能否处理前置条件和不可行任务
```

但本地材料能确认的是“与 Matter 对齐的仿真”，尚未证明这些任务已经在多种真实 Matter 设备上完成独立复现。论文声称迁移只需较少适配，这属于作者声明，不能等同于第三方真机验证。

### 10.4 SMH-Bench 与美的的关系

SMH-Bench 的作者单位包括：

```text
Midea Group
Beijing University of Posts and Telecommunications
Donghua University
The University of Sydney
Peking University
```

但在本地论文正文与附录中，没有看到“HomeEnv 的设备、属性和服务来自美的真实产品目录、米家/MIoT 协议或美的生产系统”的直接说明。

因此现有证据更支持：

```text
SMH-Bench 由美的研究人员参与完成
HomeEnv 是一套论文自定义的可执行设备状态 schema
设备类型、属性、服务与约束为 benchmark 设计和生成服务
```

暂不能写成：

```text
SMH-Bench 直接基于美的真实设备系统
SMH-Bench 的设备 schema 等于美的产品 API
```

除非作者代码、数据或后续文档明确证明 HomeEnv 的设备规格来自美的真实产品目录。

### 10.5 哪种迁移更强

```text
设备控制语义
  HomeBench 可提供初步信号

厂商 feature 级工具调用
  PersonalHomeBench 更接近现实

环境状态、空间与组合约束
  SMH-Bench 更完整

跨厂商通用协议与设备结构
  SimuHome 更接近现实

真实物理执行
  四个 benchmark 本地材料都不能单独证明
```

对我们的项目，建议把验证拆成三级：

```text
L1 语义评测
  设备/能力/参数选择是否正确
  可参考 HomeBench、PersonalHomeBench、SMH-Bench

L2 协议与模拟器评测
  HA Entity / Matter Cluster / 小米 DP / 厂商 API 映射是否正确
  以 SimuHome、Matter 和本地插件源码为依据

L3 真机迁移评测
  实际设备写入、落定、回读、失败恢复、超时和权限
  必须接真实 HA 或真实设备，不能由上述 benchmark 代替
```

这也能解释为什么 SimuHome 的方向更合适作为基础：它保留了设备结构、前置条件和状态迁移；而 HomeBench、PersonalHomeBench、SMH-Bench 更适合作为不同层级的辅助评测，不能只看一个总分推断真实家庭操作能力。
