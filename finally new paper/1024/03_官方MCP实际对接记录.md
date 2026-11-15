# 03：官方 MCP 实际对接记录

实测时间：2026-10-27 +08:00

本记录来自一次连续运行。英文自然语言已译成中文；Tool 名、参数名、JSON 字段和实体值保持原样。

当前 Home Assistant `tools/list` 返回的 12 个 Tool：

| Tool | 作用 | 一行例子 |
|---|---|---|
| `HassTurnOn` | 打开、激活、启用或按下设备；对锁执行上锁 | 打开客厅空气净化器。 |
| `HassTurnOff` | 关闭、停用、禁用或合上设备；对锁执行解锁 | 关闭厨房洗碗机。 |
| `HassCancelAllTimers` | 取消当前全部计时器 | 取消家里所有计时器。 |
| `HassFanSetSpeed` | 按百分比设置风扇速度 | 把厨房风扇调到 50%。 |
| `HassBroadcast` | 通过家庭广播设备播放一条消息 | 向全屋广播“开饭了”。 |
| `HassClimateSetTemperature` | 设置 `climate` 设备的目标温度 | 把客厅空调设定为 24°C。 |
| `HassListAddItem` | 向指定的待办清单添加一项 | 把“买牛奶”加入购物清单。 |
| `HassListCompleteItem` | 将待办清单中的一项标记为完成 | 把购物清单里的“买牛奶”标记为完成。 |
| `HassListRemoveItem` | 从待办清单删除一项或多项 | 从购物清单删除“买牛奶”。 |
| `GetDateTime` | 获取当前日期和时间 | 现在几点？ |
| `todo_get_items` | 查询待办清单中符合状态条件的事项 | 看看购物清单里还有什么未完成。 |
| `GetLiveContext` | 读取设备、传感器、实体或区域当前的实时状态、值或模式 | 客厅空调现在开着吗？ |

这 12 个 Tool 是当前 Assist API 和实体暴露配置的结果，不属于 MCP 固定标准；配置或设备变化后，`tools/list` 会随之变化。

交互层面的三个角色：

```text
user
  任务发起方

DeepSeek
  执行任务的模型

MCP
  一个由多个内部角色组成的系统
```

MCP 内部角色与网络两端：

```text
客户端侧
  MCP Host
    负责组织用户、模型和 MCP 会话
    内部包含 MCP Client

  MCP Client
    由 MCP Host 创建，负责通过 MCP 协议连接 MCP Server

服务端侧
  MCP Server
    当前就是 Home Assistant 官方的 mcp_server
```

```text
客户端侧 [MCP Host（包含 MCP Client）] ── MCP 协议 ──> 服务端侧 [MCP Server]
```

MCP Host 和 MCP Client 都位于客户端这一端。
MCP Server 位于服务端这一端。不存在第三种网络端。

## 1 三个交互角色与 MCP 内部角色

```text
外部 user
    │
    ▼
┌────────────────────────────────┐          ┌────────────────────────────────┐
│ 客户端侧                        │          │ 服务端侧                        │
│                                │          │                                │
│ MCP Host                       │          │ MCP Server                     │
│  本地 Python 测试程序           │          │  Home Assistant 官方 mcp_server│
│  ├─ 调用 DeepSeek API          │          │    ├─ prompts  Assist          │
│  ├─ 组织 messages 与 tools     │          │    ├─ tools    12 个           │
│  └─ MCP Client ────────────────┼─ MCP ───▶│    ├─ resources 1 个           │
│                                │ 协议      │    └─ tool result              │
└────────────────────────────────┘          └───────────────┬────────────────┘
                                                            │
                                                            ▼
                                                   Home Assistant Core
                                                   设备、实体与服务
```

### MCP 自身的三个角色

```text
MCP Host
  运行 AI 应用并协调整个会话的应用程序
  位于客户端侧

MCP Client
  由 MCP Host 创建，负责与一个 MCP Server 建立连接
  位于 MCP Host 内部
  当前实验为 1 个 MCP Host、1 个 MCP Client、1 个 MCP Server

MCP Server
  暴露 prompts、tools、resources 并提供 tool result
  位于服务端侧，当前由 Home Assistant 提供
```

