# L2 四个 Benchmark 与 Miloco 如何组织家庭记忆

## 0 本文范围

本文并列讨论五个对象：

```text
四篇论文
├─ HomeBench
├─ PersonalHomeBench
├─ SMH-Bench
└─ SimuHome

一个实际系统
└─ 小米 Miloco 2.0
```

Xiaomi Home HA 插件属于设备接入适配层，不是本文五个研究对象之一；它的设备映射机制见 `08`、`09`、`11`。

这五个对象对“记忆”的处理差异很大：

```text
PersonalHomeBench
  记忆对象 + 记忆工具

SMH-Bench
  记忆对象 + 提示词上下文

Miloco
  长期家庭档案 + 感知事件记忆 + 习惯建议状态
  自动注入、Host 工具、CLI 写入混合使用

HomeBench
  不建模家庭记忆

SimuHome
  不建模长期家庭记忆
```

先把最容易混淆的结论放前面：

| 对象 | 模型是否看到记忆内容 | 是否有专门记忆工具 | 记忆从哪里来 |
|---|---:|---:|---|
| HomeBench | 否 | 否 | 任务只提供设备、房间、方法和用户指令 |
| PersonalHomeBench | 是 | 是 | 家庭/成员记忆对象，可由提示词提供，也可调用工具检索 |
| SMH-Bench | 是 | 否 | `memory_list` 作为 `User Memory` 放入提示词 |
| SimuHome | 否 | 否 | 公开任务定义只围绕当前环境状态与查询 |
| Miloco | 是 | 混合 | 家庭档案自动注入；感知用 Host 工具；完整档案用 Skill + CLI |

## 1 五个对象怎样建模记忆

### 1.1 总表

| 对比轴 | HomeBench | PersonalHomeBench | SMH-Bench | SimuHome | Miloco |
|---|---|---|---|---|---|
| 核心目标 | 指令到设备方法映射 | 个性化长程家庭智能体 | 环境接地、多轮与个性化评测 | 动态环境与时间推理 | 可持续运行的家庭管家 |
| 记忆实体 | 无 | 家庭级记忆、成员级记忆、成员画像 | 短期对话事实、长期用户偏好 | 无独立长期记忆实体 | 感知事件、家庭档案、习惯建议状态 |
| 主要记忆载体 | 无 | 结构化 household 数据 | 任务字段 `M` 与提示词 `memory_list` | 当前环境状态 | `memory/` Markdown、JSON 档案、建议状态文件 |
| 模型初始可见 | 设备与指令 | 任务提示词中的相关记忆 | 完整或任务给定的 `memory_list` | 当前状态观测 | 自动注入的 `profile.md` 摘要 |
| 检索方式 | 无 | `get_household_memories`、`get_member_memories` | 无工具，直接读取上下文 | 查当前设备/环境状态 | Host `memory_search/get` 搜索感知记忆 |
| 写入/晋升 | 无 | 数据集构造与记忆更新操作 | 基准预先定义 | 无 | 用户直告 Skill/CLI；观察先进候选区再晋升 |
| 是否真实设备 | 否；虚拟设备 | 否；虚拟家电接口 | 否；HomeEnv | 否；Matter 风格模拟器 | 是；MIoT/米家设备 |
| 对设备语义的作用 | 验证方法选择与无效指令识别 | 用偏好补齐个性化目标 | 用记忆补齐省略参数 | 用当前状态约束动作 | 用长期偏好提升主动控制和建议质量 |

### 1.2 四种不同位置的“记忆”

```text
HomeBench
  记忆不在系统里
  重点是“当前指令能不能映射成合法设备方法”

PersonalHomeBench
  记忆是数据库
  模型可以主动搜索家庭或成员记忆

SMH-Bench
  记忆是提示词材料
  基准把 memory_list 放进任务，模型直接读

SimuHome
  当前状态是主要上下文
  设备和环境随时间变化，但没有长期人物档案

Miloco
  记忆是运行时基础设施
  有采集、压缩、候选、晋升、注入、检索和维护流程
```

### 1.3 记忆相关接口清单

这里只列与记忆读取、记忆检索和记忆写入直接相关的模型表面。完整设备、场景和工作流工具见 `L3_四个Benchmark与Miloco完整工具面.md`。

