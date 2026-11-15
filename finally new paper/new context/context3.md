# context3：v2 八工具、可移植部署与 HA 跨协议语义调研（2026-09-27 ~ 11-15 全过程记录）

本文件用于新对话接续，接在 `context2.md`（09-17 ~ 09-27）之后。
写法沿用 context1/context2：记录关键决策、实测数字、踩坑与待办，不是交付清单。

```text
工作目录      D:\D_coderesource\simuhome
本轮跨度      2026-09-27 ~ 10-28
主线          把 v2 MCP 收口成 8 个可交付工具；把仓库变成别人可复现的方案；
              补齐 1012 三份接口/协议调研；完成论文任务类型梳理与目录归档；
              10-25 ~ 10-28 继续核验 HA 官方 MCP、社区 MCP、插件映射与设备功能面
当前状态      10-24 的 v2 状态保留在原正文；10-28 新增跨协议能力调研与
              common/custom 功能面规范，投影实现仍待落地
远端仓库      https://github.com/juezhan340/simuhome（private，main；推送需代理 7897）
```

## 0. 一分钟结论

```text
工具面        v2 开放 8 个：observe_home / inspect_room / inspect_device / execute_action /
              finish / inspect_time / time_control / memory
              隐藏：check_goal / query_events；未实现：manage_rule
时间语义      time_control = 一次性预约（schedule / list / cancel）；
              list 只列还没执行的单；取消、执行完、失败的单留库审计但对模型不可见
记忆语义      memory = 整段文本 read / write；write 上限 200 token；写成功只回 {"code":"OK"}
验收          离线 36/36；真机 22/22；DeepSeek stdio 12/12（8 个接口全部真实调用）
部署          deploy/ 五份文档 + 通用 compose；脚本改相对路径 + python 自动探测；
              broker 凭证可一键生成；别人拿仓库 + 自己的 HA 就能跑
调研          1012/01 模型可见接口与完整样例；1012/02 五篇论文协议与能力；
              1012/03 主流协议与设备能力模型
任务梳理      16 号：五篇 bench；SMH 22 子类全表；HomeFlow 5 大类、无自动化规则任务
已知问题      小米云链路偶发 15s 超时（容器 TLS 3~3.7s，某 IP 不通）；
              HA 浏览器会话按 origin（localhost ≠ 127.0.0.1）；
              文件夹日期前缀中 UbiComp 是否应最早还未定口径
仓库状态      最后推送 3c246c4；之后 1012 文档、16 号更新、目录重命名等尚未提交
```

## 1. 时间线

### 1.1 09-27 ~ 10-06：v2 从 M1/M2 推到 8 工具

```text
09-27   context2 结束时：v1 冻结；v2 M1（认领）/M2（角色）完成，
         米家冰箱拆成 cold/freeze 两台逻辑设备；M3（命名泛化/第三方集成抽样）暂缓
之后    补齐 execution.perform_action：execute_action 与预约到点共用同一执行内核
        新增 finish / inspect_time / time_control / memory
        契约收敛到 10 个 ToolSpec；主配置开放 8 个（原 5 + inspect_time + time_control + memory）
        memory 最终定为整段文本 read/write（不再走 key-value、不再只读）
        time_control 定为绝对分钟级时间；去掉 advance / 相对时间 / 快进
```

### 1.2 10-07 ~ 10-09：设计定稿 + list/cancel 语义反转

```text
14 号《记忆与时间调度接口详细设计》与 15 号《目前方案与交付接口》成稿
10-09  list 语义修订（关键决策）：
        原设计 list 显示 pending + 已结束单；用户指出这对小模型是负担
        新语义 list 只列未执行单；cancel / completed / failed 全部从 list 消失
        内部 sqlite 行保留（审计），调度器到点检查 status，取消单不会执行
        代码：ScheduleStore.list_pending（SQL 过滤 pending）；取消 _status_for_model
        文档：14/15 同步（T3 反转）；测试新增 test_list_only_shows_pending
        实测：离线 36/36、真机 22/22、真机到点回归、DeepSeek 12/12
```

真机到点回归（10-09）：19:58 建单（at=19:59 开灯）→ 到点灯确实打开 → 该单从 list 消失 → 灯状态还原。

### 1.3 10-10：仓库可移植化 + deploy 文档

用户要求“别的开发者拿仓库 + 自己的 HA 能跑起来”。审计结论：

```text
我们没有改 HA 源码：运行时是官方镜像 ghcr.io/home-assistant/home-assistant:stable
  compose 在仓库里；ha_core_official 只是干净的官方参考克隆
真正缺的是：
  ① SimuHome 内核不在仓库（上游公开：Mr-luo-q/Simuhome_experiment，实测 f8a813f，
     home_full.json 在仓库里；论文官方 holi-lab/SimuHome 不含该文件）
  ② broker 凭证生成步骤缺失
  ③ mosquitto.conf / 启动脚本写死 D:\ 路径
  ④ v2 README 过期（还写着“改造尚未开始”）
```

已做的修复：

```text
deploy/01~05：部署总览 / 快速接自己的 HA / 沙盒全链路 / 米家真机 / 排错
deploy/compose.yaml：${HA_CONFIG_DIR:?}，强制显式 HA 配置目录
mosquitto.conf 改相对路径；start_mosquitto.ps1 固定工作目录
所有启动脚本改 $PSScriptRoot + python 自动探测
新增 broker/make_credentials.ps1：生成 mosquitto.passwd + credentials（无 BOM）
requirements 移除已退役的 amqtt
ha_mcp_universal/README.md 重写为 v2 现状
.gitignore 放行 deploy/；source.md 重新生成
```

实测：broker 相对路径重启后 HA 25 实体 / 0 unavailable；一键脚本 exit 0；离线 36/36；真机 22/22。

### 1.4 10-12 ~ 10-13：1012 调研文档

```text
10-12  建 finally new paper\1012\
       01_模型可见接口面与完整交互样例.md
          tools/list 逐字 8 工具 + 返回外壳 + 4 组完整样例 + 真机 DeepSeek 记录
          §1 设备能力现状：六字段 + 八动作；当前 14 台设备 / 43 个动作位
10-13  02_协议适配与设备能力-五篇论文调研.md
          SimuHome Matter / SMH schema / HomeFlow YAML / HomeBench 手工 15 类 /
          PersonalHomeBench 厂商文档 40+；横向对比 + 我们的三条扩展路线 A/B/C
       03_主流智能家居协议与设备能力模型调研.md
          Matter/Zigbee/Z-Wave/Thread/Wi-Fi/BLE Mesh/KNX/Modbus 的能力模型；
          占比数字（TechInsights/Statista/CSA，口径各异）+ 定性地图 + 检索限制
```

