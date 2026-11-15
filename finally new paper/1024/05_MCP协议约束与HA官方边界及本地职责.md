# 05：MCP 协议约束、HA 官方边界与本地职责

核验日期：2026-10-26

本文回答三个问题：

```text
MCP 通用协议要求 Host、Client、Server 共同遵守什么
Home Assistant 官方 MCP 提供了什么，又额外限制了什么
本地 Host、Client、自建 MCP Server 与语义压缩层各自负责什么
```

## 一页总览

```text
┌─────────────────────────────────────────────────────────────────────┐
│ MCP 通用协议层                                                       │
│ JSON-RPC 2.0 / 初始化握手 / 能力协商 / tools / prompts / resources   │
│ 这一层由双方共同遵守，不能随意改消息格式                             │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
┌───────────────────────────────┐   ┌─────────────────────────────────┐
│ 客户端侧                       │   │ 服务端侧                        │
│                                │   │                                 │
│ 我们的 MCP Host                │   │ HA 官方 MCP Server              │
│  用户任务                      │   │  /api/mcp                       │
│  模型调用                      │   │  prompts / tools / resources    │
│  Prompt 组织                   │   │  Assist 工具执行                │
│  Tool 筛选与压缩               │   │  HA 实体与服务                   │
│  权限和审计                    │   │                                 │
│                                │   │                                 │
│  MCP Client ───── MCP 协议 ────┼──▶│                                 │
└───────────────────────────────┘   └─────────────────────────────────┘

Host 与 Client 都在客户端一侧，Client 是 Host 管理的协议连接。
HA 官方在当前链路中只提供 Server。
```

## 1 MCP 通用协议约束

当前本机协商的 MCP 协议版本是 `2025-11-25`。协议按版本演进，下面的结论以该版本为准。

### 1.1 角色与连接结构

```text
Host
  一个 AI 应用容器和总协调器
  管理用户、模型、完整对话、权限和多个 Server

Client
  由 Host 创建和管理
  一个 Client 与一个 Server 保持 1:1 协议会话
  负责初始化和能力协商，不承担业务决策

Server
  暴露上下文和执行能力
  不读取完整对话
  不替 Host 调用模型
```

硬约束是：

```text
Host 创建 Client
Client 连接 Server
Server 响应 Client
Client 把响应交回 Host
一个 Host 可以管理多个 Client
一个 Client 只对应一个 Server
```

Host 和 Client 可以运行在同一个进程里。物理上不要求拆成两个程序，协议角色仍然分明。

### 1.2 消息格式

MCP 所有 Client 与 Server 之间的消息必须使用 JSON-RPC 2.0，并使用 UTF-8 编码。

```text
Request
  jsonrpc: "2.0"
  id: 非 null，同一会话内不能重复
  method: 例如 tools/list
  params: 可选参数

Response
  jsonrpc: "2.0"
  id: 与 Request 相同
  result: 成功结果

Error Response
  jsonrpc: "2.0"
  id: 与 Request 相同
  error: 包含整数 code 和 message

Notification
  jsonrpc: "2.0"
  method: 例如 notifications/initialized
  不带 id
  接收方不能回复 response
```

### 1.3 初始化和能力协商

初始化必须是第一次协议交互：

```text
Client → Server    initialize
Server → Client    initialize result
Client → Server    notifications/initialized
                   ↓
                 正常操作
```

它和 TCP 三次握手在形式上相似，作用层级不同：

```text
TCP 三次握手
  SYN
  SYN-ACK
  ACK
  作用：建立传输连接并确认双方收发状态

MCP 初始化
  initialize request
  initialize result
  notifications/initialized
  作用：确认协议版本、交换能力，并确认 Client 已完成初始化
```

完整时序：

```text
T1 Host → MCP Client
   Host 调用 Client.initialize()

T2 MCP Client → MCP Server
   发送 initialize request
   参数包含 protocolVersion、capabilities、clientInfo

T3 MCP Server → MCP Client
   返回 initialize result
   包含协议版本、Server 能力、serverInfo 和可选的 instructions

T4 MCP Client 内部记录
   保存协商后的 Server 能力
   检查 Server 返回的协议版本是否受支持

T5 MCP Client → MCP Server
   发送 notifications/initialized
   该消息不带 id，不期待 JSON-RPC response

T6 HA Streamable HTTP 传输层
   接收通知并返回 HTTP 202 Accepted

T7 MCP Client → Host
   Client.initialize() 完成并返回 InitializeResult
   Host 现在可以发起正常 MCP 请求

T8 Host 开始业务交互
   tools/list
   prompts/list
   prompts/get
   resources/read
   tools/call
```