| 对象 | 模型可见的记忆接口 | 是否真的可调用 | 记忆内容怎样出现 |
|---|---|---:|---|
| HomeBench | 无 | 否 | 只有当前设备状态、房间和设备方法 |
| PersonalHomeBench | 读取：`get_household_memories(search_string, num_memories)`<br>`get_member_memories(member_id, search_string, num_memories)`<br>计划层写入：`update_memories(household_id, memories)` | 读取接口直接可调用；`update_memories` 属于计划生成动作面 | 提示词可先放 `{memories}`；模型可检索更多记忆，也可在主动程序里生成 `update_memories` 步骤 |
| SMH-Bench | 无 | 否 | `{{memory_list}}` 直接放在 `## User Memory` 提示段 |
| SimuHome | 无 | 否 | 公开任务没有长期家庭记忆输入 |
| Miloco | `memory_search(query, maxResults, minScore, corpus)`<br>`memory_get(path, from, lines, corpus)` | 是；属于 OpenClaw Host | 家庭档案 `profile.md` 自动注入；感知事件通过这两个 Host 工具检索 |

PersonalHomeBench 的家庭资料接口和设备接口不属于记忆检索接口，虽然它们可以辅助理解个性化上下文。`update_memories` 不在 PersonalHomeTools 的检索提示中，而在 “Toolbox for generating plans” 中，语义是添加或更新该家庭的记忆。Miloco 除记忆工具外还有通知和习惯建议相关工具，它们不属于通用家庭记忆读写。

Miloco 的完整档案写入没有专用 Tool：

```text
模型可见：
  profile.md 摘要
  memory_search / memory_get

模型通过 Skill 和 CLI 做：
  home-profile list
  home-profile profile-write
  home-profile commit

模型看不到专用：
  home_profile_read
  home_profile_write
  remember
  recall
```

## 2 HomeBench：没有用户记忆层

### 2.1 它测什么

HomeBench 关注的是：

```text
用户指令
  ↓
识别设备、房间和方法
  ↓
处理有效、多设备无效、混合无效指令
  ↓
生成正确的设备 API 调用
```

它的虚拟环境包含设备和房间，提示词会提供设备、方法及状态，但论文没有建模成员偏好、历史事件、长期画像或对话记忆。

论文局限性一节直接写明：

```text
品牌选择往往取决于个人偏好
这超出了本研究范围
```

因此 HomeBench 中的“上下文”主要指：

```text
当前场景有哪些设备
设备有哪些方法
方法需要哪些参数
用户这次要求操作什么
```

这些属于任务上下文和设备能力信息，不属于家庭记忆。

### 2.2 一个例子

任务：

```text
把卧室灯打开。
```

模型需要看到：

```text
卧室存在于当前家庭
卧室里有灯
灯有“开启”方法
调用参数为空或符合方法要求
```

它不需要知道：

```text
谁喜欢暖光
爸爸通常几点睡觉
昨晚卧室灯开到几点
过去是否因为灯光发生过争执
```

### 2.3 模型是否有记忆工具

没有。HomeBench 的重点是静态指令和虚拟设备 API，论文中的 RAG 也是用于补充设备/方法上下文，不是长期家庭记忆。

它的强项是判断“指令能否执行”，弱项是“理解这是不是一个具体家庭、具体成员的偏好”。因此在我们的统一语义 MCP 中，HomeBench 更适合参考设备能力约束和无效能力拒绝，不适合参考长期记忆设计。

## 3 PersonalHomeBench：记忆是显式数据，也是显式工具

### 3.1 记忆在家庭对象中的位置

PersonalHomeBench 的家庭环境同时包含：

```text
household
├─ members
├─ appliances
├─ sensors
├─ memories
└─ background
```

论文给出的记忆条目结构是：

```text
memory
subject / subjects
additional_information
time
```

按字段含义可以写成这个示意：

```json
{
  "memory": "爸爸希望冰箱冷藏室平时保持 4°C",
  "subjects": ["dad_id"],
  "additional_information": "温度偏好",
  "time": "2026-10-31"
}
```

论文中的 Memory Retriever 会检索与整体家庭或特定成员相关的情境感知记忆。Transcriber 则从观察结果中提取和更新记忆、事件及上下文属性。

### 3.2 模型实际看到的记忆面

