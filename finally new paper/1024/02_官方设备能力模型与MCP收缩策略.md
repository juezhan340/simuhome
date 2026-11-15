# 02：官方设备能力模型与 MCP 收缩策略

日期：2026-10-25

## 图一：官方如何组织设备与能力

```text
                  抽象层级                                    例子：米家冰箱

┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│ Integration Config Entry         │          │ xiaomi_home 接入实例              │
│ 来源集成、账号、连接方式           │          │ 小米账号 + 云端/局域网连接         │
└────────────────┬─────────────────┘          └────────────────┬─────────────────┘
                 │ 1 个实例可创建多台设备                       │
                 ▼                                             ▼
┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│ Device                           │          │ 米家对开门冰箱                    │
│ 型号、厂商、区域、物理身份         │          │ midjd6.fridge.6101               │
└────────────────┬─────────────────┘          └────────────────┬─────────────────┘
                 │ 1 台设备可拥有多个实体                       │
                 ▼                                             ▼
┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│ Entity                           │          │ number 冷藏目标温度               │
│ 设备对外暴露的功能面               │          │ number 冷冻目标温度               │
│                                  │          │ select 冰箱模式                  │
│                                  │          │ sensor 冷藏/冷冻/环境温度         │
│                                  │          │ sensor 故障码/门未关              │
│                                  │          │ event 门开提醒                   │
│                                  │          │ button 恢复出厂设置               │
└───────────┬──────────────────────┘          └───────────┬──────────────────────┘
            │                                             │
            ├─ 读取 state + attributes                     ├─ state=5
            │  当前值、范围、枚举、supported_features       │   min=2, max=8, step=1
            │                                             │
            └─ 被 Domain Service 调用                      └─ number.set_value
               动作由领域服务执行                              set_value(5)
                              │
                              ▼
                    ┌─────────────────────────┐
                    │ Derived Capability      │
                    │ 不是官方注册表对象        │
                    │                          │
                    │ 冷藏目标温度 + 2~8°C     │
                    │ + 步长 1 + set_value     │
                    └─────────────────────────┘
```

```text
严格父子关系
  Integration Config Entry → Device → Entity

能力推导关系
  Entity + State + Attribute + Domain Service → Capability
```

图一的重点是：官方注册表只明确 `Config Entry → Device → Entity` 的归属；`Capability` 是读取能力位、状态属性和服务后推导出来的，不是另一个可以直接查询的官方注册表对象。

## 图二：官方 MCP 如何组织暴露链路

```text
                  抽象链路                                    当前空调实例

┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│ HA Entity + 暴露设置              │          │ climate.simuhome_living_room_ac  │
│ 哪些实体允许 Assist 看见           │          │ state=cool                       │
│                                  │          │ temperature=24                   │
│                                  │          │ hvac_modes=[off,cool,heat]       │
└────────────────┬─────────────────┘          └────────────────┬─────────────────┘
                 │                                             │
                 ▼                                             ▼
┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│ Assist LLM API                   │          │ HassTurnOn / HassTurnOff         │
│ 按 domain、能力和暴露结果生成工具  │          │ HassClimateSetTemperature        │
│                                  │          │ GetLiveContext                   │
└────────────────┬─────────────────┘          └────────────────┬─────────────────┘
                 │                                             │
                 ▼                                             ▼
┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│ Official MCP Server              │          │ /api/mcp                         │
│ mcp_server 集成                   │          │ tools / prompts / resources      │
│ 把 Assist 工具投影成 MCP          │          │ 当前共 12 个工具                  │
└────────────────┬─────────────────┘          └────────────────┬─────────────────┘
                 │                                             │
                 ▼                                             ▼
┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│ MCP Client                       │          │ LLM / Agent 客户端                │
│ Streamable HTTP 或 SSE           │          │ 调用工具改变 HA 世界              │
└──────────────────────────────────┘          └──────────────────────────────────┘

反向方向：外部 MCP Server → 官方 mcp (Client) 集成 → Assist
           把外部工具接入 HA，不等同于 mcp_server 的暴露链路
```

```text
官方 Server 方向的层级是：
  Assist 可见实体
    → Assist LLM API 工具
    → mcp_server 集成
    → MCP Client

当前实例的 `supported_features=385` 解码为：
  TARGET_TEMPERATURE + TURN_ON + TURN_OFF

所以 Agent 最终看到的是：
  HassTurnOn / HassTurnOff / HassClimateSetTemperature
  而不是 Matter cluster、MIoT property 或 HA 的全部内部服务
```

