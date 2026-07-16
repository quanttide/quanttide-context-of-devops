# CLI 架构师视角评估

> 来源：2026-07-16 架构师对话，对 qtcloud-devops-cli 的能力评估

## 一、背景

从架构师视角审视 qtcloud-devops-cli 的能力全景，识别"能做"和"缺什么"。CLI 定位是 DevOps 管道的视图层 + 操作层，服务于量潮单仓子模块管理。

## 二、能做到的

### 每日全局健康状态

```bash
qtcloud-devops status
```

一行命令聚合：子模块同步状态、构建状态、测试状态、发布状态、契约配置、环境工具链。不需要逐个仓库 `git status`。

### 安全发布流水线

```bash
qtcloud-devops release audit -v v0.2.0    # 预检
qtcloud-devops release publish -v v0.2.0 -y  # 发布
```

五阶段门禁流水线（detect → plan → precheck → audit → publish）保证版本号、CHANGELOG、tag 冲突、远程可达性都在创建 release 前检查完毕。最怕的"版本号忘了改"或"CHANGELOG 没写"场景被 gate 住了。

### 代码质量基线审计

```bash
qtcloud-devops code audit
```

9 条规则一轮扫描：scope 目录完整性、TODO 密度、函数长度、API 文档覆盖率、嵌套深度、圈复杂度、import 数、模块文档、语法检查。适合 CI 门禁。

### 测试质量审计

```bash
qtcloud-devops test audit --all -v
```

函数级覆盖率和错误变体覆盖率的交叉检查——不只看行覆盖率，还看"每个 pub 函数有没有被调用"、"每个 Error 变体有没有被测到"。比行覆盖率更严格的维度。

### ROADMAP 演化管理

```bash
qtcloud-devops plan status        # 看当前进度
qtcloud-devops plan clean          # 清理已完成条目
qtcloud-devops plan doctor         # 修复格式
```

ROADMAP 不只是一份文档，是可操作的规划——进度跟踪、格式校验、自动化清理。架构师不用手动编辑 markdown chore。

### 项目结构理解（契约）

```bash
qtcloud-devops contract status
```

scope 边界、语言、CI 配置、版本一致性一览。这是架构师问得最多的问题之一。

---

## 三、做的不够好的

### 3.1 没有架构依赖图

Monorepo + submodules，但没法问 CLI 类似这样的问题：

- "模块 A 依赖了模块 B 的哪些接口？"
- "如果重构 `contract::Contract`，哪些模块会受影响？"
- "代码里的循环依赖在哪里？"

现有的 `code audit` 是单文件级别的规范检查，不是架构级别的依赖分析。这是架构师最需要的功能之一。

### 3.2 Audit 规则不可按 scope 定制

`code audit` 的 9 条规则对所有 scope 一视同仁。但实际上：

- 工具库 scope 应有更高的 API 文档覆盖率要求（如 90%）
- 应用层 scope 的函数长度阈值可以更宽松
- 某些 scope 可能不需要 unsafe 检查（非 Rust）

契约（`contract.yaml`）已有 scope 定义，但 audit 规则没有利用这个分层信息。

### 3.3 没有跨 scope 一致性检查

Monorepo 的核心架构问题——一致性，没有被覆盖：

- 所有 scope 用同一版本的 formatter 配置吗？
- lint 规则一致吗？
- CI 模板一致吗（还是每个 scope 自己写了一份）？
- 依赖版本有没有漂移（A scope 用 serde 1.0，B scope 用 serde 2.0）？

目前只有版本号一致性检查（`release::audit` 检 tag/config 对齐），但工具链一致性和依赖版本一致性没有。

### 3.4 审计结果没有趋势和基线

`code audit` 跑完就输出到终端，没有存历史。架构师想知道：

- 本周的 TODO 密度比上周高了还是低了？
- function length violation 在变多还是变少？
- 覆盖率是上升还是下降趋势？

没有 dashboard，没有趋势线，只有 point-in-time 快照。

### 3.5 没有"架构规则"的声明式配置

架构师想声明约束让 CI 持续检查，比如：

- `code/` 模块不能依赖 `plan/` 模块（分层约束）
- 只有 `contract/` 模块可以 import `quanttide-devops` 这个外部 crate（适配层隔离）
- 所有 `pub fn` 必须有文档注释
- 所有 Error 类型必须实现 `std::error::Error`

`contract.yaml` 现在只声明 scope 边界和构建配置，没有声明模块间的依赖规则。这部分是架构师最关心的"防腐层"和"依赖方向"，CLI 完全没有覆盖。

### 3.6 Plan 和 Audit 之间的闭环不够完整

`plan todo-from-audit` 能从 audit JSON 生成 TODO，这很好。但缺少：

- 自动关闭已完成的 TODO（需手动去 ROADMAP.md 打勾）
- Audit 重新跑完后自动更新 TODO 状态
- "建议优先级"的排序逻辑（`classify_priority` 是硬编码的 if-else）

### 3.7 没有"架构决策记录"集成

架构师做决策（ADR, Architecture Decision Record）是常见场景。CLI 没有：

- 创建 ADR 的模板命令
- 列出最近的 ADR
- 检查代码实现是否偏离了 ADR

---

## 四、成熟度总览

| 层面 | 成熟度 | 说明 |
|------|--------|------|
| **操作层**（发布、构建、测试） | 🟢 好 | Release pipeline 很成熟，门禁体系完整 |
| **监控层**（状态、审计） | 🟡 中等 | 点状覆盖好，缺趋势和聚合 dashboard |
| **架构层**（依赖、约束、一致性） | 🔴 缺乏 | 完全没有依赖分析、规则声明、一致性检查 |
| **治理层**（ADR、演进记录） | 🔴 缺乏 | 完全没有 |

## 五、核心差距

CLI **知道项目有哪些模块（scope），但不知道模块之间的关系（依赖）和约束（规则）**。补齐这两层，才能真正回答架构师的问题。
