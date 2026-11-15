# 06：高星社区 Home Assistant MCP 工具面调研

调研日期：2026-10-27

## 先看结论

```text
官方 HA MCP
  12 个 Assist intent 工具
  面向“说出设备或区域，执行常见家庭动作”

homeassistant-ai/ha-mcp
  87 个细粒度工具
  面向“配置、管理、调试整个 Home Assistant”

tevonsb/homeassistant-mcp
  11 个宽工具
  每个宽工具内部用 action/command 分流

voska/hass-mcp
  29 个显式工具
  重点覆盖状态查询、设备控制、历史、日志和 Dashboard
```

三条路线分别代表三种扩展方向：

```text
靠更多 Tool 增加覆盖面
靠更少 Tool 包住更多动作
靠更细的操作工具和查询过滤提升可控性
```

三个项目都没有直接提供“按设备 profile 压缩高频能力，长尾按需展开”的通用语义层。它们解决了官方 HA MCP 的部分能力缺口，也产生了新的上下文、参数路由和权限问题。

## 1 调研范围与排名口径

本文限定为直接实现 Home Assistant MCP Server 的社区仓库。

GitHub 搜索和星数排序采用：

```text
"home-assistant mcp server in:name,description"
按 stargazers_count 降序
核验日期：2026-10-27
```

本次核验针对源码和 README 中的实际 Tool 注册与 schema，没有把三个社区 Server 分别接入当前 HA 做端到端兼容性测试。

入选仓库：

| 排名 | 仓库 | 星数 | 当前核验 commit | 工具数 |
|---:|---|---:|---|---:|
| 1 | `homeassistant-ai/ha-mcp` | 5272 | `9f5460e` | 87 |
| 2 | `tevonsb/homeassistant-mcp` | 576 | `da5c30b` | 11 |
| 3 | `voska/hass-mcp` | 350 | `bff8453` | 29 |

`Coolver/home-assistant-vibecode-agent` 的星数是 645，但它本身是运行在 HA 内的 Agent，实际 MCP Server 位于另一个仓库 `Coolver/home-assistant-mcp`，该仓库星数为 58。本文比较的是 MCP Server 工具面，因此没有把 Agent 的星数算到 MCP Server 上。

## 2 homeassistant-ai/ha-mcp

```text
定位
  Home Assistant 的配置、管理和调试服务器

默认工具面
  87 个 Tool 直接出现在 tools/list

可选紧凑模式
  4 个检索/代理 Tool + 可配置的 pinned Tool
```

### 2.1 工具面

完整目录按类别划分：

| 类别 | 数量 | 工具 |
|---|---:|---|
| Apps | 2 | `ha_get_app`, `ha_manage_app` |
| Areas & Floors | 3 | `ha_list_floors_areas`, `ha_remove_area_or_floor`, `ha_set_area_or_floor` |
| Assist | 1 | `ha_manage_pipeline` |
| Automations | 3 | `ha_config_get_automation`, `ha_config_remove_automation`, `ha_config_set_automation` |
| Blueprints | 1 | `ha_manage_blueprints` |
| Calendar | 3 | `ha_config_get_calendar_events`, `ha_config_remove_calendar_event`, `ha_config_set_calendar_event` |
| Camera | 1 | `ha_get_camera_image` |
| Dashboard | 1 | `ha_get_dashboard_screenshot` |
| Dashboards | 6 | `ha_config_delete_dashboard_resource`, `ha_config_delete_dashboard`, `ha_config_get_dashboard`, `ha_config_list_dashboard_resources`, `ha_config_set_dashboard_resource`, `ha_config_set_dashboard` |
| Developer | 2 | `ha_dev_manage_server`, `ha_dev_manage_settings` |
| Device Registry | 3 | `ha_get_device`, `ha_remove_device`, `ha_set_device` |
| Energy | 1 | `ha_manage_energy_prefs` |
| Entity Registry | 4 | `ha_get_entity_exposure`, `ha_get_entity`, `ha_remove_entity`, `ha_set_entity` |
| Files | 4 | `ha_delete_file`, `ha_list_files`, `ha_read_file`, `ha_write_file` |
| Groups | 3 | `ha_config_list_groups`, `ha_config_remove_group`, `ha_config_set_group` |
| HACS | 2 | `ha_get_hacs_info`, `ha_manage_hacs` |
| Helper Entities | 3 | `ha_config_list_helpers`, `ha_config_set_helper`, `ha_remove_helpers_integrations` |
| History & Statistics | 3 | `ha_get_automation_traces`, `ha_get_history`, `ha_get_logs` |
| Integrations | 3 | `ha_get_integration`, `ha_get_system_health`, `ha_set_integration` |
| Labels & Categories | 6 | `ha_config_get_category`, `ha_config_get_label`, `ha_config_remove_category`, `ha_config_remove_label`, `ha_config_set_category`, `ha_config_set_label` |
| Matter | 1 | `ha_manage_radio` |
| Scenes | 3 | `ha_config_get_scene`, `ha_config_remove_scene`, `ha_config_set_scene` |
| Scripts | 3 | `ha_config_get_script`, `ha_config_remove_script`, `ha_config_set_script` |
| Search & Discovery | 3 | `ha_get_overview`, `ha_get_state`, `ha_search` |
| Service & Device Control | 5 | `ha_bulk_control`, `ha_call_event`, `ha_call_service`, `ha_get_operation_status`, `ha_list_services` |
| System | 9 | `ha_config_get_yaml`, `ha_config_set_yaml`, `ha_manage_backup`, `ha_manage_custom_tool`, `ha_manage_security_policy`, `ha_manage_theme`, `ha_manage_updates`, `ha_reload_core`, `ha_restart` |
| Todo Lists | 3 | `ha_get_todo`, `ha_remove_todo_item`, `ha_set_todo_item` |
| Utilities | 2 | `ha_eval_template`, `ha_report_issue` |
| Zones | 3 | `ha_get_zone`, `ha_remove_zone`, `ha_set_zone` |

