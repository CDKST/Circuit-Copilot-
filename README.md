# Circuit Copilot · 代码优先的电路设计工作台

**用一句中文说你要什么电路，或者直接用 Python 把电路写出来** —— 两条路都落到同一个确定性内核上：
真实 ngspice 仿真、可复核的达标判定、能去市场上买到的元件值。

纯 Python 标准库 + ngspice，**零第三方依赖**。**桌面为主**（电脑浏览器里打开就是 VS Code 风格的 IDE），手机（Termux）同样能一条命令跑起来。

---

## 一个仓库，三层东西

| 层 | 入口 | 你得到什么 | 代码位置 |
|---|---|---|---|
| ① **CLI 参数化工具** | `python3 main.py "二阶低通，截止1kHz，Q=0.707"` | 自然语言 → 拓扑选型 → E 系列邻域搜索 → 真仿真 → 容差成品率 → Markdown 报告 | `copilot/` + `main.py` |
| ② **CCL 代码库** | `from ccl import Circuit, GND` | 电路就是 Python 代码：器件 / 子电路块 → 网表 → 仿真 → `c.expect` 断言 → `c.lint()` 检查 → 报告 | `ccl/` |
| ③ **IDE** | 浏览器打开 `http://127.0.0.1:3081/ide` | VS Code 风格界面：代码编辑器 + **电路图**（带真数据驱动的**工作态与瞬态动画**、**开关可点重跑**）、波形/相位、问题、指标、工作点、网表、报告七个面板 + AI 生成/修复/解释 | `web/ide/` + `web/ccl_api.py` |

三层的依赖是**单向**的，互不拖累：

```
web/ide/*.js ──HTTP──► web/ccl_api.py ──► ccl/ ───────────┐
                                                          ├──► copilot/sim.py ──► ngspice -b
main.py ──► copilot/agent.py（拓扑库 + 搜索 + 容差 + 判定）─┘
```

两条链唯一的共享物是 `copilot/sim.py`（ngspice 封装）与 `copilot/llm.py`（模型客户端）。

- **CLI** 那层是「参数化拓扑库」：结构由人写死，元件值由搜索决定，判定由代码决定，语言交给模型。
- **CCL** 那层是「代码即电路」：没有图形编辑器，电路是一段可 diff、可断言、可参数化的 Python。
- **IDE** 只是 CCL 的壳：它把同一段代码送到 `POST /api/ccl/run`，拿回波形、问题列表、规格核对结果画出来。

`web/index.html`（根路径 `/`）仍是第一层的参数化工具界面；`/ccl` 会 302 到 `/ide`。

---

## 为什么需要它

做电源、滤波、运放电路的日常是这样的：翻书找公式，`f = 1/(2πRC)`，计算器按出 R=1591.5Ω →
翻 E24 表找最近的 1.6kΩ → 搭面包板 → 示波器上看波形 → 觉得「好像有点振」→ 把电容换成 220nF 再试一遍。

这里面其实是两件性质完全不同的事被混在一张桌上：

- **算**：给定 fc 求 RC，是确定的算术。但它算的是理想元件，也不管你能不能买到那个值。
- **试**：换值、看波形、凭感觉判断「行不行」，可靠但慢，而且**判断全凭经验**。

这个工具把这两步合成一条命令，并且补上教科书从不回答的第三个问题：
**标称值上完美达标的设计，量产时有多少台会掉出规格？**
（答案经常很难看：1% 电阻 + 5% 电容，一个「刚好达标」的设计成品率可能只有 90%；
设计点再往规格边界贴一点，就掉到 70% 以下。）

它不替你做架构决策，也不假装懂你的负载和布局。它负责把**能算清楚的部分算清楚**。

---

## 快速上手：三条命令

```bash
# 0) 唯一的外部依赖
pkg install -y ngspice        # Termux / Android；桌面用 apt install ngspice / brew install ngspice

# 1) 起服务（默认 3081，探活 /api/health，日志 out/server.log）
bash start.sh

# 2) 浏览器打开 IDE（主要界面）
#    http://127.0.0.1:3081/ide

# 3) 改完代码想确认没砸坏东西：一条命令跑全部验证（约 3~6 分钟）
bash selfcheck.sh            # 加 --fast 跳过最慢的真实模型基准
#    语法 → 库(188) → 端到端(133) → HTTP(42) → 电路图(51) → 拦截基准(闸门 98 + 真实模型 14)
#    失败会点名是哪个套件，日志在 out/selfcheck.log

# 4) 或者直接用命令行
python3 main.py "二阶低通，截止1kHz，Q=0.707"
```

其它 CLI 开关：

```bash
python3 main.py --check        # 自检：ngspice 版本 / 拓扑数 / 大模型是否配置
python3 main.py --catalog      # 列出所有支持的拓扑（参数与指标）
python3 main.py --json "…"     # 只输出结构化 JSON，方便脚本化调用
python3 main.py --key sk-xxxx  # 写大模型配置（也支持 --base / --model）
python3 main.py --serve 3081   # 等价于 bash start.sh 起的那套服务
```

第 3 条命令是真跑出来的（本机，大模型可用）：

```
**拓扑**：Sallen-Key 二阶有源低通（`sallen_key_lowpass`）
| 截止频率 −3dB | 1kHz  | 998Hz  | 900Hz ~ 1.1kHz  | ✅ |
| 品质因数 Q    | 0.707 | 0.7414 | 0.6009 ~ 0.813  | ✅ |
BOM：R1 11kΩ / R2 11.5kΩ / C1 22nF / C2 10nF / X1 运放 1 个
```

---

## 核心原理：AI 只做两件事，其余交给数值计算

这是第一层（参数化工具）最重要的一条设计边界。**LLM 的输出里没有一个数字进入网表。**

```
   用户：「做一个 1kHz 的 RC 低通滤波器，要求阻带衰减好一点」
                    │
                    ▼
        ┌──────────────────────────────┐
        │ ① LLM：自然语言 → 结构化需求   │  只输出 JSON：
        │    选拓扑 + 目标指标           │  {"topology":"rc_lowpass",
        └──────────────────────────────┘   "spec":{"fc":1000}}
                    │                      ↑ 没配 Key 时由关键词规则顶上
                    ▼
        ┌──────────────────────────────┐
        │ ② 确定性闸门（不看模型脸色）    │  物理可行性预判 + 语义对账：
        │    feasibility / verify        │  「一阶 RC 要 60dB 衰减」当场说做不到
        └──────────────────────────────┘  「12V 分压到 3.3V」不许变成 6V
                    │
                    ▼
        ┌──────────────────────────────┐
        │ ③ 拓扑模板：解析式给搜索初值    │  R = 1/(2π·fc·C)，先定 C 再算 R
        └──────────────────────────────┘  纯 Python 算术，不吸附标称值
                    │
                    ▼
        ┌──────────────────────────────┐
        │ ④ 数值搜索：E 系列邻域候选     │  每个参数在初值 ±1 倍频程内取标称值，
        └──────────────────────────────┘  组合后按离初值远近排序（budget=40）
                    │
                    │  每个候选 ──► ngspice 真跑一次 AC 分析（实测 50~165 ms）
                    ▼
        ┌──────────────────────────────┐
        │ ⑤ 判定：确定性代码             │  每个指标一条 Rule(mode, tol, kind)
        │    允许区间 [lo, hi]，✅/❌    │  → 报告里直接印出「≥ 90kHz」供复核
        └──────────────────────────────┘
                    │
                    ▼
        ┌──────────────────────────────┐
        │ ⑥ 蒙特卡洛：20 次容差扰动      │  R±1% · C±5% · L±10% · GBW±20% · A0±50%
        └──────────────────────────────┘  → 成品率 + 各指标 p5/均值/p95
                    │
                    ▼
        ┌──────────────────────────────┐
        │ ⑦ LLM：仿真结果 → 工程评审文字 │  只解释取舍与风险，不产生任何数字
        └──────────────────────────────┘
                    │
                    ▼
        网表 .cir ＋ BOM ＋ Markdown 报告（成品率、短板指标、下一步建议）
```

四件事各有归属，互不越界：

| 事情 | 谁做 | 为什么 |
|---|---|---|
| 理解人话、写评审 | LLM | 规则写死会很僵硬，模型成本极低 |
| 元件值 | E 系列邻域搜索 + 真实仿真 | 可枚举、可验证、误差可量化 |
| 指标计算 | Python 从波形算（不用 `.meas`） | 解析稳定、可单测 |
| 达标判定 | `copilot/judge.py` 的规则表 | 必须确定性、可复核 |

### 为什么不让 LLM 直接给元件值

不是保守，是它在三个地方会**静默地错**（错了还不报警）：

1. **给出买不到的值**。它写 `R1 1k`、`R2 2.2k`、`R3 3.2k`，语法合法、仿真能跑，
   但 3.2kΩ 在 E24 里不存在——用户拿着 BOM 买不到料。这个错误不会让任何一行代码报错。
2. **误差没有边界**。同样是算 `1/(2π·1000·100nF)`，它可能给 1.59k 也可能给 1.6k，
   两个答案在它看来一样合理。你没法在文档里承诺精度。
3. **算得比仿真慢**。一次 ngspice 仿真实测 **50~165 ms**（手机 Termux，随负载波动），一次 LLM 调用以秒计。
   用搜索比用猜**又快又准**——这不是信仰，是成本对比。

于是模型被限制在两个它真正擅长、且错了看得见的位置：把含糊的人话翻译成
`{topology, spec}`，以及把冷冰冰的指标解释成工程取舍。
万一它选错拓扑或给出物理上做不到的目标，后面的闸门会把结论变成「❌ 未达标」并说明原因——
**错误是可见的，而不是伪装成成功。**

### 这套闸门到底拦得住多少：实测数字

「我们会拒绝坏设计」是一句话，可以随便说。所以这里把它做成了**可复跑的基准**（`bench/`），
任何人都能自己敲一遍核对：

```bash
python3 bench/run_bench.py --run-gates   # 64 条对抗用例，1.5 秒，零 API Key
python3 bench/run_llm.py                 # 真实模型端到端 14 条，约 3 分钟
```

| 层 | 拦截率 | 误杀率 |
|---|---|---|
| **确定性闸门**（98 条对抗用例，10 类） | **59/60 = 98.1%** | **0/38 = 0%** |
| **真实模型端到端**（14 条完整需求） | **B 类 4/4 = 100%** | **A 类 1/7** |
| **合计** | **55/56 = 98.2%** | **1/42 = 2.4%** |

「拦截率」= 该拦的坏需求真的被拦下的比例；「误杀率」= 合法设计被误判为坏的比例。

