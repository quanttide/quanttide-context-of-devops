# provider 与 studio 的产品定位澄清

> 来源：qtcloud-devops provider/studio 代码评审（2026-07-31 之后）
> 触发原因：评审发现 provider 无消费者、studio 绕开 provider 直连 CLI、
> 注释声称"对齐 qtcloud-secret"但职责实质不同——定位从未被正式记录。

## 一、现状事实（证据）

| 组件 | 语言 | 现状 | 证据 |
|------|------|------|------|
| `src/cli` | Rust | 唯一被文档化的产品：docs/index.md "封装量潮发布规范为可执行命令"；有 CHANGELOG、ROADMAP、CI（release-cli） | `apps/qtcloud-devops/docs/index.md` |
| `src/provider` | Go | HTTP 服务，仅封装 `code status` 为 `GET /api/scan` + `GET /health`；**无任何消费者**；无 README、无 CHANGELOG、无 CI、无 ROADMAP | `provider/cmd/server/main.go`、`.github/workflows/` |
| `src/studio` | Flutter | 桌面端直连 CLI（`Process.run qtcloud-devops code status`）；Web 端 stub 抛"仅支持桌面端"；无 CI | `studio/lib/api/cli_client_io.dart`、`cli_client_stub.dart` |

关键证据链：

1. **studio 不消费 provider**：`cli_client_io.dart` 直接 `Process.run` 调用 CLI，候选顺序为 PATH → 预构建 → cargo；provider 的 `/api/scan` 无人调用。
2. **provider 注释表述误导**：`provider/cmd/server/main.go` 注释"结构对齐 qtcloud-secret/src/provider"，而 cli CHANGELOG 称"参考 qtadmin 结构"——两处来源说法不一致。
3. **参考架构（qtcloud-secret）的 provider 是业务服务端**：有 JWT 认证、OSS 存储、加密、CORS 中间件（为浏览器 Web 客户端服务）；其 studio 通过 `provider_client.dart` 走 HTTP API 消费。
4. **qtcloud-devops 的 provider 无 CORS、无认证、无业务逻辑**——纯 CLI 适配层，未为 Web 客户端接线（studio web 端是 stub）。
5. **架构文档已把三组件列为 scope**（`cli/docs/architecture.md` 格栅：cli/studio/provider 各走 build/test/release），但**没有 contract.yaml 实际定义**，只有 `docs/contract/*.md`。
6. **无任何规划**：顶层 ROADMAP 已 per-scope 化，cli 有 `ROADMAP.md`；provider/studio 无 ROADMAP、无 CI workflow、studio 无自身 CHANGELOG。

## 二、定位澄清结论

### 三层职责

```
cli（引擎）     唯一执行层 / 事实源，封装 DevOps 规范为可执行命令
    │
    ├─ 桌面端直连（studio：本地进程可启动，无网络往返）
    │
provider（网关） CLI 能力的 HTTP 适配层，服务「无法启动进程」的消费端
    │             （Web 浏览器、远程部署、CI 服务）
    │
studio（视图）   面向人的客户端：桌面端直连 CLI；Web 端应经 provider
```

- **cli = 能力引擎**：唯一的执行层与事实源（所有真实操作都在这里）。
- **provider = CLI 的 HTTP 网关/适配层**：不做业务逻辑（无认证/存储/加密），只是把 `code status` 等能力远程化。当前无消费者 = **预留未接线**，不是业务服务端。
- **studio = 客户端视图层**：状态驱动 UI（AppState），桌面端直连 CLI 是正确选择（本地进程更简单、无网络依赖）；Web 端走 provider 是既定方向但**尚未实现**（stub 占位）。

### 与 qtcloud-secret 的同与不同

- **结构对齐**：`cmd/server/main.go` + `internal/{model,handler}` 分层，studio `api/` + `ui/` 分层——骨架相同，这是"对齐"的真实含义。
- **职责不同**：qtcloud-secret 的 provider 是**业务服务端**（认证、加密、存储、CORS）；qtcloud-devops 的 provider 是**纯适配网关**（无业务逻辑）。同名 provider，含义不同——注释应写明"结构对齐、职责不同"，避免误导读者认为 devops provider 也承载安全与存储。

## 三、待决策项（需要产品/架构确认）

1. **Web 端是否经 provider 上线**？若走：需补 CORS 中间件（参考 qtcloud-secret `handler/cors.go`）、把 studio web 端 stub 换成 HTTP 客户端（`provider_client.dart` 模式）、决定部署形态（provider 服务部署在哪、与 QtCloud 云平台关系）。若不走：provider 应标记为实验性，避免"看起来是产品"的误导。
2. **provider/studio 是否纳入 per-scope 规划**？当前架构格栅已把它们列为 scope，但无 contract.yaml 定义、无 ROADMAP、无 CI。至少应补：`contract.yaml` scope 定义 + 各组件 CI（provider: go vet+test；studio: dart analyze+test）。
3. **provider 的超时接线**：`cliScanTimeout` 常量已声明未使用（评审 MUST），无论定位如何都应接上 `context.WithTimeout`。

## 四、一句话定位

> **qtcloud-devops 的 provider/studio 是 CLI 的「远程化」与「可视化」两翼：provider 把 CLI 能力变成 HTTP 服务（面向 Web/远程消费端），studio 把 CLI 能力变成人可用的界面（桌面端直连、Web 端待经 provider）。二者都是 cli 的视图/适配层，不是独立业务系统。**