PersonalHomeBench 是两层结合：

```text
任务提示词
  └─ 先放入一组相关记忆 {memories}

PersonalHomeTools
  ├─ get_household_memories(
  │    search_string,
  │    num_memories
  │  )
  └─ get_member_memories(
       member_id,
       search_string,
       num_memories
     )
```

这一点与 SMH-Bench 不同。SMH-Bench 只把记忆直接写进提示词；PersonalHomeBench 还给模型提供主动检索家庭级和成员级记忆的接口。

示意交互：

```text
用户任务
  “帮我建议一套爸爸晚上回家后适合的客厅环境。”

模型先看到
  家庭成员 ID
  当前设备
  部分相关记忆

模型需要更多偏好时
  get_member_memories(
    member_id="dad_id",
    search_string="爸爸 晚上 客厅 温度 灯光",
    num_memories=5
  )

拿到记忆后
  生成环境建议
  必要时继续查设备状态和可执行特征
```

### 3.3 记忆读取与记忆更新不在同一个工具面

```text
PersonalHomeTools：读取
  get_household_memories
  get_member_memories

计划生成工具面：写入
  update_memories(household_id, memories)
```

读取工具负责回答“这个家庭成员喜欢什么、经历过什么”。`update_memories` 的用途是“添加或更新关于该家庭的记忆”，但它是主动计划可生成的动作，不是 PersonalHomeTools 的通用检索函数。

设备和场景接口负责回答“家里有什么、现在是什么状态、能执行什么”。完整设备与场景工具见 `L3`。

模型必须把两者合起来，才能从“爸爸怕冷”和“空调当前 25°C”推导出“把目标温度调到 26°C”之类的动作。

### 3.4 接地边界

PersonalHomeBench 的设备是面向 LG、三星等公开资料建模的虚拟家电特征接口，不是当前真实住宅里的在线设备。记忆和背景可以来自视频与数据构造，但任务执行仍在虚拟环境内完成。

所以它可以证明：

```text
显式记忆对象和检索工具能否帮助个性化推理
```

不能直接证明：

```text
同一记忆工具迁移到真实米家、HA 或 Matter 设备后一定有效
```

## 4 SMH-Bench：有记忆内容，没有记忆工具

### 4.1 记忆是任务的一部分

SMH-Bench 把任务写成：

```text
τ = (H0, u, C, M, g)

H0  当前家庭初始状态
u   当前用户指令
C   对话历史
M   用户记忆
g   评测目标
```

模型在提示词中看到的记忆形式是：

```markdown
## User Memory

{{memory_list}}
```

它不是通过 `search_memory` 之类的工具去查，而是基准在构造任务时直接把 `memory_list` 放进模型上下文。

### 4.2 短期记忆与长期记忆

SMH-Bench 的 Personalized Memory 子类分成两种：

```text
Short-term Memory
  最近轮次发生的事情、刚执行的动作、上一轮修改

Long-term Memory
  存储的用户偏好、习惯、日常参数
```

论文例子：

```text
短期记忆
  上一轮刚调过客厅空气 Cube
  这一轮用户说“算了，不要调客厅的，改开客卫风扇”
  模型需要撤销/延续最近操作上下文
```

```text
长期记忆
  用户只说“该睡觉了”
  memory_list 里记录：
    空气 Cube 目标湿度 50%
    关闭呼吸灯
    进入入睡模式
  模型据此补齐省略参数
```

### 4.3 模型工具面没有记忆接口

SMH-Bench 的直接工具面只覆盖设备查询、设备控制和自动化，没有 memory search、memory write 或 profile 工具。三个设备的完整参数见 `L3`。

这意味着它的记忆是：

```text
预先给出的文本材料
  ↓
模型在上下文中阅读
  ↓
直接用于补全目标和生成设备调用
```

不是：

```text
模型发现信息不足
  ↓
主动搜索长期记忆库
  ↓
读取更多记忆
```

### 4.4 接地边界

SMH-Bench 的执行环境是 HomeEnv，能够验证最终设备状态和无关状态是否保持，但不是实际米家或 HA 家庭。

它的价值在于把“记忆使用”独立成 TC6 子类，并用真实状态最终值判定是否成功。它证明的是模型能否使用给定记忆，而不是一个长期记忆系统能否在真实家庭中采集、更新和维护记忆。