> 那条 1/35 的误杀是真事，没有抹掉：模型给「无源 RLC 带通 100kHz」写了一段
> 自带的规格断言，把**带通的中心频率当成 −3dB 截止点**（那是低通的定义），
> 于是电路峰值明明落在 100kHz（实测达标），却被自己写错的断言否掉了。
> 这是已知的失效模式——模型自加的断言可能比用户要求更严甚至写错。
>
> **已修（原先记为待办）**。现象：一个实测达标的电路，因为模型自己在代码里写错的
> `c.expect(...)` 被整条拒收。修法：`ccl/ai/generate.py` 把「用户指标」和「模型自加断言」
> 分开判——用户指标不达标照旧进修复循环；用户指标全达标、只有自加断言没过，
> 就**照常交付**，并把可疑断言逐条写进 `Generation.warnings` 明示（报告里提示「这几条断言
> 不一定代表需求」）。基准侧同步记账（`bench/run_llm.py` 的 `warnings` / 自加断言告警计数），
> 不再把它算成误杀。复现：喂一个「电路达标、自加断言故意写错 5000Hz」的桩模型，
> 实测 `ok=True` 且 warnings 里点明「要求 5000Hz、实测 994.7Hz」。
**两个必须一起看**——把所有输入都拒绝的系统，拦截率是 100%，但没人能用。

真实模型层（`cn:deepseek-v4-flash`，14 条全跑完，194 秒）的具体表现：

| 现象 | 实测 |
|---|---|
| 幻觉拦截 | **8 次**（AI 文本里的数字与仿真实测不符，被指标对账抓出） |
| 硬凑（B 类假装达标） | **0 条** |
| 假成功（A 类报成功但不达标） | **0 条** |
| 编造且不自知（C 类模糊需求） | **0 条** |
| 一次通过 | 64%（9/14），平均修复 0.43 轮 |

几个真跑出来的案例：

- 「无源 RC 做 Q=20」物理不可能。模型硬给了一个电路，**实测 Q=4.2×10⁴（偏 21 万倍）**，
  被仿真实时揭穿 → 拦住。
- 「单级运放 100dB 增益 + 1MHz 带宽」违反增益带宽积。3 轮修复实测增益只有 49~60dB
  （偏差 4000~5000%），每轮都被抓，最后**如实报「没做到」而不是硬凑**。
- 「1kHz RC 低通」第 1 轮带宽偏 29.4%，被指标对账抓出 → **修复 1 轮后达标**。

> 这个基准开发过程中自己抓出了 3 个真 bug（`f3db` 在扫频起点不对时会**编造**一个假交点、
> `verify` 从不核对分压器的输入电压、从不比对 `<带宽>`），以及一处**修复建议方向反了**
> 的物理提示（Sallen-Key 的 Q 应靠 C1/C2 抬，原文却让人去调 R2/R1）。
> 基准的价值在于它真的会让坏东西变红，而不是证明我们做得好。

---

## 支持的电路（参数化工具这一层）

清单以命令输出为准（拓扑数量不写死在文档里）：

```bash
python3 main.py --catalog     # 当前 13 个
```

| 拓扑 | id | 参数 | 可判定指标 |
|---|---|---|---|
| RC 一阶低通滤波 | `rc_lowpass` | R, C | 增益 (dB), 截止频率 −3dB (Hz), 阻带衰减 (dB) |
| RC 一阶高通滤波 | `rc_highpass` | R, C | 增益 (dB), 截止频率 −3dB (Hz), 阻带衰减 (dB) |
| RLC 串联带通滤波 | `rlc_bandpass` | L, C, R | 中心频率 (Hz), 品质因数 Q, 带宽 (Hz), 增益 (dB) |
| 电阻分压网络 | `divider` | R1, R2, Vin | 输出电压 (V), 输出电流 (A), 功耗 (W), 输出阻抗 (Ω) |
| 运放同相放大 | `opamp_noninverting` | R1, R2, GBW, A0 | 电压增益 (倍), 增益 (dB), 截止频率 −3dB (Hz), 单位增益频率 (Hz), 相位裕度 (°), 增益峰化 (dB) |
| 运放反相放大 | `opamp_inverting` | R1, R2, GBW, A0 | 电压增益 (倍), 增益 (dB), 截止频率 −3dB (Hz), 单位增益频率 (Hz), 相位裕度 (°), 增益峰化 (dB) |
| Sallen-Key 二阶有源低通 | `sallen_key_lowpass` | R1, R2, C1, C2, GBW, A0 | 增益 (dB), 截止频率 −3dB (Hz), 中心频率 (Hz), 品质因数 Q, 增益峰化 (dB) |
| Sallen-Key 二阶有源高通 | `sallen_key_highpass` | R1, R2, C1, C2, GBW, A0 | 增益 (dB), 截止频率 −3dB (Hz), 中心频率 (Hz), 品质因数 Q, 增益峰化 (dB) |
| 差分放大 / 减法器 | `difference_amp` | R1, R2, R3, R4, GBW, A0, CMRR_OPA | 电压增益 (倍), 增益 (dB), 共模抑制比 (dB), 截止频率 −3dB (Hz), 相位裕度 (°) |
| 有源低通滤波器综合 | `filter_lowpass_synth` | order(2/4/6), kind(butterworth/chebyshev/bessel), ripple_db, 各级 R/C/Q, GBW, A0 | 增益 (dB), 截止频率 −3dB (Hz), 增益峰化 (dB), 阻带衰减 (dB) |
| 有源高通滤波器综合 | `filter_highpass_synth` | order(1/2/3 级), 各级 R/C, GBW, A0 | 增益 (dB), 截止频率 −3dB (Hz), 增益峰化 (dB), 阻带衰减 (dB) |
| 有源带通滤波器综合 | `filter_bandpass_synth` | order(2/4/6 → 每侧 1/2/3 级), 高通段 R/C, 低通段 R/C, GBW, A0 | 中心频率 (Hz), 带宽 (Hz), 品质因数 Q, 增益 (dB) |
| Twin-T 双 T 陷波器 | `twin_t_notch` | R1, R2, C1, C2, R3(=R/2), C3(=2C), GBW, A0 | 中心频率 (Hz), 凹陷深度 (dB), 品质因数 Q, 带宽 (Hz), 增益 (dB) |

运放类拓扑的 `GBW`/`A0` 是**元件选型参数**（不是器件型号），它们**不参与 E 系列搜索**——
由解析式按目标带宽反推（同相放大：`GBW ≈ fc·增益·1.3`，差分为 `fc·(1+Ad)·1.3`），`A0` 默认 1e5。
报告里给出的「运放 GBW = 1.3MHz」就是你挑器件时要满足的下限。

`filter_lowpass_synth` 是综合型拓扑：给它阶数与响应类型，它按极点表反解各级 Sallen-Key 的 f0 与 Q
（Butterworth 最平坦、Chebyshev 用通带波纹换陡度、Bessel 群延迟最平），再逐级算出元件值。
`order`/`kind`/`ripple_db` 是**设计选择**（`Topology.design_keys`）：进 `design()`，但不进判定表。

---

## 跑两个真实需求（真跑出来的输出）

### 例 1 · 最简单的一条路：RC 一阶低通

```bash
python3 main.py "做一个1kHz的RC低通滤波器"
```

输出（规则模式，本机端到端约 2.5 秒；时间几乎全花在 ngspice 子进程上，
并行跑测试时会长到 8 秒以上——报告末尾的「累计仿真耗时」是工具自己记的账）：

````text
# 电路设计报告

**需求**：做一个1kHz的RC低通滤波器

**结论**：✅ 全部达标

**拓扑**：RC 一阶低通滤波（`rc_lowpass`）

> 最简单的无源低通，−20dB/十倍频滚降，常用于电源去耦、信号抗混叠前置。

**选型理由**：按关键词规则匹配（未启用大模型）

## 设计规格与实测

| 指标 | 目标 | 实测 | 允许区间 | 结论 |
|---|---|---|---|---|
| 截止频率 −3dB | 1kHz | 995Hz | 900Hz ~ 1.1kHz | ✅ |

## 元件清单（BOM）

| 位号 | 类型 | 规格 | 数量 |
|---|---|---|---|
| R1 | 电阻 | 1.6kΩ | 1 |
| C1 | 电容 | 100nF | 1 |

## 设计参数

- 电阻 R：**1.6kΩ**
- 电容 C：**100nF**

## 容差分析

按 **电阻 ±1% · 电容 ±5% · 电感 ±10% · 运放 GBW ±20% · 开环增益 ±50%** 高斯扰动跑了 **20 次**：

- **成品率**：20/20 = **100%**

| 指标 | 目标 | p5 | 均值 | p95 | 5–95% 波动 |
|---|---|---|---|---|---|
| 截止频率 −3dB | 1kHz | 914Hz | 1kHz | 1.07kHz | ±8.0% |
| 增益 | — | -0.00dB | -0.00dB | -0.00dB | ±0.0% |
| 阻带衰减 | — | 20.04dB | 20.04dB | 20.04dB | ±0.0% |

**结论**：成品率 100%（20/20）≥95%，批次一致性良好，可直接打样。

## 仿真统计

- 参数组合尝试：3 次
- 累计仿真耗时：189 ms
- 单次平均：63.0 ms
- 需求解析来源：规则引擎

## 仿真网表

```spice
* RC 一阶低通
Vin in 0 AC 1
R1 in out 1600
C1 out 0 1e-07
.ac dec 200 0.0994718 9.94718e+06
.end
.control
set filetype=ascii
run
write c.raw all
.endc
.end

```
````

看点：

- **E 系列吸附**起了作用：理想值 1.59kΩ 被吸到 1.6kΩ，实测截止频率 995Hz 而不是 1000Hz。
  这 0.5% 的偏差就是「必须仿真验证」的由来——不是代码不准，是市场只卖标称值。
- 允许区间 `900Hz ~ 1.1kHz` 是判定规则算出来的（±10% 相对容差），不是事后解释。
- **扫频卡不是固定 1Hz~100MHz**：`.ac dec 200 0.0994718 9.94718e+06` 是以设计截止频率为中心
  展开 ±4 个十倍频（`copilot/topologies/` 包的 `ac_sweep`）。这样 1Hz 低通和 10MHz 低通用的是
  **同样的 200 点/十倍频密度**，`db[0]` 也永远落在通带里。
- 容差分析：成品率 100%，但 **p5 已经掉到 914Hz**。1% 电阻 + 5% 电容的组合，
  在这个设计点上是够的——因为设计点正好压在 1kHz 中心。

### 例 2 · 标称全绿，成品率只有 90%：10 倍同相放大器

```bash
python3 main.py "10倍同相放大器 需要100kHz带宽"
```

输出（截取关键段）：

````text
| 截止频率 −3dB | 100kHz | 128kHz | ≥ 90kHz | ✅ |
| 电压增益 | 10 倍 | 10.09 倍 | 9.5 倍 ~ 10.5 倍 | ✅ |

BOM：R1 976Ω / R2 8.87kΩ / X1 运放 1 个
容差分析：**成品率 18/20 = 90%**，主要短板：截止频率 −3dB（2/20 次落在规格外）

| 指标 | 目标 | p5 | 均值 | p95 | 5–95% 波动 |
|---|---|---|---|---|---|
| 截止频率 −3dB | 100kHz | 83.3kHz | 131kHz | 175kHz | ±46kHz |
| 电压增益 | 10 倍 | 9.923 倍 | 10.11 倍 | 10.38 倍 | ±2.3% |
| 相位裕度 | — | 90.3° | 90.3° | 90.3° | ±0.0% |
| 增益峰化 | — | 0.00dB | 0.00dB | 0.00dB | ±0.00dB |

