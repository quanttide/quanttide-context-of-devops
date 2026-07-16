# 实现 `code status v2` 暴露的架构缺陷

> 分析：以 v2 设计为"探针"，反推当前架构中阻碍实现的固有缺陷。

---

## 缺陷 1：没有 Cargo.toml 解析抽象

**症状**：`deps` section 需要解析多个 scope 的 `Cargo.toml` 获取依赖信息，但当前代码中 Cargo.toml 的解析方式是**人工逐行扫描字符串**（`build/mod.rs::check_dependencies()`）：

```rust
// 当前做法——逐行匹配 [dependencies] 段落
for line in content.lines() {
    let t = line.trim();
    if t.starts_with('[') { in_deps = ... ; continue; }
    if t.contains("path = \"") { issues.push("path"); }
}
```

**根源**：没有一个 `source::manifest` 模块。`cargo metadata` 的输出是 JSON，有标准 schema，但没有任何一层封装它。`deps` section 不仅要读 Cargo.toml，还要关联 `cargo metadata` 的输出、映射 workspace members 到 scope、检测循环依赖——当前架构假设"每个模块自己处理文件解析"。

**v2 需要的**：
- `Manifest::from_path(path)` — 解析单个 Cargo.toml
- `Workspace::from_metadata(repo_path)` — 封装 `cargo metadata` 输出
- `Manifest::shared_dependencies(&self, other: &Manifest)` — 比较跨 scope 的依赖版本

---

## 缺陷 2：Scope 迭代逻辑重复 6 次

**症状**：遍历 scope 的逻辑在代码库中重复了至少 6 次，每次都自己写相同的"有 scope 就遍历，没有就用 root"模式，但互不共享：

| 位置 | 迭代方式 | 是否处理空 scope |
|------|---------|-----------------|
| `build/status.rs` | `for scope in &c.scopes` + `append_root_status` | ✅ |
| `test/audit.rs` | `if all { ... } else { auto-detect cwd }` | ✅ |
| `release/audit.rs` | `audit_all` 内部迭代 | ✅ |
| `release/detect.rs` | `detect_single_scope` | ✅ |
| `code/audit.rs` | `walk_scope_files` | ✅ 但用 `c.scopes` 遍历 |
| `build/mod.rs` | `ScopeInfo` struct（仅被 build/ 使用） | ✅ 但不通用 |

**根源**：没有一个 `ScopeIter` 或 `for_each_scope()` 公共函数。每个模块自己决定遍历策略（all、by cwd、by name filter）。当 v2 的 `code status` 需要同时遍历 scope 做三件事（读取 Cargo.toml、对比版本、构建依赖图）时，重复的迭代逻辑会迫使你选择：写第四个重复版本，还是重构出一个共享迭代器——后者牵涉到改动 6 个模块的调用方。

**v2 需要的**：
```rust
// 一个共享的 scope 迭代器，所有模块复用
pub struct ScopeIter<'a> {
    contract: &'a Contract,
    strategy: IterStrategy,  // All, ByCwd, ByName(&str)
    context: &'a Path,
}

impl ScopeIter {
    pub fn scopes(&self) -> Vec<ScopeContext>;
}

pub struct ScopeContext<'a> {
    pub scope: &'a Scope,
    pub dir: PathBuf,
    pub manifest: Manifest,  // 预加载的 Cargo.toml
}
```

---

## 缺陷 3：输出抽象不一致 —— 3 种风格并存

**症状**：当前有三种输出方式混用：

| 风格 | 使用方 | 可测试 |
|------|--------|--------|
| `status_to(writer)` | `build/status`、`release/status`、`contract/status` | ✅ 可截获输出断言 |
| `println!` 散落在模块内 | `code/audit`、`test/audit`、`doctor` | ❌ |
| `println!` 在 `main.rs` | `code/status`（`print_report`）、`plan/` | ❌ |

`code status` 的输出函数 `print_report()` 在 `main.rs` 里，不在 `code/status.rs` 里。这意味着：

- 无法单元测试输出格式
- 无法复用 render 逻辑
- 添加 v2 的四个 section 渲染会导致 `main.rs` 继续膨胀

