# 自举：应用与库的两种范式

`qtcloud-devops` 和 `quanttide-devops-toolkit` 是量潮 DevOps 体系下的两个核心仓库，分别代表**应用**和**库**两种范式。它们都用自举来验证自身，但路径截然不同。

## 应用范式：qtcloud-devops

qtcloud-devops 是一个命令行工具，覆盖 Plan → Code → Build → Test → Release 五个阶段。

**自举路径：用自己发布自己**

它的起点是发布自动化。v0.1 时还只是一个 Python 脚本，做的事情很简单——自动打 Tag、更新 CHANGELOG、推送到 PyPI。后来发现一个悖论：如果发布工具本身还需要手动发布，那它解决的根本不是自己的问题。

于是从 v0.4 开始，团队强制要求：**所有版本必须通过 `release publish` 命令发布**。命令发布工具，工具发布命令——形成一个闭环。

```
qtcloud-devops release stage -v cli/v0.9.2-rc.1
  → 预发布版本，自动生成 CHANGELOG
qtcloud-devops release publish -v cli/v0.9.2 -y
  → 正式发布，打 Tag、创建 Release、推送包
```

最直接的验证发生在 v0.9.0 → v0.9.1 → v0.9.2 的 24 小时三轮迭代中：

| 版本 | 事件 |
|------|------|
| v0.9.0 | 发布后立即发现 Cargo.lock 漏提交 |
| v0.9.1 | 用 v0.9.0 修复漏提交问题，但子模块计数异常 |
| v0.9.2 | 修复后重发，新增 `--force` 参数自动清理已存在的 Tag/Release |

三轮连发，每一轮都在"用上一版本发布下一版本"。这不是设计好的剧本，而是工具在真实使用中暴露缺陷、修复缺陷、迭代自身的自然过程。

**应用自举的闭环条件：**
- 发布命令必须稳定到足以管理自己的发布流程
- 出问题后修复流程不能比手动更慢
- CHANGELOG、Tag、Release 三者必须一致且可追溯

截至 v0.9.x，patch 版本级别的发布已基本无需人工干预。

## 库范式：quanttide-devops-toolkit

quanttide-devops-toolkit 是一个 monorepo，同时维护 Python 和 Rust 两个语言包的 SDK。

**自举路径：被依赖者验证自身**

库的自举与应用不同。应用自举是"我自己用自己"，库的自举是"我被别人用，别人用得好说明我靠谱"。

toolkit 的主要消费者就是 qtcloud-devops 本身。当 CLI 工具依赖 toolkit 的能力（如版本号解析、Git 操作封装、契约扫描），每一次 CLI 的发布都在隐式地验证 toolkit 的可用性。

这种自举反馈回路更间接但同样有效：

```
toolkit 提供 SDK
  → qtcloud-devops 消费 SDK 构建命令
  → qtcloud-devops 用 release 命令发布自己
  → 发布成功 → toolkit 的 SDK 被验证
  → 发布失败 → 原因可能是 CLI 的问题，也可能是 toolkit 的问题
```

**两种范式的对照**

| 维度 | 应用 (qtcloud-devops) | 库 (quanttide-devops-toolkit) |
|------|----------------------|------------------------------|
| 自举方式 | 用自己发布自己 | 被消费者验证 |
| 验证信号 | 发布成功 | 依赖方运行正常 |
| 失败影响 | 自己发不出来 | 下游全部阻塞 |
| 迭代周期 | 快速，patch 可独立发 | 受制于消费者的发布节奏 |
| 核心关注 | 流程完整性 | API 稳定性 |

**为什么两种范式都要做？**

应用自举解决的是"工具是否真的可靠"——一个连自己都管不好的发布工具不值得信任。库自举解决的是"抽象是否正确"——当一个库被真实消费者反复调用而不用频繁改接口时，说明它的抽象边界是合理的。

两者合在一起，形成了从底层 SDK 到上层 CLI 再到业务交付的完整自举链条：toolkit 提供能力 → CLI 组装能力 → CLI 发布自己 → 发布过程反过来验证 toolkit。