本机实测使用 Python MCP SDK 的高层 `ClientSession.initialize()`。该方法在收到初始化结果后会发送 `notifications/initialized`，然后在初始化完成后把结果返回给 Host。因此 `03` 记录里的首轮交互确实执行了完整初始化。

初始化是每个逻辑 MCP Client-Server 会话执行一次，不需要每次模型请求或每次 `tools/call` 前重新执行。HA `/api/mcp` 采用无状态传输，表示 HA 不在两个 HTTP 请求之间保存会话对象；Client 的逻辑会话和协议版本管理仍由 Host/SDK 维护。新建 Client、客户端重启或连接需要重建时，再重新初始化。

初始化时必须交换：

```text
支持的协议版本
Client 能力
Client 实现信息
Server 能力和版本
```

双方只能使用成功协商过的能力。只有 Server 声明了 `tools`、`prompts`、`resources`，Client 才能按对应能力发起操作。

本机实测协商结果：

```text
协议版本    2025-11-25
Server      home-assistant
SDK/版本    mcp==1.26.0
```

### 1.4 传输约束

MCP 支持多种传输。Home Assistant 当前主要使用 Streamable HTTP 和旧的 HTTP+SSE。

以 Streamable HTTP 为例，通用协议要求：

```text
MCP endpoint 是一个固定 HTTP 路径
Client 发送 JSON-RPC 时使用 POST
每个 POST 请求体只放一个 JSON-RPC 请求、通知或响应
Client 的 Accept 通常要同时声明 application/json 和 text/event-stream
Server 可以返回单个 application/json 响应
Server 也可以返回 text/event-stream 流
HTTP 初始化后，后续请求要携带协商出的 MCP-Protocol-Version
```

会话是逻辑概念。Server 可以选择分配 `MCP-Session-Id`，也可以实现无状态模式。HA 当前 `/api/mcp` 采用无状态模式，每个 HTTP 请求都会新建 MCP Server 对象。

### 1.5 Tools 的协议约束

Server 提供工具时必须声明 `tools` 能力，并通过两个主要方法工作：

```text
tools/list
  返回工具名、description、inputSchema、可选 outputSchema 和 annotations

tools/call
  接收工具名和 arguments
  返回 content blocks
  用 isError 表示工具执行失败
```

`inputSchema` 必须是合法 JSON Schema。默认 dialect 是 JSON Schema 2020-12。无参数工具应使用合法的空对象 schema。

工具调用的错误分两类：

```text
协议错误
  未知工具、请求格式错误、Server 内部协议错误
  使用 JSON-RPC error response

工具执行错误
  参数值非法、API 失败、业务条件不满足
  返回 result.isError = true
  错误内容应让模型有机会修正参数并重试
```

### 1.6 Prompts 与 Resources

```text
Prompts
  prompts/list 发现模板
  prompts/get 获取具体模板
  返回 PromptMessage，包含 role 和 content

Resources
  resources/list 发现资源
  resources/read 读取内容
  资源由 URI 标识
  subscriptions 属于可选能力
```

Server 声明了能力才可用。HA 当前同时声明 tools、prompts、resources。

### 1.7 安全与边界

MCP 通用协议要求 Server：

```text
校验所有工具输入
实施访问控制
限制调用频率
清洗工具输出
```

Client 和 Host 应当：

```text
对敏感操作请求用户确认
在执行前展示工具和参数
校验工具结果后再交给模型
为工具调用设置超时
记录工具调用审计日志
```

Server 不应看到完整对话。Host 只向 Server 提供完成当前工具调用所需的信息。

使用 Streamable HTTP 时，Server 必须校验请求的 `Origin`，防止 DNS rebinding。HA 端点还位于 Home Assistant 的 HTTP 鉴权体系内；若要把服务暴露到局域网外，仍需另外检查代理、Origin 和访问控制。

### 1.8 协议不规定的内容

以下内容属于 Host 的应用设计，MCP 协议不替我们决定：

