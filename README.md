# RV32IM 五级流水线 RISC-V 处理器

> 全链路自研的 32 位 RISC-V 处理器系统：五级流水线 + 片上存储 + 分层总线互联 + MNIST 手写识别，全部数字可复现。

本仓库以压缩包形式交付完整工程与可复现验证包（含技术文档）。压缩包 `riscv-cpu-repro.rar`
为单层扁平结构，解压后直接得到工程根目录的内容。

| 项目 | 说明 |
|---|---|
| 指令集 | RV32I（含 Zicsr）+ M 扩展，共 40 余条指令 |
| 流水线 | IF-ID-EX-MEM-WB 五级，多级前递 + 选择性停顿 |
| 控制流预测 | 256 项两位饱和 BHT + 8 层 RAS |
| 存储 | 32 KiB ITCM + 32 KiB DTCM，程序与权重驻留片内 |
| 总线互联 | AXI-Lite → toggle 同步 CDC 桥 → APB（UART / 按键 / PWM / GPIO） |
| 目标器件 | AMD Artix-7 `xc7a200tfbg484-2`，Vivado 2025.2 |
| 参赛组别 | 高级组（按规则 3.3.1.1 以队内最高学历归组） |
| 应用演示 | CPU 执行 INT8 定点 CNN 完成 MNIST 手写识别（无专用 AI 加速器） |

## 主要特性

- **完整 SoC 而不止于核**：RTL 手写到板级落地，含约束、跨时钟域、固件烧写与上位机交互，不是仿真通过即止。
- **性能量化可对照**：CoreMark 与 CSR 计数器给出 CPI、CoreMark/MHz；分支/返回预测做成开关对照实验，调用返回程序由 107 周期降至 77 周期（提升 28.0%）。
- **通用处理器跑 AI**：INT8 卷积网络由 CPU 的 MUL/MULH 完成重缩放，上位机提供单字练习、六位记忆挑战与 MNIST 批量测试三种递进模式。
- **验证四层全覆盖**：模块单元测试 → CPU 核心集成仿真 → 系统软件测试 → 板级演示，45 条回归用例，波形、日志、问题清单全部保留。

## 性能指标

| 指标 | 数值 |
|---|---|
| CoreMark（3200 次迭代） | CPI 1.124、CoreMark/MHz 3.08、seedcrc/crclist/crcmatrix/crcstate/crcfinal 五项 CRC 校验通过 |
| 分支预测准确率 | 92.83%（BHT）/ 98.71%（RAS 返回预测） |
| 手写识别（MNIST 批量） | 100 样本正确 98 幅，端到端平均 146.3 ms（其中 UART 传输约 114.7 ms、板内推理约 30 ms） |
| FPGA 资源 | LUT 5680（4.25%）、FF 3800（1.42%）、BRAM 16（4.38%）、DSP 7（0.95%） |
| 时序（100 MHz） | WNS +0.416 ns，无 setup 违例 |
| 预测器开关对照 | CoreMark 负载下关闭 BHT/RAS：2,575,471 → 3,362,623 周期（+23.41%），两组 CRC 一致 |

## 仓库内容

```
riscv-cpu-repro.rar（解压后直接得到工程根目录，无额外文件夹）
├── report/design-report.pdf  完整技术文档（50 页）
├── README.md           复现步骤与验收基线
├── LICENSE / notice.md MIT 许可与第三方声明
├── src/                RTL（rtl/）、软件（software/）、补丁（patches/）
├── sim/                测试平台（tb/）、仿真脚本（scripts/）
├── build/              复现脚本（scripts/）、报告（reports/）、位流（bitstream/）
├── board/              上位机（host/）、约束（constraints/）、演示位流（bitstream/）、上板实测日志（logs/）
├── data/               内存镜像（mem/）
├── skill/              可复现验证技能包（SKILL.md / README.md + references/）
└── report/             验证报告、AI 协作记录与配图、已知失败清单（known-failures.md）
```

## 快速开始

**环境**：Windows 10/11 + PowerShell 5.1 + Vivado 2025.2（含 XSim）+ `riscv-none-elf-gcc` 工具链。

