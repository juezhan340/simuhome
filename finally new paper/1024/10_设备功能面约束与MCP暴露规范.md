# 10：设备功能面约束与 MCP 暴露规范

日期：2026-10-27

## 结论

端侧 MCP 只保留两类可见功能面：

```text
common
  基础 + 高频功能，统一语义，默认暴露

custom
  型号特有、家庭自定义或低频但安全的功能，按需暴露
```

后台另有第三层：

```text
hidden
  恢复出厂、初始化、配对、升级、协议调试、内部诊断、原始字段写入
  始终不向端侧模型暴露
```

`hidden` 不属于 MCP 功能面分类，只属于后台存储。

```text
完整能力事实
  ↓
Device Profile
  ↓
common / custom 投影
  ↓
端侧模型

hidden
  × 端侧模型不可见
```

这套结构适合端侧模型：

```text
每类设备先看到少量稳定语义
常见任务全部由 common 完成
只有确实需要时才展开 custom
危险和维护操作不进入模型上下文
```

## 1 两类功能面

### 1.1 common

`common` 包含原来的基础信息和日常高频能力。

```text
common.base
  device_id
  profile
  name
  area
  online
  fault_summary

common.observe
  高频只读状态

common.control
  高频可写属性

common.action
  高频一次性动作

common.event
  高频事件
```

可见功能只有两类，但 `common` 内部仍要明确读、写、动作和事件。

### 1.2 custom

`custom` 用来承载：

```text
某品牌特有但语义清楚的能力
某住宅特有的组合动作
用户定义的场景
低频但安全的设备功能
```

例如：

```text
air_conditioner.sleep
refrigerator.quick_cool
television.movie_mode
lighting.home_mode
camera.privacy_mode
vacuum.clean_kitchen
```

Custom 不是原始能力的无序转储。每个 Custom 能力仍需要：

```text
稳定 ID
明确输入输出
参数范围
真实来源
安全等级
```

### 1.3 hidden

以下能力保留在后台，但永不进入端侧 MCP：

```text
factory_reset
initialize
pair
unpair
network_config
firmware_update
device_delete
integration_config
debug_action
raw_protocol_write
credentials
internal_test
```

设备的主电源关闭是否进入 common/custom，要按设备判断：

```text
电视 / 灯 / 风扇
  开关通常属于 common.control

冰箱 / 洗衣机
  整机断电可能中断冷冻、清洁或安全流程
  默认不暴露；确有需要时进入 custom 并二次确认

门锁 / 摄像头 / 燃气阀
  即使属于 common，也必须受安全策略约束
```

## 2 二十类设备的功能约束

下表是端侧视图的默认建议。每个实例只展示真实存在的能力。

| Profile | common.observe | common.control / action | custom 示例 |
|---|---|---|---|
| `air_conditioner` | 当前温度、开关状态、运行状态 | 开关、目标温度、模式、风速 | `sleep_mode`、`quick_cool`、`auto_comfort` |
| `refrigerator` | 各温区当前温度、门状态、模式 | 各温区目标温度、模式 | `quick_cool`、`quick_freeze`、`holiday_mode` |
| `television` | 电源、输入源、播放状态、标题 | 开关、音量、静音、输入源、播放/暂停 | `movie_mode`、`night_mode` |
| `light` | 开关、亮度、色温、颜色 | 开关、亮度、色温、颜色 | `reading_mode`、`movie_lighting` |
| `switch_plug` | 开关、功率、电量 | 开关 | `plug_timer`、`auto_cutoff` |
| `curtain_cover` | 当前位置、运行方向、开合状态 | 打开、关闭、停止、目标位置、倾斜位置 | `morning_open`、`privacy_position` |
| `fan` | 开关、风速、摆头 | 开关、风速、摆头、方向 | `sleep_wind`、`natural_wind` |
| `air_purifier` | PM2.5、滤芯寿命、模式 | 开关、模式、风速 | `auto_clean_air`、`quiet_mode` |
| `humidity_controller` | 当前湿度、水箱状态 | 开关、目标湿度、模式、风速 | `dry_laundry`、`sleep_humidity` |
| `vacuum` | 电量、状态、地图、当前房间 | 开始、暂停、继续、回充、定点清扫 | `clean_kitchen`、`deep_clean` |
| `washer` | 状态、剩余时间、门锁、故障 | 选择程序、开始、暂停、停止 | `quick_wash`、`night_wash` |
| `dryer` | 状态、剩余时间、滤网状态 | 选择程序、开始、暂停、停止 | `quick_dry`、`anti_wrinkle` |
| `dishwasher` | 状态、剩余时间、门状态 | 选择程序、开始、暂停、停止 | `eco_wash`、`quick_wash` |
| `water_heater` | 当前水温、加热状态、在线 | 开关、目标温度、模式 | `shower_ready`、`away_mode` |
| `lock` | 锁状态、电量、最近事件 | 上锁、解锁 | `lock_all`、`guest_access` |
| `camera` | 在线、录像状态、隐私模式 | 实时画面、抓图、录像或隐私模式 | `patrol_home`、`watch_entry` |
| `speaker` | 开关、播放状态、音量、静音 | 音量、静音、播放、暂停、TTS/广播 | `announce_dinner`、`music_mode` |
| `thermostat` | 当前温度、目标温度、运行状态 | 开关、目标温度、模式 | `away_heating`、`morning_warmup` |
| `door_window_sensor` | 开/关、电量、在线、最近变化 | 无 | `door_open_warning`、`night_security` |
| `environment_sensor` | 温度、湿度、PM2.5、CO2、空气质量 | 无 | `air_alert`、`comfort_check` |

