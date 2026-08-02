# MA-VRC-Skill

用于 opencode 的 Modular Avatar agent skill —— 在 Unity/VRChat 项目中安全地检查、配置和自动化 Modular Avatar。

## 前置条件

- **推荐使用 opencode（非必备）**：本 skill 配合 opencode 使用，通过 Unity MCP 服务器提供的原生 `unityMCP_*` 工具进行 Unity 的实时检查和修改。没有 Unity MCP 时无法进行编辑器实时操作，但 `scripts/` 目录下的本地检查脚本（PowerShell / Python）仍可独立运行。配置和使用方法见 `references/unity-mcp-playbook.md`。
- 已安装 Modular Avatar 包的 Unity 项目。
- （可选）VRChat SDK、NDMF、VRCFury 及相关包。

## 使用方法

在 opencode 中加载本 skill，然后按照 `SKILL.md` 中的工作流执行。它作为工作流导航器 —— 每个任务都从发现开始，并按指定格式报告变更。

## 目录结构

- `SKILL.md` —— 主要工作流和策略
- `references/` —— 组件指南、工作流、版本说明、安全规则、官方来源
- `scripts/` —— 本地包检查脚本（PowerShell / Python）
