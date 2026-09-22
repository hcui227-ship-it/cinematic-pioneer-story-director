<a id="generation-engine"></a>
# 镜头生成引擎（AI Shot Generation Engine）：导演意图 → 控制需求 → 控制信号 → 路线 → Prompt

> 本文件解决镜头层面最容易被忽略的一件事：**AI 导演不是把镜头"写成提示词"，而是决定这个镜头该用什么控制信号、走什么生成路径，才能以最低试错成本拿到画面。**
> 核心命题：**Prompt 不是中心，是最后一公里。** 身份、构图、运镜、空间、衔接这些能被确定性信号锁定的东西，不要交给形容词。
> 本文件细化导演状态机的 S4–S6：Shot Contract → **Control Map** → **Generation Router** → **Generation Recipe** → Generate → **Director Review（Accept/Repair/Reroute）**。

> **小白提示（对外沟通用白话，内部存档用专业名）**：本文件是镜头内部机制、术语较密；讲给用户听时统一换大白话，权威对照见 [`plain-language-glossary.md`](plain-language-glossary.md)。核心几个：Shot Contract＝镜头说明书、Control Analysis＝判断这镜最难做对什么、Control Map＝控制点清单（必须钉死 / 随便发挥）、Control Budget L0–L3＝花几分功夫锁画面、Control Signal＝用来钉住画面的东西、Router＝选哪条做法、Proxy＝动作小样、Recipe＝镜头配方、Accept/Repair/Reroute＝能用 / 局部修 / 换路子。
>
> **9 步白话版**：①这镜要什么情绪和关键瞬间 → ②判断这镜最难做对什么（脸 / 构图 / 表演 / 运镜 / 站位 / 衔接 / 形变）→ ③列"必须钉死的"和"随便 AI 发挥的" → ④定花几分功夫（L0 只写文字 … L3 全套参考+动作小样+反复修）→ ⑤给每个必须钉死的项选"用什么钉"（参考图 / 开场图 / 结尾图 / 动作小样 / 音频 / 蒙版）→ ⑥选生成做法（文字生 / 图生 / 首尾帧 / 视频改视频 / 局部修 / 拆开做）→ ⑦把能钉的控制从提示词里拿走，提示词只留动作、克制表演、氛围质感 → ⑧存成镜头配方 → ⑨生成后判定：能用就进粗剪 / 小毛病局部修 / 做法错了换路子或拆镜。

## 1. 引擎总流程

```
导演意图 (Director Intent)
      ↓
Shot Contract（叙事任务 / 首态→末态 / 不可失败项）
      ↓
Control Analysis：这镜最难控制的是什么？（身份/构图/表演/运镜/空间/衔接/形变）
      ↓
Control Map：MUST CONTROL（要素+强度） ＋ CAN DRIFT（明确放手）
      ↓
Control Budget：L0–L3（控制预算，决定最多上多少信号）
      ↓
Generation Router：为每个 HIGH 项选控制信号 → 选生成方法 → 备齐输入资产
      ↓
把能锁的控制从 Prompt 中拿出去（Prompt 只留意图/氛围/表演基调）
      ↓
Generation Recipe（镜头完整配方）→ GENERATE
      ↓
Director Review：┌ ACCEPT 进粗剪 ／ REPAIR 局部修 ／ REROUTE 换方法·拆控制链·改戏
```

顺序不可颠倒：**没有做 Control Map 和信号分配，不许直接写 Prompt。**

## 2. 控制信号分类学（Control Signals Registry，开放可注册）

一个 AI 镜头实际能被这些东西控制。先认识信号，再谈提示词。

| 控制信号 | 主要锁定什么 |
|---|---|
| 文本 Prompt | 行为意图、氛围、摄影/运动意图、表演基调、难以信号化的细微变化 |
| 角色参考图 Character Ref | 人物身份、脸、发型、体征 |
| 服装/造型参考 | 服装、妆造、随身标志物 |
| 场景参考图 Environment Still | 空间、美术、环境光线基调 |
| 道具参考 | 关键道具外形与归属 |
| 构图静帧 / 首帧 Start Frame | 景别、机位、构图、镜头从哪一帧开始 |
| 尾帧 End Frame | 镜头最终必须到哪、末态、出画状态 |
| 首尾帧对 | 起点→终点的运动/变化方向 |
| 视频参考 / V2V | 动作、节奏、表演、镜头运动；或风格化/重绘 |
| Motion Reference | 人或物体的运动规律 |
| **运动/空间代理 Motion/Spatial Proxy**（见第7节） | 相机轨迹、空间关系、blocking、走位节奏 |
| Pose / Depth / Edge / Lineart / Normal 等结构控制 | 姿态、深度、轮廓、空间结构 |
| Mask / 局部区域 | 只改人物、背景或某个物体（局部修复） |
| 多图 Reference 组合 | 身份+服装+场景+道具的组合约束 |
| 音频 Audio | 口型、表演节奏、动作节奏 |
| 前一镜成片/尾帧 | 下一镜连续性的视觉锚点 |
| 种子/模型版本/参数 | 可复现性（弱控制，但必须记入 Recipe） |