## 5 SimuHome：重点是动态状态，不是家庭记忆

### 5.1 任务定义里没有独立记忆变量

SimuHome 把任务定义为 POMDP：

```text
(S, A, O, T, R)

S  设备状态 + 环境状态
A  Matter 风格命令
O  模型可见的部分状态观测
T  状态转移
R  评测奖励
```

公开论文的任务定义与工具表中，没有像 SMH-Bench 的 `M` 那样的用户记忆变量，也没有 PersonalHomeBench 的记忆检索工具。本地 `0524` 中译本写 12 类查询、13 个工具；较新的 ICLR 2026 英文稿写 17 个工具。两版都没有长期家庭记忆接口。

论文中出现的“动作历史”主要用于判断智能体是否完成了所需动作，不是提供给模型的长期用户记忆。

### 5.2 一个例子

任务：

```text
我快做饭了，厨房现在湿度是多少？
```

模型需要：

```text
找到厨房
执行设备发现
查当前湿度
返回带单位的当前值
```

它不需要：

```text
妈妈过去喜欢厨房湿度多少
上个月做饭时设过什么目标
谁对干燥空气敏感
```

另一个任务：

```text
客厅感觉太闷。
```

模型需要从当前环境状态推断“闷”可能对应湿度或空气质量，再查设备并采取动作。这是“当前状态接地”，不是“历史偏好召回”。

### 5.3 接地边界

SimuHome 的设备、房间和环境变量都在模拟器中，环境会随时间连续变化。它比 HomeBench 多了动态状态和时间演化，但没有长期人物档案。

因此它适合参考：

```text
状态怎样随时间变化
动作前怎样验证当前状态
时间约束怎样评测
```

不适合直接参考：

```text
家庭成员长期偏好怎样存储
感知事件怎样沉淀
记忆工具怎样分层暴露
```

## 6 Miloco：把记忆做成持续运行的基础设施

### 6.1 三套记忆

```text
感知记忆
  家里发生过什么
  例如：07:31 爸爸从玄关出门

家庭档案
  这个家长期是什么样
  例如：爸爸通常 7:30 出门上班

习惯建议状态
  哪些习惯值得推荐成任务
  例如：工作日早上出门时自动关闭全屋灯
```

三者不是同一张表：

```text
感知事件
  ↓ 聚合与推断
候选知识
  ↓ 证据累积、夜间晋升
正式家庭档案
  ↓
系统提示词摘要
  +
习惯建议状态机
```

### 6.2 感知记忆

```text
摄像头画面与声音
  ↓
Miloco 感知日志
  ↓ 每 15 分钟
miloco-perception-digest
  ↓
memory/YYYY-MM-DD-miloco-perception.md
  ↓
模型用 memory_search 搜索
必要时用 memory_get 精读
```

感知记忆回答的是“最近发生了什么”，不是“这个成员长期喜欢什么”。

### 6.3 家庭档案

家庭档案分成八类：

```text
member_persona       成员画像
member_health        健康与禁忌
member_routine       作息和日常规律
member_entertain     娱乐习惯
member_preference    个体偏好
family               全家共同规则
space                空间特征
device               设备经验
```

一条正式条目包含：

```text
id
type
subject_id
subject_name
content
confidence
evidence_count
first_seen
last_seen
source
evidence_log
archived
```

示意条目：

```json
{
  "id": "p_example",
  "type": "member_preference",
  "subject_id": null,
  "subject_name": "爸爸",
  "content": "冷藏室温度设定为 4°C 更舒服",
  "confidence": 1.0,
  "evidence_count": 1,
  "first_seen": "2026-10-31",
  "last_seen": "2026-10-31",
  "source": "user_told",
  "evidence_log": [
    "2026-10-31 20:10: 用户告知爸爸觉得冷藏室 4°C 更舒服"
  ],
  "archived": false
}
```

正式档案写入 `profile.json`，再渲染成 `profile.md`：

```markdown
# 家庭档案

## 家庭成员

- name: 爸爸

### 爸爸

- 冷藏室温度设定为 4°C 更舒服
```

主 Agent 每次普通对话前，`before_prompt_build` Hook 会读取 `profile.md` 并把摘要追加到 system context。

模型看到的因此是：