图二的重点是：官方 MCP 已经在 HA 实体之上做了一次“Assist 工具级”收缩，但它仍然没有把动作参数范围、设备 profile 和写后验证组织完整。

### 用冰箱走一遍官方 MCP

用户问：

```text
“冰箱现在冷藏和冷冻分别是多少度？”
```

模型先用区域和 domain 缩小查询：

```json
GetLiveContext({
  "area": "2646196763的家",
  "domain": "sensor"
})
```

官方 MCP 当前实际返回的核心内容：

```text
冷冻室 温度
  domain: sensor
  state: -19
  unit: °C
  device_class: temperature

冷藏室 温度
  domain: sensor
  state: 3
  unit: °C
  device_class: temperature

室内环境温度
  domain: sensor
  state: 24
  unit: °C
  device_class: temperature
```

这时模型能看到：

```text
有一台冰箱
它位于“2646196763的家”
冷藏当前 3°C
冷冻当前 -19°C
室内环境温度 24°C
```

模型可以直接回答：

```text
“冷藏当前是 3°C，冷冻当前是 -19°C。”
```

接下来用户说：

```text
“把冷藏室调到 5°C。”
```

模型在当前官方 MCP 中会遇到一个明确断点：

```text
它知道冰箱当前是 3°C
但它看不到 domain 为 number 的“冷藏室设定温度”实体
当前 12 个工具里也没有 number.set_value 或 select.select_option 的入口
因此它不能完成这次温度设定
```

从 HA 内部看，真正的执行目标本来存在：

```text
entity:
  number.midjd6_cn_911432173_6101_target_temperature_p_3_2

state:
  5

attributes:
  min=2
  max=8
  step=1
  unit=°C

service:
  number.set_value(value=5)
```

但这条能力没有进入当前 Assist 暴露面，也没有对应的官方 MCP 动作，所以模型“看得到温度，却操作不了温度”。

这正是需要设备 profile 和按需能力展开的地方。我们的 MCP 可以把它收缩成：

```json
{
  "id": "midjd6.fridge.6101",
  "type": "refrigerator",
  "state": {
    "cold_current_c": 3,
    "freezer_current_c": -19,
    "ambient_c": 24
  },
  "actions": {
    "set_cold_temperature": {
      "min": 2,
      "max": 8,
      "step": 1,
      "unit": "°C"
    },
    "set_freezer_temperature": {
      "min": -24,
      "max": -15,
      "step": 1,
      "unit": "°C"
    }
  }
}
```

模型此时只需要看到一台冰箱、三个当前值和两个设定动作。它执行：

```text
execute_capability(
  device="midjd6.fridge.6101",
  capability="set_cold_temperature",
  value=5
)
```

后端映射到真实 HA 实体：

```text
number.set_value(
  entity_id="number.midjd6_cn_911432173_6101_target_temperature_p_3_2",
  value=5
)
```

执行层再回读该实体：

```json
{
  "ok": true,
  "verified": true,
  "state_after": {
    "cold_target_c": 5
  }
}
```

完整对比如下：

```text
官方 MCP 当前
  看到 3 个当前温度
  看不到设定温度、模式、门事件和故障
  没有冰箱设定动作

收缩后的 MCP
  看到 3 个当前温度
  看到 2 个核心设定动作及范围
  默认隐藏模式、故障码、压缩机转速和恢复出厂
  执行后回读并返回 verified
```

目标：先说明 Home Assistant 官方如何表示一台设备及其能力，再讨论怎样把庞大的真实能力面收缩成 Agent 可以低成本使用的 MCP 视图。

## 1 一句话结论

HA 官方没有把一台设备定义成“一台设备 = 一组固定动作”，而是分成四层：

```text
Config Entry
  这是哪个集成、用哪套账号或连接配置
        ↓
Device
  这是现实中的哪个物理设备
        ↓
Entity
  这台设备暴露出的功能面
        ↓
State / Attribute / Service
  现在是什么状态、有哪些能力、能调用什么动作
```

因此，“设备能力”不是单独存在的一份静态表。它由实体、状态属性、能力位和 HA 服务共同组成。

Agent 不需要看到这四层的全部内容。MCP 应该把四层内部保留，对外只暴露当前设备和当前任务真正需要的能力。

## 2 官方是怎么定义设备的

