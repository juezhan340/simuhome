# L3 四个 Benchmark 与 Miloco 完整工具面

## 0 文档边界

现有 `13_论文中的设备能力约束与Matter组件分析.md` 讨论的是设备如何被约束，本文不改写、不覆盖它。`L3` 专门整理模型直接可见的工具或函数接口。

这里的“完整”指论文或项目公开的模型侧工具面：

```text
包括
├─ Tool / Function 名称
├─ 主要参数
├─ 作用
└─ 是否属于记忆接口

不包括
├─ 后端内部 REST API
├─ 未暴露给模型的数据库表
├─ 所有 OpenClaw Host 通用工具
└─ Skill 内部的全部 CLI 子命令
```

五个对象的工具面规模差异很大：

| 对象 | 固定工具数量 | 接口形式 | 记忆接口 |
|---|---:|---|---|
| HomeBench | 无固定全局数量 | 每台设备动态给出方法签名 | 无 |
| PersonalHomeBench | 28 个辅助函数 + 6 个计划生成动作 | Tool-Box 函数与计划步骤 | 读取：`get_household_memories`、`get_member_memories`；写入：`update_memories` |
| SMH-Bench | 3 个 | 三种设备控制工具 | 无；`memory_list` 是上下文 |
| SimuHome | 最新英文稿 17 个 | 环境查询、Matter 命令、工作流工具 | 无 |
| Miloco | 自有 3 个；另依赖 Host 2 个记忆工具 | Plugin Tool + Skill/CLI | Host `memory_search/get` |

## 1 HomeBench：设备方法就是接口

HomeBench 没有 `query_device`、`control_device` 这样的统一工具名。它把当前家庭每台设备的方法签名直接放进提示词，模型输出：

```text
<房间>.<设备>.<方法>(参数)
```

非法设备、属性或方法统一输出：

```text
error_input
```

### 1.1 基础设备方法全集

论文表 7 给出的 15 类设备及基础方法如下。具体家庭还会随机增加方法，所以这不是一个跨环境永久固定的函数注册表。

| 设备类型 | 论文给出的基础方法 |
|---|---|
| 灯光设备 | `turn_on`、`turn_off`、`set_brightness`、`set_color` |
| 空调设备 | `turn_on`、`turn_off`、`set_temperature`、`set_mode`、`set_fan_speed`、`set_swing` |
| 加热设备 | `turn_on`、`turn_off`、`set_temperature`、`set_mode`、`set_fan_speed` |
| 风扇设备 | `turn_on`、`turn_off`、`set_speed`、`set_swing` |
| 车库门设备 | `open`、`close` |
| 百叶窗设备 | `open`、`close` |
| 窗帘设备 | `open`、`close`、`set_openness` |
| 空气净化器设备 | `turn_on`、`turn_off`、`set_mode`、`set_fan_speed` |
| 热水器设备 | `turn_on`、`turn_off`、`set_temperature`、`set_mode` |
| 媒体播放器设备 | `play`、`pause`、`stop`、`set_volume`、`set_song`、`set_artist`、`set_genre` |
| 扫地机器人设备 | `start`、`pause`、`stop`、`charge`、`set_mode`、`set_cleaning_area` |
| 香薰设备 | `turn_on`、`turn_off`、`set_intensity`、`set_interval` |
| 垃圾桶设备 | `pack` |
| 加湿器设备 | `turn_on`、`turn_off`、`set_intensity`、`set_mode` |
| 除湿器设备 | `turn_on`、`turn_off`、`set_intensity`、`set_mode` |

上面的英文方法名是按论文中文译名归一化后的写法。HomeBench 的评测重点是模型是否选中合法设备、方法及参数，而不是 MCP Tool 名称是否固定。

### 1.2 模型看到的上下文

```text
<家庭状态>
房间、设备、可调属性、阈值、当前值
</家庭状态>

<设备方法>
房间.设备.方法(参数)
</设备方法>

<用户指令>
...
</用户指令>
```

模型输出一个或多个机器指令。它没有记忆查询步骤。

### 1.3 例子

用户：

```text
把主卧灯的亮度设置为 50。
```

模型接口：

```text
主卧.灯.set_brightness(50)
```

这属于设备方法调用，不属于 Tool-Box 查询接口。

## 2 PersonalHomeBench：28 个辅助函数与 6 个计划生成动作

PersonalHomeBench 的最新论文附录 L 给出统一的 PersonalHomeTools 函数表。样例提示词只展示其中一个常用子集，完整表包含 28 个辅助函数。

### 2.1 家庭、成员与记忆

