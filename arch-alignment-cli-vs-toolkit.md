# 架构对齐度分析：qtcloud-devops-cli × quanttide-devops-toolkit

> 分析范围：
> - A: `apps/qtcloud-devops/src/cli`（v0.11.0）
> - B: `packages/quanttide-devops-toolkit/packages/rust`（v0.3.3）
>
> 关系：CLI 通过 Cargo dependency `quanttide-devops = { path = "..." }` 依赖 toolkit。

---

## 1. 整体架构定位

| 维度 | CLI (A) | Toolkit (B) | 对齐度 |
|------|---------|-------------|--------|
| 角色 | 可部署二进制 + CLI 库（含 PyO3 binding） | 领域模型库（纯） | 互补 |
| 用户 | 终端用户、CI 流水线 | CLI 开发者、其他消费方 | — |
| 顶层模块 | `10 个`：build / code / contract / doctor / plan / platform / python / release / source / test | `3 个`：contract / source / stage | 不对齐，A 模块粒度过细 |
| 文档体系 | `19 篇`（architecture.md + 各模块文档） | `4 篇`（contract + 3 source 文档） | B 远不如 A 丰富 |

**结论**：角色互补但不是分层清晰的关系。A 直接依赖 B，但大量 A 独有逻辑未下沉到 B。

---

## 2. 核心对齐：contract 模块

### 2.1 模块映射

```
Toolkit (B)                      CLI (A)                           对齐方式
─────────────────────────────────────────────────────────────────────────────
src/contract/                   src/contract/                      适配层
├── core.rs  (Contract 结构体, auto_detect, validate)    ├── core.rs (load, status)   A 加 display
├── error.rs (ContractError)   ├── (隐含在 mod.rs)                 A 未独立暴露 error
├── platform.rs (枚举类型)     ├── platform.rs (pub use)           A 纯 re-export
├── scope.rs (Scope 结构体)    ├── scope.rs (pub use + load_scopes) A re-export + 加包装
├── source.rs (Source 枚举)    ├── source.rs (pub use + detect)    A re-export + 加 detect_by_files
├── stage.rs (Stage 枚举)      ├── stage.rs (pub use)              A 纯 re-export
├── version.rs (版本号工具)    ├── version.rs (pub use + status)   A re-export + 加 version_status
└── mod.rs (load, load_from_str) └── mod.rs (适配说明)
```

### 2.2 评价

- **对齐度：高（90%）**。A 的 `contract/` 全部是薄适配层（re-export + 少量 display/fallback 逻辑）。
- 偏离：A 的 `contract/core.rs` 中的 `status()` / `status_to()` 含有大量显示格式逻辑，这些是 presentation 层职责，不应在 toolkit 里。
- 偏离：A 不直接暴露 `ContractError`，而是在包装函数中静默 fallback。

---

## 3. 部分对齐：source 模块

### 3.1 模块映射

```
Toolkit (B)                      CLI (A)
─────────────────────────────────────────────
src/source/                     src/source/
├── mod.rs                      ├── mod.rs
├── changelog.rs                ├── changelog.rs
├── config_file.rs              ├── (CLI 通过 contract/source.rs 使用)
├── git/                        ├── git/
│   ├── mod.rs                  │   ├── mod.rs  (git/git_check + re-export)
│   ├── repo.rs                 │   ├── repo.rs  (is_git_repo + ref_exists)
│   └── tag.rs                  │   ├── status.rs (is_working_tree_dirty)
├── roadmap.rs                  │   ├── log.rs (rev_list_count + parse_commit_messages)
                                │   ├── diff.rs (get_changed_paths_since_last_tag)
                                │   ├── tag.rs (create_tag + parse_tag + collect_tags)
                                │   └── submodule.rs  (B 中无对应)
                                └── roadmap.rs
```

> **2026-07-16 更新**：双方统一了 `git/` 目录结构。B 从 `git_repo.rs`/`git_tag.rs` 移入 `git/repo.rs`/`git/tag.rs`。A 同步调整 import 路径，并将 `git_tag.rs` → `git/tag.rs`、`git_submodule.rs` → `git/submodule.rs`。A 的 `doctor`（原 `diagnostics`）命令名也已完成替换。

### 3.2 偏离项

