# 致谢与来源

本仓库是衍生研究。以下按贡献层次排列，一手来源在前。

---

## 一手来源：本仓库的基础

### luyii-code-1 — 原始研究、工具与证据框架

- GitHub: [@luyii-code-1](https://github.com/luyii-code-1)
- 仓库: [luyii-code-1/dji-ocusync-droneid-research](https://github.com/luyii-code-1/dji-ocusync-droneid-research)
- 许可: GPL-3.0

本项目直接继承的成果：

| 内容 | 原始位置 | 本仓库位置 |
|---|---|---|
| HackRF 扫描器与频点预设框架 | `src/droneid_hackrf_scanner.py` | `src/droneid_hackrf_scanner.py` |
| O2 PHY/FEC 参考解码器 | `src/o2_droneid_decode.py` | `src/o2_droneid_decode.py` |
| O4 逻辑包工具（CRYP/AA 结构、已知 note 解 INFP/87） | `src/o4_packet_tool.py` | `src/o4_packet_tool.py` |
| TurboFEC 适配层 | `tools/remove_turbo_soft.c` | `tools/remove_turbo_soft.c` |
| 单元测试 | `tests/` | `tests/` |
| 事实与证据台账（中英） | `FACTS.md` / `FACTS.zh-CN.md` | `docs/` |
| 研究进展表（中英） | `STATUS.md` / `STATUS.zh-CN.md` | `docs/` |
| 贡献与隐私约定 | `CONTRIBUTING.md` | `docs/CONTRIBUTING.md` |
| 证据分级方法论 | README「Evidence rules」段 | 本仓库 README 同名段，升级为强制产物规范 |

没有这份工作，这里的代码和证据规则都不存在。协议的第一手描述以原仓库为准。

原作者在 O4 密码链路上的披露尺度（只讲方向，不发敏感参数和真实会话密钥）也直接影响了这里的隐私约定。

---

## 上游公开研究（由原仓库与本仓库共同引用）

### EdwardBlair — 协议规范与 v3 RF/PHY

- 仓库: [EdwardBlair/dji-droneid-spec](https://github.com/EdwardBlair/dji-droneid-spec)
- 关键提交: [`55bb886`](https://github.com/EdwardBlair/dji-droneid-spec/tree/55bb88676dbc75913f074d7816ed5f7a3ab92454)（2026-10-22）
- 贡献: O4 协议结构、`C1‖C3‖C2` 语义、v3 八符号波形与七频点跳频表、合成测试向量与 dummy 私钥往返测试
- 本仓库用到的具体事实：七频点表（2414.5 / 2429.5 / 2444.5 / 5756.5 / 5776.5 / 5796.5 / 5816.5 MHz，排除 2459.5）、640 ms 发布周期、`r=(c+255)%256`、`i=(r+6)%7`、163.84 s 超周期

### alphafox02 — CRYP/INFP 语义的独立确认

- 仓库: [alphafox02/antsdr_dji_droneid](https://github.com/alphafox02/antsdr_dji_droneid)
- 关键讨论: [Issue #27](https://github.com/alphafox02/antsdr_dji_droneid/issues/27)
- 贡献: 独立确认 CRYP 承载 SM2 封装的会话密钥、INFP 为 AES-CTR 遥测；离线/在线实现的公开报告

### proto17 — 接收机与跳频研究

- 仓库: [proto17/dji_droneid](https://github.com/proto17/dji_droneid)
- 关键讨论: [Issue #60](https://github.com/proto17/dji_droneid/issues/60)、[Issue #63](https://github.com/proto17/dji_droneid/issues/63)
- 贡献: 接收机实现讨论、跳频行为报告、机型覆盖报告（Air 3S / Avata 2 / Neo / Neo 2 / Mavic 3 Thermal）
- 注意: Issue #63 中"O4 解密密钥"的说法随后被本人澄清为在线服务密钥，**不是** SM2 私钥

### King-Of-Knights — 频点表与 dongle 演示

- 上游 Issue #1 的 AeroScope dongle 控制台记录
- 上游 Issue #2 的实测频点表（对应本仓库 `--preset issue2`）
- 说明: dongle 的 USB 命令协议与脚本未公开，本仓库不涉及

### Bender 等（2022）— DroneID 频点历史数据

- 对应本仓库 `--preset paper2022`（Table II 去重条目）
- 用途: 历史对照，**不是** O4 频点图
- 注意: 正文提到的 2474.5 MHz 上限与 Table II 不一致，本仓库按 Table II 处理并显式标注

### Edgesource / Aerial Defence（2024）— AeroScope 升级模块白皮书

- [Security Risks of the Aeroscope Upgrade Module (2024-03)](https://www.aerial-defence.com/wp-content/uploads/2024/03/Security-Risks-of-the-Aeroscope-Upgrade-Module-Whitepaper-March-2024.pdf)
- 贡献: 升级模块硬件结构、主机↔dongle 认证与通信会话密钥、CRYP 解封装流程（pp. 10–12）
- 注意: 该文**未**证明 dongle 使用 TEE

---

## 本仓库自身的工作

| 内容 | 位置 |
|---|---|
| 研究问题重构（链路打通 → 取证方法学） | `README.md`、`RESEARCH.md` §1 |
| 分层模型与结论层级规则 | `RESEARCH.md` §1.3 |
| 研究问题与可证伪假设 RQ1–RQ5 / H1–H5 | `RESEARCH.md` §2 |
| 实验设计与因素矩阵 | `RESEARCH.md` §3 |
| 指标定义（DR / P / FR / Lat / RC / EB） | `RESEARCH.md` §3.4 |
| 产物规范与 `decode.json` 字段要求 | `RESEARCH.md` §4 |
| 里程碑 M0–M4 与判据 | `RESEARCH.md` §5 |
| 风险登记与自我约束 | `RESEARCH.md` §7 |

---

## 引用建议

引用本项目时，请同时给出上游一手来源，避免读者误认一手结论归属：

```text
本仓库：<repo-url>（衍生自 luyii-code-1/dji-ocusync-droneid-research）
原始：https://github.com/luyii-code-1/dji-ocusync-droneid-research
```

协议与 PHY 的一手规范请引 EdwardBlair 的 `dji-droneid-spec`，不要引本仓库。