### 2.1 Config Entry：设备属于哪个接入实例

同一台空调可能通过以下任一集成进入 HA：

```text
Matter
小米米家
美的局域网插件
LocalTuya
MQTT discovery
HomeKit Controller
```

Config Entry 记录连接方式、账号、token、区域或设备发现信息。它回答的是“这台设备从哪里来”，不直接回答“它能做什么”。

### 2.2 Device Registry：现实中的设备对象

Device Registry 保存：

```text
device_id
name / name_by_user
manufacturer / model
area_id
identifiers / connections
config_entry_id
```

一个 Device 可以理解成现实中的“整台机器”。冰箱是一台 Device，空调也是一台 Device。

### 2.3 Entity Registry：设备向 HA 暴露的功能面

Entity 才是 HA 操作和观测的主要单位。常见的功能面包括：

```text
climate       温度目标、模式、冷暖状态
number        某个数值参数
select        离散选项
switch        开关能力
light         亮度、颜色、色温
cover         开合、位置、倾斜
fan           风速、摆头、预设模式
sensor        只读测量值
binary_sensor 是/否状态
event         离散事件
button        一次性动作
```

同一个物理设备可能被拆成很多 Entity：

```text
冰箱
├─ number 冷藏设定温度
├─ number 冷冻设定温度
├─ select 冰箱模式
├─ sensor 冷藏温度
├─ sensor 冷冻温度
├─ sensor 环境温度
├─ sensor 故障码
├─ sensor 门未关提醒
├─ sensor 压缩机转速
├─ event  门开提醒
└─ button 恢复出厂设置
```

当前本地米家冰箱就是一台 Device 关联 12 个 Entity。Entity 的粒度由插件决定，不由物理设备的外观决定。

### 2.4 State 与 Attribute：当前状态和可选能力

Entity 的 `state` 表示当前值，`attributes` 补充能力范围：

```text
number
  state=5
  min=2
  max=8
  step=1
  unit=°C

select
  state=手动
  options=[手动, 智能, 假日, 速冷, 速冻]

climate
  state=cool
  hvac_modes=[off, cool, heat]
  current_temperature=26
  temperature=24
  min_temp=7
  max_temp=32
  target_temp_step=0.5
  supported_features=385
```

`supported_features` 是位标志。当前空调的 `385` 解码后为：

```text
TARGET_TEMPERATURE
TURN_ON
TURN_OFF
```

当前厨房风扇的 `49` 解码后为：

```text
SET_SPEED
TURN_ON
TURN_OFF
```

这正是官方能力定义的一个特点：同一个 `climate` 类型可以有完全不同的能力组合，Agent 不能只看域名字。

### 2.5 Service：HA 的动作接口

HA 通过服务执行动作：

```text
climate.set_temperature
climate.set_hvac_mode
fan.set_percentage
light.turn_on
number.set_value
select.select_option
```

具体 Entity 是否支持某个能力，仍要结合域、`supported_features` 和属性判断。

## 3 官方 MCP 当前怎么暴露

本地 Core 同时有两套官方 MCP 集成：

```text
mcp
  HA 作为 MCP Client，连接外部 MCP Server

mcp_server
  HA 作为 MCP Server，把 Assist 能力暴露给外部 MCP Client
```

当前运行实例暴露的是 `mcp_server`，入口为：

```text
http://localhost:8123/api/mcp
```

实测返回 12 个工具：

```text
HassTurnOn
HassTurnOff
HassCancelAllTimers
HassBroadcast
HassClimateSetTemperature
HassFanSetSpeed
HassListAddItem
HassListCompleteItem
HassListRemoveItem
GetDateTime
todo_get_items
GetLiveContext
```

这些工具来自 HA 的 Assist LLM API，并非直接罗列 HA 的全部服务。官方已经做了一层意图级收缩：

```text
HA 更丰富的实体与能力
        ↓ 只保留暴露给 Assist 的实体
        ↓ 按 domain 和能力生成工具
官方 MCP 的 12 个工具
```

但这层收缩仍然不够适合小模型：

```text
工具参数 required 全部为空
工具 schema 没有温度范围、枚举、步长
HassTurnOn/Off 使用自然语言 name + area/floor，不直接使用稳定 id
GetLiveContext 仍可能返回较大的全屋上下文
写调用没有内嵌回读和 verified
```

## 4 两个真实例子

### 4.1 空调：一个实体里藏着一整张能力表