| 函数 | 主要参数 | 返回或用途 |
|---|---|---|
| `get_household_profile()` | 无 | 家庭规模、居住类型、布局、宠物、成员 ID |
| `get_household_member_info(member_ids)` | `member_ids: List[str]` | 成员姓名、角色、年龄等画像 |
| `get_household_member_major_events(member_id)` | `member_id` | 单个成员的重大人生事件 |
| `get_household_memories(search_string, num_memories)` | 查询文本、返回数量 | 搜索家庭级记忆 |
| `get_member_memories(member_id, search_string, num_memories)` | 成员 ID、查询文本、返回数量 | 搜索成员级记忆 |

记忆数据结构是：

```json
{
  "memory": "爸爸希望冷藏室平时保持 4°C",
  "subjects": ["dad_id"],
  "additional_information": "温度偏好",
  "time": "2026-10-31"
}
```

### 2.2 家电与传感器

| 函数 | 主要参数 | 返回或用途 |
|---|---|---|
| `get_owned_appliances()` | 无 | 家庭内全部家电及类型、位置 |
| `get_owned_sensors()` | 无 | 家庭内全部传感器及类型、位置 |
| `get_weather(...)` | 室内/室外筛选 | 当前天气或环境读数 |
| `get_appliances(appliance_names)` | 家电名列表 | 指定家电的详细状态 |
| `get_sensors(sensor_names)` | 传感器名列表 | 指定传感器状态或日志 |
| `get_appliances_by_location(locations)` | 地点列表 | 指定地点内的家电 |
| `get_sensors_by_location(locations)` | 地点列表 | 指定地点内的传感器 |
| `get_appliance_and_sensor_events(...)` | 查询或时间条件 | 按时间排序的家电与传感器事件 |
| `get_appliance_features(appliance_names)` | 家电名列表 | 支持的功能及参数 |
| `run_appliance_feature(appliance_name, feature_method, feature_kwargs)` | 家电、功能、参数 | 执行具体家电特征 |
| `update_appliance(appliance_name, state)` | 家电名、状态字典 | 更新家电状态 |

PersonalHomeBench 的设备控制不是统一 MCP 工具，而是：

```text
先查 appliance
  ↓
查 appliance feature
  ↓
run_appliance_feature
```

不同家电可以有不同 feature 和参数。

### 2.3 事件、空间、人物与多模态

| 函数 | 主要参数 | 返回或用途 |
|---|---|---|
| `get_contextual_event_descriptions(compact, sort_by_time, indent)` | 压缩、排序、缩进选项 | 当前家庭事件的文字描述 |
| `get_scene_summary()` | 无 | 源视频中的家庭场景摘要 |
| `get_layout_zones()` | 无 | 家庭空间区域列表 |
| `get_landmarks_in_house()` | 无 | 屋内固定地标 |
| `get_members_appearance(member_ids)` | 成员 ID 列表 | 成员外观描述 |
| `get_appliance_descriptions(appliance_names)` | 家电名列表 | 家电自然语言描述 |
| `get_object_information()` | 无 | 环境内物体信息的扁平 JSON |
| `extract_captions(start, end, query, return_text, join_with)` | 时间段、查询、输出格式 | 按时间或语义提取视频字幕 |
| `extract_event_captions(num_events, query)` | 事件数、查询 | 最近或相关事件字幕 |
| `inspect_video_segment(start, end)` | 秒级开始和结束 | 生成视频片段并返回文件句柄 |
| `get_audio_events(start, end, query)` | 时间段和查询 | 音频事件片段 |
| `extract_frames(start, end, fps)` | 时间范围和帧率 | 抽取视频帧，最多建议 300 帧 |

### 2.4 模型交互特点

```text
提示词中可先给相关记忆 {memories}
  ↓
模型信息不足时调用家庭/成员记忆工具
  ↓
再调用家电、传感器、事件或视频工具
  ↓
最后回答或规划操作
```

PersonalHomeBench 的工具使用规则要求逐步调用：

```text
call tool
  ↓
read TOOL_RESPONSE
  ↓
decide next tool
```

它不支持预先假设工具返回，也不支持把多次未执行的工具调用当作已经得到结果。

### 2.5 计划生成动作面

PersonalHomeBench 的主动计划生成提示词另有一套 “Toolbox for generating plans”。它不是前面 PersonalHomeTools 的检索函数表，而是模型生成主动程序时可以使用的动作词汇：