## 仿真统计
- 参数组合尝试：9 次
- 累计仿真耗时：743 ms
- 单次平均：82.5 ms

```spice
* 运放同相放大
Vin src 0 AC 1
R1 inn 0 976
R2 out inn 8870
X1 src inn out OPA
Rload out 0 1Meg
.subckt OPA inp inn out
Rin inp inn 10Meg
E0 ox1 0 inp inn 100000
R0 ox1 ox2 1
C0 ox2 0 0.0122427
Rout ox2 out 50
.ends
.ac dec 200 12.8865 1.28865e+09
.end
```
````

看点（这是整个项目想让你看到的东西）：

- **标称值上一切都很宽松**：带宽 128kHz，规格是「不低于 90kHz」，余量看起来有 42%。
- **容差分析立刻推翻这个印象**：`p5 = 83.3kHz`，20 次扰动里有 2 次掉出规格，成品率 **90%**，
  短板正是带宽。原因是运放 GBW 的批次离散（按 ±20% 建模）+ 增益对带宽的杠杆
  （带宽 ≈ GBW/增益，增益 10 倍时 GBW 的抖动被 1:1 转成带宽抖动）。
- 网表里的 `Rload out 0 1Meg` 是**仿真专用负载**（给运放一个直流通路），
  `copilot/report.py::bom()` 用 `SIM_ONLY_REFS` 把它从物料清单剔掉——所以上面 BOM 只有 3 行。
- 注意 `相位裕度 90.3°`：这是单极点运放 + 纯电阻反馈的真实结果，稳定但保守；
  报告没有把它当成「性能好」来吹。

---

## CCL 代码库：把电路写成代码

`ccl/` 是独立的库（`import ccl` 就行）：它不依赖 `copilot/agent.py` 那套搜索逻辑，
也不依赖任何大模型，只复用 `copilot/sim.py` 作为 ngspice 后端。

```python
from ccl import Circuit, GND

c = Circuit("rc_lp")
c.V("in", GND, ac=1)              # 交流源，幅度 1V
c.R("in", "out", 1.6e3)
c.C("out", GND, 100e-9)
sol = c.ac(1, 1e5, 200)           # .ac dec 200 1 1e5
print(round(sol.f3db("out"), 1))  # 994.7
```

### 分析能力：AC / OP / TRAN / DC

`Circuit` 上一共四个分析入口，`dc` 有「一维 / 两维」两种用法：

| 分析 | 调用 | 网表卡 | 你拿到什么 |
|---|---|---|---|
| 交流扫频 | `c.ac(f1, f2, points)` | `.ac dec N f1 f2` | 复波形 → `f3db` / `peak_db` / `db` / `phase` / `value_at` |
| 直流工作点 | `c.op()` | `.op` | 每个节点电压（`Solution.kind == "op"`） |
| 瞬态 | `c.tran(t_step, t_stop)` | `.tran step stop` | 时域波形 + **5 个瞬态指标**（见下） |
| 直流扫描（一维） | `c.dc(src, start, stop, step)` | `.dc` | 传输特性（Vout–Vin）、二极管伏安曲线 |
| 直流扫描（两维） | `c.dc(..., second=(src2, s, e, st))` | `.dc` | 曲线族（BJT 输出特性 Ic–Vce 这类） |

被扫的必须是**独立源**（`V` / `I`）的位号；写错位号会报
`CCLConfigError` 并列出可用位号。降序扫描（`stop < start`、步长为负）也支持。

```python
from ccl import Circuit, GND

c = Circuit("divider")
c.V("in", GND, dc=0)              # 直流源，等下被扫
c.R("in", "out", 10e3)
c.R("out", GND, 10e3)
c.I("out", GND, dc=0)             # 第二维：从 out 抽走的电流

s1 = c.dc("V1", 0, 10, 1)                                 # 传输特性
s2 = c.dc("V1", 0, 5, 1, second=("I1", 0, 1e-3, 0.5e-3))  # 曲线族
print(len(s1.x), [round(v, 1) for v in s1.dc_curves("out")[0]["y"]])
print(len(s2.dc_curves("out")), [k["label"] for k in s2.dc_curves("out")])
```

实测（本机 ngspice-47）：

```
11 [0.0, 0.5, 1.0, 1.5, 2.0, 2.5, 3.0, 3.5, 4.0, 4.5, 5.0]     ← 一条直线，斜率 0.5
3 ['I1=0A', 'I1=0.5mA', 'I1=1mA']                              ← 3 条曲线 × 每条 6 点
```

曲线族的物理含义：每加 0.5mA 抽流，整条 `out` 下移 2.5V——正好是 `I1 × 5kΩ`
（两个 10kΩ 并联的等效源内阻）。三段 y 分别是
`[0, .5, 1, 1.5, 2, 2.5]` / `[-2.5, -2, -1.5, -1, -.5, 0]` / `[-5, -4.5, -4, -3.5, -3, -2.5]`。

瞬态这一侧：

```python
from ccl import Circuit, GND

c = Circuit("rc_step")
c.PULSE("in", GND, 0, 5, 0, 1e-9, 1e-9, 1, 2)   # 单次阶跃（pw ≥ t_stop）
c.R("in", "out", 1e3)
c.C("out", GND, 100e-9)                          # τ = 1kΩ × 100nF = 100µs
s = c.tran(1e-6, 5e-3)
print(s.rise_time("out"))       # 0.00021972…  → 219.7µs
```

**两维扫描必须用 `dc_curves(node)` 取数据，别按点号直接连线**：
ngspice 的 `.dc` 两维输出是「外层固定、内层扫完」的分段结构（18 点 = 3 段 × 6 点），
按点号连会画出折返锯齿。`dc_curves()` 负责切段并附上 `"I1=0.5mA"` 这样的 label。

### 瞬态指标：五个数，测不到就是 `None`

`c.tran()` 拿到的波形可以继续问五个指标（全部在 Python 侧算，不用 `.meas`）：

| 指标 | 口径 | 实测（RC 阶跃，τ=100µs） |
|---|---|---|
| `rise_time(node)` | 10% → 90%，**第一次**穿越，相邻样本线性插值 | `219.7µs`（= 2.197τ，一阶系统的理论值） |
| `fall_time(node)` | 90% → 10%，只有下降沿才有 | `None`（单次上升阶跃里没有下降沿） |
| `overshoot_pct(node)` | `(峰值 − 稳态)/\|稳态\| × 100`，无超调给 `0.0` | `0.0`（浮点残差 1.8e-14，前端显示 `0.00%`） |
| `settling_time(node)` | 最后一次离开 ±2%·稳态带且**再没回来**的时刻 | `392.1µs`（2% 带 ≈ 4τ） |
| `steady_state(node)` | 最后 10% 时间窗的**均值**（不是末点） | `5.0000 V` |

二阶欠阻尼电路的 `overshoot_pct` 与 `exp(−πζ/√(1−ζ²))·100`（ζ = 1/(2Q)）对得上，
四组 Q 交叉验证（RLC 串联，L=10mH / C=1µF）：

| Q | 实测（`t_stop=20ms`） | 理论 |
|---|---|---|
| 0.707 | 4.33% | 4.33% |
| 1 | 16.30% | 16.30% |
| 2 | 44.43% | 44.43% |
| 5 | 72.93% | 72.92% |

⚠️ **Q 高的时候要跑得够久**：超调的分母是 `steady_state`（末 10% 窗均值），
Q=5 的振铃很长，窗口落在振铃里就会同时低估稳态与超调——同一个电路 `t_stop=5ms` 给
`72.56%`（稳态 5.0106V，还在振铃），`t_stop=20ms` 才是 `72.93%`（稳态 5.0000V）。
指标对不上时，**先查测量口径，再怀疑模型**。

**测不到就返回 `None`，绝不编数**：纯正弦、恒值、还没走到 90% 电平就结束的波形，
五个指标会逐项给 `None`，前端指标卡片直接跳过这一行（不显示一串「—」）。

### 五层结构

| 层 | 文件 | 职责 |
|---|---|---|
| 数据模型 | `ccl/core.py` | `Pin` / `Component` / `Circuit`；节点是字符串，地固定为 `"0"` |
| 编译 | `ccl/compile.py` + `ccl/format.py` | `Circuit` → SPICE 网表；数值怎么写成不歧义的形式 |
| 仿真 | `ccl/runner.py` + `ccl/solution.py` | 生成分析卡（`.ac`/`.op`/`.tran`/`.dc`）→ 调 `copilot.sim` → `Solution`（`v/db/phase/f3db/peak_db/value_at` + `dc_curves` + 5 个瞬态指标） |
| 检查 | `ccl/lint.py` + `ccl/assertions.py` | 静态+工作点检查；`c.expect` 规格断言 |
| 复现 | `ccl/blocks.py` + `ccl/report.py` | 可复用子电路块；Markdown 报告导出 |
| AI（可选） | `ccl/ai/` | `generate` / `fix` / `review` / `explain`；核心库不 import 它们 |

### 器件清单（引脚顺序 = SPICE 端子顺序）

下表直接对着 `ccl/devices.py` 的 docstring 抄——**引脚顺序错了仿真照样跑通，只是结果全错**：

| 器件 | 构造签名 | 引脚顺序 | 网表形态 |
|---|---|---|---|
| 电阻 | `Resistor(a, b, value)` | `a` → `b` | `R1 in out 1600` |
| 电容 | `Capacitor(a, b, value)` | `a` → `b` | `C1 out 0 1e-07` |
| 电感 | `Inductor(a, b, value)` | `a` → `b` | `L1 in out 0.001` |
| 电压源 | `VoltageSource(a, b, dc=0.0, ac=0.0)` | `a`(＋) → `b`(−) | `V1 in 0 DC 0 AC 1` |
| 电流源 | `CurrentSource(a, b, dc=0.0, ac=0.0)` | `a` → `b` | `I1 out 0 DC 0.001 AC 0` |
| 运放 | `OpAmp(inp, inn, out, *, gbw=1e6, a0=1e5)` | `inp`(＋) → `inn`(−) → `out` | `X1 src inn out OPA_100000_1p3e06` + 一段 `.subckt` |
| 二极管 | `Diode(a, k, model=None, *, model_params=None, area=None)` | `a`阳极 → `k`阴极 | `D1 a 0 D1N4148` + `.model` |
| 三极管 | `Bjt(c, b, e, model=None, *, model_params=None)` | `c` → `b` → `e` | `Q1 c b 0 Q2N3904` + `.model` |
| MOSFET | `Mosfet(d, g, s, bulk=None, model=None, ...)` | `d` → `g` → `s` → `b`（省略则接源极） | `M1 d g 0 0 M2N7002` + `.model` |
| 脉冲源 | `PulseSource(a, b, v1, v2, td, tr, tf, pw, per)` | `a`(＋) → `b`(−) | `V1 in 0 PULSE(0,5,0,1e-09,1e-09,0.001,0.002)` |
| 正弦源 | `SinSource(a, b, vo, va, freq, td, theta)` | `a`(＋) → `b`(−) | `V1 in 0 SIN(0,1,1000,0,0)` |
| **开关** | `Switch(a, b, ctrl, *, on_voltage=2.5, ron=0.01, roff=1e9)` | `a` → `b` → `ctrl` → `ref`（**固定接地 `0`**，所以是 **4 个引脚**） | `SW1 a b ctl 0 SW_0p01_1e09_2p5` + `.model SW_0p01_1e09_2p5 SW(Ron=0.01 Roff=1e9 Vt=2.5)` |