```text
MCP Host 不直接向 MCP Server 发送 MCP 协议消息
MCP Host 调用自己内部的 MCP Client
MCP Client 通过 MCP 协议与 MCP Server 通信
MCP Server 的响应先返回 MCP Client，再交给 MCP Host
```

### 不属于 MCP 三个内置角色的对象

```text
任务发起方
  user，位于 MCP 外部

DeepSeek
  由 MCP Host 调用的语言模型，不承担 MCP Host/Client/Server 角色
```

### 组装方式

MCP Host 发起一次 DeepSeek 请求，将官方 prompt、用户任务和 tools 按模型 API 的字段组织：

```text
messages[0]
  role: assistant
  content: 官方 Assist prompt

messages[1]
  role: user
  content: 任务发起方的目标

tools
  从官方 MCP tools/list 取得的完整 Tool schema
```

### 实测配置

```text
endpoint       http://localhost:8123/api/mcp
transport      Streamable HTTP
protocol       2025-11-25
server         home-assistant
server_version 1.26.0
client         codex-live-case
model          deepseek-flash
temperature    0
thinking       disabled
```

## 2 实际时间线

本次完整实测按以下顺序发生：

```text
01  MCP Host → MCP Client
     initialize()

02  MCP Client → HA MCP Server
     initialize request

03  HA MCP Server → MCP Client
     initialize result
     protocolVersion = 2025-11-25
     server = home-assistant
     server version = 1.26.0
     capabilities 包含 tools、prompts、resources

04  MCP Client → HA MCP Server
     notifications/initialized

05  HA MCP Server → MCP Client
     HTTP 202 Accepted

06  MCP Client → HA MCP Server
     tools/list

07  HA MCP Server → MCP Client
     12 个 Tool schema

08  MCP Client → HA MCP Server
     prompts/list

09  HA MCP Server → MCP Client
     Assist Prompt

10  MCP Client → HA MCP Server
     resources/list

11  HA MCP Server → MCP Client
     homeassistant://assist/context-snapshot

12  MCP Client → HA MCP Server
     prompts/get(name="Assist")

13  HA MCP Server → MCP Client
     role=assistant 的完整 Assist prompt

14  MCP Host → DeepSeek
     REQUEST 1
     官方 Assist prompt
     用户任务
     12 个 Tool schema

15  DeepSeek → MCP Host
     RESPONSE 1
     GetLiveContext tool_call

16  MCP Host → MCP Client
     call_tool(name="GetLiveContext", arguments={...})

17  MCP Client → HA MCP Server
     tools/call

18  HA MCP Server → MCP Client → MCP Host
     tool result，isError=false

19  MCP Host → DeepSeek
     REQUEST 2
     原对话
     assistant tool_call
     tool result
     同样的 12 个 Tool schema

20  DeepSeek → MCP Host → user
     RESPONSE 2
     最终判断
```

初始化发生在第 02–05 步，完成以后才执行第 06 步的 `tools/list`。后面的工具调用复用这次逻辑初始化会话，不需要在每次 `tools/call` 前重新初始化。

### T0 user → MCP Host

```text
请把米家对开门冰箱的冷藏室目标温度设置为5°C。
优先使用可用工具获取实时信息并尝试完成任务；
如果当前工具无法完成，请明确说明原因。
```

### T1 MCP Host → MCP Client

```text
initialize()
```

### T2–T6 MCP 内部初始化、上下文获取与返回