工具设计以资源对象和 CRUD 动作为中心：

```text
ha_get_*
  读取一个对象

ha_set_*
  创建或修改

ha_remove_* / ha_delete_*
  删除

ha_manage_*
  一个工具包住多类管理动作

ha_call_service
  通用底层转发入口
```

典型用例：

```text
“把所有灯都关掉”
  ha_call_service

“创建一条日落后打开门廊灯的自动化”
  ha_config_set_automation

“查看某个自动化为什么失败”
  ha_get_automation_traces / ha_get_logs

“给客厅仪表盘加一张天气卡”
  ha_config_get_dashboard / ha_config_set_dashboard
```

### 2.2 87 个工具带来的问题

项目自己在 README 中承认，完整工具目录会给不支持的模型造成明显上下文压力，因此提供了第二种工具面：

```text
ENABLE_TOOL_SEARCH=true
  ↓
tools/list 不再直接列出 87 个 Tool
  ↓
ha_search_tools        BM25 搜索工具
ha_call_read_tool      执行只读 Tool
ha_call_write_tool     执行写入 Tool
ha_call_delete_tool    执行删除 Tool
  +
若干 pinned Tool
```

使用顺序变成：

```text
模型不知道具体 Tool
  ↓
ha_search_tools("automation logs")
  ↓
获得少量候选 Tool 的名称、参数摘要
  ↓
ha_search_tools(tools=["ha_get_logs"])
  ↓
获得完整 schema
  ↓
ha_call_read_tool(name="ha_get_logs", ...)
```

这种方案把 87 个 schema 的固定成本换成了检索步骤和额外上下文。`ha_search_tools` 当前主要使用英文 BM25 关键词，中文设备词和中文意图需要经过翻译或别名处理。

### 2.3 可以借鉴的部分

```text
读、写、删除分开代理
  Host 可以按风险等级采用不同确认策略

Tool 可按连接或配置启停
  普通家庭控制不需要暴露 HACS、文件、系统管理

Pinned Tool
  高频入口固定出现，长尾 Tool 延迟发现

Read Only 模式
  控制模型自动执行敏感写入
```

这个项目的优势是覆盖面，不擅长低成本设备语义。87 个工具中的大多数面向 HA 管理员和自动化开发者，普通家庭任务只需要其中一小部分。

## 3 tevonsb/homeassistant-mcp

```text
定位
  Home Assistant 通用控制和管理服务器

工具面
  11 个宽工具

核心设计
  一个 Tool 内放多个 action，由参数做二次路由
```

### 3.1 11 个工具