这个表的目的不是规定所有品牌能力完全相同，而是固定端侧模型使用的标准槽位。

```text
Profile 定义允许出现什么
设备实例决定实际有什么
```

## 3 Entity 内部复杂性

HA 的 Entity 是适配层对象，不能直接当作最终语义能力写给模型。

### 3.1 Entity 和 Capability 不是一一对应

```text
一个 Entity 可以包含多个 Capability
  climate
    on/off
    target_temperature
    hvac_mode
    fan_mode
    swing_mode

一个 Capability 也可能跨多个 Entity
  refrigerator.target_temperature
    冷藏室 number
    冷冻室 number
    变温室 number

一个 Entity 可能只是事件或动作
  event
  button
```

因此不能把 HA Entity 原样转换成 Tool。

### 3.2 一台设备对应多个 Entity

当前米家冰箱就是一个直接例子：

```text
同一 Device
  ├─ sensor 冷藏室当前温度
  ├─ sensor 冷冻室当前温度
  ├─ number 冷藏室目标温度
  ├─ number 冷冻室目标温度
  ├─ select 冰箱模式
  ├─ event 门开提醒
  ├─ sensor 压缩机转速
  └─ button 恢复出厂设置
```

端侧模型应当看到：

```text
一台 refrigerator
```

模型不需要看到每个 `midjd6_...` entity ID。

### 3.3 Domain 不等于设备类型

```text
switch 可能是
  插座
  灯
  空气净化器
  洗衣机

climate 可能是
  空调
  恒温器
  地暖

number 可能是
  目标温度
  亮度校准
  童锁延时
```

Device Profile 必须先由以下信息共同判断：

```text
integration
device registry 中的 manufacturer / model
device_class
entity domain 组合
entity names
configuration entry
```

单看 domain 会产生错误语义。

### 3.4 同一 Entity 的能力藏在多个字段里

以 `climate` 为例：

```text
state
  off / cool / heat / auto / dry / fan_only

attributes.current_temperature
attributes.temperature
attributes.hvac_modes
attributes.fan_modes
attributes.swing_modes
supported_features
supported services
```

最终要拆成：

```text
power               write
current_temperature read
target_temperature  read/write
hvac_mode           read/write
fan_speed            read/write
swing_mode           read/write, custom
```

`state` 和 `attributes` 不能只作为一段 JSON 塞给模型。

### 3.5 supported_features 是位掩码

HA 的 `supported_features` 可能同时表示多个能力：

```text
climate
  TARGET_TEMPERATURE
  FAN_MODE
  SWING_MODE

cover
  OPEN
  CLOSE
  SET_POSITION
  STOP

media_player
  VOLUME_SET
  PLAY
  PAUSE
  NEXT_TRACK
```