```text
┌────────────────────────────────────────────┐        ┌────────────────────────────────────────────┐
│ 客户端侧                                    │        │ 服务端侧                                    │
│                                            │        │                                            │
│ T2 建立 MCP 会话                            │        │                                            │
│   MCP Host 调用 MCP Client.initialize()     │        │                                            │
│   MCP Client ───── initialize ──────────────┼───────▶│ MCP Server 接收初始化请求                  │
│                                            │        │                                            │
│ T3 初始化完成                               │        │                                            │
│   MCP Client ◀──── initialize result ───────┼────────│ MCP Server 返回初始化结果                  │
│   Client 记录协议版本、Server 名称和能力      │        │                                            │
│   MCP Client ───── initialized ────────────┼───────▶│ MCP Server 接收初始化完成通知              │
│   MCP Client ◀──── HTTP 202 ────────────────┼────────│ 通知不产生 JSON-RPC response               │
│   MCP Host ←──── MCP Client                 │        │                                            │
│   Host 获得初始化结果，可以开始正常请求      │        │                                            │
│                                            │        │                                            │
│ T4 Host 请求 MCP 上下文                     │        │                                            │
│   MCP Host → MCP Client                     │        │                                            │
│   请求 tools/list                           │        │                                            │
│   请求 prompts/list                         │        │                                            │
│   请求 resources/list                       │        │                                            │
│   请求 prompts/get(name="Assist")           │        │                                            │
│                                            │        │                                            │
│ T5 MCP Client 发出协议请求                  │        │                                            │
│   MCP Client ───── tools/list ──────────────┼───────▶│ MCP Server 查找可用工具                    │
│   MCP Client ───── prompts/list ────────────┼───────▶│ MCP Server 查找可用 Prompt                 │
│   MCP Client ───── resources/list ──────────┼───────▶│ MCP Server 查找可用 Resource               │
│   MCP Client ───── prompts/get("Assist") ───┼───────▶│ MCP Server 生成 Assist prompt              │
│                                            │        │                                            │
│ T6 MCP 上下文返回                           │        │                                            │
│   MCP Host ← MCP Client                     │        │                                            │
│   Host 获得：                               │        │                                            │
│     server info                             │        │                                            │
│     Assist prompt                           │        │                                            │
│     resource list                           │        │                                            │
│     tool list                               │        │                                            │
│   MCP Client ◀──── MCP responses ───────────┼────────│ MCP Server 返回 tool list、prompt、resources│
│                                            │        │                                            │
│ T6 完成后的 Host 状态                        │        │                                            │
│   已持有 Assist prompt                      │        │                                            │
│   已持有 Tool schema                        │        │                                            │
│   prompt 与 tools 尚未拼成文本               │        │                                            │
│   下一步在 T7 发起 DeepSeek 请求             │        │                                            │
└────────────────────────────────────────────┘        └────────────────────────────────────────────┘
```

原始初始化请求：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {},
    "clientInfo": {
      "name": "codex-live-case",
      "version": "1.0.0"
    }
  }
}
```

原始初始化响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "experimental": {},
      "prompts": {
        "listChanged": false
      },
      "resources": {
        "subscribe": false,
        "listChanged": false
      },
      "tools": {
        "listChanged": false
      }
    },
    "serverInfo": {
      "name": "home-assistant",
      "version": "1.26.0"
    }
  }
}
```

初始化完成通知：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

```text
initialize                  HTTP 200
notifications/initialized   HTTP 202
tools/list                  HTTP 200
prompts/list                HTTP 200
resources/list              HTTP 200
prompts/get                 HTTP 200
```

`tools/list` 返回的 12 个完整 Tool schema 位于 T7 的 `tools` 字段；Host 只增加了模型 API 的 `type/function` 包装，Tool 名、description 和 inputSchema 来自 MCP。

`prompts/list` 的实际响应：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "prompts": [
      {
        "name": "Assist",
        "description": "Home Assistant Assist API 的默认 Prompt。"
      }
    ]
  }
}
```

`resources/list` 的实际响应：

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "resources": [
      {
        "name": "assist_context_snapshot",
        "title": "Assist context snapshot",
        "uri": "homeassistant://assist/context-snapshot",
        "description": "当前 Assist 上下文的快照，内容与 GetLiveContext 工具输出一致。",
        "mimeType": "text/plain"
      }
    ]
  }
}
```