| Tool | 作用 | 参数要点 | 调用例子 |
|---|---|---|---|
| `list_devices` | 列出全部实体，并按 domain 分组 | 无参数 | `list_devices({})` |
| `control` | 控制设备或调用服务 | `command`, `entity_id`，按设备类型附带亮度、温度、位置等 | `control({command:"set_temperature", entity_id:"climate.living_room", temperature:24})` |
| `get_history` | 查询实体历史 | `entity_id`, `start_time`, `end_time` | `get_history({entity_id:"sensor.living_room_temperature", start_time:"2026-10-26T00:00:00Z"})` |
| `scene` | 列出或激活场景 | `action=list/activate`, `scene_id` | `scene({action:"activate", scene_id:"scene.movie"})` |
| `notify` | 发送通知 | `message`, `title`, `target`, `data` | `notify({message:"洗衣完成"})` |
| `automation` | 管理自动化开关或手动触发 | `action=list/toggle/trigger`, `automation_id` | `automation({action:"trigger", automation_id:"automation.morning"})` |
| `addon` | 管理 Home Assistant App/Add-on | `action=list/info/install/uninstall/start/stop/restart` | `addon({action:"restart", slug:"core_mosquitto"})` |
| `package` | 管理 HACS 包 | `action=list/install/uninstall/update`, `repository` | `package({action:"update", repository:"hacs/integration"})` |
| `automation_config` | 创建、更新、删除或复制自动化配置 | `action`, `automation_id`, `config` | `automation_config({action:"delete", automation_id:"automation.test"})` |
| `subscribe_events` | 订阅 HA 事件 | `token`, `events`, `entity_id`, `domain` | `subscribe_events({token:"...", entity_id:"binary_sensor.front_door"})` |
| `get_sse_stats` | 查看 SSE 连接统计 | `token` | `get_sse_stats({token:"..."})` |

### 3.2 工具体积与路由压力

`control` 工具把多类设备写入一个 schema：

```text
command
  turn_on / turn_off / toggle
  open / close / set_position / set_tilt_position
  set_temperature / set_hvac_mode / set_fan_mode / set_humidity

entity_id
  决定实际 domain

其他参数
  brightness / rgb_color / color_temp
  position / tilt_position
  temperature / target_temp_high / target_temp_low
  hvac_mode / fan_mode / humidity
```

模型需要同时判断：

```text
选哪个 Tool
  ↓
选哪个 command/action
  ↓
当前 entity_id 是什么 domain
  ↓
哪些附加参数属于该 domain
```

这减少了 `tools/list` 的条目数量，参数路由压力仍然存在。

### 3.3 主要问题

```text
list_devices
  无过滤参数，直接读取 /api/states 并把全部实体按 domain 分组。
  设备规模上来后，单次结果可能明显膨胀。

automation_config.config
  trigger / condition / action 使用宽泛结构接收，缺少设备能力级约束。
  模型生成错误结构的概率需要靠外部检查兜底。

control
  只对代码列出的 domain 做专门映射。
  未覆盖的 domain 会直接返回 unsupported domain/operation。

subscribe_events
  把认证 token 放进 Tool 参数，安全边界和上下文泄露风险较重。

写入
  多数工具只返回 service 调用是否成功，
  没有统一的目标状态回读与 verified 结果。
```

这个项目的 11 个工具更接近“管理 API 集合”，不是面向小模型的低成本家庭语义层。

## 4 voska/hass-mcp

```text
定位
  状态查询、设备控制、历史统计、日志和 Dashboard 管理

工具面
  当前源码共 29 个 Tool

设计
  显式动作工具 + 查询过滤 + Dashboard 专用工具
```

README 的 Available Tools 列出了较早的一组工具；当前源码还包含 `system_overview`，所以以源码枚举的 29 个为准。

### 4.1 状态与发现，7 个

```text
get_version
get_entity
list_entities
get_entities_by_area
search_entities_tool
domain_summary_tool
system_overview
```

调用例子：

```json
{"name":"get_entity","arguments":{"entity_id":"light.living_room","fields":["state","attr.brightness"]}}
```

这组工具提供了多种上下文收缩手段：

```text
单实体读取        get_entity
字段筛选          fields
按区域读取        get_entities_by_area
关键词检索        search_entities_tool
domain 摘要       domain_summary_tool
全屋概览          system_overview
```

### 4.2 控制与自动化，4 个

```text
entity_action
call_service_tool
list_automations
restart_ha
```

调用例子：

```json
{"name":"entity_action","arguments":{"entity_id":"light.living_room","action":"on","params":{"brightness":128}}}
```

`restart_ha` 是高风险管理操作；`call_service_tool` 是通用底层出口，覆盖范围很大。

### 4.3 历史、统计与日志，5 个

```text
get_history
get_history_range
get_statistics
get_statistics_range
get_error_log
```

调用例子：

```json
{"name":"get_error_log","arguments":{"level":"ERROR","integration":"mqtt","lines":50}}
```

日志过滤放在 Server 侧执行，可以避免把整份 HA 日志塞进模型上下文。

### 4.4 Dashboard，13 个

```text
list_dashboards
get_dashboard_config
set_dashboard_config
add_card
update_card
remove_card
move_card
list_view_sections
add_view
remove_view
update_view
list_dashboard_backups
restore_dashboard
```

调用例子：

```json
{"name":"add_card","arguments":{"url_path":"lovelace","view":"home","card":{"type":"weather-forecast"},"dry_run":true}}
```

Dashboard 工具占全部 29 个工具中的 13 个，接近一半。普通家庭控制场景不需要把这 13 个工具全部交给模型。

### 4.5 关键特点