当前客厅空调的 HA 实体是：

```text
climate.simuhome_living_room_ac

state=cool
current_temperature=26
temperature=24
hvac_modes=[off, cool, heat]
min_temp=7
max_temp=32
target_temp_step=0.5
supported_features=385
```

这台设备在 HA 中只有一个 Entity，但 Agent 实际需要的能力已经很清楚：

```text
observation
  power
  mode
  current_temperature
  target_temperature

action
  turn_on / turn_off
  set_mode
  set_target_temperature
```

压缩后可以只交给模型：

```json
{
  "id": "living_room.ac",
  "type": "air_conditioner",
  "state": {
    "power": "on",
    "mode": "cool",
    "target_c": 24.0,
    "current_c": 26.0
  },
  "actions": {
    "set_power": ["on", "off"],
    "set_mode": ["cool", "heat"],
    "set_temperature": {
      "min": 7.0,
      "max": 32.0,
      "step": 0.5,
      "unit": "°C"
    }
  }
}
```

默认视图不需要携带：

```text
supported_features 的位运算细节
device_id、config_entry_id、unique_id
integration 名称和底层协议
风速、风向、预设、故障位图
```

对于“把客厅调凉快一点”，这些已经足够。

只有用户明确说“风别对着我吹”或“开最大风”，才按需展开：

```text
fan_speed
swing_mode
preset_mode
```

### 4.2 冰箱：一个设备被拆成 12 个实体

当前米家冰箱是一台 Device，关联 12 个 Entity：

```text
number 冷藏室设定温度   2~8°C，步长 1
number 冷冻室设定温度   -24~-15°C，步长 1
select 冰箱模式         手动 / 智能 / 假日 / 速冷 / 速冻
sensor 冷藏室温度
sensor 冷冻室温度
sensor 环境温度
sensor 门未关提醒
sensor 故障码
sensor 门开异常事件
sensor 压缩机转速
event  门开提醒
button 恢复出厂设置
```

对普通家庭控制，真正高频的只有：

```text
冷藏目标温度
冷冻目标温度
冷藏当前温度
冷冻当前温度
```

压缩后的相关能力：

```text
refrigerator
├─ set_cold_temperature
├─ set_freezer_temperature
├─ observe_cold_temperature
├─ observe_freezer_temperature
└─ door_open_event（只在异常或用户主动询问时展开）
```

默认隐藏：

```text
压缩机转速
故障位图
恢复出厂设置
环境温度
内部实体 id
MIoT siid/piid
```

这些信息仍然保留在完整能力图谱中，出故障或排查时才查询。

官方当前还有一层“Assist 暴露”设置。这个冰箱目前只把三个温度传感器设为暴露，设定温度和模式实体都没有暴露给 Assist。这说明：

```text
设备真实能力
≠
插件暴露到 HA 的能力
≠
HA 暴露给 Assist 的能力
≠
最终应暴露给 Agent 的能力
```

我们不能把所有层混成一张表，也不能只信任某一层的裁剪结果。

## 5 收缩的对象不只是工具数量

真正需要收缩的有六个维度。

### 5.1 设备数量

全屋有几十到上千个 Entity 时，默认只给 Agent：

```text
房间摘要
每台设备的名称、类型、可用状态
```

需要操作时再查具体设备。

### 5.2 实体数量

把一台设备下的多个实体合成一个“设备 profile”：

```text
12 个冰箱实体 → 1 个 refrigerator profile
1 个 climate 实体 → 1 个 air_conditioner profile
```

Entity 仍作为内部执行目标，不直接塞进模型上下文。

### 5.3 能力数量

每类设备只保留高频能力：

| 设备 | 默认能力 | 按需展开 | 默认隐藏 |
| --- | --- | --- | --- |
| 空调 | 开关、模式、目标温度、当前温度 | 风速、风向、预设、故障码 | 协议字段、内部 id |
| 冰箱 | 冷藏/冷冻目标温度、当前温度 | 模式、门事件、故障信息 | 压缩机转速、恢复出厂 |
| 灯 | 开关、亮度 | 色温、颜色、特效 | 原始属性 |
| 窗帘 | 开、关、位置 | 倾斜、方向、端点校准 | 协议字段 |
| 风扇 | 开关、档位 | 摆头、预设模式 | 内部状态 |
| 扫地机 | 开始、暂停、回充、状态 | 地图、清扫模式、耗材 | 调试字段 |
| 传感器 | 当前测量值 | 历史、统计、诊断 | 配置与原始消息 |