`prompts/get` 返回一个 `role=assistant` 的 PromptMessage，`content.type=text`。完整自然语言译文见 T7 的 `messages[0]`。

```text
Assist prompt 是 MCP Server 返回的完整文本。
Tool schema 是 MCP Server 返回的独立结构化字段。
T7 中 MCP Host 将它们与用户任务一起组织为一次 DeepSeek 请求。
```

### T7 MCP Host → DeepSeek · REQUEST 1

#### messages[0]

```text
role: assistant
content:

控制 Home Assistant 时始终调用 intent 工具。
锁门使用 HassTurnOn，解锁使用 HassTurnOff。
控制单个设备时，优先只传 name 和 domain。
控制一个区域时，优先只传 area 名称和 domain。

你具备使用 GetLiveContext 回答家庭实时状态的能力。这是主要功能。
涉及实时数据时，不要声称自己缺少该功能。

如果用户询问设备是否存在或设备类型：
  使用下面的静态上下文回答。

如果用户询问当前状态、当前值或当前模式：
  1. 识别这需要实时数据。
  2. 必须调用 GetLiveContext。
  3. 使用工具返回结果准确回答。

与家庭无关的通用知识问题：
  根据内部知识如实回答。

静态上下文：当前智能家居中的区域和设备概览。

- names: Air Conditioner Air Conditioner
  domain: climate
  areas: living_room
- names: Air Conditioner Air Conditioner
  domain: climate
  areas: utility_room
- names: Air Purifier Air Purifier
  domain: switch
  areas: living_room
- names: Dimmable Light Dimmable Light
  domain: switch
  areas: living_room
- names: Dishwasher Dishwasher
  domain: switch
  areas: kitchen
- names: Fan Fan
  domain: fan
  areas: kitchen
- names: Laundry Dryer Laundry Dryer
  domain: switch
  areas: utility_room
- names: Laundry Washer Laundry Washer
  domain: switch
  areas: utility_room
- names: On/Off Light On/Off Light
  domain: switch
  areas: living_room
- names: On/Off Light On/Off Light
  domain: switch
  areas: utility_room
- names: On/Off Light On/Off Light
  domain: switch
  areas: kitchen
- names: TV TV
  domain: switch
  areas: living_room
- names: kitchen PM10
  domain: sensor
  areas: kitchen
- names: kitchen 温度
  domain: sensor
  areas: kitchen
- names: kitchen 湿度
  domain: sensor
  areas: kitchen
- names: living_room PM10
  domain: sensor
  areas: living_room
- names: living_room 温度
  domain: sensor
  areas: living_room
- names: living_room 湿度
  domain: sensor
  areas: living_room
- names: utility_room PM10
  domain: sensor
  areas: utility_room
- names: utility_room 温度
  domain: sensor
  areas: utility_room
- names: utility_room 湿度
  domain: sensor
  areas: utility_room
- names: 米家对开门冰箱（适用610L/606L/616L)  冷冻室 温度
  domain: sensor
  areas: 2646196763的家
- names: 米家对开门冰箱（适用610L/606L/616L)  冷藏室 温度
  domain: sensor
  areas: 2646196763的家
- names: 米家对开门冰箱（适用610L/606L/616L) * 数据上传 室内环境温度
  domain: sensor
  areas: 2646196763的家
- names: 购物清单
  domain: todo

用户要求打开某一类型的所有设备时：
  除非家里只有一个该类型设备，否则要求用户指定区域。

该设备不能启动 timers。
```

#### messages[1]

```text
role: user
content:

请把米家对开门冰箱的冷藏室目标温度设置为5°C。
优先使用可用工具获取实时信息并尝试完成任务；
如果当前工具无法完成，请明确说明原因。
```

#### tools

`tools/list` 实际返回 12 个 Tool。以下保留完整 `name`、`description` 的中文含义和 `inputSchema`。