`Circuit` 上的便捷方法一一对应：`c.R / c.C / c.L / c.V / c.I / c.OpAmp / c.D / c.Q / c.M / c.PULSE / c.SIN / c.SW`。

**开关（`c.SW`）**是 M6 加的，三个必须知道的事实（实测数据见下一节与 `ccl/CONTRACT.md` 的 M6 节）：

- 第 3 个参数是**控制节点**（`V(ctrl) > Vt` 导通），第 4 端固定接地；`Vt` 总是显式写进 `.model`
  —— 实测 ngspice 省略 `Vt` 时默认 0，会让「控制端悬空/0V」的开关处于**导通**状态。
- **模型名由 `(ron, roff, Vt)` 派生**（默认 `SW_0p01_1e09_2p5`），不是固定名：实测同一个 model 名声明两次且参数不同时，
  ngspice **不报错、不警告、静默只认后一份**，共用固定名会让一个开关的 `ron` 被另一个偷偷改掉。
  **`Vt` 也必须在名字里**：只有 `ron`/`roff` 时会撞名，实测 `Vt=2.5` 与 `Vt=4.0` 两个开关撞在一起后，
  控制 3V 时**两个都导通**（本该只有 2.5V 那个导通）。
- **断开不是理想开路**：`roff=1e9Ω` 下实测漏电 4.951 nA（5V 回路）。这是刻意的物理诚实——真实机械/半导体开关也漏。
  要更小就自己把 `roff` 调到 1e12 以上。

几条实测过的约定：

- **两端件** `(a, b, value)`；**三端及以上**按引脚顺序做位置参数，且同名关键字可用：
  `c.OpAmp("src","sum","out", gbw=1.3e6)` 与 `c.OpAmp(inp="src", inn="sum", out="out")` 等价。
  `gbw` / `a0` 是关键字-only。
- **PULSE / SIN 用逗号分隔参数**，不用空格。`compile` 层有「值 token 内部不得含空白」的保护
  （防止真把网表行拆错列），空格写法会直接撞上它；ngspice-47 对两种写法都接受，所以统一输出逗号形式。
- **内置真实器件模型卡**（`ccl/devices.py::MODEL_CARDS`）：
  `D1N4148` `D1N4007` `D1N5819` `Q2N3904` `Q2N3906` `M2N7002` `MIRF540`。
  模型名不能以数字开头（SPICE 限制），所以 1N4148 写成 `D1N4148`。
  MOSFET 用 Level-1（Shichman-Hodges）：收敛优先，教学/工程估算够用。
- **自定义模型可用**：`c.D("a", GND, "MYDIODE", model_params={"Is": "1e-14", "N": "1.8"})`
  会生成实例行 `D1 a 0 MYDIODE` + `.model MYDIODE D(Is=1e-14 N=1.8)`，实测可跑通（D/Q/M 均验证）。
  注意模型参数只进 `.model` 卡，**不会**被追加到实例行（SPICE 不认实例级参数）。
  不想自己写参数就直接用内置 7 张模型卡。

### `c.expect`：把规格写进代码

```python
from ccl import Circuit, GND

c = Circuit("rc_lp")
c.V("in", GND, ac=1)
c.R("in", "out", 1.6e3)
c.C("out", GND, 100e-9)
c.expect("fc", "out", 1e3, tol=0.05)      # 截止 1kHz ±5%（相对容差）
c.expect("gain", "out", 1.0, tol=0.01)    # 通带增益 1 倍 ±1%

rep = c.verify()      # → Report，逐项「要求 vs 实测」
print(rep)
c.assert_ok()         # 不通过就抛 CCLAssertionError
```

实测输出：

```
全部通过 ✅
  ✅ −3dB 截止频率(out)               要求 = 1000Hz  实测 994.7Hz
  ✅ 电压增益(out)                    要求 = 1  实测 1
```

改一个电阻，几十条规格自动重跑。`kind` 可用：
`fc` `gain` `gain_db` `peak_db` `q` `bw` `v` `i` `p` `value_at`；比较方式 `eq`（默认）/ `min` / `max`。

```python
c.expect("peak_db", "out", 0.5, mode="max")   # 峰化不超过 0.5dB（dB 类用绝对容差）
c.expect("v", "b", 0.7, tol=0.1)              # 工作点：节点 b 电压 0.7V ±10%
c.expect("p", "R1", 0.2, mode="max")          # R1 功耗 ≤ 0.2W
```

**容差语义是踩过坑才定下来的**：频率/增益/电压/功耗用**相对**容差（`tol=0.05` = ±5%），
`gain_db`/`peak_db` 用**绝对**容差（`tol=0.6` = ±0.6dB）。混用会让「1000Hz ±10%」变成「≥999.9Hz」，
一个合格的 990Hz 设计被判失败。`verify()` 会自动判断该跑 AC 还是 OP（两类都有就都跑）；
**测不到的指标判 `NaN` → 未达标，绝不编数据**。

### 参数扫描：`sweep` / `with_`

```python
for r, variant in c.sweep("C1", [100e-9, 47e-9, 22e-9]):
    print(f"{r*1e9:>5.0f}nF -> fc = {variant.ac(1, 1e5).f3db('out'):.1f} Hz")
```

实测：

```
 100nF -> fc = 994.7 Hz
  47nF -> fc = 2116.4 Hz
  22nF -> fc = 4521.4 Hz
```

- `c.with_(R1=2e3)` 返回**副本**，不动原电路（键是位号，必须精确：`R1` 不是 `R`）。
- `sweep` 是生成器，可叠加固定参数：`c.sweep("R1", [...], C1=47e-9)`。

### 子电路库：12 个可复用块

```python
from ccl import Circuit, GND
from ccl.blocks import BLOCKS

c = Circuit("two_stage")
c.V("src", GND, ac=1)
c.use_ports(BLOCKS["rc_lowpass"](fc=1e3),  {"in": "src", "out": "n1"})
c.use_ports(BLOCKS["noninv_amp"](gain=10), {"in": "n1",  "out": "y"})

c.expect("fc", "y", 1e3, tol=0.05)
c.verify()
```

实测：`fc(y) = 999.9Hz`、`gain(y) = 9.999`，6 个器件内联成
`['V1', 'U1.R1', 'U1.C1', 'U2.R1', 'U2.R2', 'U2.X1']`，内部节点自动加前缀（`U2.sum`）。

| 块 | 参数 | 端口 |
|---|---|---|
| `rc_lowpass(fc, *, r=None, c=None)` | 截止频率 | in, out |
| `rc_highpass(fc, *, r=None, c=None)` | 截止频率 | in, out |
| `divider(ratio, *, total_r=10e3)` | 分压比 ∈(0,1) | in, out |
| `rlc_bandpass(f0, q=10)` | 中心频率 / Q | in, out |
| `noninv_amp(gain, *, r1=1e3, bw=None, gbw=None)` | 增益 ≥1 | in, out |
| `inv_amp(gain, *, r1=1e3, bw=None, gbw=None)` | \|增益\| | in, out |
| `sallen_key_lowpass(fc, q=0.7071, *, gbw=None)` | 截止 / Q | in, out |
| `bjt_ce_bias(rc, rb, *, model="Q2N3904")` | 集电极/基极电阻 | vcc, in, out |
| `emitter_follower(rb, re, *, model="Q2N3904")` | 基极/发射极电阻 | vcc, in, out |
| `sallen_key_highpass(fc, q=0.7071, *, gbw=None)` | 截止 / Q（Q 可到 5+，GBW 按 Q 预算） | in, out |
| `difference_amp(gain, *, r_in=10e3, gbw=None)` | 增益 | inp, inn, out |
| `twin_t_notch(f0=50, *, c=None)` | 陷波频率 | in, out |

两条硬约束：

1. **块里不放独立源**。内联（`use`）后与外部共节点，两个电压源接同一对节点就是 singular matrix。
   供电和激励留在顶层，块只暴露端口。
2. **块必须声明 `ports`**。组合时用 `use_ports(sub, {"端口名": "外部节点"})`，不用记内部节点叫什么。
   端口写错会报错并列出可用端口：
   `CCLConfigError: 电路 'rc_lowpass' 没有这些端口：['nope']；可用：['in', 'out']`

端口还有一个必须的副作用：`validate()` 与 `lint()` 会**自动放行已声明的端口**，
不把它们报成「悬空节点 / 无直流通路」。所以 12 个块单独检查**没有 error**，
只有一条「没有独立源」的 warn——因为块里刻意不放源。

### `c.lint()`：不跑模型、不跑仿真也能抓掉一半问题

```python
c2 = Circuit("bad2")
c2.V("in", GND, dc=5, ac=1)
c2.R("in", "a", 10)
c2.D("a", GND)
sol = c2.op()
for i in c2.lint(sol):
    print(i)
```

实测输出：

```
[错误] [R1] R1 功耗 1.54W 严重超限：R=10Ω 上压降 3.92V，功耗接近或超过 1W，普通贴片电阻会烧。请增大阻值或换用功率电阻。
```

- **静态检查**（不跑仿真）：空电路 / 无地 / 悬空节点 / 节点无直流通路 / 电压源并联或自短路 /
  参数越界 / 无 AC 激励 / 孤立子网络。
- **工作点检查**（需要 `solution`）：节点电压发散（|V|>1kV）/ 电阻功耗超限（≥0.25W warn、≥1W error）/
  三极管放大·饱和·截止 / MOSFET 截止·线性·饱和。

`Issue` 带 `code` 字段（如 `resistor_power`、`bjt_saturated`、`mos_off`），可程序化判断。

### AI：`generate` / `fix` / `explain`

```python
from ccl.ai import generate, fix, explain

gen = generate("二阶低通，截止 10kHz，Q=0.707")
gen.ok        # 成功了吗
gen.source    # 生成的**可执行 Python 代码**（不是网表）
gen.circuit   # 在沙箱里 exec 出来的 Circuit
gen.solution  # 真跑过一次仿真的结果
gen.attempts  # 每一轮的代码与错误，事后能判断是「模型不行」还是 prompt 不行

fixed = fix(your_code)            # 按实际检查结果修正现成代码
text  = explain(circuit, sol)     # 中文解释这段电路在做什么、实测如何
```

三条硬性要求（写在 `ccl/ai/generate.py`、`ccl/ai/sandbox.py` 的 docstring 里）：

1. **生成代码而不是网表**。网表错一处整个崩，代码错能定位到行号。
2. **生成后必须真跑**：沙箱 exec → 拿到 `Circuit` → 真跑一次仿真 → 不通过就把
   具体错误（行号 + 异常 + 正确签名对照表）回喂模型重试（`max_repair` 次）。
