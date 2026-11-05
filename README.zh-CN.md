# 无人机射频取证 — DJI OcuSync DroneID

这个仓库做无人机射频信号的被动接收与取证。收到信号只是第一步，后面要回答的是抓全了没有、解对了没有、换个机型还认不认。

工具与证据框架取自 [luyii-code-1 的 DroneID 研究](https://github.com/luyii-code-1/dji-ocusync-droneid-research)，来源与致谢见 [CREDITS.md](CREDITS.md)。

[English](README.md) | [简体中文](README.zh-CN.md) | [研究方案](RESEARCH.md) | [本仓库进度](docs/PROGRESS.md) | [证据台账](docs/FACTS.zh-CN.md) | [致谢与来源](CREDITS.md)

---

## 研究问题

低空监测项目里被追问最多的一句是：凭什么说探测到了，又凭什么证明就是那架。频谱上有能量不算，解出帧头也不算。能站住的证据链至少要答四件事：

| 编号 | 研究问题 | 判定标准（我们给自己定的） |
|---|---|---|
| RQ1 | 被动接收抓到的是什么，抓全了没有 | 给定采集参数，能算出该时段理论应到的帧数与实到帧数；漏帧有归因（跳频未覆盖 / 信噪比不足 / 采集丢样），不是一句"信号弱" |
| RQ2 | 解出来的东西是不是真的解对了 | 双重 CRC（传输块 CRC24A + 逻辑包 DJI CRC16）全过才算解出；只过一层算候选，不进结论 |
| RQ3 | 同一架机器、跨会话，能不能稳定关联 | 会话标识（4 字节 hash）在 CRYP 与 INFP 之间一致；会话刷新后旧密钥失效；关联结论不依赖单次捕获 |
| RQ4 | 不同机型、不同固件，上面三条还成不成立 | 每个结论带机型 + 固件 + 采集参数三要素；未做跨机型验证的结论一律标注为单点观测 |

所以这里产出的不是一个"能解密"的结论，是带误差和适用边界的方法。具体方案、实验矩阵、里程碑在 [RESEARCH.md](RESEARCH.md)。

### 为什么这么划分

原项目公开的主线是两条：会话密钥可预测性，和围绕 AeroScope dongle 的硬件路线。两条都在解决"把明文拿回来"。理解协议绕不开这两条，但布监测、出报告、过合规审查要的不是明文，是漏了多少、误报多少、换机型还认不认、报告别人能不能复算。

重心就落在可复现和可审计上：每层结论要有对应中间产物（IQ → 符号 → LLR → 传输块 → 逻辑包 → 明文），缺哪层，结论就停在那一层。这条来自原项目的证据分级，我们把它当主流程，不只是文档末尾的一段规则。

---

## 跟原项目各自做什么

| 事项 | 上游（luyii-code-1） | 本仓库 |
|---|---|---|
| O2 PHY / FEC / CRC 参考实现 | 建立并本地验证 | 继承，作为回归基线 |
| O4 逻辑包结构（CRYP / INFP） | 建立并跨源确认 | 继承，用于包层校验 |
| 会话密钥可预测性（原仓库称"方法二"） | 主线研究方向 | **不复现、不延伸**；只在取证语境下讨论其披露边界 |
| AeroScope dongle 硬件路线 | 记录第三方演示 | 不涉及 |
| 检测侧方法学（检出率 / 漏帧归因 / 误报） | 未系统展开 | **本仓库主线** |
| 跨机型 / 跨固件适用边界 | 列为待办 | **本仓库主线**，落实验矩阵 |
| 证据可审计性（中间产物留存、第三方复算） | 有分级规则 | 升级为强制产物规范 |

---

## 技术基线

以下是从上游继承的事实基线，用来做对齐，不在此重复推导。

### 逻辑包通用头

```text
offset  size  meaning
0       1     packet_length_minus_3
1       1     message type / subtype
2       4     ASCII marker (`CRYP`, `INFP`, ...)
6       4     hashcode（会话关联标识）
...     ...   type-specific body
last-2  2     DJI CRC16, little-endian
```

逻辑长度 = `packet[0] + 3`。DJI CRC16 初值 `0x3692`，反射多项式 `0x8408`，完整逻辑包余数为零。

### CRYP / AA：会话密钥封装

```text
AA 13 43 52 59 50 [hashcode]
      C  R  Y  P

offset    size   interpretation
10:74     64     C1 = X || Y，两个 32 字节大端坐标
74:106    32     C3
106:122   16     C2（明文长度 = 16 字节会话密钥）
122:...   ...    其余协议字段 / 尾部数据
```

C1 满足标准 SM2 曲线方程（`a`、`b`、`p` 见上游文档）。多个独立会话的 C1 均落在曲线上。概念上：

```text
S    = d × C1
mask = SM2_KDF(S.x || S.y, 16)
note = C2 XOR mask
C3   = SM3(S.x || note || S.y)
```

`d` 是接收端私钥，本仓库不含、也不去获取。

### INFP / 87：遥测加密

```text
cipher = AES-128-CTR
key    = note（会话密钥）
IV     = nonce8 || 0x00×8
```

拿到同会话的 `note` 后，多个连续的 87 包解出稳定字段结构，序号、时间戳、经纬度、高度彼此连贯。

### O2 帧结构与 FEC 链

```text
sample rate       15.36 MS/s
FFT               1024
subcarrier space  15 kHz
active carriers   600, DC excluded
symbols           9
normal CP         72
long CP           80
symbol 3          ZC root 600
symbol 5          ZC root 147
data symbols      1,2,4,6,7,8
```

```text
6 QPSK symbols × 600 carriers × 2 bit = 7200 soft bits
→ Gold 解扰
→ LTE 反向速率匹配 (E=7200, RV=0)
→ LTE Turbo (K=1408, D=1412)
→ 176 字节传输块
→ CRC24A
→ 逻辑载荷
→ DJI CRC16
```

两条 CRC 都过才认一帧。

### 采样率是硬约束

HackRF 原始文件是交织的 signed-int8 I/Q（每复数样点 2 字节）。20 MS/s 采集要跑 15.36 MS/s 检测器，必须按 `20 × 96 / 125 = 15.36` 重采样。采样率错了，ZC 时间尺度、CP/FFT 对齐、CFO 估计、CRC 结果会一起错，而且错得很像"信号不好"。

---

## 仓库布局

```text
src/o4_packet_tool.py          CRYP/AA 结构检查；已知 note 时解 INFP/87
src/o2_droneid_decode.py       O2 PHY/FEC 参考解码器
src/droneid_hackrf_scanner.py  HackRF 实时 / 回放扫描器
tools/remove_turbo_soft.c      TurboFEC 适配层
tests/                         包工具与频点预设的单元测试
docs/PROGRESS.md               本仓库里程碑进度
docs/FACTS.zh-CN.md            事实与证据台账（继承，标注来源）
docs/STATUS.zh-CN.md           研究进展（继承，标注来源）
RESEARCH.md                    研究方案：问题、假设、实验矩阵、指标、里程碑
CREDITS.md                     原作者与全部上游来源
NOTICE                         GPL-3.0 衍生声明与修改记录
```

---

## 快速开始

依赖：Python 3.10+、numpy、scipy、OpenSSL 命令行。

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt

# 检查一个完整 AA/CRYP 包
python3 src/o4_packet_tool.py '<complete-AA-hex>'

# 已知 16 字节会话密钥时解同会话 87 包
python3 src/o4_packet_tool.py --note '<16-byte-note-hex>' '<complete-87-hex>'

# 被动扫 5.8 GHz（含 5816.5 MHz）
python3 src/droneid_hackrf_scanner.py --band 5g --preset baseline

# 两个频段，用上游 Issue #2 的频点表
python3 src/droneid_hackrf_scanner.py --band both --preset issue2

# 自定义中心频率（覆盖 --preset / --band）
python3 src/droneid_hackrf_scanner.py --freqs 2474.5
```

公开工具只覆盖包校验与"密钥已知时的 AES 步骤"。不含任何私有商用密钥材料，也不含 `d`。

### 频点预设

| `--preset` | 含义 | 2.4 / 5.8 GHz 条目数 |
|---|---|---|
| `baseline`（默认） | 上游既有扫描顺序，5.8 GHz 段加入 5816.5 MHz | 5 / 4 |
| `issue2` | 上游 Issue #2 报告的频点表 | 4 / 4 |
| `paper2022` | Bender 2022 Table II 去重条目；**历史数据，不是 O4 频点图** | 5 / 7 |

三个预设来自不同时期的观测，**不能合并成一张"通用频点表"**。2022 年论文正文提到 2474.5 MHz 上限，但 Table II 重复的是 2459.5 MHz，所以 `paper2022` 里没有 2474.5 MHz。

v3 公开规范给出的是 8 符号波形与七频点跳频（2414.5 / 2429.5 / 2444.5 / 5756.5 / 5776.5 / 5796.5 / 5816.5 MHz，`2459.5` 被排除）。**本仓库扫描器没有实现 v3 PHY，也没有实现跳频跟随**，`--dwell` 不是同步。

---

## 证据规则

判定顺序（自上游继承，本仓库强制执行）：

```text
原始 IQ
> 本地复现的双 CRC 结果
> 本地复现的明文字段
> 独立公开的一手协议确认
> 公开代码
> 讨论区陈述
> 假设
```

推论：

- 有能量不等于 DroneID；
- ZC/CP 对上只是 PHY 候选，不是解出；
- CRC24A 通过 = 传输块可信；
- DJI CRC16 通过 = 逻辑包边界与内容可信；
- 合法但加密的包仍是密文，直到密码步骤被复现；
- 第三方声称"有离线解码器"只证明实现存在，不证明方法公开可复现。

每次实验必须留下的中间产物：

```text
candidate.iq          原始候选段
symbols_fd.npy        频域符号
equalized.npy         均衡后
llr_7200.npy          软比特
descrambled.npy       解扰后
transport_176.bin     传输块
logical_payload.bin   逻辑载荷
decode.json           参数与判定结论
```

---

## 里程碑

| 阶段 | 目标 | 完成判据 |
|---|---|---|
| M0 | 环境与回放链路跑通 | 上游测试全过；给定捕获能稳定复现双 CRC |
| M1 | 采集参数敏感性 | 采样率 / 增益 / 时长三因素矩阵，给出检出率曲线与漏帧归因 |
| M2 | 频点覆盖与跳频跟随 | 按公开七频点表实现最小可行跟随，报告实测捕获率 |
| M3 | 跨机型适用边界 | Air 3S / Mini 4 Pro / Avata 2 等机型各 ≥1 组双 CRC 有效样本，标注固件 |
| M4 | 取证方法学成文 | 一份可被第三方按文档复算的检测与取证流程，含误报分析 |

当前进度见 [docs/PROGRESS.md](docs/PROGRESS.md)。M0 之外的阶段都还没有样本。

---

## 发布与隐私

示例一律使用合成或匿名化标识与坐标。不发布：私人服务凭据、会话密钥、真实序列号、操作者或住宅精确坐标、可定位到个人的采集元数据。

## 使用边界

本仓库支持互操作研究、频谱分析与授权安全研究。仅用于被动接收与经明确授权的测试，遵守所在辖区的无线电、隐私、航空与计算机安全法规。

DJI、OcuSync、AeroScope 及相关产品名称归各自权利人所有。本项目独立，与 DJI 及文中提及的第三方无隶属或背书关系。

---

## 来源

工具与证据框架来自 [luyii-code-1](https://github.com/luyii-code-1) 的 [dji-ocusync-droneid-research](https://github.com/luyii-code-1/dji-ocusync-droneid-research)（GPL-3.0）：DroneID 的 PHY / 逻辑包 / 密码链路研究框架，`src/` 下三个工具（HackRF 扫描器、O2 解码器、O4 包工具），`tools/remove_turbo_soft.c` 的 TurboFEC 适配层，以及 `docs/` 下的双语证据台账。协议的一手描述在那个仓库，比这里全。

我们改的是研究问题本身：原项目问"链路能不能打通"，我们问"打通之后，把它当证据用成不成立"。逐文件来源归属和修改记录见 [NOTICE](NOTICE)，完整致谢见 [CREDITS.md](CREDITS.md)。

---

## 许可与致谢

GPL-3.0。原始版权归 luyii-code-1 及其许可声明中的权利人；本仓库的衍生部分同样以 GPL-3.0-or-later 发布。逐文件来源与修改记录见 [NOTICE](NOTICE)。

**原始研究、工具与证据框架：[luyii-code-1 / dji-ocusync-droneid-research](https://github.com/luyii-code-1/dji-ocusync-droneid-research)。** 这里的代码和证据规则都出自那里。

同时致谢本项目引用到的上游公开工作：

- **EdwardBlair** — [dji-droneid-spec](https://github.com/EdwardBlair/dji-droneid-spec)：O4 协议、v3 RF/PHY 规范、合成测试向量
- **alphafox02** — [antsdr_dji_droneid](https://github.com/alphafox02/antsdr_dji_droneid)：CRYP/INFP 语义的独立确认
- **proto17** — [dji_droneid](https://github.com/proto17/dji_droneid)：接收机与跳频讨论
- **King-Of-Knights** — 上游 Issue #1 的 dongle 控制台演示、Issue #2 的频点表
- **Bender 等（2022）** — DroneID 频点历史数据
- **Edgesource / Aerial Defence（2024）** — AeroScope 升级模块硬件与协议白皮书

完整引用（含具体评论链接与页码）见 [CREDITS.md](CREDITS.md)。