### 1.5 10-14 ~ 10-15：服务、登录与冰箱

```text
10-14  停过一轮服务，再一键拉起：SimuHome 3 房间 / 25 实体 / 0 unavailable
        HA 登录要求重新输密码：查实是浏览器 origin 问题——
        会话令牌 client_id = http://localhost:8123/（有效至约 2027-01-08）；
        我给的链接是 127.0.0.1，浏览器视为另一个站点，所以看不到登录态
        长期令牌（接口/MCP 用）不受影响：GET /api/ 200

10-14  冰箱控制失败：日志是 xiaomi_home → miot_cloud 写请求 TimeoutError（写超时 15s）
        容器→api.io.mi.com：TLS 握手 2.98s / 3.65s，IP 39.156.44.8 直接超时；
        宿主机直连同一域名 0.65s → 问题在容器到小米云的链路（挑到坏 IP/握手过慢）
        等值写入 4 次成功（2.9 / 4.2 / 1.8 / 1.0s），不是“完全不能控”

10-15  冷藏设定 8→5°C，连续 5 次写入全部成功（2.4 / 0.8 / 0.6 / 0.9 / 0.9s），
        state=5；说明连上正常路径后连接复用，写入又快又稳
        系统代理仍指着 127.0.0.1:7897 但代理程序未开（半开状态，未清理）
```

### 1.6 10-15 ~ 10-17：论文任务类型与 16 号文档

```text
SimuHome 数据合成（论文 §4.2 图 2 + 仓库实证）：
  ① 初始家庭状态：先造不同房间/设备布局，再从全关随机操作到合理状态；种子可复现
  ② 目标与所需动作：按查询类型生成（附录 A.5）；目标从预定义有效状态集采样，
     同时生成“必须出现的工具调用序列”
  ③ 查询合成：先定可验证目标，再用 GPT-5-mini 合成自然语言查询；
     两名研究生全量复核，Cohen κ=0.92
  ④ 情境生成：12 条件（QT1/QT2/QT3/QT4-1/2/3 × 可行/不可行）×50 = 600 回合
  仓库：gen_spec.example.yaml（simuhome-gen-spec-v1）；data/benchmark 正好 600 个 JSON；
       每个回合只有一个 query 字符串，没有对话历史 → SimuHome 是单轮
  对比：HomeFlow 才是多轮（TC4 + 用户模拟器 + MCTS）；SMH-Bench 有 TC5 多轮

16 号文档（现位于 0520_方法相关的论文\16_四个Bench任务类型梳理.md）：
  五篇 bench：HomeBench / SimuHome / SMH-Bench / PersonalHomeBench / HomeFlow
  SMH 补全 22 子类（3+5+3+3+4+2+2）并给代表指令；
    数量口径：22×50=1,100；正文分类句相加 1,150（TC7=150 与附录冲突，
    按 22×50 口径 TC7 应为 100）
  HomeFlow 补 5 大类（QT1~QT5 / 附录 TC1~TC5）与子模板；
    明确它没有 SMH TC4 那种“写自动化规则”任务——
    automation 只在画像提示词 “Habits → automation scheduling” 出现，
    rule 全是验证器/奖励/生成规则；pyexec 能跑代码但评测只看回合末状态
  编号陷阱：HomeFlow QT4/TC4=上下文多轮 ≠ SimuHome QT4=时间、SMH TC4=调度
```

### 1.7 10-20 ~ 10-24：目录日期前缀与收尾

按用户规则“每个文件夹内取最早修改文件”，对 `finally new paper` 下除 `new context`
的文件夹加 MMDD 前缀：

```text
0520_方法相关的论文                 最早文件 2026-05-20 21:34
0524_4个bench                     最早文件 2026-05-24 20:08
0902_给老师汇报方向                 最早文件 2026-09-02 20:52
0909_UbiComp三支柱讨论              最早文件 2026-09-09 22:03
0916_HomeAssistant双仓库接口分析     最早文件 2026-09-16 23:58
0918_外界大模型与HA交互方式调研        最早文件 2026-09-18 23:55
1004_另一端对齐文档                 最早文件 2026-10-04 19:04
1005_10_04V2接入小米后设计方案        最早文件 2026-10-05 01:44
1012                              本身已是日期（2026-10-12），保持不动
new context                       按要求不动
```

同时更新了 27 个文件里的 69 处旧路径引用，重生成 `source.md`，扫描确认 0 处旧路径残留。

未定：用户认为 `UbiComp三支柱讨论` 应该最早，但文件系统显示它创建于 09-09 14:32、
最早修改文件 09-09 22:03；比它更早的是两个资料库（PDF 保留 5 月时间）和
`给老师汇报方向` 里 09-02 的实验输出。需要确认“最早”的口径（文件时间 vs 项目时间），
再决定是否改 0909 前缀。

## 2. 当前系统状态（2026-10-24 核对）

```text
端口        8123 / 8000 / 1883 全部关闭（服务当前是停的）
发布器      仍有一个 publisher.py 进程在跑（PID 38040）——下次拉起 broker 后它会自动重连；
            注意启动脚本会把它当成“已在运行”而跳过启动
冰箱        最后已知冷藏设定 5°C；实体 number.midjd6_cn_911432173_6101_target_temperature_p_3_2
HA 登录     浏览器会话按 origin 区分；用 localhost 保持原会话，长期令牌供接口使用
```

## 3. 关键决策与理由

### 3.1 list/cancel 只保留“当前有效单”

```text
模型侧三件事：schedule 写单、list 看还没执行的单、cancel 取消
取消/完成/失败的单对模型不可见；内部 sqlite 仍保留，供审计与调度器判断
理由：端侧小模型不需要看到已结束状态；list 的语义收敛成“当前还有效的预约”
```

### 3.2 8 个工具就是当前统一能力面

```text
observe_home / inspect_room / inspect_device / execute_action / finish
inspect_time / time_control / memory
check_goal / query_events 保留隐藏；manage_rule（持久规则）仍未实现
设备能力：六字段 on/mode/target/level/position/current；八动作
  switch 3 / light 3~4 / fan 3~4 / climate 5 / cover 1 / select 1 / number 1 / sensor 0
当前部署 14 台设备，合计 43 个动作位
```

### 3.3 memory / time 的最终口径

```text
memory   整段文本；read 原文；write 整段替换；写入上限 200 token；
         成功只回 {"code":"OK"}，不带 updated_at
time     绝对分钟级时间 "YYYY-MM-DD HH:MM"；最多 8 步；最远 7 天；
         无 advance / 无相对时间 / 无快进
```

