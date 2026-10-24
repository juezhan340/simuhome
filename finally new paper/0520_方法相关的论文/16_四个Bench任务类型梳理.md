# 16：五个 Bench 的任务类型梳理（按 T 计数）

2026-10-07 稿，2026-10-13 增补 HomeFlow（方法论文 19，自带 SmartHome-Bench）为第 5 篇。
目的：把五篇 benchmark（01 HomeBench、04 SimuHome、03 SMH-Bench、02 PersonalHomeBench、
19 HomeFlow）的任务类型按"有多少个 T"列清楚，并给出每一类的实例数量。
数据来自 `0524_4个bench\` 与 `0520_方法相关的论文\19_2026_arXiv_HomeFlow-Data-Flywheel-Smart-Home_EN.txt`；
分类口径参考同目录的《4个Bench分类对比.md》《4个Bench总结.md》。

## 1 一屏总览：每篇有多少个 T

```text
Bench              类型命名          类型数                        实例总量
-----------------------------------------------------------------------------
HomeBench          VS/IS/VM/IM/MM    5 类                          17 万+ 条指令
SimuHome           QT1~QT4           4 大类 / 6 查询类型 / 12 条件   600 回合
SMH-Bench          TC1~TC7           7 大类 / 22 子类               1,100 任务（分布口径见 §4）
PersonalHomeBench  IG/CF/MC/FR/PG   5 类（3 反应 + 2 主动）        9,168 任务
HomeFlow           QT1~QT5（附录 TC1~TC5） 5 大类（声称 18 子类）     1,678 条测试样本
```

每篇的"T"命名不一样：HomeBench 是双字母缩写（VS/IS/…），SimuHome 是 QT，SMH-Bench 是 TC，
PersonalHomeBench 是任务缩写（IG/CF/…），HomeFlow 正文用 QT、附录又换成 TC（同一套内容两套编号，
且编号与 SimuHome / SMH-Bench 会撞车，见 §6）。

## 2 HomeBench：5 类

```text
VS  Valid Single        有效单设备指令：单设备、单方法，指令正确且可执行
IS  Invalid Single      无效单设备指令：单设备、单方法，但指令错误、不能执行
VM  Valid Multi         有效多设备指令：多设备多方法，全部正确可执行
IM  Invalid Multi       无效多设备指令：多设备多方法，全部错误、不能执行
MM  Mixed Multi         混合多设备指令：多设备多方法，有些正确有些错误
```

论文表 2 给出的指令数量（训练 / 验证 / 测试三档）：

```text
类型    训练集     验证集    测试集
VS      49,048     6,145     6,187
IS      53,771     6,731     6,765
VM       1,829       232       245
IM         826       124        97
MM      33,443     4,132     4,072
ALL    138,917    17,364    17,366

合计约 17.4 万条指令；100 个场景，每场景至少 47 台设备。
```

要点：5 类里两类是无效指令（IS / IM），一类是混合（MM）；这三类是 HomeBench 的核心考点（无效指令识别与拒绝）。

## 3 SimuHome：4 大类、6 个查询类型、12 个评测条件

```text
QT1  环境感知        正确感知环境条件与设备状态，并如实回答
QT2  隐含意图        从抱怨/间接表达里推断潜在目标
QT3  显式意图        准确执行指定设备 + 目标值的命令
QT4  时间推理（拆 3 个子类）
  QT4-1  未来调度     把多个设备调度到未来某个绝对时刻
  QT4-2  依赖调度     按设备完成时间建立依赖（如洗碗机洗完再开灯）
  QT4-3  并发调度     多个运行设备在时间约束下不冲突
```

```text
结构
  4 大类（QT1~QT4）
  其中 QT4 拆成 3 个子类 → 共 6 个查询类型
  每个查询类型再分可行（F）/ 不可行（IF）→ 12 个评测条件

数量
  每个评测条件 50 个回合
  12 × 50 = 600 回合（论文口径：12 种查询类型，600 回合）
```

要点：QT4 是唯一的时间/调度类，也是四篇里与我们的 time_control / 预约设计最直接对应的一类；QT1 是"先查状态再动作"的考点。

## 4 SMH-Bench：7 大类、22 子类

```text
TC1  Atomic Control                       原子控制
TC2  Compositional Control                组合控制
TC3  Ambiguous Intent                     歧义意图
TC4  Automated Task Scheduling            自动化任务调度（持久规则）
TC5  Context-Aware Multi-turn Interaction 上下文多轮
TC6  Personalized Memory                  个性化记忆
TC7  Environment-Grounded Query           环境接地查询（只读）
```

22 个子类（附录 C 表 15，每子类 50 条；括号内为代表性指令）：

```text
TC1 原子控制（3）
  1 Clear Commands        清晰命令      "把西厨房热水器目标水温设为 45 度"
  2 Colloquial Requests   口语化请求    "把主卧风扇调到吹风模式"
  3 Noisy Utterances      带噪措辞      "把卧室温湿度传感器，不对，等等，客厅传感器改别名……"

