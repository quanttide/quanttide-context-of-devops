你选择 yoagent 是非常明智的。它确实是一个 专注于核心代理循环 的轻量级库，完美契合你 基于文件系统管理 Skill 的需求，并且避免了 MCP 的复杂性。
下面我来详细解释为什么 yoagent 适合你，以及如何使用它来为你的智能体添加 Skill 能力。
🧠 yoagent 核心价值与设计哲学
yoagent 的核心设计哲学是 “循环即产品”。它认为一个 AI Agent 本质上就是一个循环：发送消息给 LLM → 获取文本和工具调用 → 执行工具 → 将结果反馈回 LLM → 重复直到完成。它实现了这个核心循环，并提供了流式传输、取消、上下文管理等必要功能，而不引入过度复杂的规划、反思或 RAG 层。
这意味着你可以 完全掌控 如何处理 Skill，并将其作为 工具 或 系统提示的扩展 自然地融入这个循环。
✨ yoagent 关键特性（与你的需求高度相关）
🛠️ 如何用 yoagent 实现 Skill 支持
你可以通过以下两种主要方式将 Skill 集成到 yoagent 代理中，它们可以结合使用：
1. 将 Skill 作为 自定义工具
这是最直接的方式。你可以定义一个工具，当 LLM 决定使用某个 Skill 时，该工具负责加载并返回 Skill 的内容，或者执行 Skill 中定义的脚本。
use yoagent::{Agent, AgentTool, ToolDefinition};
use serde_json::json;
// 1. 定义一个“加载 Skill”的工具
struct LoadSkillTool;
impl AgentTool for LoadSkillTool {
    fn definition(&self) -> ToolDefinition {
        ToolDefinition {
            name: ”load_skill“.to_string(),
            description: ”根据技能名称加载其完整指令内容。当需要执行特定领域任务时调用此工具。“.to_string(),
            parameters: json!({
                ”type“: ”object“,
                ”properties“: {
                    ”skill_name“: {
                        ”type“: ”string“,
                        ”description“: ”要加载的技能名称，例如 ’csv_analyzer‘“
                    }
                },
                ”required“: [”skill_name“]
            }),
        }
    }
    fn execute(&self, params: serde_json::Value) -> Result<serde_json::Value, String> {
        let skill_name = params[”skill_name“].as_str().unwrap();
        // 根据名称查找并读取 Skill 文件 (例如 ./skills/csv_analyzer/SKILL.md)
        let skill_path = format!(”./skills/{}/SKILL.md“, skill_name);
        let content = std::fs::read_to_string(&skill_path)
            .map_err(|e| format!(”无法加载技能 ’{}‘: {}“, skill_name, e))?;
        // 返回技能的完整内容作为工具结果
        Ok(json!({
            ”skill_content“: content
        }))
    }
}
// 2. 定义一个“执行 Skill 脚本”的工具（如果 Skill 包含可执行脚本）
struct ExecuteSkillScriptTool;
impl AgentTool for ExecuteSkillScriptTool {
    fn definition(&self) -> ToolDefinition {
        ToolDefinition {
            name: ”execute_skill_script“.to_string(),
            description: ”执行技能中定义的脚本文件。仅当技能明确包含脚本时使用。“.to_string(),
            parameters: json!({
                ”type“: ”object“,
                ”properties“: {
                    ”skill_name“: {
                        ”type“: ”string“,
                        ”description“: ”技能名称“
                    },
                    ”script_name“: {
                        ”type“: ”string“,
                        ”description“: ”要执行的脚本文件名（如 run.sh）“
                    },
                    ”args“: {
                        ”type“: ”array“,
                        ”items“: {”type“: ”string“},
                        ”description“: ”传递给脚本的参数列表“
                    }
                },
                ”required“: [”skill_name“, ”script_name“]
            }),
        }
    }
    fn execute(&self, params: serde_json::Value) -> Result<serde_json::Value, String> {
        let skill_name = params[”skill_name“].as_str().unwrap();
        let script_name = params[”script_name“].as_str().unwrap();
        let args: Vec<String> = params[”args“].as_array()
            .map(|a| a.iter().map(|v| v.as_str().unwrap().to_string()).collect())
            .unwrap_or_default();
        // 构建脚本路径 (例如 ./skills/csv_analyzer/scripts/run.sh)
        let script_path = format!(”./skills/{}/scripts/{}“, skill_name, script_name);
        // 在沙箱中执行脚本（这里简化了，实际应使用沙箱机制）
        let output = std::process::Command::new(&script_path)
            .args(&args)
            .output()
            .map_err(|e| format!(”执行脚本失败: {}“, e))?;
        let stdout = String::from_utf8_lossy(&output.stdout).to_string();
        let stderr = String::from_utf8_lossy(&output.stderr).to_string();
        Ok(json!({
            ”stdout“: stdout,
            ”stderr“: stderr,
            ”success“: output.status.success()
        }))
    }
}
// 3. 构建代理并注册工具
let agent = Agent::builder()
    .with_model(”claude-3-5-sonnet-20241022“) // 指定模型
    .with_tool(LoadSkillTool)
    .with_tool(ExecuteSkillScriptTool)
    // ... 其他配置
    .build();
