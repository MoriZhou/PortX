# PortX

> **让 AI 在可验收的工程约束内，生成平台化、可落地、可生产、可测试的嵌入式软件。**

[![CI](https://img.shields.io/badge/CI-gate.sh-blue)](./ci/gate.sh)
[![Spec](https://img.shields.io/badge/spec-v0.1-green)](./portx-spec/)
[![License](https://img.shields.io/badge/license-Apache--2.0-lightgrey)](./LICENSE)
[![Status](https://img.shields.io/badge/status-MVP%20in%20progress-orange)]()

---

## 为什么需要 PortX

嵌入式软件用 AI 生成，目前有三个绕不过去的问题：

- **能编译，不能生产**：AI 生成的代码没有平台抽象，换块板子就废。
- **能运行，不能测试**：业务逻辑和寄存器操作缠在一起，主机上跑不起来。
- **能交付，不能追溯**：需求、代码、测试之间没有链路，出了问题查不到源头。

PortX 的答案是：**不要用文档去“要求”AI 遵守规范，而要用模板、契约和门禁去“强制”AI 只能产出合规代码。** AI 是受约束的生成器，不是架构师。

---

## PortX 是什么

PortX 不是一个代码生成器，而是一套 **AI 可执行的嵌入式工程体系**，由六部分组成：

| 组成 | 作用 |
|---|---|
| **规范与元模型** | 需求、接口、代码、测试、发布的唯一合法结构 |
| **平台抽象层** | HAL / OSAL / BSP，让上层代码与芯片无关 |
| **需求分析流水线** | 原始输入 → 澄清 → 可行性预检 → 人工确认基线 |
| **受控生成流水线** | 模板 + 任务卡 + AI 生成 + 自证报告 |
| **验证门禁** | 编译、静态分析、单测、分层检查、追溯检查 |
| **发布与追溯** | SBOM、版本包、需求↔代码↔测试追溯 |

最终产出满足四个目标：

| 目标 | 含义 |
|---|---|
| **平台化** | 芯片、RTOS、板级、通信、存储全部抽象为可配置能力 |
| **可落地** | 接入现有 GCC/CMake、链接脚本、烧录和调试流程 |
| **可生产** | 代码规范、静态分析、评审、版本锁定、SBOM |
| **可测试** | 接口契约、Mock、仿真、单测、HIL 全链路可执行 |

---

## 架构总览

```text
┌─────────────────────────────────────────────┐
│ Layer 0  需求分析层                          │  原始输入 → 需求基线
├─────────────────────────────────────────────┤
│ Layer 1  需求与配置层                        │  结构化需求 + 平台配置
├─────────────────────────────────────────────┤
│ Layer 2  PortX 元模型与规范层                │  portx-spec
├─────────────────────────────────────────────┤
│ Layer 3  生成器与模板层                      │  portx-templates / portx-gen
├─────────────────────────────────────────────┤
│ Layer 4  平台抽象与运行时层                  │  HAL / OSAL / BSP / services
├─────────────────────────────────────────────┤
│ Layer 5  验证与质量门禁层                    │  tests / ci/gate.sh
├─────────────────────────────────────────────┤
│ Layer 6  发布与追溯层                        │  SBOM / trace / package
└─────────────────────────────────────────────┘
```

统一入口是 `portx` CLI：

```text
原始输入 → req ingest → req clarify → req check → req baseline
        → plan → task run → build → test → verify → release
```

---

## 快速开始（30 分钟）

### 环境准备

```bash
# 安装 CLI
pip install portx-cli

# 验证
portx --version
# PortX v0.1.0
```

### 拉取示例项目

```bash
git clone https://github.com/portx/examples.git
cd examples/sensor_node_demo
```

目录结构：

```text
sensor_node_demo/
├── portx.yaml                        # 项目与平台配置
├── raw_input/                        # 原始输入（访谈/文档/截图）
│   └── customer_interview_2026-11-10.md
└── requirements/                     # 待生成
```

### 走完一条需求全流程

```bash
# 1. 采集原始输入
portx req ingest --source ./raw_input/

# 2. AI 生成澄清问题
portx req clarify --domain sensor_node
# → 生成 12 个问题，P0 必答 5 个
# 填写 answers.yaml 后：
portx req answer --file answers.yaml

# 3. 平台可行性预检
portx req check --feasibility --against board_stm32f4_freertos

# 4. 结构化 + 缺口检测
portx req structure --output requirements/
portx req check --gaps

# 5. 人工确认基线
portx req review
portx req baseline --version v1.0 --confirm

# 6. 拆任务并 AI 生成
portx plan --baseline requirement_baseline_v1.0.yaml
portx task run --all

# 7. 构建、测试、发布
portx build --target release
portx test --unit --mock
portx test --hil --board stm32f4_demo
portx verify --static --trace
portx release --sbom --package
```

30 分钟后，你会得到一个**可烧录、可追溯、有测试**的固件。

详见 [`docs/getting-started.md`](./docs/getting-started.md)。

---

## 核心概念

### 1. 需求基线（Requirement Baseline）

任何需求进入生成流程前，必须满足：

- 有**来源引用**（文件+页码/时间戳），防止 AI 幻觉；
- 有**可测量的验收标准**；
- 经过**平台可行性预检**（feasible / degraded / infeasible）；
- 经过**人工确认**并冻结版本。

> 无来源不立项，无验收标准不确认，无预检不生成。

### 2. 分层架构

```text
L4 应用层  app/          业务逻辑，不含任何硬件细节
L3 组件层  components/   可复用功能组件
L2 服务层  services/     OSAL、通信、存储、日志
L1 HAL层   hal/          芯片外设契约（纯头文件）
L0 BSP层   bsp/<platform> 平台实现
```

依赖方向单向：**上层可依赖下层，下层禁止依赖上层。** 由 `tools/check_layering.py` 自动检查。

### 3. 接口契约先行

平台化的本质是**接口稳定、实现可换**。`hal/include/*.h` 为受保护文件，AI 只能实现契约，不能修改契约。

### 4. 生成区 / 手写区隔离

```text
generated/      # AI 全权负责，可重新生成覆盖
src/            # 人工维护，AI 只读
  hal/include/  # 契约
  bsp/          # 平台实现
  app/          # 业务组装
```

AI 任务卡中写死 `allowed_paths` 和 `forbidden_paths`。

### 5. 质量门禁

每个 AI 产物必须通过 `ci/gate.sh` 才能合并：

```bash
== 1. 格式检查 ==      clang-format
== 2. 静态检查 ==      clang-tidy
== 3. 分层检查 ==      check_layering.py
== 4. 自定义规则 ==    check_portx_rules.py
== 5. host 编译 ==     可移植性试金石
== 6. 目标板编译 ==    交叉编译
== 7. 单测 + 覆盖率 == coverage >= 80%
== 8. 追溯检查 ==      portx verify --trace
```

**本地和 CI 共用同一脚本**，避免“本地过 CI 不过”。

---

## 仓库结构

```text
PortX/
├── portx-spec/          # 规范与元模型
│   ├── 00-principles.md
│   ├── 01-requirements/ # 需求元模型、七类模板、澄清问题库、预检规则
│   ├── 02-platform-model/
│   ├── 03-interface-contracts/
│   ├── 04-code-standard/
│   ├── 05-test-standard/
│   ├── 06-build-release/
│   ├── 07-safety-security/
│   ├── 08-ai-generation-rules/
│   └── 09-acceptance-gates/
├── portx-platform/      # HAL / OSAL / BSP / services
├── portx-templates/     # 组件、测试、任务卡模板
├── portx-ai/            # 提示模板、任务编排、自证报告
├── portx-cli/           # CLI 实现
├── portx-tests/         # 单测框架、Mock、HIL
├── portx-examples/      # 端到端示例
├── tools/               # 检查脚本（分层、规则、覆盖率）
├── ci/                  # gate.sh 门禁脚本
└── docs/                # 上手、培训、设计文档
```

---

## 规范体系索引

| 文档 | 内容 |
|---|---|
| [`00-principles.md`](./portx-spec/00-principles.md) | 设计原则、AI 边界、生成区规则 |
| [`01-requirements/`](./portx-spec/01-requirements/) | 需求元模型、七类模板、澄清问题库、可行性预检规则 |
| [`02-platform-model/`](./portx-spec/02-platform-model/) | 平台能力矩阵、组件清单 |
| [`03-interface-contracts/`](./portx-spec/03-interface-contracts/) | HAL/OSAL 契约、错误码、生命周期 |
| [`04-code-standard/`](./portx-spec/04-code-standard/) | 分层规则、命名、`.clang-format`、`.clang-tidy` |
| [`05-test-standard/`](./portx-spec/05-test-standard/) | 单测/Mock/HIL 规范、覆盖率门槛 |
| [`06-build-release/`](./portx-spec/06-build-release/) | 构建、烧录、版本、SBOM |
| [`07-safety-security/`](./portx-spec/07-safety-security/) | 看门狗、断言、安全启动、OTA |
| [`08-ai-generation-rules/`](./portx-spec/08-ai-generation-rules/) | 任务卡格式、提示模板、禁止事项 |
| [`09-acceptance-gates/`](./portx-spec/09-acceptance-gates/) | 每阶段门禁定义 |

---

## AI 任务卡示例

AI 的每次生成都由结构化任务卡驱动，而不是自由提示：

```yaml
task_id: PORTX-TASK-0001
component: temp_sampler
requirement_ids: [REQ-SENSOR-001]
template: portx-templates/component/
allowed_paths:
  - generated/components/temp_sampler/**
  - generated/config/**
forbidden_paths:
  - hal/include/**
  - bsp/**
  - CMakeLists.txt
rules:
  - 通过 clang-format / clang-tidy
  - 通过 check_layering.py / check_portx_rules.py
  - 单测覆盖率 >= 80%
  - host 与目标板均编译通过
acceptance:
  - bash ci/gate.sh
```

AI 产出必须附带 `GENERATION_REPORT.yaml`，列明所有检查项结果，否则 PR 不可合并。

---

## 路线图

| 里程碑 | 目标 | 状态 |
|---|---|---|
| **M0** 规范骨架 | A 的 P0 规范 + `hal/` 契约头文件 + `bsp/host` | 🚧 进行中 |
| **M1** 端到端 Demo | 自然语言 → 基线 → AI 生成 → 门禁绿 → 可烧录 | ⏳ 计划中 |
| **M2** 真实项目 | 用真实小项目跑完整流程 | ⏳ 计划中 |
| **M3** 平台可移植 | 第二个 BSP，验证上层零改动 | ⏳ 计划中 |
| **M4** 生产化 | SBOM、追溯报告、HIL、CI 全通 | ⏳ 计划中 |

**M1 是分水岭。** M1 之前不扩模块，M1 之后按 P1 优先级逐个补。

---

## 度量指标

PortX 是否真的成立，看这几个数：

| 指标 | 目标 |
|---|---|
| 需求基线可追溯率 | 100% must 需求有来源 + 验收标准 |
| **AI 生成一次通过门禁率** | **≥ 70%** |
| host 可编译率 | 100% 组件 |
| 换平台上层改动量 | 0 行业务代码 |
| 需求 → 代码 → 测试追溯覆盖 | 100% must 需求 |
| 新项目复制示例工时 | ≤ 2 人天 |

**最重要的一个指标：AI 生成一次通过率。** 它低说明模板和任务卡没做好，不是 AI 不行。

---

## 参与贡献

### 开发环境

```bash
git clone https://github.com/portx/portx.git
cd portx
pip install -e portx-cli
pre-commit install
```

### 提交前必须

```bash
bash ci/gate.sh        # 与 CI 完全一致
```

### 贡献方向

- **规范**：补充某个领域（电机、网关、BMS）的澄清问题库和预检规则
- **平台**：新增 BSP（nRF52 / ESP32 / 国产 MCU）
- **模板**：新增组件类型模板
- **工具**：新增 `tools/check_*.py` 检查规则
- **示例**：新的端到端示例项目

详见 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。

---

## 设计原则

1. **PortX 先定义“可验收的约束”，再让 AI 生成。**
2. **平台抽象、接口契约、测试替身必须先行。**
3. **每个 AI 产物必须通过构建、测试、静态分析和追溯检查，否则不算完成。**
4. **规范写在文档里是给人看的，写在 `.clang-tidy`、`check_*.py`、`gate.sh` 和模板里才是给 AI 用的。**
5. **不能用脚本检查的规范，只是建议。建议对 AI 无效。**

---

## License

[Apache-2.0](./LICENSE)

---

## 致谢

PortX 的设计吸收了 AUTOSAR 分层思想、MISRA/CERT 编码规范、以及现代 AI 辅助工程的实践。特别感谢所有在早期讨论中提出“AI 应该被关进笼子”这一共识的贡献者。