```text
压缩后的自然语言文字
```

而不是：

```text
完整 profile.json
所有条目 ID
全部 evidence_log
候选区内容
```

### 6.4 候选区与夜间 Dreaming

观察到的知识不会直接进入正式档案：

```text
感知记忆 + 每日交互记忆
  ↓ 每天 00:00
Observe
  提取候选知识
  ↓
candidates.json
  ↓
Promote
  达到条件的候选晋升
  ↓
profile.json
  ↓
Prune
  统一主体、清理过期、commit
  ↓
profile.md 重新渲染
```

核心规则：

```text
候选置信度达到 1.0
  可以走快速晋升

普通观察
  至少 3 次证据
  证据跨度至少 2 天

候选超过 30 天且证据不足 3 条
  清理
```

正式档案的限制：

```text
max_profile_tokens   2000
max_profile_entries  300
max_candidates       500
max_evidence_log     10
```

用户直接告知的知识 `source=user_told` 权重更高，并豁免静默过期。

### 6.5 模型能否看到记忆工具

Miloco 的模型可见面需要分成三层：

```text
OpenClaw Host
├─ memory_search
└─ memory_get

自动上下文
└─ profile.md

Skill + CLI 记忆写入
└─ miloco-cli home-profile
   ├─ list
   ├─ profile-write
   └─ commit
```

`memory_search` 与 `memory_get` 来自 OpenClaw 默认的 `memory-core` 插件，不是 Miloco 插件注册的工具。Miloco 系统提示词要求模型搜索感知记忆时使用 `memory_search`，但本地 Miloco 仓库没有实现这个工具。

如果 Host 没有启用 memory 插件，`memory_search` 不会因为提示词里写了这个名字就存在。

Miloco 插件另有三个非记忆直接工具，负责通知和习惯建议状态；它们不属于家庭档案读取、搜索或写入。完整工具面见 `L3`。

### 6.6 冰箱例子

```text
用户说
  爸爸觉得冷藏室 5°C 太冷，平时设 4°C 更舒服。

模型判断
  这是 member_preference

加载 Skill
  miloco-home-profile

读取完整的结构化真源
  miloco-cli home-profile list --target profile

写入
  miloco-cli home-profile profile-write --user-edit --ops '[...]'

提交
  miloco-cli home-profile commit

下一轮普通会话
  profile.md 自动进入 system context
```

实际写入示意：

```bash
miloco-cli home-profile profile-write --user-edit --ops '[
  {
    "op": "add",
    "entry": {
      "type": "member_preference",
      "subject_name": "爸爸",
      "content": "冷藏室温度设定为 4°C 更舒服",
      "evidence_log": [
        "2026-10-31 20:10: 用户告知爸爸觉得冷藏室 4°C 更舒服"
      ]
    }
  }
]'

miloco-cli home-profile commit
```

后续用户如果说“把冰箱调成我平常的温度”，模型可以从家庭档案取出 4°C；真正读取冰箱当前温度并设置目标温度，仍要调用设备能力。

```text
记忆负责
  “为什么选 4°C”

设备能力负责
  “哪台冰箱、哪个温区、当前值、可接受范围、怎么下发”
```

### 6.7 接地边界

Miloco 可以接真实米家设备、摄像头和 OpenClaw Agent，因此它比四个 benchmark 更接近持续运行的家庭系统。

但本地仓库能证明的是：

```text
记忆结构、注入 Hook、CLI、候选晋升、调度和工具注册怎样实现
```

不能仅凭源码证明：

```text
每个真实家庭都有完整感知覆盖
长期自动提取一定准确
memory_search 在所有 OpenClaw 配置下默认可用
```

## 7 五个对象的模型交互顺序

### 7.1 HomeBench

```text
用户指令 + 设备/方法上下文
  ↓
模型直接选择设备方法
  ↓
虚拟设备 API 调用
```

没有记忆检索步骤。

### 7.2 PersonalHomeBench

```text
用户任务
  ↓
提示词中的家庭信息 + 部分相关记忆
  ↓
信息不足时调用 get_household_memories / get_member_memories
  ↓
查设备能力和状态
  ↓
生成回答、建议或设备操作
```

记忆工具与设备工具并列暴露。

### 7.3 SMH-Bench