2. 将 Skill 内容 注入系统提示
另一种更轻量的方式是，在代理启动时或根据用户请求，动态地将相关 Skill 的指令内容拼接进系统提示中。这不需要 LLM 主动调用工具，而是“潜移默化”地影响其行为。
use yoagent::Agent;
use std::fs;
// 1. 扫描技能目录，加载所有技能的 SKILL.md 内容
fn load_all_skills(skills_dir: &str) -> Vec<String> {
    let mut skills = Vec::new();
    if let Ok(entries) = fs::read_dir(skills_dir) {
        for entry in entries.flatten() {
            let path = entry.path();
            if path.is_dir() {
                let skill_md_path = path.join(”SKILL.md“);
                if let Ok(content) = fs::read_to_string(&skill_md_path) {
                    skills.push(format!(
                        ”## 技能: {}\n\n{}“,
                        path.file_name().unwrap().to_str().unwrap(),
                        content
                    ));
                }
            }
        }
    }
    skills
}
// 2. 构建包含技能指令的系统提示
fn build_system_prompt(skills: &[String]) -> String {
    let skills_text = if skills.is_empty() {
        ”无可用技能。“.to_string()
    } else {
        skills.join(”\n\n—\n\n“)
    };
    format!(
        ”你是一个强大的 AI 助手。你拥有以下技能，请根据用户请求判断是否需要使用以及如何使用它们：\n\n{}\n\n—\n请根据用户请求，自然地决定是否调用这些技能的能力。“,
        skills_text
    )
}
// 3. 初始化代理
let skills = load_all_skills(”./skills“);
let system_prompt = build_system_prompt(&skills);
let agent = Agent::builder()
    .with_model(”claude-3-5-sonnet-20241022“)
    .with_system_prompt(&system_prompt) // 注入包含技能的系统提示
    // .with_tool(...) // 仍可配合工具使用
    .build();