```text
使用 DeepSeek、Qwen 还是其他模型
模型 system prompt 怎么写
是否使用 Server 返回的 prompt
本轮给模型看哪些 Tool
Tool description 和 schema 如何压缩
是否给 Tool 起模型友好的别名
模型调用失败后重试几轮
是否要求用户确认写操作
如何保存长期记忆
如何把多个 Server 的结果汇总给模型
```

这些是本地可以改良的部分。约束边界在于：Host 对模型做的重新包装，最终仍要通过 MCP Client 映射回 Server 实际声明的工具名和参数。

## 2 Home Assistant 官方 MCP 提供了什么

### 2.1 提供的组件

当前链路：

```text
Home Assistant 2026.6.4
  └─ 官方集成 mcp_server
       └─ MCP Server: home-assistant
            mcp Python SDK 1.26.0
```

官方实现位于容器内：

```text
/usr/src/homeassistant/homeassistant/components/mcp_server/
  http.py          传输、鉴权入口、Streamable HTTP、旧 SSE
  server.py        prompts、resources、tools 的实际注册
  config_flow.py   选择 llm_hass_api
  __init__.py      注册 HTTP 端点
  manifest.json    声明 mcp==1.26.0
```

HA 官方在 `/api/mcp` 链路中提供 MCP Server。MCP Host、MCP Client、模型调用循环由外部应用负责。

Home Assistant 代码库另有外部 MCP Client 集成，用于让 HA 调用别人的 MCP Server。它和本文讨论的 `/api/mcp` Server 是独立方向。

### 2.2 传输、端点与鉴权

```text
现代端点
  POST http://localhost:8123/api/mcp
  Streamable HTTP
  无 MCP-Session-Id
  每个请求独立创建 Server
  单请求单 JSON 响应
  超时 60 秒
  GET 同一端点返回 405

旧端点
  GET  /mcp_server/sse
  POST /mcp_server/messages/{session_id}

请求约束
  Accept 必须包含 application/json
  Content-Type 必须是 application/json
  请求体必须是单个 JSON-RPC 消息
  通知和客户端响应返回 202

鉴权
  HA Bearer Token
  可由支持 OAuth 的 MCP Client 使用 HA OAuth
```

通用 MCP 的 Streamable HTTP Client 一般同时发送：

```http
Accept: application/json, text/event-stream
Content-Type: application/json
MCP-Protocol-Version: 2025-11-25
Authorization: Bearer <token>
```

HA 的入口检查只需要 `Accept` 中包含 `application/json`，当前实现返回普通 JSON，不返回 SSE 流。

### 2.3 HA 实际暴露的三类对象

```text
prompts
  一个 Prompt
  name = 当前 LLM API 名称
  get 后返回该 API 的 api_prompt
  实测角色是 assistant，内容包含被暴露实体的静态清单和操作规则

tools
  来自选中的 llm_hass_api
  default = assist
  工具集随被暴露实体和 API 动态变化

resources
  homeassistant://assist/context-snapshot
  mimeType = text/plain
  实际调用 GetLiveContext 取得当前上下文快照
```

HA 源码中的直接映射：

```text
prompts/list       → llm_api.api.name
prompts/get        → llm_api.api_prompt，role=assistant
tools/list         → llm_api.tools
tools/call         → llm_api.async_call_tool()
resources/read     → async_call_tool("GetLiveContext", {})
```

### 2.4 当前实例的工具面

当前 HA 配置和实体暴露情况产生了 12 个工具：

```text
HassTurnOn
HassTurnOff
HassCancelAllTimers
HassFanSetSpeed
HassBroadcast
HassClimateSetTemperature
HassListAddItem
HassListCompleteItem
HassListRemoveItem
GetDateTime
todo_get_items
GetLiveContext
```

这 12 个工具属于当前实例快照，不属于 MCP 标准。更换 LLM API、调整 Assist 暴露实体或接入新设备后，`tools/list` 会变化。

### 2.5 HA 对设备控制的额外约束

```text
定位方式
  Assist 工具主要用 name / area / floor 定位
  模型侧不使用 entity_id

调用上下文
  llm.LLMContext 固定 platform=mcp_server、language="*"、device_id=None
  模型不知道“我现在站在哪个房间”

可见性
  工具和上下文受实体是否暴露给 Assist 约束
  没有暴露的实体不会自动进入工具能力

工具来源
  默认是 Assist intent API
  可以通过多个 llm_hass_api 组合官方 API 与自定义 LLM API

服务状态
  /api/mcp 无服务端会话状态
  Host 需要自己维护多轮对话、模型上下文和任务状态
```

