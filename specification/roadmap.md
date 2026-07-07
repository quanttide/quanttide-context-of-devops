在开源社区中，ROADMAP.md（或 ROADMAP.rst、存放在 docs/ 目录下的路线图文件）是一个具有特殊意义的项目文件。如果说 README.md 是项目的“产品说明书”，CHANGELOG.md 是项目的“历史档案”，那么 ROADMAP.md 就是项目的“导航仪”。
它主要用于向用户和贡献者清晰地传达项目未来的发展方向、承诺将要实现的功能以及长期的愿景。
以下是开源社区中 ROADMAP 特殊文件的常见用法和规范：
1. 核心作用与使用场景
对齐社区预期：告诉使用者“我们未来会做什么，不会做什么”。这能极大减少社区里重复的 Issue（比如“你们什么时候支持 XXX 功能？”）。
指导贡献者：告诉外部贡献者哪些功能是核心团队正在筹划的，欢迎大家在哪些方向提交 PR（Pull Request），避免贡献者做无用功。
拒绝不合理需求：当用户提出偏离项目核心定位的需求时，维护者可以直接引用 ROADMAP.md 来礼貌拒绝（“这个功能不在目前的 Roadmap 范围内”）。
吸引赞助：对于商业化或寻求赞助的开源项目，一个清晰、宏大的 Roadmap 能让赞助商看到项目的发展潜力和资金投入方向。
2. 常见的内容结构
虽然格式没有绝对标准，但优秀的 ROADMAP.md 通常包含以下几个部分：
项目愿景：一两句话说明这个项目最终想成为什么。
核心原则：指导开发优先级的原则（例如：稳定性大于新功能、性能优先等）。
阶段/里程碑划分：
按版本划分：如 v2.0、v2.1、v3.0 分别计划做什么。
按时间划分：如 2024 Q1、2024 Q2。
按状态划分：使用状态标签，如：
🤔 Exploring（探索中）：还在讨论，不一定做。
🚧 In Progress（开发中）：正在动手做。
👀 Under Review（审核中）：代码已提交，正在 Review。
✅ Shipped（已发布）：已完成（通常也会同步到 CHANGELOG）。
明确“不做什么”：这是非常高级的做法，明确列出项目永远不会考虑的功能，划定边界。
3. 典型示例 (Markdown 格式)
# Project Roadmap
欢迎来到 OurProject 的路线图！这里列出了我们未来的开发计划。
## 🎯 Our Vision
让数据处理变得前所未有的简单和快速。
## 🗺️ Roadmap
### v2.0 - 核心引擎重写 (预计 2024 Q3)
- [x] 重构底层调度器，提升 50% 性能
- [ ] 支持原生的 GPU 加速 🚧
- [ ] 提供全新的 Python SDK 👀
### v2.1 - 生态集成 (预计 2024 Q4)
- [ ] 添加对 Kafka 的原生连接器支持 🤔
- [ ] 可视化监控 Dashboard
### 💡 Future Ideas (远期想法)
- 支持 WebAssembly 运行时
- 去中心化计算节点支持
### 🚫 What‘s Not on the Roadmap
- **图形化界面 (GUI) 编辑器**：本项目坚持 CLI/代码优先，不打算开发内置 GUI。
- **支持 Python 2.x**：我们已全面拥抱 Python 3.8+。
4. 现代开源社区的演变趋势
近年来，纯文本的 ROADMAP.md 正在与其他项目管理工具结合使用：
结合 GitHub Milestones / Projects：
很多项目不再维护长篇大论的 ROADMAP.md，而是直接在 GitHub 的 Projects (看板) 或 Milestones 中管理路线图，然后在 ROADMAP.md 中放一个链接（如 查看我们的路线图看板 ->）。这样状态可以自动同步，不用手动维护文本。
RFC (Request for Comments) 机制：
对于大型基础设施项目（如 Rust, React, Vue），Roadmap 中的重大功能通常会先发起一个 RFC 提案。社区在讨论通过后，该 RFC 才会被正式列入 Roadmap。
Monorepo 中的多组件 Roadmap：
在像 LangChain 这样的庞大家族中，会有多个子项目的 Roadmap（如 langchain-core/ROADMAP.md, langchain-community/ROADMAP.md）。
5. 如何在你的项目中用好它？
保持更新：过时的 Roadmap 比没有 Roadmap 更可怕。每次发版后应同步更新状态。
不要过度承诺：路线图不是法律合同，尽量使用“计划”、“探索中”等词汇，而不是“保证会做”。
允许社区参与：可以通过 Issue 讨论来决定哪些功能应该被加入 Roadmap。