📊 两种方式对比与选择建议
🚀 快速开始：最小可运行示例
确保你已安装 Rust 工具链，然后在你的项目中添加 yoagent 依赖：
cargo add yoagent
以下是一个完整的、可运行的最小示例，它结合了上述两种方法：
use yoagent::{Agent, AgentTool, ToolDefinition};
use serde_json::json;
use std::fs;
// — 1. 定义加载技能的工具 —
struct LoadSkillTool;
impl AgentTool for LoadSkillTool {
    fn definition(&self) -> ToolDefinition {
        ToolDefinition {
            name: ”load_skill“.to_string(),
            description: ”根据技能名称加载其完整指令内容。当需要执行特定领域任务时调用此工具。“.to_string(),
            parameters: json!({
                ”type“: ”object“,
                ”properties“: {
                    ”skill_name“: {”type“: ”string“, ”description“: ”技能名称“}
                },
                ”required“: [”skill_name“]
            }),
        }
    }
    fn execute(&self, params: serde_json::Value) -> Result<serde_json::Value, String> {
        let skill_name = params[”skill_name“].as_str().unwrap();
        let path = format!(”./skills/{}/SKILL.md“, skill_name);
        let content = fs::read_to_string(&path)
            .map_err(|e| format!(”技能 ’{}‘ 加载失败: {}“, skill_name, e))?;
        Ok(json!({”skill_content“: content}))
    }
}
// — 2. 辅助函数：加载所有技能名列表 —
fn list_skill_names(skills_dir: &str) -> Vec<String> {
    fs::read_dir(skills_dir)
        .map(|entries| {
            entries
                .filter_map(|e| e.ok())
                .filter(|e| e.path().is_dir())
                .filter_map(|e| e.file_name().into_string().ok())
                .collect()
        })
        .unwrap_or_default()
}
// — 3. 主函数 —
#[tokio::main]
async fn main() {
    // 假设你的技能放在项目根目录的 skills 文件夹下
    let skills_dir = ”./skills“;
    let skill_names = list_skill_names(skills_dir);
    // 构建一个简单的系统提示，告知有哪些技能可用
    let skills_list = if skill_names.is_empty() {
        ”无“.to_string()
    } else {
        skill_names.join(”, “)
    };
    let system_prompt = format!(
        ”你是一个助手。你当前可用的技能有: {}。如果需要执行特定任务，可以调用 `load_skill` 工具来获取详细指令。“,
        skills_list
    );
    // 创建代理
    let agent = Agent::builder()
        .with_model(”claude-3-5-sonnet-20241022“) // 替换为你的模型和 API Key
        .with_system_prompt(&system_prompt)
        .with_tool(LoadSkillTool)
        .build();
    // 模拟用户请求
    let user_input = ”请分析一下 sales_data.csv 文件。“;
    println!(”用户: {}“, user_input);
    // 运行代理并处理事件流
    use futures::StreamExt;
    let mut event_stream = agent.prompt(user_input);
    while let Some(event_result) = event_stream.next().await {
        match event_result {
            Ok(event) => {
                // 打印事件，实际应用中可根据事件类型更新 UI
                println!(”事件: {:?}“, event);
                // 如果是工具执行事件，可以打印工具结果
                if let yoagent::AgentEvent::ToolExecution { tool_name, result, .. } = &event {
                    println!(”工具 {} 执行结果: {:?}“, tool_name, result);
                }
            }
            Err(e) => eprintln!(”错误: {}“, e),
        }
    }
}
📁 项目结构建议
为了组织好你的 Skill 文件，建议采用以下目录结构：
your_agent_project/
├── Cargo.toml
├── src/
│   └── main.rs
├── skills/                  <— Skill 根目录
│   ├── csv_analyzer/        <— 单个 Skill 文件夹
│   │   ├── SKILL.md         <— 核心说明文件 (必需)
│   │   ├── scripts/         <— 可执行脚本 (可选)
│   │   │   └── analyze.py
│   │   └── references/      <— 参考文档 (可选)
│   │       └── data_format.md
│   ├── git_workflow/
│   │   └── SKILL.md
│   └── ...
└── ... 其他项目文件
SKILL.md 文件格式：通常包含 YAML 元数据和 Markdown 正文。
—
name: csv_analyzer
description: 分析 CSV 文件并生成摘要统计信息
version: 1.0.0
author: Your Name
—
# CSV 分析技能
## 指令
1. 使用 `load_skill` 工具或直接读取此文件内容。
2. 调用 `execute_skill_script` 工具执行 `scripts/analyze.py` 脚本。
3. 脚本接受 CSV 文件路径作为参数，并输出 JSON 格式的统计信息。
## 参数
- `file_path`: 要分析的 CSV 文件路径 (必需)
## 示例
用户: ”分析 data.csv“
助手: (调用 execute_skill_script 工具，参数: skill_name=”csv_analyzer“, script_name=”analyze.py“, args=[”data.csv“])
⚠️ 重要注意事项与最佳实践
安全第一：如果 Skill 包含可执行脚本，务必在沙箱环境中运行。yoagent 本身不提供沙箱，你需要自行实现或使用如 bollard（Docker API）等库创建容器隔离环境。切勿直接执行用户提供的脚本。
错误处理：工具的 execute 方法应返回 Result，提供清晰的错误信息，让 LLM 能理解失败原因并尝试修复或回退。
上下文管理：当加载大量 Skill 时，利用 yoagent 的上下文管理功能（自动摘要和截断旧消息）来控制 token 使用。
技能发现：除了固定目录，你还可以实现更复杂的技能发现机制，如从 GitHub 仓库拉取（类似 agentskills 工具），或通过 API 查询。
版本控制：将你的 skills/ 目录纳入版本控制（如 Git），方便团队协作和技能迭代。
🔗 进阶探索：基于 yoagent 的完整框架
如果你希望获得一个开箱即用的、基于 yoagent 构建的完整代理框架，可以关注 yoclaw。它已经实现了：
SQLite 持久化：会话、消息队列、长期记忆（FTS5 全文搜索）。
崩溃恢复：消息处理前先入队，kill -9 重启不丢数据。
安全策略：工具权限、Shell 命令黑名单、路径白名单。
预算跟踪：每日/每会话 token 和轮次限制。
技能系统：正是你所需的，将 Markdown 文件放入 ~/.yoclaw/skills/ 即可扩展能力。
yoclaw 可以作为你项目的起点或参考实现。
💎 总结
选择 yoagent 是正确的。它将 核心代理循环 的控制权完全交给你，让你能以最符合需求的方式（作为工具或注入提示）灵活地实现 Skill 系统。
下一步建议：
从上面的最小可运行示例开始，确保 yoagent 能在你的环境中跑起来。
创建你的第一个 skills/csv_analyzer/SKILL.md 文件。
实现 LoadSkillTool，并让代理调用它。
根据需要，逐步添加 ExecuteSkillScriptTool 和更复杂的安全沙箱机制。
如果你在实现过程中遇到具体问题，或者想了解如何更精细地控制代理的流式输出，随时可以再问我。