### 2.6 默认 Assist 工具面的能力缺口

```text
偏控制
  动作工具较多，通用读状态主要依靠 GetLiveContext

无完整设备 profile
  冰箱多个实体仍然分散出现在上下文中
  空调参数范围、能力组合和写后验证没有形成统一 profile

无空间拓扑
  area 只是实体归属关系
  没有房间邻接、门、路径和设备移动语义

无虚拟时间
  没有 advance / wait_until 这类时间推进接口

无目标断言
  没有统一的 check_goal

无长期记忆
  没有用户画像和跨会话任务记忆

长尾能力不可达
  例如当前冰箱冷藏目标温度是 number 实体
  当前 Assist 工具面没有 number.set_value
  模型能在 GetLiveContext 里看到温度，却无法设置目标温度
```

这些缺口属于 HA 官方 Server 的业务边界，MCP 协议无法自动补齐。

## 3 我们本地已经组织了什么

### 3.1 官方 HA MCP 的实测 Host

`03` 文档中的 Host 是为这次实测临时运行的本地 Python 测试程序。它不是 HA 官方组件，也不是仓库里现成封装的长期 Host 服务。

它实际负责：

```text
任务发起
  ↓
本地 Python Host
  ├─ 创建 MCP Client
  ├─ initialize
  ├─ tools/list
  ├─ prompts/get("Assist")
  ├─ 调 DeepSeek API
  ├─ 组织 messages 和 tools
  ├─ 解析 DeepSeek 的 GetLiveContext 调用
  ├─ 通过 MCP Client 发 tools/call
  ├─ 把 MCP 返回结果交回 DeepSeek
  └─ 得到最终回答
```

实测模型请求的关键结构：

```json
{
  "messages": [
    {
      "role": "assistant",
      "content": "HA 官方 Assist prompt"
    },
    {
      "role": "user",
      "content": "请把米家对开门冰箱的冷藏室目标温度设置为5°C"
    }
  ],
  "tools": [
    "HA 官方 tools/list 返回的 12 个 Tool schema"
  ]
}
```

这说明当前组织权已经在 Host：

```text
HA Server 决定能提供什么 Tool
Host 决定这次模型实际看到什么
模型输出 tool call
Host 映射成合法的 MCP tools/call
```

### 3.2 本地自建 MCP Server

仓库里有两个独立的自建 Server，名称中的 `ha_mcp` 容易与 HA 官方 Host 混淆：

```text
ha_mcp_server
  v1 基线
  通过 HA REST/WebSocket 适配真实 HA
  契约定义 8 个工具
  当前 README 标记 6 个已实现
  time_control 和 memory 仍为 NOT_IMPLEMENTED

ha_mcp_universal
  v2 通用化主线
  面向任意设备类型和设备来源
  tools.expose 白名单控制模型可见工具
  当前开放 observe_home / inspect_room / inspect_device / execute_action / finish /
  inspect_time / time_control / memory
  check_goal / query_events 已实现但隐藏
```

它们不是 HA 官方 `/api/mcp` Server，也不是现成的 MCP Host。它们是本地自建的 MCP Server 实现，向 Host/Client 暴露自己的工具契约，后端通过 HA REST 和 WebSocket 访问设备。

### 3.3 本地语义压缩设计

`02` 文档确定的目标结构：

```text
HA 的真实设备与能力
  ├─ Integration
  ├─ Device
  ├─ Entity
  ├─ State / Attributes
  └─ Service
        ↓
本地能力索引
  保留完整能力、参数范围、设备型号差异和长尾能力
        ↓
按设备类型生成 profile
        ↓
Host 按当前任务选择可见 profile / Tool
        ↓
模型只看到压缩后的高频能力
```

当前已经明确方向，设备 profile 和按需能力展开还未形成完整落地的稳定 Host 接口。后续实现要避免把完整 HA 能力表直接塞进模型上下文。

### 3.4 模型层

DeepSeek 或 Qwen 都不属于 MCP Host、Client、Server 三个角色。Host 通过模型 API 发起推理，再把模型产生的 tool call 转换为 MCP 的 `tools/call`。