| 动作 | 参数 | 作用 |
|---|---|---|
| `update_user_information` | `user: HouseholdMember` | 更新成员姓名、年龄、职业等资料 |
| `update_memories` | `household_id: str`、`memories: List[Memory]` | 添加或更新家庭记忆 |
| `update_sensor_status` | `sensor_name`、`sensor_location`、`status` | 更新传感器状态 |
| `update_appliance_status` | `appliance_name`、`appliance_location`、`status` | 更新家电状态 |
| `update_pet_information` | `pet: Pet`、`household_id: str` | 更新宠物信息 |
| `create_routine` | `List[Function]` | 把一组功能串成例程并返回例程名称 |

`update_memories` 的调用示意：

```json
{
  "program_id": "record_fridge_preference",
  "tool": "update_memories",
  "args": {
    "household_id": "household_001",
    "memories": [
      {
        "memory": "爸爸希望冰箱冷藏室平时保持 4°C",
        "subjects": ["dad_id"],
        "additional_information": "温度偏好",
        "time": "2026-11-01"
      }
    ]
  }
}
```

提示词要求模型谨慎、低频使用 `update_user_information`、`update_memories` 和宠物更新，避免把主动计划变成频繁改档案。

## 3 SMH-Bench：只有三个直接工具

SMH-Bench 的 Environment-Interactive Agent 设置中，模型直接看到三个工具。

### 3.1 `query_device`

用途：

```text
查找设备
查能力
查状态
同时查能力与状态
```

参数：

| 参数 | 是否必填 | 作用 |
|---|---:|---|
| `what` | 是 | `brief`、`spec`、`status`、`spec_status` |
| `did` | 条件必填 | 已确定设备时使用 |
| `room` | 条件必填 | 按房间查找 |
| `tags` | 可选 | 设备类型或特征过滤 |

四种模式：

```text
brief        查候选设备
spec         查服务、参数类型和取值范围
status       查当前状态
spec_status  同时查能力与状态
```

### 3.2 `control_device`

```json
{
  "did": "fridge_101",
  "locator": "fresh.set_target_temperature",
  "arguments": {
    "temperature": 5
  }
}
```

参数：

| 参数 | 作用 |
|---|---|
| `did` | 设备 ID |
| `locator` | 服务名或 `component.service` |
| `arguments` | 与目标服务参数严格匹配 |

### 3.3 `create_automation`

```json
{
  "conditions": {
    "time_cron": "0 0 20 ? * * *",
    "state_expr": null
  },
  "actions": [
    {
      "did": "fridge_101",
      "locator": "fresh.set_target_temperature",
      "arguments": {
        "temperature": 5
      }
    }
  ]
}
```

它处理定时或条件触发任务，不处理立即控制。

### 3.4 记忆不是工具

```text
## User Memory
{{memory_list}}
```

`memory_list` 是提示词变量。模型没有第四个记忆工具；它只把提示词里已有的短期或长期记忆用于补全当前动作。

## 4 SimuHome：最新英文稿的 17 个工具

本节以本地较新的 ICLR 2026 英文稿 `simuprocject/07_2025_arXiv_SimuHome-Temporal-Environment_智能家居代理基准_英.txt` 的 Appendix A 为准。

本地 `0524` 中译本对应较早版本，只有 13 个工具。最新英文稿增加了：

```text
get_environment_control_rules
ask_user
cancel_workflow
get_workflow_status
```

### 4.1 完成与澄清

| 工具 | 参数 | 作用 |
|---|---|---|
| `finish` | `answer: str` | 完成当前回合并返回最终自然语言答案 |
| `ask_user` | `question: str` | 信息不足、目标含糊或偏好不明确时向用户提问 |

### 4.2 房间与环境

| 工具 | 参数 | 作用 |
|---|---|---|
| `get_environment_control_rules` | `state: str` | 查询温度、湿度等环境状态的控制规则 |
| `get_rooms` | 无 | 返回家庭全部房间及显示名 |
| `get_room_devices` | `room_id: str` | 返回房间内设备 |
| `get_room_states` | `room_id: str` | 返回房间的温度、湿度、照度和 PM10 |

`get_room_devices` 是动作前验证房间内设备是否存在的关键工具。SimuHome 会把必要的查询动作列入 prerequisite actions。

### 4.3 Matter 设备

| 工具 | 参数 | 作用 |
|---|---|---|
| `execute_command` | `device_id`、`endpoint_id`、`cluster_id`、`command_id`、`args` | 执行 Matter 命令 |
| `write_attribute` | `device_id`、`cluster_id`、`attribute_id`、`value` | 直接写属性 |
| `get_all_attributes` | `device_id` | 读取设备全部属性 |
| `get_attribute` | `device_id`、`cluster_id`、`attribute_id` | 读取指定属性 |
| `get_device_structure` | `device_id` | 返回端点、集群、属性和命令结构 |
| `get_cluster_doc` | `query: str`、`top_k: int` | 在 Matter Cluster 文档中做语义搜索 |