**根源**：`code/` 模块一直没有采用 `status_to(writer)` 模式。早期的 `code/status.rs` 设计选择了"返回到 main.rs 再渲染"，而这个设计从未被修正。

**v2 需要的**：要么把所有 `print_report` 从 main.rs 移回 `code/status.rs` 并统一为 `status_to`，要么为 `code/` 设计全新的输出抽象（比如 Renderer trait）。不管选哪条路，都涉及跨文件重构。

---

## 缺陷 4：没有持久化层

**症状**：`health` section 需要存储 audit 历史来计算趋势。但当前代码没有通用的持久化机制：

- `test/` 模块有自己的 `summary.rs` 缓存（`.quanttide/devops/test-summary.json`），但它在 `test/` 内部，不是共享模块
- 其他模块（`code/audit`、`build/audit`）跑完结果直接丢弃到 stdout
- 没有一个 `store/` 或 `persist/` 模块定义"谁可以存什么、存到哪里"

**根源**：架构从设计之初就没有为"审计历史"做存储预留。每个 stage 的 audit 都是瞬时计算 + 终端输出，没有考虑"上次的结果去哪了"。

**v2 需要的**：
```rust
// 需要一个通用的存储抽象
pub trait SnapshotStore {
    fn save(&self, key: &str, snapshot: &AuditSnapshot) -> Result<()>;
    fn load_latest(&self, key: &str, n: usize) -> Result<Vec<AuditSnapshot>>;
}
```
以及一个明确的存储位置决策——现有 `data/history/` 是 git 管理的（适合归档），但审计快照应该是机器生成的中间产物。

---

## 缺陷 5：错误类型不统一 —— `code/` 还是 bare `String`

**症状**：`code/status.rs` 返回 `Result<StatusReport, Box<dyn std::error::Error>>`——这是整个代码库里最粗糙的错误类型。其他模块：

| 模块 | 错误类型 | 严重程度 |
|------|---------|---------|
| `release/detect.rs` | `DetectError`（thiserror，3 个变体） | 🟢 好 |
| `plan/mod.rs` | `PlanError`（thiserror，2 个变体） | 🟢 好 |
| `code/status.rs` | `Box<dyn Error>` | 🔴 粗放 |
| `code/audit.rs` | `Vec<RuleResult>`（无错误返回） | 🟡 无错误路径 |
| `test/audit.rs` | `Result<(), String>` | 🟡 无类型 |
| `build/audit.rs` | 全 `bool`（完全不返回错误） | 🟡 丢失信息 |

v2 的 `deps` section 要调用 `cargo metadata`（可能失败）、解析 JSON（可能失败）、遍历 20+ scope 的 Cargo.toml（可能部分失败）。如果没有合适的错误类型，v2 会陷入两种烂选择之一：要么用 `String` 一路 propagate 丢失上下文，要么写一个 10 变体的巨大 enum 跟谁都不兼容。

**根源**：早期对 `code/` 模块的错误处理没有要求。`Box<dyn Error>` 是"先跑起来再说"的遗留债务。

**v2 需要的**：`CodeStatusError`（thiserror），且与 `PlanError`/`DetectError` 等有清晰的转换关系。

---

## 缺陷 6：`contract` 是被动的数据包，不是主动的查询层

**症状**：`contract::Contract` 是一个纯数据结构——`load()` 读 YAML 反序列化，然后就放在那里。没有查询方法。

当前如果有人想知道"所有 Rust scope 的公共依赖有哪些版本漂移"，他得：
1. 自己遍历 `c.scopes`
2. 过滤 `language == "rust"`
3. 对每个 scope 读 Cargo.toml
4. 手动对比版本

这些步骤**完全在 contract 之外做**。v2 的 `consistency` section 需要 3 种不同的跨 scope 对比（Rust 版本、公共依赖、CI 模板），每种都意味着写一遍 "遍历 → 过滤 → 读取 → 对比" 的模板代码。

**根源**：`contract` 模块的设计思路是"描述项目结构"，而不是"回答关于项目结构的问题"。缺少对 scope 的查询 API：