### 5.4 参数数量

参数 schema 只保留执行所需信息：

```text
target_temperature
  type=number
  min=7
  max=32
  step=0.5
  unit=°C
```

不需要把 `supported_features=385`、中文说明、图标和协议地址发给模型。

### 5.5 状态更新频率

虚拟时间、电流、功率、压缩机转速这类高频信号默认不进入对话上下文。Agent 需要时再查：

```text
当前值
最近一次变化
指定时间段统计
```

### 5.6 事件与历史

事件默认不应持续推给模型。

```text
冰箱门开
设备离线
洗衣机完成
门窗传感器触发
```

应先保留在事件索引里，由目标、订阅或安全策略决定是否唤醒 Agent。

## 6 建议的 MCP 收缩结构

### 6.1 内部保留完整能力图

```text
device
├─ physical identity
├─ integration / protocol
├─ entity list
├─ capabilities
├─ constraints
├─ events
├─ evidence source
└─ confidence
```

这一层可以不进入模型上下文，只存在本地索引或数据库。

### 6.2 对外只暴露设备 profile

```json
{
  "device_id": "living_room.ac",
  "profile": "air_conditioner",
  "name": "客厅空调",
  "room": "living_room",
  "available": true,
  "state": {
    "power": "on",
    "mode": "cool",
    "target_c": 24.0,
    "current_c": 26.0
  },
  "actions": {
    "set_power": ["on", "off"],
    "set_mode": ["cool", "heat"],
    "set_temperature": {
      "min": 7.0,
      "max": 32.0,
      "step": 0.5
    }
  }
}
```

### 6.3 提供按需展开入口

```text
observe_home
  房间和主要设备的极简状态

inspect_device
  当前设备的 profile、核心能力、异常

inspect_capabilities
  用户明确需要额外功能时展开风速、风向、模式、历史等

execute_capability
  只执行当前 profile 中明确列出的动作
```

这样固定 MCP 工具面可以很小，而设备动作由 `inspect_device` 的动态结果补充。

## 7 压缩的目标预算

建议不是按“所有 token 都不能超过多少”做硬编码，而是给不同视图设预算：

```text
全局家庭摘要
  每个主要设备一行，默认不展开属性

单设备默认 profile
  只包含当前状态 + 3~6 个核心动作

按需详情
  仅在明确任务需要时查询

历史与事件
  由查询触发，不进入固定系统提示
```

在本地当前这种小型环境里，`GetLiveContext` 一次全屋快照约 700 token 还能接受；设备规模扩大后，全量 dump 会线性膨胀，而设备 profile 的默认成本基本稳定。

## 8 官方实现与我们方案的分工

```text
官方 HA
  负责协议接入、设备注册、实体状态、服务调用

官方 MCP Server
  负责把 Assist 暴露的工具转成 MCP 工具

我们的能力层
  负责完整能力索引、设备 profile、按需展开和写后验证

我们的 MCP
  只暴露压缩后的设备视图与执行入口
```

这套分工允许我们继续利用 HA 的广覆盖，同时不把 HA 的全部内部字段和所有第三方集成能力直接塞给模型。

## 9 还需要保留的边界

```text
默认隐藏不等于删除
  长尾能力仍保存在能力图里，只是不进入固定上下文

设备 profile 不等于协议真相
  profile 是面向 Agent 的投影

官方 exposure 不等于安全策略
  当前冰箱设置温度未暴露给 Assist，并不代表它物理上不可控

压缩参数不等于限制能力
  可以先展示核心参数，再按任务展开长尾参数

不同型号不能总是共享同一 profile
  同一台空调若无风速能力，profile 也不能伪造 fan_speed
```

## 10 下一步

第一版可以按以下顺序实现：

```text
1 读取 HA 的 device/entity/state/service
2 为当前已接入设备生成完整 capability graph
3 实现 5 个 profile
   air_conditioner、refrigerator、light、cover、fan
4 为每个 profile 定义核心状态与核心动作
5 用 inspect_device 返回压缩 profile
6 用 inspect_capabilities 做按需展开
7 在 execute_capability 中加入范围校验与写后回读
8 用真实空调、冰箱和 SimuHome 设备验证
```

这套方案的核心取舍是：

```text
完整能力留在后台
高频能力放进固定视图
低频能力按需展开
安全策略独立于能力本体
```