**注册表是开放的**：未来出现任何新生成能力，只需在本表新增一行"信号名 | 锁定什么 | 输入形式 | 适用控制难题 | 局限"，引擎流程不变（见第12节）。

## 3. Control Analysis：先问"这镜最难控制的是什么"

收到 Shot Contract 后不写词，先判断主难题落在哪些维度（可多选，标出最重的）：

| 控制难题 | 典型症状 | 优先控制信号 |
|---|---|---|
| **identity 身份** | 脸/人漂、换人 | 角色参考、多图、首帧 |
| **composition 构图/机位** | 景别机位不对、主体占比错 | 构图静帧、首帧、结构控制 |
| **performance 表演** | 表情假、情绪错、口型对不上 | 视频/表演参考、音频驱动、克制的表演 Prompt |
| **camera 运镜** | 乱晃、推拉变焦无理由、轨迹错 | 运动代理（白膜/相机路径）、视频参考、明确"固定" |
| **spatial 空间/blocking** | 多人站位乱、穿模、地理关系错 | 场景参考、空间代理、depth、首尾帧 |
| **transition 衔接** | 动作重演、相位断、左右反、越轴 | 前镜尾帧当首帧、首尾帧对、复述末态 |
| **transformation 形变/特效** | 状态变化、受伤、变老、物质变化失败 | 首尾帧、结构控制、分层合成、V2V |

## 4. Control Map：MUST CONTROL 与 CAN DRIFT

每个要生成的镜头先产出一张控制地图，再谈方法与 Prompt。

```
SHOT: SH_023
DIRECTOR INTENT
  情绪弧：慌张 → 突然犹豫
  叙事功能：她第一次产生逃跑的念头
  关键瞬间：门口回头
MUST CONTROL（控制强度 HIGH / MEDIUM / LOW）
  角色身份      HIGH
  服装          HIGH
  便利店空间    MEDIUM
  回头动作      HIGH
  相机后拉      HIGH
  雨            MEDIUM
CAN DRIFT（明确允许 AI 自由发挥，不重生）
  雨滴形态、路人、背景小物、头发细节、灯牌细节
```

- **MUST CONTROL**：漂了这条镜头就废，逐项映射到第2节的信号；HIGH 项必须有确定性信号兜底，不能只靠 Prompt。
- **CAN DRIFT**：显式放手清单。写出来是为了**阻止自己为不可见的细节烧重生**（观众看不到、不影响理解的就不投资）。
- 对白片同样适用：正脸台词镜的"身份/口型/末态"是 HIGH，背景陈设通常 CAN DRIFT。

## 5. Control Budget L0–L3（控制预算）

不是每个镜头都塞满参考。先按四个因子给镜头定级，再决定最多上多少控制信号：

- **Narrative Importance** 叙事重要度（拿掉是否影响理解）
- **Identity Sensitivity** 身份敏感度（是否主角正脸/定妆资产）
- **Motion Complexity** 运动复杂度（运镜+动作+形变）
- **Continuity Sensitivity** 衔接敏感度（是否承接相邻镜动作/末态/轴线）

| 预算 | 信号配置 | 适用 |
|---|---|---|
| **L0 Prompt only** | 仅 Prompt | 过场、空镜、无主角、无衔接（如 2 秒夜高架车流） |
| **L1 Prompt + 1 reference** | 单张角色或场景参考 | 单人、轻运动、非相邻、远景 |
| **L2 Multi-ref + controlled start frame** | 多参考 + 受控首帧 | 主角戏、相邻连续对话、中等运镜、正脸口型 |
| **L3 Reference package + start/end frame + motion/spatial proxy + iterative repair** | 全套参考 + 首尾帧 + 运动/空间代理 + 迭代局部修复 | 高潮、复杂运镜、严格动作接点、多人 blocking、奇观形变 |

- 连续性投入并入本预算：衔接敏感度是定级因子之一，具体的末态/轴线/持物手别作为 MUST CONTROL 项，用尾帧/前镜成片信号锁定。
- 口型 A+/A/B/C 是表演维度内的分级，与预算正交：A+ 正脸长台词通常把镜头整体推到 L2/L3。
- **预算封顶**：L0 镜头不允许升级成 L3 去追求完美；超预算说明镜头设计不经济，回 Cut Map。