转换时要把位掩码解码成 capability，不能把整数直接交给模型。

### 3.6 服务决定可写能力

读属性和写能力来自不同位置：

```text
state / attributes
  当前值和可选值

domain service
  是否能写

entity_id + service data
  实际调用路径
```

例如：

```text
number.set_value
  写数值

select.select_option
  写枚举

climate.set_temperature
  写目标温度

cover.set_cover_position
  写位置

lock.unlock
  解锁动作
```

可写性必须结合服务确认。

### 3.7 状态值存在不可用和缺失

```text
unavailable
unknown
None
空 attributes
```

这些状态不能解释为设备没有该能力。

```text
能力是否存在
  来自 registry / service / profile

能力当前是否可用
  来自 state / online / availability
```

两者必须分开。

### 3.8 Entity ID 不稳定

Entity ID 可能随名称、重命名、迁移或用户修改变化：

```text
number.midjd6_..._target_temperature_p_3_2
```

端侧语义不应依赖 entity_id。

```text
稳定层
  device_id + capability_id

适配层
  entity_id
```

### 3.9 不同协议暴露方式不同

```text
MIoT
  属性、事件、动作，由 SPEC 驱动

Matter
  cluster / attribute / command

Zigbee / Z-Wave
  cluster / command class

Tuya
  category / DPCode / status_range

Midea
  设备类型 / 字节协议 / 固定属性
```

最终都要收敛为同一套 capability。

## 4 从 Entity 到 Capability

### 4.1 先按 Device 聚合 Entity

```text
遍历 HA Entity Registry
  ↓
按 device_id 分组
  ↓
一个 device_id 得到一组相关 entity
```

示例：

```text
device_id = 冰箱设备
  sensor.*temperature_p_3_1
  sensor.*temperature_p_4_1
  number.*target_temperature_p_3_2
  number.*target_temperature_p_4_2
  select.*mode_p_2_2
  event.*fault_open_e_8_1
  button.*reset_a_7_1
```

后续所有转换都在这台 Device 上进行。

### 4.2 判定 Device Profile

建议优先级：

```text
1 设备注册表中的型号/设备类显式映射
2 插件提供的 device_class / category / model 规则
3 实体组合模式
4 integration 默认推测
5 用户手工覆盖
```

例如：

```text
model = midjd6...
integration = xiaomi_home
entities 包含冷藏/冷冻 target-temperature
  → refrigerator

entity = climate.*
attributes.hvac_modes 包含 cool/heat
  → air_conditioner

entity = media_player.*
device_class = tv
  → television
```

判定结果要允许用户覆盖，并提供置信度或来源。

### 4.3 Entity 类型到能力原语

| HA Entity | 读取能力 | 写入/动作能力 |
|---|---|---|
| `sensor` | 数值、文本或枚举状态 | 无 |
| `binary_sensor` | 是否状态 | 无 |
| `number` | 当前数值 | `set_value` |
| `select` | 当前选项 | `select_option` |
| `switch` | 开关状态 | `turn_on` / `turn_off` |
| `text` | 文本 | `set_value`，通常不暴露 |
| `event` | 事件类型和属性 | 无 |
| `button` | 无持久状态 | `press` |
| `climate` | 当前温度、目标温度、模式 | `turn_on` / `turn_off` / `set_temperature` / `set_hvac_mode` / `set_fan_mode` |
| `cover` | 位置、状态、倾斜位置 | `open` / `close` / `stop` / `set_position` / `set_tilt_position` |
| `fan` | 开关、百分比、预设、摆头 | 对应 set 服务 |
| `lock` | 锁定状态、电量 | `lock` / `unlock` |
| `media_player` | 播放状态、音量、来源 | 播放、暂停、音量、静音、源切换 |
| `camera` | 在线、录像状态 | 获取画面、抓图、录制控制 |
| `vacuum` | 电量、状态、地图 | 开始、暂停、回充、定点清洁 |
| `humidifier` | 当前湿度、目标湿度、模式 | 开关、目标湿度、模式 |
| `water_heater` | 当前温度、目标温度、模式 | 开关、设定温度 |