3. **exec 必须隔离**：AST 白名单 + 受限 `__builtins__` + 超时。导入白名单只有
   `ccl`、`ccl.blocks`、`math`；`open` / `eval` / `exec` / `__import__` 一律禁止。
   **刻意不放 `ccl.ai`**——否则生成出来的代码可以反过来调用模型。

**`fix()` 的关键设计：没问题就不烧 token。** 它先做确定性诊断（`ccl.lint` 静态+工作点 + `c.expect` 核对），
干净就直接返回原代码，一次模型都不调用。实测：合格代码 → `0 轮 / 235 ms`（沙箱 exec + 一次仿真），
期间模型的 `chat()` 从未被调用。有问题时才把**具体问题**回喂；模型不可用时也会把诊断结论留在 `attempts` 里：

```
ok= False | error= 未配置 API Key（无法自动修复；确定性诊断结果见 attempts）
attempts= 1
 - - 规格未通过：−3dB 截止频率(out)               要求 = 5000Hz  实测 994.7Hz
```

（上面这段是把模型换成一个「不可用」的桩之后跑的——证明诊断是真确定性的，不依赖网络。）

**没有 API Key 时**：`import ccl.ai` 永远成功，`generate`/`fix` 返回 `ok=False` 并给出明确原因，
`explain` 退回确定性模板；手写 CCL 代码完全不受影响。

### 报告导出

```python
from ccl import report as RPT
md = RPT.markdown(c, sol, analysis_desc=".ac dec 200 1 1e5")
```

6 个章节：**概况 / 器件清单 / SPICE 网表 / 实测结果 / 电路检查 / 规格核对**。
最后那节「规格核对」只在你声明了 `c.expect(...)` 时出现（没有 `expect` 时报告是 5 节，实测确认）。
报告里贴的网表就是仿真器实际吃进去的那份（可以另存成 `.cir` 复跑），
器件值用人类单位（`1.6kΩ` / `100nF`），规格核对表逐行给「要求 / 实测 / ✅❌」。

---

## IDE：VS Code 风格界面

```
┌──── 标题栏 36px：CCL · untitled.py ● ─────────────────────────────────────────────────── ▶ 运行 ────┐
├────┬──────────────┬─────────────────────────────────────────────────────────────────────────────────┤
│ 活 │ 侧边栏 220px │ 工具条：[交流][工作点][瞬态][直流扫描] 扫 V1  起 0  止 10  步 0.5               │
│ 动 │              │ 编辑区（行号 + 语法高亮 + 诊断标红）                                            │
│ 栏 │  📄 示例      │  from ccl import Circuit, GND                                                   │
│ 44 │  🧩 子电路块   │  def build():                                                                   │
│ px │  🔧 器件速查   │      c = Circuit("rc_lowpass")                                                  │
│    │  ✦ AI         │      c.V("in", GND, ac=1)                                                       │
│    │  ⚠ 问题       │      c.R("in", "out", 1.6e3)                                                    │
│    │  ⤓ 导出       │      c.expect("fc", "out", 1000, tol=0.05)                                      │
│    │  ⚙ 设置       │                                                                                 │
├────┴──────────────┴════════════════ 可拖动分隔条 6px ═══════════════════════════════════════════════┤
│ 结果面板： [电路图] [波形] [问题 2] [指标] [工作点] [网表] [报告]                     ⌄ 收起         │
│   电路图：由代码自动生成的只读 SVG（点图全屏、侧栏「导出」下载 .svg）                                │
│   波形：SVG Bode 图（dB 主图 + 相位副图、对数 x 轴、十字线读数、图例可点开关、导出 PNG）             │
├─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 状态栏 22px： 就绪 | DC | 6 器件 | 行 3, 列 5 ................  规格 2/2 ✓ | 79 ms | ngspice 47       │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

- **桌面优先**：主场景是在**电脑浏览器**里打开（宽视口、多面板、菜单栏、文件工作区）。
  手机/平板仍然可用——它真的跑在 Termux + ngspice 上，横屏加触屏能完整操作——
  但那是**便携场景**，不是主场景。竖屏窄屏会提示「建议横屏」。
  侧边栏和结果面板都能收起，收起后编辑区占满；窄屏（≤720px）侧边栏变成浮层。
- **顶部只有一个「▶ 运行」**。AI、导出、设置全部搬进侧栏活动栏——
  标题栏不再堆一排按钮，省下来的横向空间给文件名和运行键。
- **七个面板**：电路图 / 波形 / 问题 / 指标 / 工作点 / 网表 / 报告。问题面板按 error/warn 分色，
  点某条会跳回编辑器对应行（后端给了 `line` 时）。运行成功后**默认自动切到「电路图」**
  （先确认「我搭了个什么电路」），有 error 时切「问题」，工作点分析切「工作点」。
- **侧栏可点**：示例代码、12 个子电路块、13 条器件签名，点一下就把可直接跑的代码插到光标处
  （`snippets.js` 会顺手补上 `from ccl.blocks import BLOCKS`）。

### 侧栏分类

活动栏 8 个入口，一一对应（`web/ide/index.html` 的 `data-act` + `main.js` 的 `renderSidebar()`）：

| 活动栏 | 面板 | 里面是什么 | 点一下发生什么 |
|---|---|---|---|
| 📄 | 示例 | 11 条示例代码（`/api/ccl/example`，全部真实可跑） | 新标签页打开，分析参数一起带过来 |
| 📚 | 代码速查 | 子电路库（12 个块，`/api/ccl/blocks`）+ 器件速查（13 条器件 / API 签名）**合并在一栏** | 点某一条就把可跑片段插到光标处，自动补 import |
| ◆ | 参数化设计 | 一句话输入 +「开始设计」，结果只给**文字摘要** | 调第一层（参数化工具）；Bode 图 / 容差分布 / 报告导出要给「↗ 在完整界面打开」的按钮交给 `/` |
| ✦ | AI 助手 | 一句话输入框 + 生成 / 修复 / 解释三个按钮 | 调 `/api/ccl/ai`，结果写回编辑器 |
| ⤓ | 导出 | 「⤓ 电路图 SVG」/「⤓ Markdown 报告」 | 直接下载文件 |
| 🛡 | 拦截基准 | 闸门层**真跑**全部对抗用例（约 2 秒，不需要网络）+ 真实模型层最近一次存档 | 打开面板即自动跑一次，数字打在面板里 |
| 🔌 | 插件 | 已加载的扩展列表 | 查看 / 管理插件 |
| ⚙ | 设置 | 编辑器字号、布局重置、引擎版本，以及拦截基准 / 插件 / 参数化设计三个跳转按钮 | **不放分析类型与参数**——那些在编辑区顶部工具条（同一份状态只需一处入口） |

> 「问题」不再单独占一个活动栏入口：它是结果面板的页签，命令面板（`Ctrl/Cmd+Shift+P`）里的
> 「查看：问题列表」也能跳过去。

分析类型一共四种：交流 / 工作点 / 瞬态 / **直流扫描**，四种都在编辑区顶部工具条上；
选中「直流扫描」时工具条右侧多出 `扫 <位号> / 起 / 止 / 步` 四个参数框
（位号是字符串，其余按数字解析）。两维曲线族目前只有 Python API
（`c.dc(..., second=(...))`）和 `POST /api/ccl/run` 的 `second_source` 系列字段能表达——
`main.js` 里为它写过一组「第二维」输入框（`renderKindParams()` 写进 `#kParams`），
但 `index.html` 里已经没有 `#kParams` 这个容器，那段 UI 进不了屏，实际可用的只有第一维；
要画曲线族请直接用代码。

### 电路图：代码是真相，图是投影

**定位：只读投影。** 这张图不是第二份真相，也不是可以拖拽的编辑器——
它由 `Circuit` 对象在每次运行后重新生成（`web/ide/schematic.js`，纯字符串拼 SVG）。
所以不存在「图和代码不一致」这种状态：改代码，图跟着变。

怎么触发（全在 IDE 里，不需要配置）：

1. **运行**：`Ctrl / Cmd + Enter`（或点「▶ 运行」）。成功后结果面板自动切到 **「电路图」** 页签，
   也可以随时手动点它。
2. **全屏**：**点一下图** → 按 viewBox 放大 1.4 倍铺满屏幕，点任意处或 `Esc` 关闭
   （命令面板里也有「电路图：全屏查看」）。
3. **导出 SVG**：侧栏 `⤓` →「⤓ 电路图 SVG」，下载 `schematic-<标题>.svg`。
   样式全部内联成 SVG 属性，所以导出的文件单独打开也是对的，能直接贴进文档。

图上有什么：

- 10 种符号（R / C / L / V / I / D / Q / M / X / **S 开关**）；不认识的 `kind` 降级成方框但位号照显示
- 全正交走线（只有水平与垂直）；每个网络一条垂直母线，**网络 ≥3 个引脚时打实心圆点**
  ——没有圆点的十字交叉就是「不连接」，这是电路图的通用约定
- 深色 / 浅色两套配色，`width="100%"` 自适应缩放（复杂电路字会变小，所以才有全屏）
- **代码跑不通时电路图会被清空**，理由很直接：不能让人对着一张跟当前代码无关的图做判断

图能画成什么样、边界在哪（为什么带反馈的级联/差分对会有长线跨越）写在
`docs/ARCHITECTURE.md` 的「电路图渲染」一节。

### 电路图的「工作态」与瞬态动画（真仿真数据驱动）

这一层只做一件事：**把仿真算出的事实画出来**。不做装饰动画，也不靠「看起来合理」的假数据。

- **开关**：电路图里的开关**可以点**（`kind === "S"`，符号带透明命中区，手机上也点得准）。
  点一下 → 带着 `switch_states` **重跑一次仿真**（改的是控制电压源或该开关的 `.model Vt`，
  不动你的代码文本），灯亮/灭、电流大小全部以重新仿真的结果为准。
  这是**真切换，不是前端假动画**；工作点/直流分析下同样能点（点击不绑在播放条上）。
- **工作态（静态）**：工作点 / 直流扫描跑完，LED 按**实测电流**发光、电阻按**实测功耗**发热、
  其它器件按电流显示导通强度；图例写「工作态（真仿真数据）· 发光/LED · 发热/电阻 · 有电流 · N/M 个器件导通」
  （N = 电流 > 1µA 的器件数）。
  数据来自 `/api/ccl/run` 的 `live` 字段（每个器件的电流 / 电压 / 功耗）。
- **瞬态动画（逐帧）**：只有**瞬态分析**才有播放条（▶/⏸、可拖进度、**0.25~4×** 速度），
  标签固定写 `数据：ngspice 瞬态仿真 · N 点 / X 时长`。播放时电流沿导线流动、LED 亮度随电流变化、
  电阻颜色随功耗变化，每一帧都是 `timeline` 里**某一个真实仿真时刻**的采样点。
