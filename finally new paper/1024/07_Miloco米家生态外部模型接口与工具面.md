# 07：Miloco 米家生态外部模型接口与工具面

调研日期：2026-10-27

源码位置：

```text
D:\D_coderesource\simuhome\milocoproject\xiaomi-miloco-main
```

## 先说结论

Miloco 里存在两套不同的外部模型接入方式：

```text
A 原生 MIoT MCP 库
  backend/miot/src/miot/mcp.py

  米家手动场景 MCP     3 个 Tool
  米家设备控制 MCP     6 个 Tool + 2 个 Prompt
  米家摄像头 MCP       空占位，没有 Tool
  Home Assistant MCP   2 个 Tool，不属于米家控制链路

B Miloco 2.0 实际产品接口
  plugins/openclaw
  plugins/skills
  cli

  OpenClaw 插件
  3 个直接注册 Tool
  16 个 Skill
  大部分设备能力通过 miloco-cli 命令执行
  设备目录、家庭档案和系统提示在 Host 侧注入
```

当前仓库里，原生 MIoT MCP 类只被测试代码实例化和启动，没有找到 Miloco 主服务自动启动这些 MCP 的生产入口。Miloco 2.0 的正式路线是 OpenClaw 插件、Skill 和 CLI。

```text
原生 MCP
  更接近通用 MCP Server
  工具面是 DID + MIoT SPEC IID
  目前主要处于 SDK/测试可用状态

OpenClaw 插件
  当前产品实际使用
  依赖 OpenClaw Host 和 miloco-cli
  不是普通 MCP Client 可以直接连接的接口
```

两套接口都依托米家生态，不依赖 Home Assistant。

## 1 原生 MIoT MCP

### 1.1 服务划分

```text
MIoTManualSceneMcp
  name: Xiaomi Home Manual Scene MCP Server
  path: /mcp
  transport: Streamable HTTP

MIoTDeviceMcp
  name: Xiaomi Home Device Control MCP Server
  path: /mcp
  transport: Streamable HTTP

MIoTCameraMcp
  name: MIoT Camera MCP server
  当前只有空类，没有注册任何 Tool

HomeAssistantAutomationMcp
  name: Home Assistant automation MCP Server
  依赖 Home Assistant，不计入本次米家工具面
```

测试脚本中的启动方式：

```text
米家手动场景 MCP   http://host:8080/mcp
米家设备控制 MCP   http://host:8081/mcp
Home Assistant MCP http://host:8082/mcp
```

这些端口来自 `backend/miot/tests/test_mcp.py`，用于单独启动测试，不代表主服务端口。

### 1.2 米家手动场景 MCP：3 个 Tool

| Tool | 作用 | 参数 |
|---|---|---|
| `get_manual_scenes` | 获取米家手动控制场景列表 | 无 |
| `trigger_manual_scene` | 执行指定米家手动场景 | `scene_id` |
| `send_app_notify` | 向米家 App 发送通知 | `content` |

实际流程：

```text
get_manual_scenes
  ↓
得到 scene_id / scene_name
  ↓
trigger_manual_scene(scene_id)
```

如果 `scene_id` 不在当前场景缓存中，工具会返回错误，并要求先调用 `get_manual_scenes`。

这个 MCP 没有注册 Prompt 或 Resource。

### 1.3 米家设备控制 MCP：6 个 Tool

| Tool | 作用 | 参数 |
|---|---|---|
| `get_area_info` | 获取有设备的米家家庭和房间列表 | 无 |
| `get_device_classes` | 获取当前设备的类型列表 | 无 |
| `get_devices` | 获取米家设备列表 | 设备较少时无参数；设备较多时为 `area_id`、`device_class` |
| `get_device_spec` | 获取设备的 MIoT SPEC 定义 | `did` |
| `send_ctrl_rpc` | 控制设备属性或执行设备动作 | `did`, `iid`, `value` |
| `send_get_rpc` | 读取设备属性 | `did`, `iid` |

除了 Tool，还注册两个 Prompt：

```text
prompt_send_ctrl_rpc
  控制米家设备的分步模板

prompt_send_get_rpc
  查询米家设备属性的分步模板
```

### 1.4 设备控制的真实工作流

工具说明明确要求：

```text
1. get_devices
   确定目标设备 did

2. get_device_spec(did)
   获取该设备的 MIoT SPEC

3. 选择 SPEC 实例 iid
   prop 属性：prop.siid.piid
   action 动作：action.siid.aiid

4. 查询属性
   send_get_rpc(did, iid)

   或控制属性
   send_ctrl_rpc(did, iid, value)

   或执行动作
   send_ctrl_rpc(did, iid, [input1, input2, ...])
```

`send_ctrl_rpc` 内部的行号拆解逻辑：

```text
iid = prop.2.1
  cmd = prop
  siid = 2
  piid = 1

iid = action.2.1
  cmd = action
  siid = 2
  aiid = 1
```

属性写入会按照 SPEC 的 `format` 转换 int、float、string 或 bool。动作值必须是数组，数组元素按照 SPEC 输入参数顺序排列。

### 1.5 设备数量自适应