`notify`、`todo` 等非设备域 Entity 不直接进入 Device Profile，除非它们被绑定到明确的设备或家庭任务语义。

### 4.4 能力统一记录

```json
{
  "capability_id": "target_temperature",
  "profile": "refrigerator",
  "device_id": "midjd6_...",
  "surface": "common",
  "access": "write",
  "value": {
    "type": "number",
    "unit": "°C",
    "minimum": 2,
    "maximum": 8,
    "step": 1
  },
  "constraints": {
    "zone": ["fridge", "freezer"]
  },
  "source": {
    "entity_id": "number.midjd6_..._target_temperature_p_3_2",
    "domain": "number",
    "service": "set_value"
  },
  "safety": "normal"
}
```

模型看到的是：

```text
capability_id
profile
zone
current value
valid range
```

Host 使用的是：

```text
entity_id
service
attribute path
protocol adapter
```

### 4.5 转换算法

```text
for device in devices:
  profile = identify_profile(device, entities)

  for entity in entities:
    capabilities += extract_capabilities(entity)

  capabilities = capabilities ∩ profile.allowed_capabilities
  capabilities = attach_instance_constraints(capabilities)
  capabilities = classify_surface(capabilities, profile)
  capabilities = hide_hidden_and_unsafe_raw_capabilities(capabilities)

  device_view = build_common_view(capabilities)
  custom_view = build_custom_view(capabilities)
```

### 4.6 Common 的判定顺序

```text
1 profile 规定该能力是否属于 common
2 设备实例确认能力真实存在
3 状态/属性补齐当前值和约束
4 安全策略确认能否暴露
```

示例：

```text
Refrigerator Profile 允许 target_temperature
  ↓
实例存在冷藏、冷冻两个 number
  ↓
补齐各温区当前值和范围
  ↓
进入 common.control
```

如果实例没有制冰机：

```text
custom.quick_ice 不生成
```

不能为了保持 JSON schema 一致而制造空能力。

### 4.7 Custom 的判定

```text
设备真实具备
  +
Profile 承认其语义
  +
不属于 hidden
  +
不是默认高频
  ↓
custom
```

用户可以叠加组合动作：

```text
custom.movie_mode
  电视输入切到 HDMI2
  音量设为 18
  客厅灯亮度设为 20%
  窗帘关闭
```

组合动作可以落到多个设备，因此 Custom 不一定是单设备能力。

### 4.8 冲突和覆盖规则

```text
用户显式配置
  > Profile 自动判定

Profile 标准能力
  > integration 临时字段

真实实例约束
  > 静态默认范围

安全策略
  > common/custom 可见性

设备不可用
  保留能力定义
  但标记 available=false
```

写操作失败后必须回读；模型的口头成功不能作为 verified。

## 5 Entity 实例示例

### 5.1 米家冰箱

原始实体：

```text
number.*target_temperature_p_3_2
  state = 5
  friendly_name = 冷藏室 设定温度

number.*target_temperature_p_4_2
  state = -17
  friendly_name = 冷冻室 设定温度

sensor.*temperature_p_3_1
  state = 4

sensor.*temperature_p_4_1
  state = -20

select.*mode_p_2_2
  state = 手动

event.*fault_open_e_8_1
  门开提醒
```

转换后：

```json
{
  "profile": "refrigerator",
  "name": "米家对开门冰箱",
  "common": {
    "observe": {
      "temperature": {
        "fridge": 4,
        "freezer": -20
      },
      "mode": "手动"
    },
    "control": {
      "target_temperature": {
        "fridge": {
          "value": 5,
          "minimum": 2,
          "maximum": 8,
          "step": 1
        },
        "freezer": {
          "value": -17,
          "minimum": -24,
          "maximum": -16,
          "step": 1
        }
      },
      "mode": {
        "values": ["手动", "智能", "假日"]
      }
    },
    "events": ["door_open_alert"]
  },
  "custom": [
    "quick_cool",
    "quick_freeze"
  ]
}
```

压缩机转速、故障码、恢复出厂不进入这个投影。

### 5.2 空调