### 3.4 协议适配与设备能力（1012/02、03 的结论）

```text
SimuHome 是唯一真正把协议标准吃进能力模型的：Matter 设备类型 + 端点/集群/属性/命令，
  用发现工具（设备结构 + 集群文档）现场学，能迁真机
SMH-Bench / HomeFlow 是 schema 派：属性 + 服务 + 组件，扩展=加 spec；不绑定真机标准
HomeBench 是手工词表派：15 类设备、方法集写死
PersonalHomeBench 是厂商文档派：40+ 家电的特征级接口
我们的位置：HA 承担协议适配；六字段八动作是 HA 之上的语义收敛层
扩展路线（未定）：
  A 核心 8 动作 + 能力扩展注册表
  B 从 HA 服务清单自动映射/透传
  C 学 SimuHome 增加能力发现入口
```

### 3.5 HomeFlow 没有自动化规则任务

```text
HomeFlow 五类：Atomic / Compositional / Ambiguous / Context-Aware Multi-turn / Personalized Memory
没有 SMH TC4 那种 persistent trigger 规则；最接近的 TC2 State-dependent 是“按当下状态筛目标”
“写自动化规则的小游戏”应对齐 SMH TC4 的 Time-specific / State-driven / Time-and-state
对应我们一直未实现的 manage_rule
```

## 4. 可直接引用的实测数据

```text
离线测试       36 / 36（contract + memory + time_control + inspect_time + claim + roles）
真机 e2e       22 / 22
DeepSeek       12 / 12（8 个接口全部真实调用；list 先看到预约单、取消后消失）
到点回归       19:58 建单 at=19:59 → 灯亮 → 单从 list 消失 → 还原
一键脚本       3 房间 / 25 实体 / 0 unavailable，exit 0
broker 相对路径 重启后 25 实体 / 0 unavailable
冰箱等值写入   2.9 / 4.2 / 1.8 / 1.0 秒，全部成功
冰箱 5°C       5 次全成功（2.4 / 0.8 / 0.6 / 0.9 / 0.9 秒），state=5
小米云异常     用户 22:31 两次操作 TimeoutError（集成写超时 15s）
容器 TLS       api.io.mi.com：2.98s / 3.65s 成功，39.156.44.8 超时；宿主机 0.65s
目录重命名     8 个文件夹；更新 27 文件 / 69 处引用；旧路径残留 0
```

## 5. 踩过的坑

```text
HA 登录        “记住登录”是浏览器侧会话令牌，不是服务器记住密码；
                localhost 与 127.0.0.1 是不同 origin，登录态不共享
小米云         写超时 15s；容器到云端 TLS 偶发 3s+ / 坏 IP；系统代理半开
PowerShell     .ps1 必须 UTF-8 BOM；用 stdin 给 Python 传中文脚本会乱码
Mosquitto      相对路径依赖启动工作目录；凭证文件不能带 BOM
Git            推送需代理 7897；目录改名会产生大量 “删除+新增”，提交时留意 rename 检测
源文件时间      PDF 保留来源修改时间（5 月），不能代表加入项目的时间
```

## 6. 未做与待办

```text
① 提交并推送当前改动：1012 三份文档、16 号更新、8 个目录重命名 + 路径更新、
   source.md、README 等（当前 251 个 modified/deleted、10 个 untracked，尚未提交）
② 确认 UbiComp 目录日期口径：若按项目讨论时间，需要给一个起始日期（如 0908/0901），
   再改 0909_UbiComp三支柱讨论 前缀
③ 小米云稳定性：可选 A 固定快 IP（/etc/hosts 或 compose extra_hosts）、
   B 把写超时 15s 调大、C 清理系统代理半开状态——均未实施
④ manage_rule / 自动化规则小游戏：只有设计思路，未实现
⑤ M3（第三方集成抽样/命名泛化）：因只有一台真实设备仍暂缓
⑥ 多轮澄清：我们的 MCP 没有独立澄清工具；HomeFlow/SMH 的多轮是参照
```

## 7. 怎么拉起与验证

```powershell
# 一键拉起（HA / SimuHome / Mosquitto / 发布器）
powershell -NoProfile -ExecutionPolicy Bypass -File D:\D_coderesource\simuhome\_tools\start_simuhome_stack.ps1

# 离线 / 真机 / DeepSeek
cd D:\D_coderesource\simuhome\ha_mcp_universal
python -m pytest tests\test_contract_tools.py tests\test_memory.py tests\test_time_control.py `
       tests\test_inspect_time.py tests\test_claim.py tests\test_roles.py -q
$env:HA_TOKEN = (Get-Content D:\D_coderesource\simuhome\_secrets\ha_token.txt -Raw).Trim()
python tests\test_e2e_real_ha.py --config config.yaml
python tests\test_llm_surface.py --config config.yaml

# Web
http://localhost:8123/simuhome-dashboard
```

## 8. 关键文件索引（按新目录名）

```text
设计/交付      finally new paper\1005_10_04V2接入小米后设计方案\
                 14_记忆与时间调度接口详细设计.md
                 15_目前方案与交付接口.md
论文任务梳理   finally new paper\0520_方法相关的论文\16_四个Bench任务类型梳理.md
1012 调研      finally new paper\1012\
                 01_模型可见接口面与完整交互样例.md
                 02_协议适配与设备能力-五篇论文调研.md
                 03_主流智能家居协议与设备能力模型调研.md
HA 旧设计      finally new paper\0916_HomeAssistant双仓库接口分析\（11/12/13/14 等）
投稿策略       finally new paper\0909_UbiComp三支柱讨论\12_投稿策略-把三个支柱的故事讲圆.md
部署文档       deploy\01~05 + compose.yaml
v2 代码        ha_mcp_universal\（README.md、PLAN.md、src\home_mcp、tests）
接入层         simuhome_mqtt_publisher\（broker、publisher、make_credentials）
本地工具       _tools\start_simuhome_stack.ps1、mcp_probe.py 等
本系列上下文   finally new paper\new context\context1.md / context2.md / context3.md
```

## 9. 下次开新对话时可以直接说

```text
"读 finally new paper\new context\context3.md，先看 UbiComp 目录日期口径"
"把当前 1012、16 号和目录改名一起提交推送"
"先把小米云链路稳定下来：固定快 IP 或把 15s 写超时调大"
"设计并实现 manage_rule：SMH TC4 的三种触发规则 + 小游戏评测"
"米家又接新设备了，继续 v2 M3 的认领/角色/动作抽样"
```

## 10. 10-25 ~ 10-28 追加：从有限实例映射转向 HA 跨协议能力底座

### 10.1 这一轮真正修正的问题

此前建模主要围绕沙盒里的固定设备、米家一台真实冰箱和少量插件展开。这样能快速跑通任务，但规则依赖少数样本：换一个品牌、换一种协议、换一个型号，原映射就可能失去意义。用户把这个问题概括为“盲人摸象、管中窥豹”，本轮调查就是确认 HA 能否承担跨协议能力底座，以及端侧 MCP 应在哪一层收紧。

这轮形成的总结构：

```text
厂商协议与真实设备
  ↓ 由 HA 集成吸收
