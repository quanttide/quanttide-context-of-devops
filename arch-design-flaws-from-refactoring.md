# 从数据模型下沉重构中暴露的 CLI 架构设计缺陷

> 基于 `build::CiRun`、`test::{TestSummary,Coverage,AuditReport,is_io_fn}` 下沉到 toolkit 的过程反思。

---

## 缺陷 1：数据模型与 I/O 方法强耦合

**表现**：`RepoState`、`Submodule`、`SubmoduleStatus` 等结构体无法单独移动到 toolkit，因为它们的 `impl` 块和 I/O 方法（`scan()`、`scan_offline()`、`determine_submodule_status()`）在同一文件中定义，共用私有辅助结构体（`SubmoduleState`）。

```
// submodule.rs 1112 行 — 没有分层
pub struct RepoState { ... }        // 数据模型
impl RepoState {
    pub fn scan() -> ... { ... }    // I/O 操作
    fn determine_submodule_status()  // 内部辅助
}
```

**后果**：即使 `RepoState` 本身只有 5 个字段的纯数据，也无法剥离出来复用。

**根因**：没有遵循「数据-操作分离」原则。数据模型应定义在 toolkit，I/O 操作应作为 toolkit 中的 trait 或 A 中的独立函数。

---

## 缺陷 2：`pub(crate)` 可见性过度限制

**表现**：`CiRun` 定义为 `pub(crate)`，`is_io_fn` 定义为 `pub(crate)`。下沉时需要改为 `pub` 才能被 toolkit 导出。

```rust
// 修改前
pub(crate) struct CiRun { ... }
pub(crate) fn is_io_fn(name: &str) -> bool { ... }

// 修改后
pub use quanttide_devops::stage::build::CiRun;  // 从 B 导入
pub use quanttide_devops::stage::test::is_io_fn; // 从 B 导入
```

**根因**：默认使用 `pub(crate)` 而非 `pub`，假设代码不会在 CLI 之外被复用。但 toolkit 的设计目标恰恰是复用。应默认 `pub`，只在确实需要隐藏实现细节时才用 `pub(crate)`。

---

## 缺陷 3：CLI 自包含领域逻辑，toolkit 沦为"被挑选的工具箱"

**表现**：CLI 的 `lib.rs` 暴露 10 个顶层模块：

```
build / code / contract / doctor / plan / platform / python / release / source / test
```

但 toolkit 只有 3 个：`contract / source / stage`。

下沉过程中，CLI `test/` 模块中约 30% 的代码（`TestSummary`、`Coverage`、`AuditReport`、`is_io_fn`）被证明是纯领域逻辑 → 可以移到 toolkit。剩下 70%（`audit.rs` 的扫描逻辑、`coverage.rs` 的解析器、`run.rs` 的命令编排）理论上也应该移，只是受限于「不处理 IO」而未执行。

**根因**：开发时默认「写在 CLI 里，以后需要再抽」。这是 YAGNI 的正确使用方式，但缺少定期审查和下沉节奏。CLI → toolkit 的单向依赖应该倒逼：发现可复用代码 → 立即下沉 → CLI 消费。

---

## 缺陷 4：`stage/` 模块命名不统一，容量不均

**表现**：toolkit 的 `stage/` 原本只有 `release.rs`，但 CLI 把 stage 拆成了三个顶层模块 `build/`、`test/`、`release/`。本次下沉后 toolkit 的 `stage/` 补齐了 `build.rs` 和 `test.rs`，但 CLI 侧的结构并未同步调整。

```
Toolkit (B) stage/        CLI (A)
───────────────           ────────
stage/build.rs   ────→    src/build/     (顶层模块)
stage/test.rs    ────→    src/test/      (顶层模块)
stage/release.rs ────→    src/release/   (顶层模块)
```

**根因**：CLI 的模块结构早于 toolkit，先按 CLI 命令组织，后试图对齐 toolkit 但未完成重构。按 stage 组织的命令（build/test/release）在 CLI 是同级模块，在 toolkit 是 stage 的子模块。

---

## 缺陷 5：缺少"默认可下沉"的模块组织策略

**表现**：如果从项目开始就遵循「领域定义在 toolkit，编排在 CLI」的分层，那么：

- `build::CiRun` 应直接定义在 toolkit，CLI 只消费
- `test::is_io_fn` 的模式匹配列表应是 toolkit 的公开配置
- `source::git::submodule` 中的 `SubmoduleStatus` 类型应属于 toolkit

但实际上这些类型在 CLI 中"自然生长"，直到重构时才被发现应该下沉。

**根因**：没有建立「模块创建时的问询机制」——"这段代码是否有 CLI 以外的复用者？""这个类型是否描述了 DevOps 领域概念而不是 CLI 交互细节？"

---

## 缺陷 6：toolkit 版本滞后于 CLI，语义脱节

**表现**：A v0.11.0 vs B v0.3.3。CLI 的 major-minor 远高于 toolkit，暗示 toolkit 不是 CLI 的"基础库"，而是被"附带更新"的模块。

```
CLI  v0.11.0 → 功能丰富，频繁发布
TK   v0.3.3  → 长期未发布新版本
```

但实际上 CLI 依赖 toolkit 的核心模型（`Contract`、`Stage`、`Source`、`VersionState`），版本落差说明 toolkit 没有跟随 CLI 的开发节奏更新。

**根因**：缺少 toolkit 的发布节奏约束。每次 CLI 增加领域逻辑时，应先或同步更新 toolkit，再消费新版本。

---

## 从缺陷中提炼的架构原则

1. **数据模型优先在 toolkit 定义**：新类型第一个文件写在 toolkit 中，除非有明确理由（如与 I/O 紧密绑定）才写在 CLI 中。
2. **pub 优先于 pub(crate)**：默认外部可见，需要隐藏时再收紧。
3. **数据与 I/O 分离**：struct/enum 在 toolkit，I/O 操作作为 toolkit trait 实现或 CLI 自由函数。
4. **CLI 模块结构对齐 toolkit stage**：`src/build/` 领域逻辑逐渐下沉到 `stage/build.rs` 后，CLI 的 `src/build/` 只保留 `status/clean/audit` 等编排函数。
5. **版本号同步**：CLI 发布前先发布 toolkit 新版本（如果 toolkit 有变更），确保语义一致。

---

*生成日期：2026-07-16*
*信息来源：git 模块拆分 + 数据模型下沉 + 子模块模型剥离失败的实践反思*