```json
[
  {
    "type": "function",
    "function": {
      "name": "HassTurnOn",
      "description": "打开、激活、启用、按下设备或实体。对于锁，执行上锁。",
      "parameters": {
        "type": "object",
        "properties": {
          "name": {"type": "string"},
          "area": {"type": "string"},
          "floor": {"type": "string"},
          "domain": {"type": "array", "items": {"type": "string"}},
          "device_class": {
            "type": "array",
            "items": {
              "type": "string",
              "enum": [
                "awning", "blind", "curtain", "damper", "door",
                "garage", "gate", "shade", "shutter", "window",
                "identify", "restart", "update", "tv", "speaker",
                "receiver", "outlet", "switch", "water", "gas"
              ]
            }
          }
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassTurnOff",
      "description": "关闭、停用、禁用、合上设备或实体。对于锁，执行解锁。",
      "parameters": {
        "type": "object",
        "properties": {
          "name": {"type": "string"},
          "area": {"type": "string"},
          "floor": {"type": "string"},
          "domain": {"type": "array", "items": {"type": "string"}},
          "device_class": {
            "type": "array",
            "items": {
              "type": "string",
              "enum": [
                "awning", "blind", "curtain", "damper", "door",
                "garage", "gate", "shade", "shutter", "window",
                "identify", "restart", "update", "tv", "speaker",
                "receiver", "outlet", "switch", "water", "gas"
              ]
            }
          }
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassCancelAllTimers",
      "description": "取消 timers。",
      "parameters": {
        "type": "object",
        "properties": {
          "area": {"type": "string"}
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassFanSetSpeed",
      "description": "按百分比设置风速。",
      "parameters": {
        "type": "object",
        "properties": {
          "name": {"type": "string"},
          "area": {"type": "string"},
          "floor": {"type": "string"},
          "domain": {"type": "array", "items": {"type": "string", "enum": ["fan"]}},
          "percentage": {"type": "integer", "minimum": 0, "maximum": 100}
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassBroadcast",
      "description": "向家庭广播消息。",
      "parameters": {
        "type": "object",
        "properties": {
          "message": {"type": "string"}
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassClimateSetTemperature",
      "description": "设置 climate 实体的目标温度。",
      "parameters": {
        "type": "object",
        "properties": {
          "temperature": {"type": "number"},
          "area": {"type": "string"},
          "name": {"type": "string"},
          "floor": {"type": "string"}
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassListAddItem",
      "description": "向待办清单添加条目。",
      "parameters": {
        "type": "object",
        "properties": {
          "item": {"type": "string"},
          "name": {"type": "string"}
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassListCompleteItem",
      "description": "完成待办清单条目。",
      "parameters": {
        "type": "object",
        "properties": {
          "item": {"type": "string"},
          "name": {"type": "string"}
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "HassListRemoveItem",
      "description": "删除待办清单条目。",
      "parameters": {
        "type": "object",
        "properties": {
          "item": {"type": "string"},
          "name": {"type": "string"}
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "GetDateTime",
      "description": "获取当前日期和时间。",
      "parameters": {
        "type": "object",
        "properties": {}
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "todo_get_items",
      "description": "查询待办清单。",
      "parameters": {
        "type": "object",
        "properties": {
          "todo_list": {"type": "string", "enum": ["购物清单"]},
          "status": {
            "type": "string",
            "enum": ["needs_action", "completed", "all"],
            "description": "按状态筛选条目，默认返回未完成条目。",
            "default": "needs_action"
          }
        }
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "GetLiveContext",
      "description": "查询设备、传感器、实体或区域的当前实时状态、值或模式。可以按 name、domain、area 组合过滤。",
      "parameters": {
        "type": "object",
        "properties": {
          "name": {
            "description": "按实体名称或 alias 过滤，忽略大小写。",
            "type": "string"
          },
          "domain": {
            "anyOf": [
              {},
              {"type": "array", "items": {"type": "string"}}
            ],
            "description": "按 domain 过滤，可接受单个 domain 或 domain 数组。"
          },
          "area": {
            "description": "按 area 名称或 alias 过滤，忽略大小写。",
            "type": "string"
          }
        }
      }
    }
  }
]
```

### T8 DeepSeek → MCP Host · RESPONSE 1