TC2 组合控制（5）
  4 Explicit Multi-device 显式多设备    "次卧智能屏继续放音乐并把音量调到 40；打开再锁上卡住的门锁；
                                        餐厅彩灯调到 30%；厨房风扇调自然风"
  5 Batch Operations      批量操作      "全屋所有灯调到 30% 亮度、最暖色温"
  6 State-dependent       状态依赖      "把当前亮度高于 80% 的灯调到 40%"
  7 Room-dependent        房间依赖      "除有活动的房间外，关掉其他房间的灯，并停掉还在播放的音箱/屏幕"
  8 Top-N Selection       Top-N 选择    "把风速最高的两台风扇调成睡眠模式并打开摆头"

TC3 歧义意图（3）
  9 State-level Ambiguity 状态级歧义    "房间太暗了"
 10 Scene-level Ambiguity 场景级歧义    "我要休息一会儿"
 11 Slot-missing          缺槽位        "用烤箱热一下午饭"（缺时长 → 追问；用户说 5 分钟后开始加热）

TC4 自动化任务调度（3）——唯一真正的"写规则"类
 12 Time-specific         时间触发      "今晚 10 点锁主卧门并开启防御模式；浴室风暖调最暗暖光、风扇最低档"
                                        Trigger: time_cron = 0 0 22 ...（到点触发）
 13 State-driven          状态触发      "厨房 15 分钟没有移动时，自动关玄关风暖、把客厅智能屏音量调到 30"
                                        Trigger: device(5267).no_move_duration > 900
 14 Time-and-state        时间+状态触发  "每天下午两点，如果厨房温度仍高于 35 度，启动客厅扫地机、
                                        卧室风暖风扇调高档"
                                        Trigger: time_cron = 每天 14:00 且 device(...).current_temperature > 35

TC5 上下文多轮（4）
 15 Context Continuation  上下文延续    "把它的别名改成主卧温湿度传感器"（承接上一轮）
 16 Topic Shifts          话题切换      "打开主卧阳台风暖，风扇自然风，亮度 50%"（不继承上一轮媒体目标）
 17 Coreference Resolution 指代消解     "把那个音箱也打开"（指代上一轮的音箱）
 18 User Corrections      用户纠正      "别调次卧了，打开男孩房风扇"

TC6 个性化记忆（2）
 19 Short-term Memory     短期记忆      "算了别调客厅空气盒子了，改成打开客卫风扇"
 20 Long-term Memory      长期记忆      "该睡觉了"（检索用户睡眠偏好并应用）

TC7 环境接地查询（2）——只读，不改设备
 21 Status Inquiries      状态查询      "次卧和客卫的扫地机现在是什么状态？"
 22 Quantity Queries      数量/聚合查询  "现在有多少台风暖灯同时开了灯和风扇？"
```

数量口径（重要）：

```text
摘要/正文统计   1,100 条人工审核实例（7 大类 / 22 子类）
附录 C          每子类 50 条（25 简单 + 15 中等 + 10 复杂）→ 22 × 50 = 1,100
正文分类句      TC1 150 / TC2 250 / TC3 150 / TC4 150 / TC5 200 / TC6 100 / TC7 150
                相加为 1,150，与 1,100 不一致——按 22×50 口径 TC7 应为 100（它只有 2 个子类）
                引用时以 1,100 为准，并注明正文 TC7=150 与附录口径冲突
环境分层        550 简单 / 330 中等 / 220 复杂
                （平均 5.5 房间 5.5 设备 / 10.5 房间 35 设备 / 31 房间 135 设备、17 种设备类型）
```

要点：TC4 是唯一真正的规则/调度类（persistent time- or state-triggered rules），包含"时间特定、状态驱动、时间+状态"三种玩法；TC6 是记忆类；这两类分别对应我们的 time_control 和 memory。

## 5 PersonalHomeBench：5 类（3 反应 + 2 主动）

```text
反应式（问答）
  IG  Information Grounding   信息接地：从家庭记忆/画像里检索事实并作答
  CF  Counterfactual          反事实：改动一个条件后推断结果（成对变体）
  MC  Multiple Choice         多项选择：受限选项下的信息接地