```text
用户请求
  +
对话历史 C
  +
用户记忆 M
  +
家庭状态 H0
  ↓
模型阅读上下文
  ↓
调用设备查询、控制或自动化接口
  ↓
HomeEnv 状态迁移
```

记忆直接放在提示词里，没有单独检索工具。

### 7.4 SimuHome

```text
用户请求
  +
当前部分可观测状态
  ↓
模型查设备/环境
  ↓
执行 Matter 风格命令
  ↓
环境随时间继续演化
```

没有长期记忆步骤。

### 7.5 Miloco

```text
用户请求
  ↓
OpenClaw Host 组装 system context
  ├─ identity / capability / memory instructions
  └─ profile.md 摘要
  ↓
模型判断是否需要更多感知事件
  ├─ memory_search 搜索每日感知记忆
  └─ memory_get 读取精确片段
  ↓
需要完整档案或写入时
  └─ Skill + miloco-cli
  ↓
设备控制
  └─ Skill + miloco-cli 调 MIoT
```

Miloco 是五种方案中唯一把“自动注入、按需检索、结构化写入、异步沉淀”同时做出来的。

## 8 对我们统一设备语义 MCP 的直接启示

### 8.1 不要让所有记忆都变成 Tool

PersonalHomeBench 证明显式记忆检索工具可评测、可操作，但它同时说明工具调用本身会成为模型负担。SMH-Bench 的记忆直接放提示词，路径短，但记忆多了会挤占上下文且不能按需扩展。

更合适的分层是：

```text
稳定且高频的小摘要
  → 自动注入

近期事件与长尾知识
  → memory_search 按需检索

完整结构化真源
  → 后端查询或专用管理接口

写入与晋升
  → 后台流程或受控 CLI
```

### 8.2 记忆必须与设备能力分开

四个 benchmark 和 Miloco 都说明两件事不能混为一谈：

```text
记忆
  爸爸喜欢冷藏室 4°C

设备能力
  这台冰箱有哪些温区
  冷藏温区的当前温度是多少
  目标温度允许调到多少
  写入后能否回读验证
```

统一语义 MCP 可以先用记忆缩小候选参数，再通过真实设备能力检查合法范围。记忆不能直接生成一个设备做不到的动作。

### 8.3 端侧模型只应看到收缩后的记忆面

对于端侧小模型，最危险的暴露方式是：

```text
全部 profile.json
全部 candidates.json
全部证据日志
全部记忆写入接口
初始化、恢复出厂、重置等管理能力
```

更合理的模型面是：

```text
自动注入
  家庭成员摘要
  常见偏好
  当前家庭规则

按需工具
  search_memory
  read_memory_excerpt

受控写入
  remember_preference
  forget_memory
```

完整 ID、证据链、候选区、归档和主体重绑定仍留在后端。

### 8.4 评测要区分“有记忆”和“会用记忆”

```text
SMH-Bench
  测模型能不能使用已经给出的 memory_list

PersonalHomeBench
  测模型能不能调用记忆工具检索

Miloco
  还要处理真实系统中的采集、写入、更新、失效和隐私

HomeBench / SimuHome
  主要测设备能力、状态和时间约束
```

这四种能力不能用一个“memory score”混在一起。至少要拆成：

```text
记忆召回是否正确
记忆是否被正确应用
设备动作是否合法
最终状态是否正确
无关设备是否保持不变
```

## 9 证据边界

| 结论 | 证据等级 |
|---|---|
| HomeBench 不建模长期家庭偏好 | 论文局限性与任务设定明确排除个人偏好 |
| PersonalHomeBench 有家庭级和成员级记忆检索工具 | 论文附录列出完整函数签名 |
| SMH-Bench 的 `M` 直接进入提示词 | 论文任务定义和提示词模板均给出 `User Memory` |
| SMH-Bench 没有第四个记忆工具 | 环境交互工具面只有三个设备工具 |
| SimuHome 公开设定没有独立长期记忆 | POMDP、查询类型和 13/17 工具版本均未定义用户记忆 |
| Miloco 自动注入 `profile.md` | Hook、Skill 与知识文档均有实现 |
| Miloco 插件没有档案 CRUD Tool | 插件清单和源码只注册三个直接工具 |
| Miloco 的感知搜索依赖 Host | 本地提示词引用 `memory_search`，但本仓库不注册该工具 |
| OpenClaw 提供 `memory_search/get` | 本地锁定的 `openclaw@2026.5.20` 中 `memory-core` 清单明确声明 |