```text
我先查询一下冰箱冷藏室的实时状态，确认当前温度和相关设置。
```

```json
{
  "role": "assistant",
  "content": "我先查询一下冰箱冷藏室的实时状态，确认当前温度和相关设置。",
  "tool_calls": [
    {
      "id": "call_00_obwNe6Qpcth0D1d7qJkL2945",
      "type": "function",
      "function": {
        "name": "GetLiveContext",
        "arguments": "{\"name\": \"米家对开门冰箱（适用610L/606L/616L)  冷藏室 温度\"}"
      },
      "index": 0
    }
  ]
}
```

```text
finish_reason=tool_calls
```

### T9 MCP Host → MCP Client

```text
call_tool(
  name="GetLiveContext",
  arguments={
    "name": "米家对开门冰箱（适用610L/606L/616L)  冷藏室 温度"
  }
)
```

### T10 MCP Client → MCP Server

```json
{
  "name": "GetLiveContext",
  "arguments": {
    "name": "米家对开门冰箱（适用610L/606L/616L)  冷藏室 温度"
  }
}
```

HTTP 状态：

```text
200
```

### T11 MCP Server → MCP Client → MCP Host

```json
{
  "isError": false,
  "content": [
    {
      "type": "text",
      "text": "{\"success\": true, \"result\": \"实时上下文：当前智能家居中的区域和设备概览：\\n- names: 米家对开门冰箱（适用610L/606L/616L)  冷藏室 温度\\n  domain: sensor\\n  state: '4'\\n  areas: 2646196763的家\\n  attributes:\\n    unit_of_measurement: °C\\n    device_class: temperature\\n\"}"
    }
  ]
}
```

### T12 MCP Host → DeepSeek · REQUEST 2

#### messages[0]

```text
role: assistant
content: 与 T7 messages[0] 的完整 Assist prompt 相同，未改写。
```

#### messages[1]

```text
role: user
content: 与 T7 messages[1] 的用户任务相同，未改写。
```

#### messages[2]

```json
{
  "role": "assistant",
  "content": "我先查询一下冰箱冷藏室的实时状态，确认当前温度和相关设置。",
  "tool_calls": [
    {
      "id": "call_00_obwNe6Qpcth0D1d7qJkL2945",
      "type": "function",
      "function": {
        "name": "GetLiveContext",
        "arguments": "{\"name\": \"米家对开门冰箱（适用610L/606L/616L)  冷藏室 温度\"}"
      },
      "index": 0
    }
  ]
}
```

#### messages[3]

```text
role: tool
tool_call_id: call_00_obwNe6Qpcth0D1d7qJkL2945
content: 与 T11 TOOL RESULT 相同，含 state='4'
```

#### tools

```text
与 T7 的 12 个 Tool schema 相同，顺序不变。
Tool schema 仍位于模型请求的独立 tools 字段。
```

### T13 DeepSeek → MCP Host → user · RESPONSE 2

```text
很抱歉，这个任务无法完成。原因如下：

1. 该设备只是温度传感器，不是可控的温控设备：
   米家对开门冰箱的“冷藏室 温度”实体属于 `sensor` 域，
   只用于读取当前温度，目前显示为 4°C，
   并不具备设置目标温度的功能。

2. 没有可用的温控实体：
   当前智能家居中，冰箱相关实体只有三个传感器：
   冷冻室温度、冷藏室温度、室内环境温度。
   没有对应的 `climate` 温控实体，
   也没有可写入目标温度的服务接口。

3. 工具能力限制：
   现有工具中，`HassClimateSetTemperature` 只能用于 `climate` 域的温控设备，
   例如空调，无法作用于冰箱传感器。
   其他工具也无法向冰箱下发温度设定指令。

建议：
如需调整冷藏室温度，请通过米家 App 或冰箱本体的控制面板操作。
如果希望以后通过语音或自动化设置，
需要该冰箱接入支持温控写入的集成，例如暴露为 `climate` 实体。
```

```text
finish_reason=stop
tool_calls=none
```
