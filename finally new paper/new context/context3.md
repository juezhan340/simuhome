# context3：v2 八工具、可移植部署与 1012 调研（2026-09-27 ~ 10-24 全过程记录）

本文件用于新对话接续，接在 `context2.md`（09-17 ~ 09-27）之后。
写法沿用 context1/context2：记录关键决策、实测数字、踩坑与待办，不是交付清单。

```text
工作目录      D:\D_coderesource\simuhome
本轮跨度      2026-09-27 ~ 10-24
主线          把 v2 MCP 收口成 8 个可交付工具；把仓库变成别人可复现的方案；
              补齐 1012 三份接口/协议调研；完成论文任务类型梳理与目录归档
当前状态      代码与文档已成型；目录已按日期前缀重命名；大量改动尚未提交
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