- **哪些分析没有工作态**：**交流分析（AC）没有** —— 频域没有「瞬时工作点」也没有时域序列，
  所以没有光晕、没有播放条，界面上不演；`live` 与 `timeline` 都是 `null`。**OP / DC 有工作态但没有动画**
  （没有时间轴），只有静态快照。仿真时长常在微秒级，播放是把一段真实波形压到约 6 秒播完
  ——**这 6 秒不是物理时间的倍率**，真实时刻看播放条上那个时间读数。

### 快捷键

| 快捷键 | 作用 |
|---|---|
| `Ctrl / Cmd + Enter` | 运行当前代码（按顶部工具条选的分析：交流 / 工作点 / 瞬态 / 直流扫描） |
| `Ctrl / Cmd + Shift + P`（或 `Ctrl / Cmd + P`） | 命令面板：运行、导出报告、AI 生成/修复/解释、**电路图全屏**、插入块/器件、切分析类型、切面板 |
| 点一下电路图 / `Esc` | 电路图全屏 / 关闭全屏 |
| `Ctrl / Cmd + /` | 注释 / 取消注释当前行或选中的多行 |
| `Tab` / `Shift+Tab` | 缩进 / 反缩进（多行选中时整体移动） |
| `Ctrl / Cmd + Z`、`Shift+Z`、`Ctrl+Y` | 撤销 / 重做（编辑器自维护快照栈） |
| `Ctrl / Cmd + S` | 清除「未保存」标记（不写磁盘，纯浏览器内存） |

### IDE 背后的接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/ide` | IDE 页面（`/ccl` 会 302 过来） |
| GET | `/api/ccl/example` | 示例代码清单（11 条，全部真实可跑） |
| GET | `/api/ccl/blocks` | 12 个子电路块的签名与说明 |
| POST | `/api/ccl/run` | `{code, analysis, require?, switch_states?}` → `{ok, circuit, analysis, lint, expect, netlist, error, line, stage}` + **`live`** / **`timeline`** / `switch_states` / `switch_apply`（见下） |
| POST | `/api/ccl/ai` | `{action:"generate"\|"fix"\|"explain", prompt?, code?}` |
| POST | `/api/ccl/report` | 生成 Markdown → `{ok, markdown, id, filename}` |
| GET | `/api/ccl/report/<id>` | 附件下载（Android WebView 里 `Blob` 下载不落文件，所以走同源 URL） |
| GET | `/api/health` | ngspice 版本 / 拓扑数 / 大模型是否配置 |

`analysis` 形如 `{kind:"ac", f_start, f_stop, points}` / `{kind:"op"}` /
`{kind:"tran", t_step, t_stop}` / `{kind:"dc", source, dc_start, dc_stop, dc_step}`。
直流扫描还认第二维（`second_source` / `second_start` / `second_stop` / `second_step`）：
返回的 `traces` 会按曲线族拆成 `"out @ I1=0.5mA"` 这样的多个 key，`sweepVar` / `secondVar`
一并回传（前端十字线读数靠它显示 `V1=3.2V`）。瞬态分析另外回 `metrics`
（`rise_time` / `fall_time` / `overshoot_pct` / `settling_time` / `steady_state`，测不到的是 `null`）。
浏览器里贴的代码和 AI 生成的代码走**同一套沙箱**（AST 白名单 + 超时），
所以「在页面里写 `import os`」会被拒。

`analysis` 里还有一个 **`reused`** 字段：`true` 表示这次**没有**再跑一遍 ngspice，直接用了
你代码里刚跑过的那次结果（判据是「分析类型 + 参数**数值**逐值相等 + 电路状态指纹一致」，
认不出的参数一律不复用）。`elapsed_ms` 仍是**那次真仿真**的耗时，不会因为省了一次就变 0。
真的少算一次的时候界面上的耗时不变，但整体等待会短一截。

四个「工作态」字段（口径的权威在 `ccl/CONTRACT.md` 的 M6 一节）：

| 字段 | 什么时候有 | 里面是什么 |
|---|---|---|
| `live` | op / tran / dc；**ac 一律 `null`** | `{kind, t, items:{位号:{kind, current, voltage, power}}}`。**`tran`/`dc` 取末点、`op` 就是那一个点**，`t` 字段说明是哪个时刻，**电流与电压严格同刻**（早先取 t=0 的行为已修）。`voltage` 是**绝对值**（丢了极性），而且**只有正好两个引脚的器件才有 `voltage`/`power`**——三端 `Q`/`M` 与四端开关 `S` 恒为 `null`，`current` 仍是真值。取不到的量是 `null`，不估算（电压源在 `live` 里没有 `current`，因为 `branch_currents()` 只解析 `i(@ref[...])`） |
| `timeline` | **只有 tran**，其它分析是 `null` | `{t, dt, points, raw_points, t_stop, series:{位号:{kind, current?, voltage?}}}`；**最多 400 点**（原始可达 10 万点，本来不足就原样给），降采样是**保峰**（每桶留 min/max，不会像均匀抽样那样把窄脉冲抽没），但 `points` **不固定是 400**（桶内极值落在同一下标时该桶只贡献 1 点，实测常见 200~400）；所有曲线共用同一组抽样下标、末点必取，不会错位；取不到 `@dev[i]` 的器件**不进 series**（不填 0）。电压源**有** `current`（`i(v1)`） |
| `switch_states` | 请求带了 `switch_states` 时 | 每个开关**最终的状态**：`{"SW1": "on"}` 或 `{"SW1": "off"}`，界面按它画开关 |
| `switch_apply` | 同上 | 后端**用什么办法**强制状态（`"source"` = 改控制源的 DC 值；`"vt"` = 改该开关的 `.model Vt`）。**调试用，不是状态** |

`switch_states` 的覆盖发生在**仿真之前、作用在电路对象上**，不改你的代码文本。
**位号不存在不再让整次运行失败**：忽略它、往 `warnings` 里写一条、`ok` 仍为 `true`
（界面上点过开关之后又把代码里的 `c.SW(...)` 删了，这是常见情形）。仍然会报错的是两类真错误：
状态词认不出（如 `"maybe"`），以及开关存在但控制端没有任何独立源、全电路也没有独立源。

---

## 工程细节（参数化工具这一层）

### 1. 元件只取 E 系列标准值

`copilot/elements.py` 内置 E6/E12/E24/E96 标称值表。参数在 `Param.series` 上声明属于哪个系列
（电阻多用 E24/E96，电容多用 E12），搜索时只在这些值里挑：

```python
Param("R1", "反馈电阻 R1", "R", 10.0, 1e6, series="E96")
Param("C1", "串联电容 C1", "C", 1e-12, 1e-3, series="E12")
```

于是「理想计算值」和「能买到的东西」之间的误差是**结构性存在**的：
你要 1.59kΩ，市场给 1.6kΩ；你要 15.9nF，市场给 15nF 或 18nF。
这个误差正是**必须仿真验证**的根本原因，也是本工具和「教科书计算器」的分界线。

搜索邻域的取法：不是只用「下/近/上」三个标称值，而是在初值 ±1 倍频程内取 k 个
（`agent._neighborhood`）。因为 10nF 与 12nF 之间没有中间档，三值邻域会让 1kHz 目标只能给出
934Hz 或 1142Hz，白扔 12% 余量。

### 2. 容差分析：回答「做 100 台能合格几台」

对每个元件的取值做高斯扰动（`σ` 取相对容差），重新仿真，统计：

- **成品率** = 所有判定行都合格的样本数 / 有效样本数；
- **每个指标的分布**（p5 / 均值 / p95 / 5–95% 波动宽度）；
- **短板指标**（是哪个指标把成品率拉下来的）。

默认容差（`tolerance.DEFAULT_TOL`）：电阻 ±1%（金属膜）、电容 ±5%（薄膜）、
电感 ±10%、运放 GBW ±20%、开环增益 ±50%。可用三层叠加覆盖：
预设（`standard`/`precision`/`tight`）→ 显式 `tol` → 逐项覆盖（`tol_R1` 这种）。
用户原话里的线索也会被捞出来：

```
「用 0.1% 的精密电阻做」→ R 的容差压到 0.1%
（同一份拓扑，0.1% 精密电阻与 1% 金属膜电阻的成品率能差出 40 个百分点）
```

跑 20 次是刻意的取舍：单次仿真 50~165ms，20 次约 2 秒，交互上还忍得住；
而 20 个样本给出的成品率分辨率就是 **5% 一格**。所以报告写 `18/20 = 90%`，
不写 `89.7%`——不希望读者误以为它有那个精度。

### 3. 为什么不用 ngspice 的 `.meas`，指标全在 Python 侧算

ngspice 自带 `.meas`，看起来正是干这个的。放弃它有三个理由：

- `.meas ... FIND vdb(x)` 的结果要从 stdout 正则抠，格式随版本与消息穿插而变，
  而且**失败时退出码仍是 0**——判定依赖这种东西，等于把确定性外包给字符串匹配。
- batch 模式下只有 `.meas` 会静默不跑（`no .plot/.print lines in batch mode`），必须先有 `.control`。
- 有些指标 `.meas` 表达不了：高通通带增益要取「f0 上方 20~300 倍频段的中位数」
  （避开低频未进平台、高频被运放滚降污染的两头）；二阶 f0 要靠相位过 ±90° 的锚点；
  CMRR 要把电阻失配与运放自身 CMRR 按误差串联合成。

所以职责是这么切的：**ngspice 只负责把波形算出来**（50~165ms 一次），
**指标、判定、报告全在 Python 里**。所有参数在 Python 侧算成纯数字再写进网表
（ngspice 不认 `pi`），`sim.run()` 会自动注入 `.control` 把波形落成 ASCII rawfile。

这条决策的直接好处：提取逻辑能被单元测试逐条钉住。`tests/test_physics.py` 用解析公式
（`1/(2πRC)`、`f0/Q`、`√(R2/R1)/2`…）核对仿真值，并且**自己实现取点与插值**——
不复用被测代码的工具函数，否则「提取算错」和「插值算错」两个 bug 会互相掩盖。

### 4. 判定：显式声明相对/绝对容差

每个指标一条规则，`mode` 是 `eq`/`min`/`max`，`kind` 必须是 `rel` 或 `abs`：

```python
SPEC_RULE = {
    "fc":         Rule("eq",  0.10, "rel"),   # 截止频率 ±10%（相对）
    "gain_lin":   Rule("eq",  0.05, "rel"),   # 增益 ±5%
    "gain_db":    Rule("eq",  0.6,  "abs"),   # ±0.6dB（绝对）
    "atten_db":   Rule("min", 1.0,  "abs"),   # 衰减「不低于」
    "peaking_db": Rule("max", 0.6,  "abs"),   # 峰化「不高于」
    "vout":       Rule("eq",  0.02, "rel"),
    ...
}
```

比较统一走**允许区间** `[lo, hi]`，报告把这区间印出来（`≥ 90kHz`、`0.9kHz ~ 1.1kHz`），
用户能自己复核判定。拓扑还能用 `spec_overrides` 改语义：放大器和滤波器的截止频率用 `min`
（「不低于」）比 `eq` 更符合工程意图——用户要 100kHz 带宽，交出 128kHz 是好事，不该判「超标」。