`MIoTDeviceMcp` 有一个默认阈值：

```text
prompt_device_count_max = 20
```

```text
设备数 ≤ 20
  get_devices 不接收过滤参数

设备数 > 20
  get_devices 增加 area_id / device_class
  工具说明中加入家庭、房间、设备类型清单
```

这说明 Miloco 已经意识到全量设备信息会增加上下文负担，但它只做到设备数量和区域/类型过滤，没有生成空调、冰箱、灯等设备 profile。

### 1.6 原生 MCP 的结果与风险

```text
查询结果
  send_get_rpc 返回属性原始值

控制结果
  send_ctrl_rpc 主要返回 true
  只检查 MIoT 云端 code 为 0 或 1
  没有统一写后回读

错误
  使用 ToolError，包含无效 did、无效 iid、SPEC 获取失败、设备无响应等

安全
  MCP 基类启动 HTTP 时没有传入认证机制
  测试代码绑定 0.0.0.0
  不能直接把测试启动方式暴露到不可信网络

工具语义
  位于 entity/did/spec 层
  模型需要理解 DID、SIID、PIID、AIID 和原始 SPEC 格式
```

### 1.7 摄像头 MCP

```python
class MIoTCameraMcp:
    """MIoT Camera MCP server."""

    _MCP_PATH: str = "/mcp"

    def __init__(self) -> None:
        pass
```

该类没有 Tool、Prompt、Resource，也没有启动逻辑。摄像头能力当前通过另一条路径进入：

```text
OpenClaw Skill: miloco-perception
  ↓
miloco-cli perceive devices
miloco-cli perceive query
miloco-cli perceive logs
```

## 2 Miloco 2.0 当前的 OpenClaw 接口

### 2.1 为什么它不是 MCP

Miloco 2.0 的生产链路：

```text
OpenClaw Host
  ├─ Miloco Plugin
  │    ├─ 注册 3 个 Tool
  │    ├─ 注入 Prompt
  │    ├─ 注册 Hook / Webhook / Service
  │    └─ 加载 16 个 Skill
  │
  └─ Agent
       ├─ 读取设备目录和 Skill
       ├─ 通过 shell 调 miloco-cli
       └─ 调 Miloco backend HTTP API
```

`openclaw.plugin.json` 中直接声明的 Tool contract：

```text
miloco_im_push
miloco_notify_bind
miloco_habit_suggest
```

设备查询、控制、场景、感知等没有作为 MCP Tool 注册，而是写成 Skill 指令，让 Agent 调 `miloco-cli`。

### 2.2 三个直接 Tool

| Tool | 作用 | 参数 |
|---|---|---|
| `miloco_im_push` | 向主人发送 IM 通知 | `message`, 可选 `bindHint` |
| `miloco_notify_bind` | 绑定通知 session | 可选 `sessionKey` |
| `miloco_habit_suggest` | 管理习惯建议候选库 | `action`, `key`, `subject`, `habit`, `suggestion`, `outcome`, `task_id` 等 |

`miloco_habit_suggest.action` 有四个值：

```text
list
record
mark_asked
resolve
```

它负责防重复、防骚扰和习惯建议状态流转，不是设备控制接口。

### 2.3 16 个 Skill

实际 `plugins/skills` 目录：

```text
miloco-create-task
miloco-devices
miloco-habit-suggest
miloco-home-observe
miloco-home-patrol
miloco-home-profile
miloco-home-promote
miloco-home-prune
miloco-miot-admin
miloco-miot-identity
miloco-miot-identity-register
miloco-miot-scope
miloco-notify
miloco-perception
miloco-perception-digest
miloco-terminate-task
```

README 重点列出其中 7 个用户能力：

```text
设备控制
视觉感知
身份管理
系统管理
创建任务
终止任务
主动通知
```

其余多项是 Home Dreaming、家庭档案维护、感知摘要和身份注册的内部流程。

### 2.4 设备目录注入

Miloco 在 `before_prompt_build` Hook 中，把设备目录注入系统上下文：

```text
最多 50 台高频设备
LRU 最近使用设备
每台设备带高频 SPEC 属性
每次重新读取后端快照
5 秒节流
```

Prompt 中同时说明：

```text
catalog 只是高频子集，不是全量
集合、多设备或找不到时必须 device list 拉全量
设备控制前必须先读 miloco-devices Skill
```

这已经是 Host 侧的语义压缩：

```text
全量米家设备
  ↓
最近使用 ≤50 台
  ↓
每台仅保留高频属性
  ↓
作为 Prompt 的设备目录
  ↓
长尾再通过 CLI 查询
```

## 3 miloco-devices Skill 的工具面

Skill 不直接给模型 JSON Tool schema，而是告诉模型如何组合 CLI 命令。

### 3.1 设备与场景命令

```text
miloco-cli device catalog
miloco-cli device list
miloco-cli device spec <did>
miloco-cli device props <did> [spec_name ...]
miloco-cli device control <did> <spec_name> <value>
miloco-cli device control <did> --set <spec_name> <value> --set ...
miloco-cli device action <did> <spec_name> [input ...]
miloco-cli device refresh
miloco-cli scene list
miloco-cli scene trigger <scene_id>
```