主动式（规划）
  FR  Feature Recommendation  特征推荐：按偏好做排序/推荐
  PG  Plan Generation         计划生成：生成结构化多步计划
```

论文给出的实例数量（纯文本子集 + 多模态子集）：

```text
类型    纯文本    多模态    合计
IG       1,000       742     1,742
CF       2,000     1,484     3,484
MC       1,000       742     1,742
FR       1,000       100     1,100
PG       1,000       100     1,100
合计     6,000     3,168     9,168
```

要点：5 类里 2 类是主动规划（FR/PG）；工具型 agent 设置下 IG/CF/PG 掉点最严重，"会不会主动查、会不会稳定调工具"是主要瓶颈。

## 6 HomeFlow：5 大类（QT1~QT5；附录 TC1~TC5）

HomeFlow 自带 SmartHome-Bench（基于 HomeEnv）；正文用 QT1~QT5，附录表 10 用 TC1~TC5，
两套编号指同一套内容：

```text
TC1 / QT1  Atomic Control                       原子控制
  模板：Clear Commands / Colloquial Requests / Noisy Utterances
TC2 / QT2  Compositional Control                组合控制
  模板：Explicit Multi-device / Batch Operations / State-dependent /
       Room-dependent / Top-N Selection
TC3 / QT3  Ambiguous Intent                     模糊意图
  模板：State-level Ambiguity / Scene-level Ambiguity / Slot-missing
TC4 / QT4  Context-Aware Multi-turn Interaction 上下文多轮
  模板：Context Continuation / Topic Shifts / Coreference Resolution / User Corrections
TC5 / QT5  Personalized Memory                  个性化记忆
  模板：Short-term Memory（原文此处以分号结尾，疑缺尾巴）
```

数量口径：

```text
实例总量      1,678 条测试样本（论文附录 E）
分类声称      5 大类 / 18 子类
实际可见      表 10 只列出 3+5+3+4+1 = 16 个模板，TC5 只列一条且带悬挂分号
              （已渲染 PDF 原页确认，不是抓取丢失）——引用"18 子类"时注明与可见列表不一致
每类数量      未公开；表 1 给的是各模型在 QT1~QT5 上的成功率与工具调用/Token 开销
成功率示例    HomeFlow-RL-8B：QT1 94.72｜QT2 79.63｜QT3 90.68｜QT4 88.31｜QT5 82.11｜总 87.03
              基座 Qwen3-8B：QT1 27.70｜QT2 10.19（最低）｜QT3 13.26｜QT4 68.83｜QT5 60.00｜总 31.87
```

生成扰动维度（不是任务类型，但决定任务构型，表 6）：

```text
Intent Clarity（clear / ambiguous / missing-slot）
Noise Level（clean / noisy）
Device Scope（single-device / multi-device / batch-device）
Execution Dependency（independent / dependent）
蓝图示例里每题的 Query Type 就写成这类组合，例如
  missing_slot+quiet+dependent+multiple_devices
```

原文例子（可直接引用）：

```text
原子+组合（附录 B.1.2 蓝图）：
  "If the AC is off, turn on heating and set the temperature to around 24 °C;
   also the study light is too dim—raise both brightness and color temperature."
上下文多轮（附录 MCTS 示例）：
  "The glare is strong, the fan swing distracts me. And the airflow is too strong."
  → 调窗帘角度 40°；下一轮 "The fan is still distracting me..." → 关摆头 + 风扇切 sleep 模式
个性化记忆（表 10 的例句）："sleep time" / "as usual" / "set it how I like it"
模糊意图（案例研究）："It feels a little cool. Change the air conditioner to natural breeze."
  → 先澄清房间，再把"自然风"翻译成 fan-only 模式
```

编号陷阱（引用时必须带论文名）：

```text
HomeFlow QT4 = 上下文多轮   ≠  SimuHome QT4 = 时间推理
HomeFlow TC4 = 上下文多轮   ≠  SMH-Bench TC4 = 自动化调度
HomeFlow TC5 = 个性化记忆   ≈  SMH-Bench TC6 = 个性化记忆（编号不同）
```

### 6.1 有没有 SMH-Bench TC4 那种"自动化规则"任务？——没有

全文检索 automation / rule / trigger / schedule 能确认：

```text
automation 只出现在：
  - 画像生成提示词里的设计原则 "Habits → automation scheduling"（行 979）
    ——把用户习惯字段映射到自动化调度场景，属于画像维度，不是任务类型
  - 参考文献标题、"自动生成布局"等普通用词
rule 的命中全是 rule-based verification / rule-based reward /
  generation rules / clarification rules——验证器、奖励与提示词规则