| 偏离 | 说明 | 严重度 |
|------|------|--------|
| `git/` 结构一致但内容不同 | B 的 `git/` 有 2 个文件（repo/tag），A 有 6 个（mod/repo/status/log/diff/submodule/tag）。命名对齐了，但功能量差 3 倍。 | 信息差 |
| `submodule.rs` | A 独有，用于 code/status 的子模块扫描，B 无子模块支持。 | 信息差 |
| `config_file.rs` 下沉 | B 的 `detect_languages` / `read_config_versions`，A 通过 `contract/source.rs` 间接使用。 | 良好 |
| `changelog.rs` | A 的 `release/mod.rs` 直接调用 `quanttide_devops::source::changelog::Changelog`。 | 良好 |
| `roadmap.rs` | A 的 `plan/` 通过 `use quanttide_devops::source::roadmap::RoadmapError` 使用 B 的 Roadmap 类型。 | 良好 |

### 3.3 评价

- **对齐度：中偏高（70%）**。changelog 和 roadmap 复用得好。git 目录结构已统一（双方 `source/git/`），但 B 功能量不足。子模块是 A 独有。

---

## 4. 未对齐：stage 和 CLI 独有模块

### 4.1 stage 模块

Toolkit 的 `src/stage/` 只有 `release.rs`（`GixTagSource`、tag 过滤/排序/创建/推送）。CLI 完全没有 `stage/` 模块，而是把 stage 拆成了顶层模块：

```
CLI (A)                        Toolkit (B)
─────────────────────────────────────────
src/build/    (status/clean/audit)    src/stage/ 不存在 build.rs
src/test/     (status/run/coverage/audit)   src/stage/ 不存在 test.rs
src/release/  (status/audit/detect/precheck/publish)   src/stage/release.rs
```

**对齐度：低（30%）**。

- Toolkit 的 `stage/` 只实现了 release，且 CLI 并未完全复用——CLI 的 `release/publish.rs` 直接调用 git2，而不是 toolkit 的 `Publish`。
- CLI 的 `build/` 和 `test/` 含大量可复用领域逻辑（language-specific check commands、覆盖率阈值、CI 运行记录）却未下沉。

### 4.2 CLI 独有模块（B 无对应）

| 模块 | 行数（估算） | 逻辑内容 | 是否应下沉 |
|------|-------------|----------|-----------|
| `build/` | ~300 | CiRun / ScopeInfo / check_command / check_manifest / check_dependencies / resolve_workflow | ✅ 可复用 |
| `test/` | ~400 | TestSummary / Coverage / AuditReport / IO_FN_PATTERNS | ✅ 可复用 |
| `plan/` | ~600 | ROADMAP/TODO 管理：status / clean / doctor / audit / from_audit | ⚠️ 部分复用 roadmap.rs |
| `code/` | ~400 | 子模块同步扫描/审计 | ❓ 子模块是 A 特有 CLI 功能？ |
| `platform/` | ~200 | GitHub CLI 集成（gh） | ❓ CLI 特有 |
| `doctor.rs` | ~200 | 系统命令检测（原 `diagnostics.rs`） | ❓ CLI 特有 |
| `python.rs` | ~100 | PyO3 binding | ✅ 可作为 feature 下沉 |

### 4.3 评价

CLI 的大量领域逻辑没有下沉到 toolkit，导致：
1. 其他 consumer（如 Web UI、CI 插件）无法复用。
2. toolkit 的 `stage/` 名不副实（只装了 release）。
3. toolkit 的版本号与 CLI 的依赖版本相差较大（B 0.3.3 <-> A 0.10.2）。

---

## 5. 依赖关系分析

### 5.1 共同依赖

| 包 | B 版本 | A 版本 | 一致？ |
|-----|--------|--------|--------|
| gix | 0.69 | 0.66 | ❌ 差 3 个小版本 |
| git2 | 0.19 (vendored-libgit2) | 0.19 | ✅ |
| semver | 1 | 1 | ✅ |
| serde | 1 | 1 | ✅ |
| jiff | 0.2 | 0.1 | ❌ 差 1 个大版本 |

### 5.2 独有依赖

| B 独有 | A 独有 |
|--------|--------|
| serde_yaml 0.9 | clap 4（B 作为 optional） |
| parse-changelog 0.6 | serde_json 1 |
| | thiserror 2 |
| | pyo3 0.23（optional） |
| | quanttide-agent 0.1 |
| | tempfile 3（dev） |