典型控制：

```bash
miloco-cli device control 4962 --set target-temperature 26 --set on@空调 true
```

典型查询：

```bash
miloco-cli device props 4962 target-temperature
```

典型动作：

```bash
miloco-cli device action 4981 start-charge
```

### 3.2 设备定位语义

Skill 规定的解析顺序：

```text
用户设备指代
  ↓
先查注入的高频 catalog
  ↓
找不到或有歧义
  ↓
device list，并按中文名和英文 category 一起 grep
  ↓
多候选时反问，不擅自选一台
  ↓
确定 did
```

这比原生 MIoT MCP 更接近自然语言控制，但实际执行仍落在 `did` 和设备 SPEC 上。

### 3.3 安全策略

```text
门锁 / 摄像头 / 燃气阀 / 烟雾报警器
  → 危险批，需要二次确认

其他设备
  → 普通批，可直接执行

单次控制
  → 最多 10 台，超出分轮

多候选
  → 要求用户澄清

设备不存在
  → 先 refresh，再明确报告

不支持
  → 不伪造 did / spec_name / model
```

对于控制非 `on` 属性，Skill 还要求在同一条命令中补该设备实际开关属性，例如 `on@空调=true`。

## 4 miloco-cli 的整体命令面

当前源码注册了 16 个命令组和 4 个独立命令。

```text
account      小米账号绑定、授权、解绑
admin        系统状态、家庭摘要、成本
config       配置读取与修改
dashboard    打开家庭面板
debug        调试信息
device       设备目录、列表、SPEC、属性、控制、动作、刷新
doctor       环境与网络诊断
home-profile 家庭档案 CRUD
identity     身份聚合、样本和成员画像
monitor      状态监控
notify       米家 App 推送
perceive     摄像头感知查询、日志、设备列表
person       家庭成员 CRUD
rule         自动化规则管理
scene        米家手动场景
scope        感知家庭和摄像头范围
service      Miloco 后端服务管理
task         家庭任务 CRUD
time-compute 时间计算辅助
version      版本
```

Skill 会根据用户意图加载对应说明，再调用这些 CLI 命令。

## 5 与 Home Assistant MCP 的本质区别

| 维度 | HA 官方 MCP | Miloco 原生 MIoT MCP | Miloco OpenClaw 插件 |
|---|---|---|---|
| 生态 | Home Assistant | 米家 MIoT | 米家 MIoT |
| 接口 | MCP Server | MCP Server | OpenClaw Tool + Skill + CLI |
| 是否当前生产主路径 | 是 | 未找到主服务启动入口 | 是 |
| 设备定位 | `name / area / floor` | `did` | `did`，但 Host 先注入高频目录 |
| 能力描述 | Assist intent | MIoT SPEC | SPEC + Skill 操作规范 |
| 工具数 | 12 | 场景 3 + 设备 6 | 直接 Tool 3 + Skill 16 |
| 高频上下文压缩 | 固定实体清单 | 设备超过 20 台时增加区域/类型过滤 | LRU 设备目录 ≤50 台 + 高频 SPEC |
| 写后验证 | 无统一验证 | 没有统一写后回读 | Skill 主要依赖 CLI 结果，没有统一回读门槛 |

## 6 对本地语义 MCP 的启示

Miloco 的做法说明：

```text
模型真正需要的是：
  “哪台设备 + 当前要操作哪个能力 + 合法值域”

不是：
  “全部 DID + 全部 SPEC 实例”
```

可以借鉴的结构：

```text
后台保存完整 MIoT / HA 能力
  ↓
Host 注入最近使用设备和高频能力
  ↓
设备目录只保留 name / did / room / online / 高频 SPEC
  ↓
模型按需查询完整能力
  ↓
写操作按安全等级确认
  ↓
执行后由 Host 回读验证
```

需要避免的部分：

```text
把完整 MIoT SPEC 直接塞给模型
用 DID / SIID / PIID / AIID 作为默认用户语义
把操作 SOP 大段复制进每个 Tool description
依赖特定 Agent 才能运行，导致通用 MCP Client 无法复用
```

更通用的实现应当把 Miloco 的“目录注入 + Skill SOP + CLI 执行”拆成：

```text
稳定 MCP Server
  暴露设备 profile、属性读取、动作执行和验证结果

MCP Host
  负责设备目录注入、工具裁剪、长尾展开和权限确认

Agent Skill
  可以保留任务策略和安全规范
  不承担唯一的工具协议
```

## 7 核验边界

```text
已核验
  原生 MIoT MCP 类、Tool 注册、Prompt、HTTP 路径
  OpenClaw 插件 manifest、3 个直接 Tool、16 个 Skill 文档
  设备目录注入与 miloco-cli 设备操作流程

未做
  没有用真实小米账号启动原生 MIoT MCP 做端到端调用
  没有启动 OpenClaw 网关验证完整 Agent 行为
```

当前源码证据更明确地说明：Miloco 2.0 的外部模型入口以 OpenClaw 插件、Skill 和 CLI 为主；原生 MCP 是独立 SDK 能力，尚未接入主服务启动链路。
