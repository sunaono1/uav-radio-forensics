# 本仓库进度

更新：2026-11-05

上游原始仓库的研究进展见 [STATUS.zh-CN.md](STATUS.zh-CN.md)（继承文档，反映 luyii-code-1 的工作状态）。本文件只记本仓库自己的里程碑。

## 里程碑状态

| 阶段 | 状态 | 说明 |
|---|---|---|
| M0 环境与回放链路 | **部分完成** | 依赖安装、单元测试、工具冒烟已过；自有捕获的双 CRC 复现尚无样本 |
| M1 采集参数敏感性 | 未开始 | 需硬件与采集场次 |
| M2 频点覆盖与跳频跟随 | 未开始 | 依赖 M1 参数结论 |
| M3 跨机型适用边界 | 未开始 | 依赖可用机型样本 |
| M4 取证方法学成文 | 未开始 | 依赖 M1–M3 |

## M0 已验证项（2026-11-05）

环境：macOS，Python 3.13.15，numpy 2.5.4，scipy 1.18.1，pytest 9.2.0。

| 项 | 结果 |
|---|---|
| `pip install -r requirements.txt`（经 venv） | 通过 |
| `pytest tests -q` | 8 passed |
| `o4_packet_tool.py --help` | 正常 |
| `droneid_hackrf_scanner.py --help` | 正常（含 `--preset {baseline,issue2,paper2022}`） |
| `cc -fsyntax-only tools/remove_turbo_soft.c` | 失败：缺 `turbofec/rate_match.h`，属外部依赖未安装，非代码缺陷 |

## M0 待办

- [ ] 安装 TurboFEC 并编译 `tools/remove_turbo_soft.c`，跑通 O2 解码路径
- [ ] 用一份自有 HackRF 捕获走完 L0→L3，产出完整 `decode.json`
- [ ] 连续 3 次重跑结果一致，记录 sha256
- [ ] 建立 `captures/<capture_id>/` 首个样例目录

## 说明

M0 的"通过"不包括任何加密载荷的解密。本仓库的解密分支只在已有合法同会话密钥或合成测试向量的情况下运行，见 [RESEARCH.md](../RESEARCH.md) §1.2。