### 5.3 特征重叠

B 有 `clap` 作为 optional feature，但 A 并未启用它（自己直接依赖 clap 4）。这表明 B 的 clap feature 可能是为其他 consumer 准备的，而非专门为 A 设计。

---

## 6. 文档体系对比

| 维度 | CLI (A) | Toolkit (B) |
|------|---------|-------------|
| 根 README | ✅ | ✅ |
| CONTRIBUTING | ✅ | ✅ |
| CHANGELOG | ✅ | ✅ |
| ROADMAP | ✅ | ✅ |
| docs/architecture.md | ✅（status/audit/action 三分类） | ❌ |
| docs/contract/ | 5 篇（model, config, scope, version, auto-detect, monorepo） | 1 篇（contract.md） |
| docs/source/ | 1 篇（status） | 3 篇（changelog, roadmap, submodule） |
| docs/release/ | 3 篇 | ❌ |
| docs/build/ | 1 篇 | ❌ |
| docs/test/ | 3 篇 | ❌ |
| docs/plan/ | 1 篇 | ❌ |

A 的文档体系是 B 的 5 倍。B 严重缺乏文档。

---

## 7. 测试对齐

| 维度 | CLI (A) | Toolkit (B) |
|------|---------|-------------|
| 单元测试 | 各模块 `#[cfg(test)]` | 各模块 `#[cfg(test)]` |
| 集成测试 | `tests/cli.rs`, `tests/code.rs`, `tests/release.rs` | `tests/contract_*.rs`, `tests/source_*.rs` |
| 测试文件组织 | 3 个集成测试文件 | 6 个集成测试文件 |
| examples | 无 | 6 个（contract/source*/stage_release） |
| lcov 覆盖报告 | ❌ | ✅（lcov.info） |
| 覆盖率目标 | 未声明 | > 95%（文档声明） |

B 的测试体系更完善，有 examples 和覆盖率追踪。A 缺乏系统性的测试策略。

---

## 8. 综合评分

| 维度 | 对齐度 | 说明 |
|------|--------|------|
| contract 模块 | 🟢 高（90%） | 薄适配层 + re-export，模型由 B 定义 |
| source 模块 | 🟡 中偏高（70%） | git 目录统一了，但 B 功能量不足；子模块 A 独有 |
| stage 模块 | 🔴 低（30%） | B 只有 release，A 的 build/test/release 全在 CLI 层 |
| CLI 独有逻辑下沉 | 🔴 极低（15%） | build/test/plan 中大量逻辑未下沉 |
| 依赖一致性 | 🟡 中（50%） | gix/jiff 版本不一致，serde 系一致 |
| 文档体系 | 🔴 低（20%） | B 远不如 A 丰富 |
| 测试体系 | 🟡 中（50%） | B 更好但 A 有集成测试 |

**整体对齐度：48%**

---

## 9. 建议

### 立即（低风险）

1. ✅ **统一 git 目录结构**：双方 `source/git/` 已完成。
2. **统一 gix 版本**：B 0.69 → A 0.66，至少对齐到相同 major/minor。
3. **统一 jiff 版本**：B 0.2 → A 0.1 或 A 升到 0.2。
4. **contract 保持 consistent**：A 的 `contract/` 保持薄适配层，不做额外逻辑。

### 短期（中风险）

5. **下沉 build 逻辑**：将 `build/mod.rs` 中的 `ScopeInfo`, `check_command`, `check_manifest_file`, `check_dependencies`, `resolve_workflow` 下沉到 B 的 `stage/build.rs`。
6. **下沉 test 逻辑**：将 `test/mod.rs` 中的 `TestSummary`, `Coverage`, `AuditReport`, `is_io_fn` 下沉到 B 的 `stage/test.rs`。

### 中期（高风险）

8. **填补 B 的文档空白**：至少补充 architecture.md, stage 说明, 各模块 README。
9. **评估子模块逻辑定位**：`source/git/submodule.rs` 是否应下沉到 B，取决于子模块操作是否算「DevOps 基础能力」。
10. **建立「status→audit→action」不变量契约**：将 A 的 `docs/architecture.md` 中的三分类原则写入 B，作为 toolkit 的架构约束。

---

*生成日期：2026-07-16*
*信息来源：代码审查 + 文件结构对比 + 文档分析*