设备层级是：

```text
Device
└─ Endpoint
   └─ Cluster
      ├─ Attribute
      └─ Command
```

### 4.4 时间与工作流

| 工具 | 参数 | 作用 |
|---|---|---|
| `get_current_time` | 无 | 获取虚拟家庭的当前时间 |
| `schedule_workflow` | `start_time: str`、`steps: list` | 在未来绝对时间注册顺序工作流 |
| `cancel_workflow` | `workflow_id: str` | 取消已注册工作流 |
| `get_workflow_status` | `workflow_id: str` | 查工作流状态 |
| `get_workflow_list` | 无 | 查工作流列表 |

`schedule_workflow` 的参数形状：

```json
{
  "start_time": "2026-11-01 18:30:00",
  "steps": [
    {
      "tool": "execute_command",
      "parameters": {
        "device_id": "kitchen_light",
        "endpoint_id": 1,
        "cluster_id": "OnOff",
        "command_id": "On",
        "args": {}
      }
    }
  ]
}
```

这是按字段含义构造的示意，字段名以当前代码实现为准；论文表直接给出的必需字段是 `start_time` 和 `steps`。

### 4.5 一次时间任务

用户：

```text
洗碗机结束后打开厨房灯。
```

模型一般需要：

```text
get_rooms
  → get_room_devices
  → get_device_structure / get_attribute
  → get_current_time
  → schedule_workflow
  → finish
```

它没有记忆检索工具。所有判断来自当前设备和环境观测。

## 5 Miloco：自有工具、Host 工具与 Skill

### 5.1 Miloco 插件直接注册的工具

Miloco 插件清单只声明三个直接工具：

```json
{
  "contracts": {
    "tools": [
      "miloco_im_push",
      "miloco_notify_bind",
      "miloco_habit_suggest"
    ]
  }
}
```

#### `miloco_im_push`

| 参数 | 必填 | 作用 |
|---|---:|---|
| `message` | 是 | 要推送给主人的通知正文 |
| `bindHint` | 可选 | 首次通知没有绑定渠道时，附带的通道绑定引导 |

#### `miloco_notify_bind`

| 参数 | 必填 | 作用 |
|---|---:|---|
| `sessionKey` | 可选 | 指定通知渠道；留空使用当前会话 |

#### `miloco_habit_suggest`

| 参数 | 说明 |
|---|---|
| `action` | 必填；`list`、`record`、`mark_asked`、`resolve` |
| `key` | 习惯建议的稳定语义键 |
| `subject` | 成员名或 `shared` |
| `habit` | 观察到的习惯 |
| `suggestion` | 推荐给用户的任务点子 |
| `title` | 建议标题 |
| `evidence` | 建议依据 |
| `item_id` | 来源家庭档案条目 ID |
| `outcome` | `accepted`、`rejected`、`created` |
| `task_id` | 任务创建成功后的任务 ID |
| `reason` | 拒绝原因 |

状态机：

```text
pending
  ├─ mark_asked → asked
  │                ├─ resolve(created) → created
  │                ├─ resolve(rejected) → rejected
  │                └─ 超时 → expired
  └─ 超时 → expired
```

### 5.2 OpenClaw Host 提供的记忆工具

这两个工具来自 OpenClaw `memory-core`，不是 Miloco 插件注册：

| 工具 | 参数 | 作用 |
|---|---|---|
| `memory_search` | `query` 必填；`maxResults`、`minScore`、`corpus` 可选 | 语义搜索 Markdown 记忆 |
| `memory_get` | `path` 必填；`from`、`lines`、`corpus` 可选 | 按文件路径和行范围精读记忆 |

`memory_search.corpus`：

```text
memory
wiki
all
sessions
```

`memory_get.corpus`：

```text
memory
wiki
all
```

### 5.3 Skill 不是 Tool

Miloco 2.0 有 16 个 Skill，它们是模型按需加载的操作说明：