trigger 只有奖励机制里的 "triggers a negative penalty"
schedule 只有上述 "Habits → automation scheduling" 与记忆例句里的
  "preference-triggering instructions"
```

执行空间上，A_Exec 允许生成 Python 代码片段 / API 调用（行 120-131），
Gym 环境也提供 pyexec 动作（行 722）；但评测只看"回合结束时状态是否满足目标"，
没有规则的注册、持久化、触发与校验机制——即使模型写了自动化代码，HomeFlow 也不评它。

与 SMH TC4 的区别（例子）：

```text
HomeFlow TC2 State-dependent（一次性）
  "把亮度高于 80% 的灯关掉"
  → 在当前时刻筛出满足条件的灯，执行一次，任务结束

SMH-Bench TC4 Time-and-state（持久规则）
  "每天下午两点，如果厨房温度仍高于 35 度，启动客厅扫地机并把卧室风扇调到高档"
  → 注册 trigger（cron + 温度条件）+ action；之后按时间/状态触发并接受校验
```

结论：HomeFlow 的"条件"是当下筛目标，不是自动化规则；
要做"写自动化控制的小游戏"，参照 SMH TC4 的三种子类与我们 manage_rule 的设计（未实现）。

## 7 横切对比

```text
维度              HomeBench        SimuHome        SMH-Bench         PersonalHomeBench  HomeFlow
类型命名          VS/IS/VM/IM/MM   QT1~QT4         TC1~TC7           IG/CF/MC/FR/PG     QT1~QT5（附录TC1~TC5）
类型数            5                6（12 条件）     7（22 子类）       5                  5 大类（声称 18 子类）
实例量            17万+ 指令       600 回合        1,100~1,150       9,168              1,678
时间/调度         ○                ●（QT4）        ●（TC4）          ◐                  ○
规则/自动化       ○                ◐               ●（TC4）          ○                  ○
记忆/个性化       ○                ○               ●（TC6）          ●（核心）           ●（TC5）
无效/拒绝         ●（IS/IM/MM）     ●（IF）         ●（TC3）          ◐                  ●（TC3 澄清）
多设备组合        ●（VM/IM/MM）     ◐               ●（TC2）          ◐                  ●（TC2）

（● 有明确对应类；◐ 部分/间接；○ 无。HomeFlow 无时间与规则类，
   "Habits → automation scheduling" 只在画像设计原则里出现，没有落成任务类型。）
```

## 8 与我们项目的关系

```text
SMH-Bench TC4 自动化调度     → time_control（一次性预约）+ manage_rule（持久规则，未实现）
SMH-Bench TC6 个性化记忆     → memory（整段文本读写）
SimuHome QT4 时间推理        → time_control 的时间/调度语义
HomeBench IS/IM/MM 无效指令  → finish 的 refused，以及 execute_action 的错误码路径
PersonalHomeBench IG/CF      → memory 读、check_goal 判定
HomeFlow QT1/QT2 原子/组合   → observe/inspect + execute_action（与我们 8 动作面直接对应）
HomeFlow QT4 上下文多轮      → 对话历史由宿主客户端承担；我们工具本身无跨轮状态
HomeFlow QT5 个性化记忆      → memory
HomeFlow QT3 模糊意图        → 需要澄清能力：我们目前没有单独澄清工具（只能 refused 或按唯一解释执行），
                              这是对照 HomeFlow/SMH 的一个已识别缺口
HomeFlow 自动化规则          → 无此类任务；"写规则小游戏"参照 SMH TC4 + manage_rule（未实现）
```

## 9 来源

```text
论文原文
  01  HomeBench（ACM CHI 2024，中文译本）
  04  SimuHome（2025 arXiv，中文译本）
  03  SMH-Bench（2026 arXiv，英文原文）
  02  PersonalHomeBench（2026 arXiv，中文译本）
  ——均在 finally new paper\0524_4个bench\
  19  HomeFlow（2026 arXiv，英文原文；正文 §4.1 + 表 1、附录 B.1.2、附录 E 表 10、附录 F 案例）
  ——finally new paper\0520_方法相关的论文\19_2026_arXiv_HomeFlow-Data-Flywheel-Smart-Home_EN.txt

已有梳理
  0524_4个bench\4个Bench分类对比.md        三大类分组 + 四篇任务分类体系
  0524_4个bench\4个Bench总结.md            逐篇总结（含任务类型与规模）
  三类内部对比分析-Bench四篇与方法BC.md  27 个方面的覆盖打表
  1012\02_协议适配与设备能力-五篇论文调研.md  HomeFlow 的接口/动作空间口径（含行号）
```