## 10 本地来源

四个 benchmark：

```text
finally new paper/0524_4个bench/01_2024_ACM-CHI_HomeBench-Evaluating-LLMs_HomeBench大模型评估_中.txt
finally new paper/0524_4个bench/02_2026_arXiv_PersonalHomeBench-Personalized-Eval_个性化智能家居评估_英.no_watermark.zh-CN.mono.txt
finally new paper/0524_4个bench/03_2026_arXiv_SMH-Bench-Environment-Grounded_EN.txt
finally new paper/0524_4个bench/04_2025_arXiv_SimuHome-Smart-Home-Agents_智能家居代理_英.no_watermark.zh-CN.mono.txt
simuprocject/07_2025_arXiv_SimuHome-Temporal-Environment_智能家居代理基准_英.txt
```

Miloco 记忆实现：

```text
milocoproject/xiaomi-miloco-main/plugins/openclaw/openclaw.plugin.json
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/index.ts
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/hooks/prompt.ts
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/home-profile/helpers.ts
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/home-profile/scheduler.ts
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/home-profile/suggestions.ts
milocoproject/xiaomi-miloco-main/plugins/openclaw/src/tools/notify.ts
```

Miloco 后端与 Skill：

```text
milocoproject/xiaomi-miloco-main/backend/miloco/src/miloco/home_profile/schema.py
milocoproject/xiaomi-miloco-main/backend/miloco/src/miloco/home_profile/constants.py
milocoproject/xiaomi-miloco-main/backend/miloco/src/miloco/home_profile/service.py
milocoproject/xiaomi-miloco-main/backend/miloco/src/miloco/home_profile/store.py
milocoproject/xiaomi-miloco-main/backend/miloco/src/miloco/home_profile/render.py
milocoproject/xiaomi-miloco-main/plugins/skills/miloco-home-profile/SKILL.md
milocoproject/xiaomi-miloco-main/plugins/skills/miloco-perception-digest/SKILL.md
milocoproject/xiaomi-miloco-main/plugins/skills/miloco-home-observe/SKILL.md
milocoproject/xiaomi-miloco-main/plugins/skills/miloco-home-promote/SKILL.md
milocoproject/xiaomi-miloco-main/plugins/skills/miloco-home-prune/SKILL.md
milocoproject/xiaomi-miloco-main/plugins/skills/miloco-habit-suggest/SKILL.md
milocoproject/xiaomi-miloco-main/knowledge/03-features/home-profile.md
```

OpenClaw `v2026.5.20`：

```text
https://github.com/openclaw/openclaw/blob/v2026.5.20/extensions/memory-core/openclaw.plugin.json
https://github.com/openclaw/openclaw/blob/v2026.5.20/extensions/memory-core/src/tools.shared.ts
https://github.com/openclaw/openclaw/blob/v2026.5.20/extensions/memory-core/src/tools.ts
https://github.com/openclaw/openclaw/blob/v2026.5.20/docs/concepts/memory.md
```

## 11 最终判断

如果把五个对象放在同一张图里：

```text
               显式记忆工具         长期人物档案         真实设备
HomeBench            无                  无                 无
SimuHome             无                  无                 无
SMH-Bench            无                  有，提示词给定       无
PersonalHomeBench    有                  有，结构化数据       无
Miloco               混合                有，持续维护         有
```

HomeBench 和 SimuHome 主要提供设备、环境和时间约束；SMH-Bench 提供“给定记忆如何影响任务”的评测；PersonalHomeBench 提供“记忆如何建模并作为工具暴露”的评测；Miloco 提供“记忆如何在真实运行系统中采集、沉淀、注入和搜索”的实现参考。

我们要做的统一语义 MCP 不应把五者压成一个结论，而应分别吸收：

```text
设备能力约束
  参考 HomeBench 和 Matter 风格设备模型

当前状态与时间验证
  参考 SimuHome

提示词内短记忆与个性化任务
  参考 SMH-Bench

显式记忆检索工具
  参考 PersonalHomeBench

长期档案、候选晋升和自动注入
  参考 Miloco
```