Config Entry
  ↓
Device Registry
  ↓
Entity Registry + State / Attributes + Service
  ↓ 保留完整事实
后台设备能力图
  ↓ 按设备 Profile 投影
common / custom
  ↓
端侧 MCP 与端侧模型
```

核心判断是：HA 接入层不必推倒重做。Matter、Zigbee、Z-Wave、MQTT、小米、美的、海尔、三星、LG 等差异继续交给集成处理；我们新增的工作集中在“从完整 Entity 事实生成统一 Capability，再生成受限 MCP 功能面”。

调查顺序：

```text
HA 官方设备组织方式
  → HA 官方 MCP 实测
  → MCP Host / Client / Server 角色澄清
  → Qwen3-4B / 1.7B 工具调用边界
  → 高星社区 HA MCP
  → Miloco 米家 MCP
  → Xiaomi / Midea / Haier / Tuya 等插件映射
  → 世界品牌与协议插件补充
  → 20 类设备功能面约束
```

### 10.2 HA 官方设备组织与 MCP 暴露

官方设备层的关系已经核实：

```text
Config Entry
  一个集成实例

Device Registry
  现实中的设备对象

Entity Registry
  设备向 HA 暴露的一个个功能对象

State / Attributes
  当前值、单位、可选模式、能力位

Service
  真正执行写入或动作的 HA 服务
```

一个 Device 可以有多个 Entity；一个 Entity 也可能包含多个 Capability。空调通常是一个 `climate` Entity 内含开关、目标温度、模式、风速等能力；米家冰箱则拆成多个 `sensor`、`number`、`select`、`event`、`button` Entity。

官方 MCP 的当前链路：

```text
┌──────────────────────────────────────┐       ┌──────────────────────────────┐
│ 客户端侧                              │       │ 服务端侧                      │
│                                      │       │                              │
│ 用户 / DeepSeek / Host               │       │ HA 官方 mcp_server           │
│ Host 管理模型调用、messages、tools    │       │ prompts / tools / resources  │
│   └─ MCP Client ─── MCP 协议 ────────┼──────▶│ Assist API                   │
│                                      │       │ HA Entity / Service          │
└──────────────────────────────────────┘       └──────────────────────────────┘
```

网络两端只有客户端侧和服务端侧。MCP Host 与 MCP Client 都在客户端侧，一个 Host 管理多个 Client，一个 Client 对应一个 Server。当前 HA 官方集成在实测链路中提供的是 Server；Host、模型调用和工具筛选由调用方应用负责。

### 10.3 HA 官方 MCP 的实际运行结果

实测环境与端点：

```text
HA 容器版本       2026.6.4
MCP SDK/版本      1.26.0
协议版本          2025-11-25
端点              http://localhost:8123/api/mcp
传输              Streamable HTTP
会话模式          无状态
```

连接先执行完整初始化，之后才读工具和 Prompt：

```text
MCP Client → HA MCP Server
  initialize
    协议版本 2025-11-25
    Client 能力与 clientInfo

HA MCP Server → MCP Client
  initialize result
    server = home-assistant
    version = 1.26.0
    capabilities = tools / prompts / resources

MCP Client → HA MCP Server
  notifications/initialized
    HTTP 202，不产生 JSON-RPC response