```powershell
# 以下命令均在解压后的工程根目录执行（含 README.md、src/、sim/ 等）

# 1. 环境 + 仓库体检（FAIL=0 WARN=0 为通过）
powershell -NoProfile -ExecutionPolicy Bypass -File .\build\scripts\repro\preflight.ps1

# 2. 一致性审计（注册表 / 源清单 / 固件清单 / mem 镜像 / 文档漂移）
powershell -NoProfile -ExecutionPolicy Bypass -File .\build\scripts\repro\verify_repo.ps1

# 3. 冒烟：编译 + 跑 cpu_top_tb（期望 PASS: cpu_top_tb）
powershell -NoProfile -ExecutionPolicy Bypass -File .\build\scripts\repro\repro.ps1 -Stage smoke

# 4. 全量回归（default / extended 两级）
powershell -NoProfile -ExecutionPolicy Bypass -File .\build\scripts\repro\repro.ps1 -Stage default
powershell -NoProfile -ExecutionPolicy Bypass -File .\build\scripts\repro\repro.ps1 -Stage extended
```

每步的期望输出与失败判据见压缩包内 `README.md` 与 `skill/references/00-repro-manual.md`。

## 上板演示（解压即烧，无需重新综合）

压缩包内**已附带两份固化好的位流**，解压后直接烧录即可，不必等 20–40 分钟的综合。

| 烧哪个（相对解压后的工程根目录） | 跑出来是什么 |
|---|---|
| `board/bitstream/digit-recog-feedback.bit` | 手写数字识别；答对播 1.5 秒上行 C 大调、答错播 1.5 秒下行 a 小调，三种模式提示一致 |
| `board/bitstream/coremark-3p08-3200iter.bit` | CoreMark 3200 次迭代；串口打印五项 CRC、CPI 1.124、CoreMark/MHz 3.08、`PASS` |

两份位流各带各的固件，**别混用**（仓库里 `data/mem/` 固化的镜像属于识别应用），对照表见 `board/bitstream/README.md`。

**方式 A — Vivado 图形界面（推荐）**
Hardware Manager → Open Target → Auto Connect → **Program Device**，Bitstream File 选上表中的 `.bit`（路径是**解压出来的工程根目录**下的 `board\bitstream\`）。

**方式 B — 脚本烧录**（在解压后的工程根目录执行，路径不要带中文或空格）
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\build\scripts\vivado\run_program_bit.ps1 `
    -Bit .\board\bitstream\coremark-3p08-3200iter.bit
```

**烧完做什么**
- CoreMark 版：开串口助手（**69120 波特**、8N1），板子复位后应打印 `CoreMark Size : 666` 与 `Status : PASS`。
- 识别版：先装上位机依赖，模式 3 还要补一次数据集（其余模式不需要）：

  ```powershell
  pip install pyserial pillow numpy python-mnist   # 注意包名是 python-mnist，导入名才叫 mnist
  python board\host\fetch_mnist_data.py            # 仅模式 3 需要，约 58 MB
  python board\host\draw_and_recog_enhanced.py COM14
  ```

  启动后在界面选串口（**69120** 波特、8N1）与三种模式之一：单字练习 / 六位记忆挑战 / MNIST 批量测试。三种模式都用同一套 1.5 秒旋律提示对错（答对上行 C 大调九音 C4→C5，答错下行 a 小调六音 A4→C4，全部压在一个八度 262–523 Hz 内）；模式 3 批量测试连跑 100 张图时逐张提示。旋律细节见 `report/digit-feedback.md`。

> 烧录报 `No hw_target found` 时依次检查：板卡上电、USB/JTAG 电缆、Digilent/Xilinx 驱动、是否有别的 Vivado 进程占着 `localhost:3121`。

## 波形复现

压缩包内 `README.md`「波形复现」一节给出 5 张关键波形（BHT / M 扩展 / load-use / 返回预测 / AXI-Lite CDC 桥）的手工复现步骤与观测信号清单，对应技术文档 3.3.4 节图 14~图 18。

## 技能包

`skill/` 目录把整套复现流程封装成 agent 可直接调用的技能（`skill/SKILL.md` + 9 篇 reference）：何时跑哪一步、期望输出、哪些失败是仓库自带而非环境问题。安装方式与逐项速查（适用场景 / 使用方法 / 已验证效果 / 来源 / 失效条件）见 `skill/README.md`。

## 许可证与第三方声明

- 本项目采用 [MIT License](LICENSE)。
- `src/software/apps/coremark/vendor/` 内 CoreMark 源码**原样引用、未作修改**；CoreMark® 是 EEMBC® 注册商标，使用受 EEMBC Acceptable Use Agreement 约束，不得在修改版本上使用该商标。
- MNIST 为 Yann LeCun 等维护的手写数字数据集；Vivado、Artix-7 等为 AMD/Xilinx 商标。
- 完整第三方声明见压缩包内 `notice.md`。