| Skill | 用途 |
|---|---|
| `miloco-devices` | 查设备、控制设备、场景和错误处理 |
| `miloco-perception` | 摄像头视听感知与近期事件查询 |
| `miloco-perception-digest` | 每 15 分钟压缩原始感知日志 |
| `miloco-miot-identity` | 人/宠物身份查询与管理 |
| `miloco-miot-identity-register` | 注册新成员身份 |
| `miloco-miot-admin` | 系统管理与成本统计 |
| `miloco-miot-scope` | 感知范围和作用域配置 |
| `miloco-create-task` | 创建、修改任务及关联 cron |
| `miloco-terminate-task` | 终止任务并清理关联 cron |
| `miloco-notify` | 通知分级、选人和渠道决策 |
| `miloco-home-profile` | 家庭档案查询、增删改和 commit |
| `miloco-home-observe` | 从感知/交互记忆中提取候选知识 |
| `miloco-home-promote` | 晋升候选知识 |
| `miloco-home-prune` | 合并主体、清理过期、提交档案 |
| `miloco-home-patrol` | 用感知记忆和家庭档案主动巡检 |
| `miloco-habit-suggest` | 从正式档案生成习惯建议 |

模型看到 Skill 名称和说明，真正执行仍依赖 Host 的命令执行工具或 Skill 规则中的 CLI。

### 5.4 CLI 不是专用 Tool

家庭档案的操作路径：

```text
模型读取 Skill
  ↓
执行 miloco-cli
  ↓
调用 Miloco 后端 API
```

常用命令：

```bash
miloco-cli home-profile list --target profile
miloco-cli home-profile list --target both
miloco-cli home-profile profile-write --user-edit --ops '[...]'
miloco-cli home-profile candidate-write --ops '[...]'
miloco-cli home-profile commit
miloco-cli home-profile reassign --mappings '[...]'
```

这些 CLI 命令不是 MCP Tool。它们通过 Host 的命令执行能力运行。

## 6 五个对象的工具调用差异

```text
HomeBench
  动态设备方法
  → 直接输出一个或多个方法调用

PersonalHomeBench
  固定 Tool-Box
  → 根据需要逐个调用工具
  → 每次读 TOOL_RESPONSE 再继续

SMH-Bench
  三个领域工具
  → query_device
  → control_device / create_automation

SimuHome
  Matter 环境工具
  → 查结构/属性/时间
  → execute_command / write_attribute / schedule_workflow
  → finish

Miloco
  原生 Plugin Tool 很少
  → 大量操作由 Skill 指导 miloco-cli
  → 记忆搜索由 OpenClaw Host 工具负责
```

## 7 工具面对我们意味着什么

从模型需要处理的接口数量看：

```text
SMH-Bench
  3 个工具最容易做小模型适配
  但工具内部参数仍然复杂

HomeBench
  没有固定全局工具数
  设备方法会随着家庭动态变化

PersonalHomeBench
  28 个辅助函数信息完整
  对端侧模型明显过宽

SimuHome
  17 个工具覆盖 Matter 结构、状态、时间和工作流
  查询能力完整，但发现成本高

Miloco
  直接 Tool 少
  把复杂设备流程放进 Skill 和 CLI
  对宿主能力要求更高
```

对我们自己的语义 MCP，可收缩的方向是：

```text
模型常用面
  查设备
  查能力
  查状态
  控设备
  建自动化

模型按需面
  设备结构
  详细属性
  Matter 文档
  视频和音频
  工作流历史

模型不直接面
  初始化
  恢复出厂
  原始后端 API
  全部 Skill 运维命令
```

## 8 证据位置

HomeBench：

```text
finally new paper/0524_4个bench/01_2024_ACM-CHI_HomeBench-Evaluating-LLMs_HomeBench大模型评估_中.txt
```

PersonalHomeBench：

```text
finally new paper/0524_4个bench/02_2026_arXiv_PersonalHomeBench-Personalized-Eval_个性化智能家居评估_英.no_watermark.zh-CN.mono.txt
```

SMH-Bench：

```text
finally new paper/0524_4个bench/03_2026_arXiv_SMH-Bench-Environment-Grounded_EN.txt
```

SimuHome：

```text
finally new paper/0524_4个bench/04_2025_arXiv_SimuHome-Smart-Home-Agents_智能家居代理_英.no_watermark.zh-CN.mono.txt
simuprocject/07_2025_arXiv_SimuHome-Temporal-Environment_智能家居代理基准_英.txt
```

Miloco：

```text
milocoproject/xiaomi-miloco-main/plugins/openclaw/openclaw.plugin.json
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/index.ts
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/home-profile/suggestions.ts
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/tools/notify.ts
milocoproject/xiaomi-miloco-main/plugins/skills/
```

OpenClaw `v2026.5.20`：

```text
https://github.com/openclaw/openclaw/blob/v2026.5.20/extensions/memory-core/openclaw.plugin.json
https://github.com/openclaw/openclaw/blob/v2026.5.20/extensions/memory-core/src/tools.shared.ts
https://github.com/openclaw/openclaw/blob/v2026.5.20/extensions/memory-core/src/tools.ts
```