Qwen3-4B 和 Qwen3-1.7B 的工具调用能力见 `04`。模型支持 `<tool_call>` 格式，Host 仍需负责 schema 注入、解析、参数校验和工具执行。

## 4 职责边界

| 层 | 必须遵守 | 可以提供 | 不应负责 |
|---|---|---|---|
| MCP 通用协议 | JSON-RPC、生命周期、能力协商、传输和错误格式 | 标准 Client/Server 互操作 | 模型 Prompt 组织、设备业务语义 |
| HA MCP Server | HA 鉴权、`/api/mcp` 请求约束、暴露实体和 Assist API 规则 | prompt、tools、resources、工具执行 | 调模型、维护完整对话、跨请求状态 |
| 我们的 MCP Host | 创建 Client、遵守 MCP 会话、只调用已协商能力 | 选模型、改 Prompt、筛 Tool、压缩 schema、做多轮循环 | 伪造 Server 没有声明的工具能力 |
| 我们的 MCP Client | 协议版本、请求 ID、传输、超时和通知 | 建立 1:1 Server 会话 | 业务决策和自然语言推理 |
| 自建语义 MCP Server | 声明能力、提供合法 schema、校验输入和输出 | 设备 profile、统一动作、目标断言、时间与记忆 | 决定 Host 给模型看什么 |

## 5 冰箱任务中谁能改什么

当前 HA 官方工具面的实际链路：

```text
GetLiveContext
  可以读到冰箱冷藏 sensor = 3°C

HassClimateSetTemperature
  针对 climate 实体

当前冰箱冷藏目标温度
  HA 里实际是 number 实体
  官方 Assist 当前没有暴露 number.set_value

结果
  模型能看到当前温度
  官方 MCP 工具面没有设置该 number 的执行入口
```

Host 能做：

```text
隐藏不相关工具
压缩 12 个工具的说明
只给模型看冰箱相关能力
把 GetLiveContext 调用改成更窄的参数
识别任务无法完成并返回明确原因
把后续动作映射到 Server 确实存在的工具
```

Host 不能单靠改名做到：

```text
官方 Server 没有 number.set_value
  ↓
Host 把一个不存在的工具改名为 set_fridge_temperature
  ↓
最终没有合法 tools/call 可以落到 HA
```

要真正补齐这个能力，需要改变 Server 侧能力来源：

```text
方案 A
  在 HA 内增加自定义 llm.API
  把 number 操作注册成新的 Assist Tool
  再让官方 mcp_server 选中该 API

方案 B
  在 HA 外运行我们的语义 MCP Server
  通过 HA REST/WebSocket 操作 number 实体
  由我们的 Host 调用这个自建 Server
```

## 6 实施时必须保持的闭环

```text
读取 HA 能力
  ↓
生成并压缩 Tool schema
  ↓
Host 交给模型
  ↓
模型选择 Tool
  ↓
Host 校验名称、参数、权限和风险
  ↓
MCP Client 发 tools/call
  ↓
HA 或自建 Server 执行
  ↓
Host 校验结果
  ↓
必要时回读设备状态
  ↓
把结果交回模型或报告失败
```

协议兼容、语义压缩、安全确认、执行和验证分别归不同层。实现时可以放在同一进程，职责定义仍需保持分开。

## 参考来源

```text
MCP Architecture
  https://modelcontextprotocol.io/specification/2025-11-25/architecture

MCP Lifecycle
  https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle

MCP Transports
  https://modelcontextprotocol.io/specification/2025-11-25/basic/transports

MCP Tools
  https://modelcontextprotocol.io/specification/2025-11-25/server/tools

MCP Prompts
  https://modelcontextprotocol.io/specification/2025-11-25/server/prompts

MCP Resources
  https://modelcontextprotocol.io/specification/2025-11-25/server/resources

Home Assistant MCP Server
  https://www.home-assistant.io/integrations/mcp_server/

本地协议与能力调研
  ../0916_HomeAssistant双仓库接口分析/02_HA官方对外接口.md
  ../0916_HomeAssistant双仓库接口分析/03_模型接口梳理与HA实现.md
  ../0916_HomeAssistant双仓库接口分析/06_MCP架构通用性说明.md
```