## 6. 把控制从 Prompt 里拿出去（Prompt 减负映射）

| 导演需求 | 交给确定性信号 | Prompt 里只保留 |
|---|---|---|
| 固定角色 | Character Ref | 不写长相 |
| 固定服装 | Costume Ref | 只写服装的**变化/状态**（淋湿、破损） |
| 固定场景 | Environment Still | 不堆陈设形容词 |
| 构图/景别/机位 | 首帧/构图静帧 | 镜头意图一句话 |
| 精确相机轨迹（如绕人 180°） | 运动代理/相机路径/视频参考 | 运动意图，不写轨迹分解 |
| 末态/动作接点 | 尾帧/前镜尾帧 | 不描述末态细节 |
| 动作规律 | Motion Reference/代理 | 动作目的与力度 |
| 只改一处 | Mask/局部重绘 | 只描述目标区域 |
| 口型/气口 | 音频驱动/台词内嵌 | 表演基调 |
| 氛围、光感、质感、风格、微妙情绪 | **这些才真正适合 Prompt**（或风格参考） | 主体内容 |

结果：当相机轨迹已由白膜负责、身份由参考图负责，Prompt 可以很短，例如：
`She hesitates for a fraction of a second, then turns toward the doorway. Restrained performance, subtle breathing, rain catching the fluorescent light, natural cinematic motion.`
**短而有效，胜过 500 字全包。**

## 7. Proxy-driven Generation（代理驱动生成，模型无关抽象）

凡是能提供 **时间 + 空间 + 运动** 信息的低质量源，都是运动/空间代理（Motion/Spatial Proxy），不要求精细 3D：

- Blender 白膜、Unreal 简模、简单 3D camera path
- AI 生成的粗糙视频、低质量测试视频
- 真人手机实拍、火柴人/Pose animation
- 关键帧 pose 序列、depth/blocking 草图

用途：锁定相机轨迹、多人走位与朝向、动作节奏与时长、空间关系。**方法论层只认"代理"这个抽象，不绑定某个软件**——新工具出现也不用改 Skill 底层。代理通常只进控制端，不进最终画面。

## 8. Generation Router：控制信号 → 生成方法

为 MUST CONTROL 的 HIGH 项逐项选好信号后，信号组合决定生成方法：

| 生成方法 | 用到的信号 | 解决 |
|---|---|---|
| T2V 文生 | Prompt | L0 探索/空镜 |
| I2V 首帧驱动 | 首帧（可由角色+场景合成） | 保脸保景保构图（主力） |
| 首尾帧/续写 | 首帧+尾帧 | 运动方向、动作接点、锁末态 |
| 多图/角色参考驱动 | 角色/服装/场景/道具多图 | 多主体一致 |
| 图生图合成首帧→I2V | 先合成角色入场景再驱动 | 复杂同框 |
| Reference Video / V2V | 视频参考 | 借运动/节奏/表演，或风格化 |
| **Proxy-guided I2V** | 运动/空间代理 + 角色/场景 | 精确运镜、走位、blocking |
| Str- **CAN DRIFT**：显式放手清单。写出来是为了**阻止自己为不可见的细节烧重生**（观众看不到、不影响理解的就不投资）。音频 | 口型与节奏 |
| Inpaint / Mask 局部修复 | Mask | 只修脸/手/背景，不全片重生 |
| **Chain 链式拆分**（见第10节） | 多方法串联 | 单一方法扛不住的高难题 |
| 声画分轨 | 画面 + 后期配音 | 规避内嵌口型风险 |
| 纯后期 | — | UI/数字/招牌/字幕 |

选择逻辑：按 HIGH 控制项映射信号 → 信号组合落到方法 → 方法数量受 Control Budget 封顶。**身份优先 I2V/多图；轨迹优先代理/V2V；接点优先首尾帧；局部问题优先 Mask；文字归后期。**

## 9. Generation Recipe（镜头级完整配方，而非一段 Prompt）

每个镜头保存的是"配方"不是"提示词"，这样下一镜、下一部、换模型时才能真正复现和学习：