```text
原始：
  climate.living_room
    state = cool
    current_temperature = 27
    temperature = 24
    hvac_modes = [off, cool, heat, dry, fan_only]
    fan_modes = [low, medium, high, auto]
    supported_features = TARGET_TEMPERATURE | FAN_MODE

转换：
  common.observe.current_temperature
  common.control.power
  common.control.target_temperature
  common.control.hvac_mode
  common.control.fan_speed
```

摆风具体是 vertical、horizontal 还是固定档位，由实际 capabilities 决定；没有该能力就不生成。

### 5.3 电视

```text
原始：
  media_player.tv
    state = playing
    volume_level = 0.35
    source = HDMI 2
    supported_features = TURN_ON | VOLUME_SET | PLAY | PAUSE

转换：
  common.observe.playback_state
  common.observe.source
  common.observe.volume
  common.control.power
  common.control.volume
  common.action.play
  common.action.pause
```

字幕、音轨、画质模式只有在真实支持时才作为 custom 加入。

### 5.4 门磁

```text
原始：
  binary_sensor.front_door
    device_class = door
  sensor.front_door_battery
  event.front_door_opened

转换：
  common.observe.door_state
  common.observe.battery
  common.event.door_opened

  common.control = 空
```

读取型设备不会被伪装成可控制设备。

## 6 MCP 输出结构

### 6.1 inspect_device

```json
{
  "device_id": "midjd6_...",
  "profile": "refrigerator",
  "common": {
    "observe": {},
    "control": {},
    "action": {},
    "event": {}
  },
  "custom": {
    "count": 2,
    "names": ["quick_cool", "quick_freeze"]
  }
}
```

Common 默认展开，Custom 默认只列名称。

### 6.2 inspect_capabilities

```json
{
  "device_id": "midjd6_...",
  "surface": "custom"
}
```

返回 Custom 的完整参数和约束。

### 6.3 execute_capability

```json
{
  "device_id": "midjd6_...",
  "capability": "target_temperature",
  "arguments": {
    "zone": "fridge",
    "value": 5
  }
}
```

Host 负责：

```text
校验 profile
  ↓
校验实例能力
  ↓
校验 common/custom 可见性
  ↓
校验参数
  ↓
映射到真实 HA service
  ↓
回读真实状态
  ↓
返回 verified 结果
```

## 7 存储和训练

### 7.1 存储

后台完整保留：

```text
device
entity
state
attributes
supported_features
services
integration
model
protocol metadata
event history
```

另外增加：

```text
device_profile
capability_mapping
custom_capability
visibility_override
safety_policy
```

不要删除原始 entity 和协议字段。

### 7.2 训练语义

Common 是稳定训练面：

```text
air_conditioner.target_temperature
refrigerator.target_temperature
light.brightness
cover.position
television.volume
```

模型学到的是跨品牌一致的“设备 + 能力 + 参数”语义。

Custom 只作为补充：

```text
不应成为所有设备必须具备的固定输出
训练时要记录具体 profile 和能力是否存在
要允许 empty custom
```

这样不会因为品牌差异导致输出空间爆炸。

## 8 必须保留的边界

```text
Profile 是可见性约束，不是能力真相
  真实能力始终来自实例和来源数据

Common 是高任务频率，不代表低风险
  门锁、摄像头等仍要二次确认

Custom 是安全且语义清楚的扩展
  不是所有剩余 Entity 的垃圾桶

Hidden 永不进入端侧
  恢复出厂、初始化、配对、升级、原始协议写入全部隐藏

状态不可用不等于能力不存在
  能力与 availability 分开表达

写入成功不等于目标达成
  必须由 Host 回读验证
```

## 9 最终判断

```text
接入方式可以继续沿用 HA 插件
  Matter / Xiaomi / Midea / Tuya 等协议差异留在适配层

后台保留完整能力
  不因端侧压缩而删除 Entity 和协议信息

端侧只暴露两类功能面
  common：基础和高频能力
  custom：安全、清楚的扩展能力

恢复出厂、初始化、协议调试、内部诊断
  永远不进入端侧 MCP

Entity 先转换为 Capability
  Device Profile 再做约束
  最后生成稳定的 common/custom 视图
```

这能满足“用同一套语义进行训练”，同时控制端侧模型的工具面、上下文大小和误操作风险。