> 这套显式声明是**踩过坑才有的**：早期把相对容差在 `min` 模式下当绝对用，
> 「1kHz 目标、10% 容差」被解释成 `≥999.9Hz`，990.9Hz 判不合格，
> Sallen-Key 的成品率被算成 50%（真实 95%+）。详见 `docs/ARCHITECTURE.md` §4.8。

---

## 接大模型（可选，但建议）

**不配 Key 也能用**：需求解析退化成关键词规则（`agent.parse_rules`），
评审退化成分段模板，AI 三个动作返回明确错误。功能完整、结论照样是真实仿真结果——
只是能听懂的说法变少，评审文字不如模型写得细。

配置方式二选一：

```bash
# 方式 1：环境变量（临时）
export CC_LLM_KEY=sk-xxxx
export CC_LLM_BASE=https://api.deepseek.com/v1   # 可选，默认就是这个
export CC_LLM_MODEL=deepseek-chat                # 可选，默认就是这个

# 方式 2：写进配置文件（权限 600），一次配好，之后一直有效
python3 main.py --key sk-xxxx
python3 main.py --key sk-xxxx --base https://api.openai.com/v1 --model gpt-4o-mini
# Web / IDE 里也能配，通过 POST /api/config 落盘
```

配置文件位置：`${CC_HOME:-~/.cc}/config.json`，写入后权限设为 `600`。
优先级（`llm.load_config()`）：内置默认 → 环境变量 → 磁盘配置。**磁盘配置赢**。

`copilot/llm.py` 里只有两个 prompt：`SYSTEM_PLAN`（需求 → `{topology, spec, reason, caveat}` 严格 JSON，
拓扑清单由 catalog 自动拼进去，模型不能瞎编 id）与 `SYSTEM_REVIEW`（150~250 字工程评审，
明确要求「不要说『该设计具有良好性能』这种空话」）。
`ccl/ai/` 用的是同一个 `copilot.llm.LLM` 客户端，但 prompt 是另一套
（`ccl/ai/prompts.py`：`API_CARD` 器件卡 + 7 个可运行 few-shot）。

调用失败（网络、超时、返回格式异常）会自动降级为规则模式，并把原因写进报告的 `**注意**` 一行。

---

## 项目结构

```
circuit-copilot/
├── main.py                 CLI：设计 / --serve / --key / --catalog / --check / --json
├── start.sh                一键启动 Web（检查 ngspice → 停旧实例 → 探活 /api/health）
│
├── ccl/                    ★ CCL 代码库（import ccl，不依赖大模型）12 个模块
│   ├── core.py             Pin / Component / Circuit；expect / verify / sweep / use_ports
│   ├── devices.py          器件库（12 类，含开关 SW）+ 7 张真实模型卡
│   ├── compile.py          Circuit → SPICE 网表（netlist_ref / 参数去重 / 拒收 .control）
│   ├── format.py           数值格式化（fmt_value：网表里一律纯十进制）
│   ├── runner.py           生成分析卡（ac/op/tran/dc）+ 调 copilot.sim + 异常翻译
│   ├── solution.py         Solution：v / db / phase / f3db / peak_db / value_at / dc_curves / 5 个瞬态指标
│   ├── assertions.py       c.expect 的实现（相对/绝对容差、测不到判 NaN）
│   ├── lint.py             静态 + 工作点检查（唯一真相，ccl.ai.review 从这导入）
│   ├── blocks.py           12 个可复用子电路块（只暴露端口，不含独立源）
│   ├── report.py           Markdown 报告（6 章节）
│   ├── errors.py           异常体系（报错带位号与节点名）
│   ├── __init__.py         公开导出
│   └── ai/                 ☆ 可选 AI 子包（7 个文件）
│       ├── generate.py     生成 + 自修复循环（generate / fix）
│       ├── sandbox.py      AST 白名单 + 受限 builtins + 超时
│       ├── prompts.py      API_CARD + 7 个 few-shot
│       ├── spec_check.py   从中文里抽目标数字做兜底核对
│       ├── review.py       叙事 + 波形形状检查（撞轨/峰化）
│       ├── explain.py      中文解释
│       └── __init__.py     公开导出
│
├── copilot/                参数化工具（CLI 那一层）
│   ├── sim.py              ngspice 封装：注入 .control、ASCII rawfile 解析、复数波形
│   ├── topologies/         13 个参数化拓扑（包：base/filters/amplifiers/misc）+ 运放宏模型 opamp_subckt()
│   ├── elements.py         E6/E12/E24/E96 标准值吸附 + 人类友好格式化
│   ├── agent.py            闭环主流程 run()、候选搜索、关键词路由 parse_rules()
│   ├── understand.py       需求理解编排（假设声明 + 追问）
│   ├── feasibility.py      物理可行性预判（纯解析式，不依赖模型）
│   ├── scope.py            能力边界：不做的电路（开关电源/振荡器…）当场拒绝
│   ├── verify.py           语义对账闸门（用户说的数字 vs 系统采用的 spec）
│   ├── judge.py            指标判定（Rule / SPEC_RULE / check / cost）
│   ├── tolerance.py        蒙特卡洛容差分析（成品率 + 分布 + 短板）
│   ├── llm.py              OpenAI 兼容接入（纯 urllib）
│   └── report.py           Markdown 报告 + BOM 解析（从网表反推）
│
├── web/
│   ├── server.py           纯 stdlib ThreadingHTTPServer（设计请求走 NDJSON 流式）
│   ├── index.html          参数化工具的单文件前端（原生 JS + 手写 SVG Bode 图）
│   ├── ccl_api.py          CCL 的 HTTP 适配层（run / ai / report / blocks）
│   ├── ccl_examples.py     IDE 侧栏的示例代码（必须真实可跑）
│   └── ide/                ★ VS Code 风格 IDE（8 个文件 + 4 份 CONTRACT）
│       ├── index.html      DOM 骨架（标题栏/活动栏/侧栏/主区/结果面板/状态栏）
│       ├── style.css       VS Code Dark+ 配色、布局网格、桌面与窄屏断点
│       ├── main.js         集成层：活动栏、侧栏、标签页、运行、AI、报告、电路图全屏与导出、状态栏
│       ├── editor.js       编辑器：行号、高亮、诊断、括号配对、Ctrl+/ 注释、自维护 undo
│       ├── panels.js       结果面板：电路图、SVG Bode（dB+相位）、DC 曲线族、十字线、PNG 导出、指标/工作点/网表/报告
│       ├── schematic.js    ★ 只读电路图：Circuit → SVG（分层布局 + 正交走线）
│       │                      + 工作态/瞬态动画层（applyLive / applyLiveFrame / createPlayer / wireSwitchClicks）
│       ├── commands.js     Ctrl+Shift+P 命令面板
│       └── snippets.js     块与器件的插入片段（自动补 import）
│
├── tests/                  全拓扑物理核对、判定语义、容差性质、边界鲁棒性、端到端
├── tests_ccl/              CCL 库的回归套件（器件/编译/断言/lint/AI/仿真/直流扫描/瞬态指标/电路图）
├── tests_api/              服务层与前端契约的回归套件（路由/分析/配置/插件/UI 契约）
├── tests_ui/               浏览器侧模块的 Node 自测（schematic_selfcheck.mjs：电路图渲染 + 几何连通性）
├── bench/                  AI 拦截能力基准（闸门层 + 真实模型层，含契约与存档结果）
├── plugins/                示例插件（loader.py + common-values，演示 lint 规则如何插入）
└── docs/                   ARCHITECTURE.md / DESIGN_NOTES.md / _examples/（M1 时期输出存档）
```

**技术栈约束：纯 Python 标准库 + ngspice，零第三方依赖。** 这不是洁癖，是刻意的：

它最初是为了在 Android / Termux 里一条命令跑起来而定的规矩，现在仍然成立：
安装步骤每多一行，就多一个「装不上」的理由（Python 版本、wheel、编译工具链、网络、磁盘），
而且桌面端也因此**不需要 npm install**——打开就能用。所以 HTTP 用 `http.server`、调模型用 `urllib`、
测试框架自己写（顺便得到了 `known_bug` 标记能力）、波形数组用内建 `list + math`。
前端同理：**没有 npm、没有 CDN、没有构建步骤**，`web/ide/` 全是 ES module + 原生 DOM + 手写 SVG。

---

## 跑测试

```bash
python3 tests/run_all.py                # 全量（含物理核对表、known_bug 统计）
python3 tests/run_all.py physics        # 只跑物理正确性
python3 tests/run_all.py --strict       # 把「已知缺陷」也算失败（发版闸门）
```

本机当前结果：

```
共 121 项核对：吻合 121 · 超差 0
共 133 项：通过 133 · 已知缺陷 0 · 失败 0 · 跳过 0 · 耗时 112.8s
结论：全部通过。
```

（这两行是原样抄的终端输出；耗时随机器浮动，120s 上下——数字会随用例增加而变，**用例数不是契约**。）

最有价值的两块：

- **物理核对表**：每个拓扑的实测值 vs 现算的解析值 vs 相对误差，逐行打印
  （13 个拓扑共 121 条核对，全部在各自容差内）。
- **`known_bug` 机制**：当断言的物理事实代码做不到时，纪律是**保留严格断言 + 打标记**，
  而不是把断言放宽到刚好通过（`tests/_harness.py` 里 `is_known_bug_fail` /
  `unexpectedly_passed` 两个状态）。**当前标记数为 0**——曾经标记的那批解析/设计式缺陷已经修掉，
  对应的回归用例留在 `tests/test_e2e.py` 的 `e2e_bug` 组里继续跑。

浏览器侧的自测可以脱离服务单独跑（不需要 Python、不需要 Key）：

```bash
node tests_ui/schematic_selfcheck.mjs     # 51 项：9 种符号（开关 S 尚未纳入）、空态、13 种拓扑的连接关系反推
```

> M1~M5 的里程碑自测脚本（`m2_verify` / `m3_verify` / `m5_verify`、`_viz/realcheck.mjs`）
> 随 `scratch_m1/` 目录一起下线了：它们覆盖的内容已经固化进 `tests/`、`tests_ccl/` 的正式回归，
> 留着反而会有一份「跑不通的旧断言」——`m5_verify.py` 里写死的「块数量 = 9」就是这样一个例子。

CCL 库自己的回归套件（不依赖 `tests/`，纯 `import ccl`）：

```bash
python3 tests_ccl/run_all.py       # 全量：器件/编译/断言/lint/沙箱/AI/真实仿真/直流扫描/瞬态指标/电路图
```

本机当前结果（用例数随轮次增长，别把数字当契约）：

```
共 212 项：通过 212 · 失败 0 · 跳过 0 · 已知缺陷 0 · 耗时 30.1s
```

---

## 已知限制（诚实清单）

**模型与分析的边界**

- 运放是**行为级单极点宏模型**（`Rin/E0/R0/C0/Rout`），只有 A0 与 GBW 两个自由度。
  没有电源引脚、没有输出限幅，不含噪声、压摆率、输出摆幅、输入偏置、CMRR 频率特性。
  所以「相位裕度」「带宽」可信，「失真」「噪声」「是否削顶」这类指标它给不出真实值。
  共模抑制是理想的（无限 CMRR），`difference_amp` 的 CMRR 只反映**电阻失配**那一项。