初始化完成后
  tools/list
  prompts/list
  resources/list
  prompts/get(name="Assist")
  resources/read(homeassistant://assist/context-snapshot)
```

初始化用一个逻辑 Client-Server 会话执行一次。后续模型请求和 `tools/call` 不必逐次重新初始化。HA 的 `/api/mcp` 无状态是指 HTTP 请求之间不保存 Server 对象，协议版本和逻辑会话仍由 Host/SDK 管理。

`tools/list` 实际返回 12 个 Tool：

| Tool | 当前用途 |
|---|---|
| `HassTurnOn` | 打开设备；对锁表示上锁 |
| `HassTurnOff` | 关闭设备；对锁表示解锁 |
| `HassCancelAllTimers` | 取消全部计时器 |
| `HassFanSetSpeed` | 设置风扇档位或百分比 |
| `HassBroadcast` | 通过家庭广播设备播放消息 |
| `HassClimateSetTemperature` | 设置 `climate` 目标温度 |
| `HassListAddItem` | 添加待办事项 |
| `HassListCompleteItem` | 完成待办事项 |
| `HassListRemoveItem` | 删除待办事项 |
| `GetDateTime` | 获取当前日期和时间 |
| `todo_get_items` | 查询待办事项 |
| `GetLiveContext` | 查询设备、传感器、实体或区域的实时状态 |

`prompts/get(name="Assist")` 返回 3114 字符的 Prompt，实际角色是 `assistant`，内容要求模型优先使用 intent 工具和 `GetLiveContext`。`resources/list` 返回 `homeassistant://assist/context-snapshot`，内容与实时上下文相关。

完整原始交互已整理在 `1024/03_官方MCP实际对接记录.md`，按 T0 ~ T13 记录了用户任务、初始化、工具发现、Prompt 获取、DeepSeek 调用、HA 返回和最终答复。

### 10.4 一次真实冰箱任务的结论

任务为：

```text
把米家对开门冰箱冷藏室目标温度设置为 5°C。
优先使用可用工具获取实时信息并尝试完成任务；
如果当前工具无法完成，请明确说明原因。
```

实际调用过程：

```text
Host 取得 Assist Prompt 和 12 个 Tool schema
  ↓
DeepSeek 第一次请求看到 messages + tools
  ↓
DeepSeek 选择 GetLiveContext
  arguments:
    name = 米家对开门冰箱（适用610L/606L/616L)  冷藏室 温度
  ↓
HA 返回
    domain = sensor
    state = 4
    unit_of_measurement = °C
  ↓
Host 把 tool result 回填给 DeepSeek
  ↓
DeepSeek 判断当前工具面没有可执行的冰箱目标温度写入入口
```

DeepSeek 没有调用 `HassClimateSetTemperature`，因为 `GetLiveContext` 返回的是 `sensor`，当前 Tool schema 也只把 `HassClimateSetTemperature` 指向 `climate` 域。

真实 HA 注册表里其实存在可写目标温度：

```text
number.target_temperature_p_3_2 = 5
  冷藏室设定温度

number.target_temperature_p_4_2 = -17
  冷冻室设定温度

select.mode_p_2_2 = 手动
  冰箱模式

sensor.temperature_p_3_1 = 4
  冷藏室当前温度

sensor.temperature_p_4_1 = -20
  冷冻室当前温度
```

所以缺口位于官方 Assist 暴露面和 MCP 工具面，插件已经生成了可写 `number` Entity。这个例子证明“设备能做什么”“HA 生成了什么 Entity”“MCP 当前给模型什么入口”是三件必须分开审计的事情。

### 10.5 Prompt 与 Tool schema 如何进入模型

HA MCP Server 分别提供 Prompt 和 Tool schema。MCP Host 读取后，按模型 API 的格式组织请求：

```text
DeepSeek / OpenAI-compatible API
  messages:
    role = assistant
      HA Assist Prompt
    role = user
      用户任务

  tools:
    12 个完整 Tool schema
```

这两个部分是模型 API 请求中的独立字段。MCP 不规定模型必须把它们拼成一段 system prompt。推理框架在送入模型前，会按自己的 chat template 和工具调用协议把两者渲染进同一段模型上下文；DeepSeek 服务端如何处理属于模型适配层。

本地 Qwen 的 chat template 可以观察到更底层的形式：

```text
<tools>
  工具 schema
</tools>

messages 按对话角色渲染

模型输出
<tool_call>
{"name": "...", "arguments": {...}}
</tool_call>

Host 执行工具后回填
<tool_response>
工具结果
</tool_response>
```

因此可以在我们自己的 Host 中改变 Prompt、工具筛选、权限和返回格式；MCP Client-Server 的 JSON-RPC、初始化、能力协商和 Tool schema 格式仍需遵循通用协议。

### 10.6 Qwen3-4B 与 Qwen3-1.7B

官方后训练版本都支持 tool calling：

| 模型 | Thinking BFCL v3 | Non-thinking BFCL v3 |
|---|---:|---:|
| Qwen3-4B | 65.9 | 57.6 |
| Qwen3-1.7B | 56.6 | 52.2 |

两者共用 Qwen3 工具调用 chat template，词表原生包含 `<tool_call>`、`</tool_call>`、`<tool_response>`、`</tool_response>`。模型本身不连接 MCP Server，仍由 Host 解析调用并转发。

当前证据来自官方模型卡、tokenizer/template 和官方适配文档，尚未在本机下载权重跑自定义智能家居工具集。端侧落地仍需实测工具选择准确率、参数准确率和多轮完成率。

### 10.7 高星社区 HA MCP

2026-10-27 按 GitHub 星数核验了三个直接实现 HA MCP Server 的仓库：

| 仓库 | 星数 | Tool 数 | 核验 commit | 主要路线 |
|---|---:|---:|---|---|
| `homeassistant-ai/ha-mcp` | 5272 | 87 | `9f5460e` | 配置、管理和调试整个 HA |
| `tevonsb/homeassistant-mcp` | 576 | 11 | `da5c30b` | 宽 Tool，内部用 action/command 分流 |
| `voska/hass-mcp` | 350 | 29 | `bff8453` | 状态、控制、历史、日志、Dashboard |

87 Tool 版本还在 README 中提供了紧凑检索模式：用 BM25 搜索 Tool，再用读、写、删除三个代理 Tool 执行。它解决了固定 schema 体积问题，同时增加检索步骤、英文关键词依赖和代理层复杂度。

三个项目都没有直接实现“按设备 Profile 暴露高频能力，长尾按需展开”的通用语义层。社区答案主要集中在覆盖面、宽接口和 HA 管理，不适合原样塞给端侧家庭控制模型。

### 10.8 Miloco 米家生态接口

本地源码：

```text
D:\D_coderesource\simuhome\milocoproject\xiaomi-miloco-main
```

原生 MIoT MCP：

```text
手动场景 MCP
  get_manual_scenes
  trigger_manual_scene
  send_app_notify

设备控制 MCP
  get_area_info
  get_device_classes
  get_devices
  get_device_spec
  send_get_rpc
  send_ctrl_rpc
  另有 2 个控制/查询 Prompt

摄像头 MCP
  当前为空占位
```

设备控制真实链路：

```text
get_devices
  → 得到 did
get_device_spec(did)
  → 得到 MIoT SPEC
send_get_rpc / send_ctrl_rpc
  → did + iid + value
```

这仍然是设备 ID 和 MIoT SPEC 层语义。测试代码分别把场景、设备控制和 HA 自动化 MCP 启在 `8080/8081/8082`，没有找到米家主服务自动启动这些 MCP 的生产入口。

Miloco 2.0 正式产品路线是 OpenClaw：3 个直接 Tool、16 个 Skill，大部分设备能力经 `miloco-cli` 执行。Host 预注入最近使用、最多约 50 台设备的目录，长尾设备再按需调用设备列表和 SPEC。这个设计承认了全量设备信息会给模型上下文造成压力，但仍没有抽象成跨品牌设备 Profile。

### 10.9 插件源码到底能提供什么

下载的插件源码可以分成四类信息：

```text
协议适配
  连接、鉴权、发现、收发消息

通用转换规则
  协议字段如何变成 HA Entity

型号/类别特例
  某型号改名、裁剪、单位转换、字节解析

运行时事实
  用户账号或局域网中发现到的真实设备、型号、状态、URN
```

不同插件的典型模式：

| 模式 | 代表 | 静态源码负责什么 | 动态部分来自哪里 |
|---|---|---|---|
| 规范驱动 | Xiaomi Home | MIoT 属性到 Entity 的通用规则、少量型号修补 | 账号设备 + MIoT SPEC |
| 静态协议驱动 | Midea AC LAN | 设备类型码、字节协议、固定属性集 | 局域网发现 + 型号 |
| 类别 + 动态字段 | Tuya / Smart Life | category 对应的常见 DP 与 Entity | 云端 DPCode、status、range |
| 云端产品描述 | Haier hOn | appliance key、控件映射 | hOn 账号与产品能力 |
| 云端数字模型 | Haier Uhome | readable / writable / valueRange 转换规则 | Uhome devdigitalmodels |

米家对开门冰箱没有在 Xiaomi Home 插件里写死完整型号表。真实路径是：

```text
小米云返回设备 model / did / urn
  → 获取 MIoT-Spec-V2
  → 通用属性转换
  → 生成 sensor / number / select / event / button
```

`profile_models.yaml` 中的 22 个 `viomi.fridge.*` 是当前跳过局域网直连的排除名单，不能解读成支持白名单。该冰箱仍可走云端接入。

海尔没有 HA 官方 `haier` 或 `hon` 集成。海外 hOn 使用 Andre0512/hon，核验 commit `70eb6c0`，支持 Haier / Candy / Hoover；中国区 Uhome 使用 banto6/haier，domain 为 `haier`，核验 commit `f3543b7`。hOn 当前确认的 7 个 Haier 冰箱型号是：

```text
HDPW5620ANPD
HBW5519ECM
HDW5620CNPK
HFW7720ENMB
HFW7819EWMP
HSW59F18EIPT
HTW5620DNMG
```

### 10.10 世界主流品牌与协议插件补充

第一轮 `ha_integration_sources` 有 16 个集成，没有海尔。本轮新增与它平行的目录：

```text
D:\D_coderesource\simuhome\ha_integration_sources_world
├─ official_core/     20 个官方核心集成
├─ hacs/              8 个 HACS 集成
└─ 01_世界主流品牌与协议插件索引.md
```

体积：

```text
源码本体        约 49.95 MiB
含 Git 元数据    约 78.20 MiB
```

官方核心补充覆盖 Samsung SmartThings、Samsung TV、LG ThinQ、LG webOS、Home Connect、Bosch、Whirlpool、Bauknecht、Miele、Daikin、三菱电机、富士通、海信、Panasonic、Sony、Android TV、Apple TV、ESPHome、Bluetooth、Thread。

HACS 补充覆盖：

```text
hon
haier_uhome
ge_smarthq
dreame_vacuum
samsungtv_smart
candy
dyson
smartthinq_sensors
```

三星并非只能走 Matter：

```text
SmartThings 家电
  → smartthings 云端 capability

三星电视
  → samsungtv 本地控制

支持 Matter 的三星设备
  → 通用 matter 集成

电视扩展
  → HACS samsungtv_smart
```

LG 同时有 `lg_thinq`（ThinQ 家电）、`webostv`（LG 电视）和旧设备补充 `smartthinq_sensors`。

`06_candy` 当前没有独立 LICENSE 文件。将它纳入可发布仓库前，需要单独确认授权。

### 10.11 设备功能面收紧方案

后台保留完整能力事实：

```text
Device
Entity
State
Attributes
Supported Features
Service
协议来源
实体注册信息
```

端侧 MCP 只保留两类可见功能面：

```text
common
  common.base
    device_id / profile / name / area / online / fault_summary
  common.observe
    高频只读状态
  common.control
    高频可写属性
  common.action
    高频一次性动作
  common.event
    高频事件

custom
  品牌特有、住宅组合、用户场景、低频但安全的能力
  默认只列摘要，需要时再展开完整参数
```

另有一类后台分类：

```text
hidden
  factory_reset / initialize / pair / unpair
  network_config / firmware_update / device_delete
  integration_config / debug_action / raw_protocol_write
  credentials / internal_test
```

`hidden` 不进入端侧模型的 MCP 上下文。冰箱、洗衣机等整机断电可能中断安全或运行流程，默认也不暴露主电源。

设备实例的可见能力按交集计算：

```text
设备实例真实能力
  ∩
Profile 允许能力
  =
MCP 可见能力

common = 可见能力 ∩ Profile.common
custom = 可见能力 ∩ Profile.custom
```

Profile 固定端侧模型使用的标准槽位，设备实例决定这个槽位是否真的存在。当前规范覆盖 20 类：

```text
air_conditioner
refrigerator
television
light
switch_plug
curtain_cover
fan
air_purifier
humidity_controller
vacuum
washer
dryer
dishwasher
water_heater
lock
camera
speaker
thermostat
door_window_sensor
environment_sensor
```

三个具体例子：

```text
空调
  common
    当前温度、开关、目标温度、运行模式、风速
  custom
    sleep_mode、quick_cool、auto_comfort
  hidden
    配对、固件升级、内部自检、原始属性写入

冰箱
  common
    冷藏/冷冻当前温度、各温区目标温度、门状态、模式
  custom
    速冷、速冻、假日模式
  hidden
    恢复出厂、网络配置、原始 MIoT 属性写入

电视
  common
    开关、输入源、播放状态、音量、静音、播放/暂停
  custom
    电影模式、夜间模式、字幕和音轨设置
  hidden
    系统重置、开发者调试、账号凭证
```

### 10.12 Entity 到 Capability 不能直接一一替换

HA Entity 内部比表面上复杂：

```text
一个 Entity 可能包含多个 Capability
  climate
    power / target_temperature / hvac_mode / fan_mode / swing_mode

一台设备对应多个 Entity
  refrigerator
    sensor / number / select / event / button

Domain 不等于设备类型
  switch 可能是插座、灯、净化器、洗衣机

能力藏在多个字段
  state + attributes + supported_features + service + entity_category

supported_features 是位掩码
  必须解码后才能知道当前 Entity 支持哪些动作

可写性由 Service 决定
  能读到 Entity 不代表存在合法写服务

Entity ID 会随命名和迁移变化
  端侧语义不能依赖 entity_id
```

转换步骤：

```text
1. 按 device_id 聚合 Entity
2. 结合 integration / manufacturer / model / domain 组合判定 Profile
3. 从 state、attributes、supported_features、service 抽取 Capability
4. 用集成和协议知识补齐单位、范围、枚举、可写性
5. 与 Profile 允许能力求交集
6. 生成 common / custom 投影
7. hidden 保留在后台事实层
8. 执行时再映射回真实 Entity 和 Service
```

统一 Capability 需要保留：

```text
capability_id
profile
surface
kind = observe / control / action / event
value_type
unit
range
enum
readable
writable
safety
source_entity_ids
source_service
protocol_metadata
```

冰箱最终应呈现为一个设备视图，不需要让模型逐个阅读 `midjd6_...` Entity ID。模型看到“冷藏室当前温度、冷藏目标温度、冷冻目标温度、模式、门状态”，执行时由 Host 映射回 `sensor`、`number`、`select`、`event` 和对应 Service。

### 10.13 当前完成到什么程度

已经验证：

```text
HA 2026.6.4 的官方 MCP 可连接
initialize / notifications/initialized 实际执行
tools/list 实测 12 个 Tool
prompts/get 实测返回 role=assistant 的 Assist Prompt
DeepSeek 对真实冰箱执行过连续工具调用
当前冰箱存在可写 number 目标温度实体
官方 MCP 当前工具面无法完成该写入
Matter 1.6 本地参考实现包含 Device Type / Cluster XML
米家、美的、海尔、Tuya 插件映射机制已分别核验
世界品牌与协议插件源码已下载并建立索引
```

当前仍是设计稿，尚未实现：

```text
从 HA Registry 自动生成 Capability 图的投影器
20 类 Profile 的可执行配置
common / custom / hidden 自动分类器
新的受限 MCP Tool 面
按 device_id 聚合后的写路径回读验证
端侧 Qwen3-4B / 1.7B 的真实工具选择评测
```

完整的官方层与 MCP 层图示见 `1024/02_官方设备能力模型与MCP收缩策略.md`；协议约束与职责划分见 `1024/05_MCP协议约束与HA官方边界及本地职责.md`；20 类设备约束和 Entity 转换规范见 `1024/10_设备功能面约束与MCP暴露规范.md`。

### 10.14 新增文档索引

```text
finally new paper/1024/
  01_HA跨协议能力语义调研.md
  02_官方设备能力模型与MCP收缩策略.md
  03_官方MCP实际对接记录.md
  04_Qwen3_4B与1.7B工具调用能力核验.md
  05_MCP协议约束与HA官方边界及本地职责.md
  06_高星社区HomeAssistantMCP工具面调研.md
  07_Miloco米家生态外部模型接口与工具面.md
  08_HA集成插件中的设备映射机制.md
  09_Matter协议与HA适配机制.md
  10_设备功能面约束与MCP暴露规范.md
```

### 10.15 下一次接续的直接入口

```text
先读 context3.md 第 10 节
  → 再读 10_设备功能面约束与MCP暴露规范.md
  → 读 05_MCP协议约束与HA官方边界及本地职责.md
  → 以米家冰箱为第一条真实实例实现 Capability 投影
```

第一项实现应先用现有冰箱验证：

```text
冷藏当前温度
  读取 sensor.temperature_p_3_1

冷藏目标温度
  读取/写入 number.target_temperature_p_3_2

冷冻当前温度
  读取 sensor.temperature_p_4_1

冷冻目标温度
  读取/写入 number.target_temperature_p_4_2

模式
  读取/写入 select.mode_p_2_2
```

验收任务：

```text
DeepSeek 或 Qwen 只看到 refrigerator common 功能面
  ↓
选择统一 Capability：冷藏室目标温度
  ↓
Host 映射到 number.set_value
  ↓
等待设备落定
  ↓
回读确认目标值
  ↓
返回精简的 verified 结果
```

## 11. 10-29 ~ 11-15 追加：设备能力三分、家庭记忆与完整工具面

这一阶段的主线从“HA 能否作为统一能力底座”继续推进到三个更具体的问题：

```text
一个设备的内部能力怎样组织
  ↓
四篇 benchmark 与 Miloco 分别怎样建模
  ↓
模型到底能看到哪些记忆接口和工具接口
```

### 11.1 新增文档

```text
finally new paper/1024/
  11_HA插件中的设备能力与映射关系.md
  12_设备能力三分与组件约束.md
  13_论文中的设备能力约束与Matter组件分析.md
  14_三种Benchmark与Miloco的设备建模分析.md

  L1_四个项目如何组织冰箱读数与温控.md
  L2_四个Benchmark与Miloco的家庭记忆.md
  L3_四个Benchmark与Miloco完整工具面.md
```

`13A` 曾短暂作为完整工具面文档使用，随后按用户要求改名为 `L3`。原来的 `13_论文中的设备能力约束与Matter组件分析.md` 保留，内容不与 `L3` 混用。

### 11.2 设备能力三层：Observe、Control、文字摘要

设备最终不再只按 HA Entity 平铺，也不把 Matter、MIoT 或厂商插件字段直接暴露给模型。当前设计采用三层：

```text
DeviceView
├─ Observe
│    当前温度、门状态、运行状态等只读信息
│
├─ Control
│    目标温度、模式、风速、开关等可写能力
│
└─ Summary
     给模型看的设备能力文字摘要
```

能力卡继续保留必选、可选、常见、不常见等约束，但 MCP 对端侧模型默认只暴露收缩后的常用能力：

```text
common capability
  → 默认可见

custom capability
  → 按需展开

hidden / administrative capability
  → 不交给端侧模型
```

空调的核心能力可以收成：

```text
开关
目标温度
模式
风速
```

其中开关、目标温度是基础控制；模式和风速是常见可选能力。摆风、定时、清洁等进入自定义能力面。

冰箱的能力面与空调不同：

```text
冷藏室当前温度
冷冻室当前温度
冷藏室目标温度
冷冻室目标温度
部分型号有模式
```

冰箱通常没有合理的设计性电源开关。真实使用中要靠断电而非控制面板“关机”，避免误操作造成食物腐坏。因此不能为了统一设备模型，给冰箱强行补一个 `switch`。

### 11.3 四篇 Benchmark 与 Miloco 怎样组织同一台冰箱

`L1` 专门限定在冰箱内部，不讨论客厅独立温湿度传感器。四个对象的组织方式如下：

```text
PersonalHomeBench
  家庭层有 appliances 和 sensors
  冰箱属于 ApplianceObject
  冷藏、冷冻温度属于家电内部 state / feature
  不把冰箱内部温度误放进家庭外部 SensorObject 集合

SMH-Bench
  家庭层是 rooms + devices
  一台冰箱内部用 component / attribute / service 表达
  冷藏室和冷冻室通过组件区分
  模型只看到 query_device / control_device / create_automation

SimuHome
  Refrigerator 与 Freezer 是两个逻辑设备类型
  每个设备内部仍是 Endpoint → Cluster → Attribute / Command
  冷藏与冷冻分别有 Temperature Measurement 和 Temperature Control

Miloco
  MIoT 设备 → Service / Property / Action
  一台冰箱可以有多个 siid 区分温区
  完整 MIoT-SPEC 不直接全部给模型
  中间经过 MIoT lite spec 和 device catalog 收缩
```

冰箱不只是一个 `fridge.temperature`。真正的语义至少要区分：

```text
当前读数
目标设定值
所在温区
可写范围
运行模式
门状态或其他只读状态
```

当前真实 HA 米家冰箱也已做过快照。模型层推荐把 `midjd6_...` 一类 Entity ID 聚合为单一冰箱视图，Host 再映射回：

```text
sensor.temperature_p_3_1
number.target_temperature_p_3_2
sensor.temperature_p_4_1
number.target_temperature_p_4_2
select.mode_p_2_2
```

其中冷藏、冷冻的当前温度和目标温度必须分开，不能合并成一个“冰箱温度”。

### 11.4 四篇 Benchmark 与 Matter 的约束路线

`13` 把论文约束路线和 Matter 组件模型做了横向对照，核心差别是：

```text
HomeBench
  手工 15 类设备和方法词表

PersonalHomeBench
  厂商文档派
  40+ 家电的特征级接口

SMH-Bench
  schema 派
  device → component / attribute / service

SimuHome
  Matter 标准派
  device → endpoint / cluster / attribute / command

Miloco
  MIoT-SPEC 派
  device → service / property / action
```

`14` 进一步分析了 PersonalHomeBench、SMH-Bench、SimuHome 与 Miloco 的建模示例、工具参数和实际接地边界。四篇 benchmark 的建模说明可用于参考，但不能把论文中的示例值冒充成官方完整数据或现实设备约束。

### 11.5 家庭记忆：五个对象的边界

`L2` 专门讨论记忆，不再混入完整设备工具面。

| 对象 | 记忆处理 | 专门记忆工具 |
|---|---|---|
| HomeBench | 不建模长期家庭偏好 | 无 |
| PersonalHomeBench | 家庭级记忆、成员级记忆、成员画像 | 有读取工具，也有计划层写入动作 |
| SMH-Bench | `memory_list` 直接进入提示词 | 无 |
| SimuHome | 公开任务没有长期家庭记忆 | 无 |
| Miloco | 感知记忆 + 家庭档案 + 习惯建议状态 | Host 记忆工具 + Skill/CLI 写入 |

PersonalHomeBench 的记忆读取接口是：

```text
get_household_memories(search_string, num_memories)
get_member_memories(member_id, search_string, num_memories)
```

后续核对发现 PersonalHomeBench 还存在计划生成动作：

```text
update_memories(household_id, memories)
```

它属于 `Toolbox for generating plans`，不是 `PersonalHomeTools` 的检索函数。它的作用是：

```text
添加或更新关于该家庭的记忆
```

该动作不能被遗漏成“所有 benchmark 都没有更新记忆接口”。

SMH-Bench 的记忆只以提示词变量形式出现：

```markdown
## User Memory

{{memory_list}}
```

`memory_list` 是上下文，不是 Tool。SMH-Bench 的直接工具面没有第四个记忆工具。

Miloco 的记忆分成三套：

```text
感知记忆
  memory/YYYY-MM-DD-miloco-perception.md

家庭档案
  profile.json / candidates.json / profile.md

习惯建议状态
  task-suggestions.json
```

完整数据流是：

```text
摄像头感知
  ↓ 每 15 分钟
miloco-perception-digest
  ↓
每日交互记忆 + 感知记忆
  ↓ 每天 00:00
home-dreaming
  ├─ Observe 提取候选
  ├─ Promote 晋升正式档案
  └─ Prune 统一主体、清理过期、commit
       ↓
profile.md
       ↓ before_prompt_build Hook
普通 Agent 的 system context
```

Miloco 的模型可见记忆面是：

```text
家庭档案摘要
  → profile.md 自动注入

感知事件
  → OpenClaw Host 的 memory_search / memory_get

完整档案增删改
  → miloco-home-profile Skill + miloco-cli

习惯建议状态
  → Miloco 插件的建议状态工具
```

Miloco 插件自身没有 `home_profile_read`、`home_profile_write`、`remember` 或 `recall` 这类专用 Tool。现有插件清单只注册：

```text
miloco_im_push
miloco_notify_bind
miloco_habit_suggest
```

`memory_search` 与 `memory_get` 来自 OpenClaw `v2026.5.20` 默认的 `memory-core`，不是 Miloco 插件注册的工具。

### 11.6 完整工具面单独放入 L3

用户要求 `L2` 只列记忆相关接口，完整工具面另写文档。因此：

```text
L2
  只讨论记忆读取、记忆写入、记忆注入和工具边界

L3
  完整整理四个 benchmark 与 Miloco 的模型侧工具面
```

`L3` 的核心数字：

| 对象 | 完整模型侧工具面 |
|---|---|
| HomeBench | 没有全局固定工具数；每台设备动态给出方法签名 |
| PersonalHomeBench | 28 个 PersonalHomeTools 辅助函数 + 6 个计划生成动作 |
| SMH-Bench | 3 个直接工具 |
| SimuHome | 最新 ICLR 英文稿 17 个工具 |
| Miloco | 自有 3 个直接 Tool；另依赖 Host 2 个记忆工具；另有 16 个 Skill |

PersonalHomeBench 的计划生成动作是：

```text
update_user_information
update_memories
update_sensor_status
update_appliance_status
update_pet_information
create_routine
```

SMH-Bench 的三个直接工具是：

```text
query_device
control_device
create_automation
```

SimuHome 的版本差异必须保留：

```text
本地 0524 中译本
  13 个工具

本地较新的 ICLR 2026 英文稿
  17 个工具
  新增 get_environment_control_rules
       ask_user
       cancel_workflow
       get_workflow_status
```

本地较新英文稿的直接工具面是：

```text
finish
get_environment_control_rules
ask_user
execute_command
write_attribute
get_all_attributes
get_attribute
get_device_structure
get_rooms
get_room_devices
get_room_states
get_cluster_doc
get_current_time
schedule_workflow
cancel_workflow
get_workflow_status
get_workflow_list
```

SimuHome 的工具体现 `Device → Endpoint → Cluster → Attribute / Command`，没有长期家庭记忆接口。

### 11.7 当前判断

这一阶段得到的不是一套已经落地的代码，而是对下一版统一语义 MCP 的边界修正：

```text
设备存储
  可以继续保留 HA / Matter / MIoT 的原始结构

模型暴露
  必须做 Observe / Control / Summary 投影

常用能力
  默认暴露

自定义能力
  按需展开

管理能力
  不交给端侧模型

家庭记忆
  稳定摘要自动注入
  近期事件按需检索
  完整档案走受控写入

完整工具
  单独留档，不塞进记忆设计文档
```

目前已经完成的是源码核验、论文核对、建模对照和文档落地。尚未完成的是把上述约束真正实现为新的受限 MCP Tool 面、自动 Capability 投影器和端侧工具选择评测。

### 11.8 下一次接续入口

```text
先读 context3.md 第 11 节
  ↓
设备内部组织
  读 L1

记忆设计
  读 L2

完整模型工具面
  读 L3

能力约束总设计
  回到 10 和 12

论文与 Matter 的约束依据
  读 13 和 14
```

第一项实现仍是：

```text
用当前米家冰箱
  → 聚合为一个 DeviceView
  → 只暴露常用 Observe / Control
  → Host 映射回 HA number / sensor / select
  → 执行后回读 verified
```