```text
优点
  查询过滤明确
  历史和日志支持范围、级别、integration 等服务器端过滤
  Dashboard 编辑支持 dry_run 和备份
  同时支持 stdio 与 stateless Streamable HTTP

问题
  工具仍然围绕 entity_id 和 HA 操作组织
  没有设备 profile 和家庭任务级语义
  13 个 Dashboard 长尾工具默认占据工具面
  HTTP 模式本身没有内建鉴权，README 明确要求反向代理或 VPN
```

## 5 四个工具面的横向比较

| 项目 | 工具数 | 主要组织方式 | 适合对象 | 主要成本 |
|---|---:|---|---|---|
| 官方 HA MCP | 12 | Assist intent，`name/area/floor` 定位 | 普通语音式设备控制 | 读写和长尾能力不足 |
| `ha-mcp` | 87 | 资源 CRUD、配置管理、全系统运维 | HA 管理员、自动化开发者 | 工具面过大，需要搜索或裁剪 |
| `tevonsb/homeassistant-mcp` | 11 | 宽工具 + action/command 路由 | 希望少量入口管理 HA 的开发者 | 参数复杂，容易全量 dump，弱约束 |
| `voska/hass-mcp` | 29 | 显式查询/控制/历史/Dashboard 工具 | 调试、监控、Dashboard 编辑 | 长尾工具多，仍然依赖 entity_id |

```text
官方 MCP
  会做家庭动作，但看不清完整能力，也操作不了部分长尾实体

ha-mcp
  几乎什么都能做，但把复杂系统完整展开给模型

tevonsb
  工具数量少，但每个工具内部继续展开 action 和参数分支

voska
  查询能力较好，但 Dashboard 和运维长尾占了大量 Tool
```

## 6 对我们自己的 MCP 设计意味着什么

### 6.1 不要直接复制 87 个或 29 个 Tool

普通家庭任务不需要：

```text
HACS 安装
Dashboard 卡片增删
YAML 文件编辑
重启 Core
备份恢复
主题修改
```

这些能力应该留在后台能力图谱或管理员 Tool 面，普通 Agent 默认视图只保留家庭任务需要的入口。

### 6.2 设备 profile 仍有独立价值

社区项目提供了更多底层能力，但没有提供类似的稳定视图：

```text
冰箱
  冷藏目标温度
  冷冻目标温度
  当前温度
  门状态
  故障

空调
  开关
  模式
  目标温度
  风速

灯
  开关
  亮度
  色温
```

模型需要的是“设备能做什么”，而不是先看 87 个管理 API，再猜应该调用哪一个。

### 6.3 可以借鉴的四种机制

```text
动态发现
  借鉴 ha-mcp 的 search -> describe -> call
  作为长尾能力入口，不作为所有任务的必经步骤

字段与结果过滤
  借鉴 voska 的 fields、domain summary、日志过滤
  避免把全量 state、history 和 log 塞进上下文

权限分级
  借鉴 ha-mcp 的 read / write / delete 分流
  Host 可以按风险等级采用自动、确认或拒绝

批量控制与高层动作
  借鉴社区统一入口，但参数要受设备 profile 约束
  不能只靠宽泛 action enum 让模型自由组合
```

### 6.4 推荐的后台与前台分层

```text
后台完整能力图谱
  HA entity / state / attributes / service
  第三方集成真实能力
  设备型号差异
  长尾配置与管理能力
        ↓
语义能力层
  设备 profile
  高频能力
  参数范围
  写前检查
  写后回读
        ↓
模型可见 MCP 工具面
  少量家庭任务入口
  当前设备相关能力
  必要时再搜索长尾能力
```

对 1.7B 和 4B 这类端侧模型，默认工具面应优先使用明确设备语义和小参数集合。BM25 英文 Tool 搜索、宽泛 action 路由和大量 Dashboard 工具都会增加多步决策成本。

## 7 数据与源码核验

```text
homeassistant-ai/ha-mcp
  https://github.com/homeassistant-ai/ha-mcp
  commit 9f5460e
  README 工具目录 87 项

tevonsb/homeassistant-mcp
  https://github.com/tevonsb/homeassistant-mcp
  commit da5c30b
  src/index.ts 注册 11 个 Tool

voska/hass-mcp
  https://github.com/voska/hass-mcp
  commit bff8453
  app/server.py 注册 29 个 Tool

被排除的关联候选
  Coolver/home-assistant-vibecode-agent
    https://github.com/Coolver/home-assistant-vibecode-agent
    Agent 与 MCP Server 分仓

  Coolver/home-assistant-mcp
    https://github.com/Coolver/home-assistant-mcp
    实际 MCP Server 包
```

星数和仓库内容会变化；本文工具明细按上述 commit 核验。