```rust
// 当前：被动数据
let c = contract::load(path);
c.scopes.iter().filter(|s| s.language == "rust")...

// 需要的：主动查询
c.scopes_by_language("rust")
c.shared_dependencies()  // 跨 scope 对比后返回
c.version_consistency()  // 直接给出结果
```

---

## 缺陷 7：`Cargo.toml` 不是一等公民

**症状**：`Cargo.toml` 在整个代码库里被当作"其中一个配置文件"，跟 `package.json`、`pyproject.toml` 同级别（`contract/source.rs::detect_by_files`）。但在 monorepo + Rust 的语境下，Cargo.toml 承载着**依赖图**、**workspace 结构**、**版本信息**三种职责，远不止"标志某种语言"。

当前对 Cargo.toml 的处理：
- `source::changelog` 完全不管 Cargo.toml
- `contract::version` 只读 `version` 字段
- `build::check_dependencies` 只检查 path/git 依赖
- 没有任何代码读 `[workspace.members]` 或 `[workspace.dependencies]`

**根源**：从 `detect_by_files` 的设计可以看出，Cargo.toml 被当作"语言检测器"而非"项目模型的入口"。v2 的依赖图需要在 workspace 层面理解成员关系（`cargo metadata --no-deps` 返回的 resolve 图），这是当前模型完全没有覆盖的维度。

---

## 缺陷 8：没有"部分失败"的处理模式

**症状**：v2 的 `code status` 有 4 个 section，每个可能独立失败：
- `sync`：总是成功（git 本地操作）
- `deps`：可能因 `cargo metadata` 缓存未就绪而失败
- `consistency`：可能因某 scope 目录不存在而部分失败
- `health`：可能因无历史记录而返回"未知"

但当前代码中的 status 命令都是"全有或全无"——要么全部成功输出，要么 `Err`。没有任何模式表达"部分 section 可用"。

**根源**：`Result<T, E>` 是二值的（成功/失败），但 v2 的输出是**多值**的（每个 section 有自己的可用状态）。需要类似 `PartialResult<Section>` 的模式。

**v2 需要的**：不是简单的 `Result` 链，而是一种能表达"sync=可用, deps=失败, consistency=可用, health=未知"的数据结构。

---

## 缺陷 9：缺乏输出格式化基础设施

**症状**：v2 需要输出依赖树（缩进、箭头、分支）、一致性对比表格、趋势方向箭头。当前代码库的输出能力：

- `code/status.rs`：简单的 `println!("  {:<20} {}{}", name, status, detail)`
- `code/audit.rs`：简单的 `println!("  {} {}", icon, detail)`
- `build/status.rs`：`format!()` 拼接字符串
- 没有任何模块支持**树状渲染**、**表格渲染**、或**颜色输出**

**根源**：CLI 的输出一直只用终端文本，没有考虑结构化展示。v2 的依赖图需要用树形结构展示（如 `tree` 命令风格），这部分能力完全不存在。

---

## 总结：缺陷关系图

```
缺陷 1（无 Cargo.toml 解析）
    └→ 缺陷 7（Cargo.toml 非一等公民）
        └→ 阻碍 deps section

缺陷 2（Scope 迭代重复）
    └→ 阻碍 consistency section
    └→ 阻碍 deps section

缺陷 3（输出抽象不一致）
    └→ 缺陷 9（无格式化基础设施）
        └→ 阻碍所有 section 的高质量输出

缺陷 4（无持久化层）
    └→ 阻碍 health section

缺陷 5（错误类型不统一）
    └→ 缺陷 8（无部分失败模式）
        └→ 阻碍 4-section 并行

缺陷 6（contract 被动）
    └→ 阻碍 consistency section
```

核心矛盾：**当前架构是按"单一职责命令"设计的（每个命令做一件事），但 v2 的 `code status` 是一个"聚合视图"（一个命令做四件事并关联它们）**。聚合视图需要共享基础设施（scope 迭代器、manifest 解析、输出渲染、持久化），而这些基础设施在当前架构中要么不存在，要么被重复发明。