```yaml
shot: SH_023
director_intent: { 情绪弧: 慌张→犹豫, 叙事功能: 首次起逃跑念, 关键瞬间: 门口回头 }
control_budget: L3
control_map:
  must_control: { 身份: HIGH, 服装: HIGH, 空间: MEDIUM, 回头动作: HIGH, 相机后拉: HIGH, 雨: MEDIUM }
  can_drift: [雨滴形态, 路人, 背景小物, 头发细节, 灯牌细节]
strategy: proxy_guided_i2v
inputs:
  character_ref: char_A_03
  environment_ref: store_night_02
  start_frame: SH023_start_v04
  motion_proxy: SH023_cam_v02.mp4
control_signals_used: [character_ref, environment_ref, start_frame, motion_proxy, audio]
continuity: { 口型级: B, 承接: SH_022 尾帧, 末态交接: SH_024 首帧 }
prompt:
  action: "she runs inside, stops abruptly and looks back toward the street"
  performance: "frightened but restrained"
  atmosphere: "cold fluorescent light, heavy rain outside"
  negative: [no exaggerated expression, no camera shake, no costume change]
audio: 台词/气口/环境声（内嵌或分轨注明）
model: { route: I2V, engine: 实测填, 参数/seed: 记录 }
result: { selected_take: v07, 状态: accepted }
failure_history:- 连续性投入并入本预算：衔接敏感度是定级因子之一，具体的末态/轴线/持物手别作为 MUST CONTROL 项，用尾帧/前镜成片信号锁定。: 良好
  v07: 选用
repair_reroute_log: （每次 Repair/Reroute 的诊断与动作）
```

分工：**Recipe＝单镜头完整档案**；`ai-production-measurement.md` 的生成日志＝跨镜汇总与成本统计，Asset Bible＝资产版本台账，缺陷 D1–D10＝Review 诊断码。Recipe 同时是 AI 来源披露与跨模型迁移的证据。

## 10. Director Review：Accept / Repair / Reroute

生成后按 MUST CONTROL 的可判定验收逐项核对，进三条分支之一：

- **ACCEPT**：不可失败项全过 → 标记 take 状态（accepted / candidate），进 Rough Cut。
- **REPAIR（局部修）**：问题局限在可隔离区域 → Mask 局部重绘、脸部/手部修复、短 V2V、后期；**不为一个局部缺陷重生整条**。
- **REROUTE（换方法/拆控制链/改戏）**：系统级失败，禁止继续改 Prompt 堆词。
  1. 先定位失败来自哪个控制维度：**信号缺失、信号冲突、还是模型能力上限？**
  2. 典型拆链：
     - **身份 + 复杂运镜互相打架**（代理+V2V 仍保不住人）→ 拆成 ①稳定人物+简单相机生成 → ②相机/运动变换（V2V/代理）→ ③局部脸部修复。
     - **表演/口型失败** → 降口型级、改侧脸/反应镜（B 级）、改音频驱动或声画分轨。
     - **空间/多人混乱** → 先用空间代理锁 blocking，或拆成关系清晰的双人/单人镜。
     - **形变/特效失败** → 首尾帧+结构控制，或分层合成。
  3. 同一镜 Reroute 累计到阈值仍不经济（参考 measurement 熔断，A+ 有上限）→ **回 Cut Map 改镜头设计**：并镜、换景别、换拍法、用剪辑规避，必要时删戏。这是正当的导演决策，不是失败。

## 11. 反模式：500 字 Prompt 综合症

身份不一致就加 Prompt、运镜不对就加 Prompt、动作/构图/空间错都继续加 Prompt，最终得到一段臃肿且互相干扰的长 Prompt。正确反应是先问"**这条控制该不该从 Prompt 里拿出去、交给哪个信号**"。Prompt 越长，常常说明控制策略越错——它在替本该由参考图、首尾帧、代理承担的确定性控制"用形容词祈祷"。

## 12. 开放架构原则（面向未来模型）

引擎稳定的是**流程与判断顺序**（意图→控制需求→信号→路线→精简 Prompt→评审），不稳定的是具体工具。出现新模型/新能力时：
- 新的控制输入 → 在第2节注册表加一行 Control Signal；
- 新的生成方式 → 在第8节 Router 加一行 Generation Method；
- 不需要改动 Control Map、Control Budget、Recipe、Review 的框架。
导演语义（意图、Control Map、Recipe）与平台/模型适配器严格分离，因此同一套导演决策可迁移到任意模型。

## 13. 与其他模块的关系

- **导演层的状态机/锁/Probe/Cut Map/粗剪在 `solo-workflow.md` 定义**；本文件接管并深化 S4（Contract 增出 Control Map）、S5（Router 增出信号选择与 Recipe）、S6（生成后增出 Accept/Repair/Reroute）。
- 连续性投入等级并入第5节 Control Budget（衔接敏感度是定级因子之一）。
- 平台资产、口型分级、台词内嵌、中文 UI 后期见 `ai-video-platform-production.md`；失败诊断码 D1–D10、熔断阈值、镜型→模型引擎路由、成本与台账见 `ai-production-measurement.md`。
