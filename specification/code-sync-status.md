# 代码同步状态

好的，我们来深入拆解这个状态设计。

你提出的三分法是一个很好的顶层抽象，但要把它落地成一个健壮的系统，我们需要把它展开成一个分层的、可操作的状态机。这恰恰是把“黑盒”变成“透明盒”的关键一步。

第一层：用户视角的顶层状态（对外）

这层直接对应你的三分法，是给用户和 AI 看的，极其简洁，认知负荷最低。

顶层状态 图标 含义 用户可执行的操作
Synchronized (已同步) ✅ 本地与远程完全一致，工作区干净。这是理想终态。 无需操作。
Out-of-sync (未同步) 🔄 存在差异，但可通过标准 Git 操作自动修复。这是常态。 code sync (一键修复)
Anomaly (异常) ⚠️ 出现了无法自动处理的情况，需要人工介入决策。 查看详情，根据建议手动处理。

—

第二层：机器视角的内部状态与转换（对内）

这层是状态机核心，定义了 Out-of-sync 和 Anomaly 的具体子状态，并明确了它们之间的转换路径。这解决了“报得很不准确”和“处理逻辑不透明”的问题。

我们可以把触发状态检测和转换的事件也显式地定义出来。

内部状态 描述 触发事件 顶层归类 自动修复动作 (code sync)
Clean 工作区干净，HEAD 等于 main，main 等于 origin/main 初始化、sync 成功 Synchronized -
Ahead 有本地提交未推送。main 领先 origin/main git commit Out-of-sync git push
Behind 远程有新提交未拉取。origin/main 领先 main 远程有新推送 Out-of-sync git pull —ff-only
Diverged 本地和远程历史分叉。main 和 origin/main 都有对方没有的提交。 本地变基/修改历史后 Anomaly 停止，报告冲突，建议手动 merge/rebase
Dirty 工作区或暂存区有未提交的变更。 编辑文件, git add Out-of-sync 提示用户需要先 commit，或询问是否自动 stash。
Detached HEAD 指向一个具体的 commit，而非分支。 git checkout <commit> Anomaly 停止，报告状态，建议 git switch main
Unknown 任何未预料到的 Git 状态或操作失败。 合并冲突、锁文件失败等 Anomaly 停止，完整报告原始 Git 输出，建议手动处理

关键设计原则：

1. 安全边界：code sync 操作只在 Out-of-sync 状态安全执行。一旦进入 Anomaly 状态，自动流程必须中止，并提供清晰的诊断信息和建议。
2. 检测先于动作：状态的准确是自动化正确的前提。这就是你日志中提到的 fetch 作用域问题如此核心的原因。状态检测逻辑必须万无一失。
3. 可组合性：父仓库的 Synchronized 状态，应该是由其本身和所有子模块的状态共同决定的。一个子模块 Out-of-sync，则父仓库整体是 Out-of-sync。

—

第三层：从状态到呈现（诊断信息）

当工具报告状态时，不应只说“异常了”，而应像医生一样给出诊断报告。对于 Anomaly 状态尤其重要。

一个 Diverged 状态的诊断报告示例：

```text
⚠️  Anomaly: Diverged (本地与远程历史分叉)

[本地领先的提交]
a1b2c3d 优化状态检测逻辑 (10分钟前)
e4f5g6h 修复fetch作用域问题 (30分钟前)

[远程领先的提交]
i7j8k9l 更新文档 (5分钟前)

建议操作：
1. 先拉取远程变更：`git pull —rebase origin main`
2. 如果有冲突，手动解决后执行：`git rebase —continue`
3. 最后推送：`git push origin main`
```

这种设计将 AI 和人类都摆在了协作的中心。AI 可以解析这个结构化的诊断报告来提供更智能的建议，而即使是新手人类用户，也能明白发生了什么以及下一步该怎么做。