- 参数化工具只做**小信号 AC 分析**（分压类另跑 `.op`）。开关电源的启动过冲、负载阶跃、
  环路时域响应都不在能力范围内。CCL 代码库这边有 `c.tran()`，但拓扑库不用它做指标。
- 分压拓扑的网表只写 `.ac dec 1 1 10`（2 个频点），所以前端那张 Bode 图对它只有两个点、
  几乎没有信息量——分压本来不需要频响，但界面还是会给它画一张。
- 拓扑数量有限（见 `--catalog`，当前 13 个），全是模拟小信号结构：没有开关电源、
  没有数字与混合信号、不做 PCB。
- **RLC 高 Q 的 Q/带宽测量精度取决于扫频密度**。拓扑库的 `build()` 会按 Q 自动加密
  （`ppd = min(max(200, 70·Q), 4000)`，`tests/test_e2e.py` 的 Q=50 用例在 ±15% 内通过）；
  但**手写 CCL 代码**用默认 `c.ac(..., points=200)` 时，Q 高到「−3dB 带宽 < 扫频步长」就测不准：
  Q≈316 的电路（L=1mH / C=10nF / R=1Ω）在 200 点/十倍频下给出 `bw=450Hz`（理论 159Hz），
  把点数加到 4000 后是 `164Hz`（对账时复测：扫频 10Hz~1MHz，200 点 → `549Hz`、4000 点 → `160.8Hz`；
  具体数字随扫频范围变，现象不变）。要测高 Q，自己把 `points` 调大。

**直流扫描与瞬态指标本身的边界（不是 bug，口径见 `ccl/CONTRACT.md`）**

- **多周期脉冲串测不出瞬态指标**。RC 滤过的方波平台被平滑、末段落在低相位时，
  `rise_time / overshoot_pct / settling_time` 全返回 `None`——这是**为了不误报纯正弦**而做的判据
  （要求末段安静或存在足够长的平台）。实测：τ=1ms、脉冲宽 1ms / 周期 2ms 的 RC 方波，
  四个指标全为 `None`。**注意：只把脉冲宽拉长不够**——对账复测，τ=100µs 的 RC、
  脉宽 1ms（10τ）/ 周期 10ms / `t_stop=5ms` 时仍然全 `None`（窗口末段已经回到低电平）。
  **正确用法**：用单次阶跃（`PULSE(pw ≥ t_stop)`），或把 `t_stop` 缩到第一个脉冲内——
  同一电路这样配置后 `rise_time=219.7µs`、`steady_state=5.000V`，读数就回来了。
- **瞬态指标要 `t_stop ≥ 5τ` 才准**。高电平取自末段均值，只跑 1τ 时 `rise_time` 量的是
  「这段爬了多少」：同一个 RC（τ=100µs）在 `t_stop=100µs` 下给 `74.0µs`，
  在 `t_stop=5ms` 下才是 `219.7µs`，`steady_state` 也从 3.067V 变成 5.000V。
- **电路图会有「无圆点交叉」的长线**。带反馈的级联、差分对这类结构里，个别网络会横跨好几列，
  形成看着像连接、其实没有圆点的十字交叉（实测一个典型差分对：31 段走线、20 个引脚、
  7 个连通块，图上有 7 处无圆点交叉，其中一处是 `n1` 的长线压过 `n2` 母线）。
  **可读性会打折，但连接关系保证正确**：同一张图按几何反推连通性，无悬空引脚、
  没有把两个网络画成短路、也没有把同一网络拆散（判据见 `docs/ARCHITECTURE.md`）。

**开关与动画的边界（M6）**

- **开关在 `.op` 下取不到电流**。ngspice 的 `S` 元件在 `.op` 里 `@sw1[i]` 恒为 ≈0
  （实测 1.94e-13 A），`.tran` 下才给真值（同一条电路 19.410 mA）。所以工作点分析里
  `live.items.sw1.current` 会显示 0 —— 那是**取不到**，不是没电流。瞬态动画走真序列，不受影响。
  这条只在「本来该有大电流」时看得出来：**断开态**（`roff=1e9`）op 与 tran 都测到真实漏电
  **5e-09 A**，那是真值。
- **断开态有漏电**：`roff=1e9Ω` 下实测 4.951 nA（不是恒 0）。想更小就把 `roff` 调大
  （1e12 以上）；这样做的代价是收敛更慢。
- **交流分析没有工作态**：`live` 与 `timeline` 都是 `null`，界面不发光、不建播放条。
  这不是没做，而是**频域里没有时域工作点**——拿频域数据演时域动画就是骗人。
- **`live` 的 `voltage` 是绝对值**，没有极性；**三端及以上器件（`Q`/`M`/开关 `S`）的
  `voltage` 与 `power` 直接是 `null`**——取「前两个引脚之差」实测 BJT 得到的是 `Vcb` 而不是
  `Vce`，MOSFET 功耗能差 **20 倍**，宁可不给也不给错数。`live` 取的是**末点**（`tran`/`dc`），
  不是 t=0；但它也**不是「稳态」**——末点只是仿真终点，电路未必已稳定，要看全过程用播放条。
- **动画的「流动方向」不承载物理极性**：导线虚线的跑动方向取自该网络电流的**时间导数符号**，
  稳态时固定正向；只有强弱（`|I|` 相对该网络峰值）是数据。不同网络之间的粗细也不能互相比较。
- **播放节奏不是物理时间倍率**：一整轮约 6 秒墙钟（按 0.25~4× 缩放），真实时刻看播放条读数。
- **开关没有开关速度/弹跳/寿命建模**：`S` 元件是理想电压控制开关（导通/断开两个电阻态），
  不含上升沿、触点弹跳、机械寿命、驱动功耗，也不建模控制的输入电容与延时。
- **报告里的「落地前必须确认」不覆盖开关**：那一节（`copilot/report.py`）只服务参数化工具那 13 个拓扑
  （没有开关类结构），CCL 代码路径导出的报告（`ccl/report.py`，6 章节）里没有这一节。
  所以用开关搭的电路导出报告时，不会有「开关没有速度/弹跳/寿命、断开态有漏电」这类提醒——上面这段文字就是现在的唯一提醒。

**参数化工具的边界**

- 容差是**常温高斯抽样**，`σ` 取标称容差。不建模批次偏移（整批偏 +0.5%）、温漂、老化、
  机械应力；电感与电容的温度特性差异很大，实板上可能更糟。
- 默认只跑 **20 次**样本：成品率分辨率 5%/格，p5/p95 是粗略分位。要更高精度得改
  `agent.run()` 里的 `runs=20`（代价是线性变慢：一次扰动 = 一次完整仿真）。
- 规则模式（没配 Key）的 `parse_rules()` 靠关键词，**解释可能归错因**：
  结论（达标/不达标）是仿真给的、可信；「为什么不达标」这句话不一定点对指标。
- 拓扑关键词的顺序即优先级（「二阶高通」必须先于「高通」匹配），改顺序等于改行为。
- `_candidates()` 的搜索邻域是「初值 ±1 倍频程内取 k 个标称值」，
  `design()` 的解析式一旦算错，搜索**兜不回来**——所以每个 `design()` 都配了物理核对用例。

**部署形态**

- 单用户、单机、无鉴权：`ThreadingHTTPServer`，会话只存在内存里，进程重启后
  `/api/report/<id>` 会失效（提示重新生成）。默认只监听 `127.0.0.1`。
- IDE 没有语义自动补全（`editor.js` 里**也没有** TODO 标记，就是单纯没做）：高亮、缩进、
  括号配对、注释切换都有，但没有补全候选——那是另一个量级的工程。

---

## 文档

| 文档 | 内容 |
|---|---|
| `ccl/CONTRACT.md` | CCL 的契约与决策全文（M1~M6 的接口约定、每条决策的原因、直流扫描与电路图的口径与限制、**开关器件与 live/timeline 的口径**） |
| `ccl/CONTRACT_ROUND3.md` | 第三轮：直流扫描（一维/两维）与瞬态指标的原始契约与验收标准 |
| `web/ide/CONTRACT.md` | IDE 的布局网格、模块契约、验收标准 |
| `web/ide/CONTRACT_ROUND4.md` | 第三轮优化：实测缺陷清单（竖屏/横屏溢出）+ 早期移动端编码刚需——该轮以手机为目标，定位已改为桌面优先，本文保留原貌作历史 |
| `web/ide/CONTRACT_ROUND5_DESKTOP.md` | 桌面版对标 VS Code：桌面基线实测、头号 bug、键位与布局对齐 |
| `web/ide/CONTRACT_SCHEMATIC.md` | 电路图渲染模块的契约：符号、布局算法、自测要求 + **工作态与瞬态动画层**（播放条、逐帧渲染、开关点击、导线流动的口径） |
| `docs/BANDPASS_Q_INVESTIGATION.md` | 主动带通（Q>1）调研：两轮推导与实测，**结论是暂不落地**及理由 |
| `docs/pitfall_rlc_bandpass_q.md` | 踩坑案例：RLC 带通的 Q 由拓扑+元件值定死，照着「想要的 Q」写网表会得到漂亮但错的结果 |
| `bench/README.md` | AI 拦截能力基准：两层跑法、类别覆盖、实测数字与自我约束 |
| `docs/ARCHITECTURE.md` | 模块依赖、一次请求的完整时序与各阶段耗时、CCL 五层结构、IDE 前端结构、**电路图渲染与连接正确性**、**踩坑记录专章**、扩展指南 |
| `docs/DESIGN_NOTES.md` | 为什么代码优先、为什么断言在核心库、为什么修复要先确定性诊断、为什么指标在 Python 侧算、为什么用 stdlib、这个项目不做什么 |
| `docs/COMPETITIVE.md` | 竞品分析（LTspice / SKiDL / Celus / JITX / AnalogCoder …）与差异化定位 |
| `docs/_examples/` | M1 时期两次真实运行的完整输出存档（结论数字仍对得上；其中的网表是旧的固定 1Hz~100MHz 扫频卡，现在由 `ac_sweep()` 按设计频率生成） |

---

## 这个项目是什么

对着 LTspice、SKiDL、Celus、JITX、AnalogCoder 逐项比过之后（见 `docs/COMPETITIVE.md`），
一句话是这么定位的：

> **不是「另一个电路仿真器」，而是设计意图的验证层 + 会说「不」的 AI。**

- 对着 LTspice：我比它快，而且帮你算成品率；
- 对着 SKiDL：我不只生成网表，我验证它（`c.expect` + `c.lint` + 电路图投影）；
- 对着「AI 生成电路」那类工具：我不只生成，**测不到就返回 `None`、不达标就判失败**，
  拒绝交付比交付错的东西更有价值。

边界也写在纸面上：12 类器件（R/C/L/V/I/运放/D/Q/M/脉冲/正弦/开关）、Level-1 MOS、行为级运放、
没有噪声 / THD / 温度 / 寄生，
**不做 PCB 与布局布线**——它现在是「教学 / 快速估算 / 方案预研」级工具，不是签核工具。
