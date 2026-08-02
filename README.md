# MA-VRC-Skill

Modular Avatar agent skill for opencode — inspect, configure, and automate Modular Avatar in Unity/VRChat projects.

[中文版 README](README.zh-CN.md)

## Prerequisites

- **Unity MCP** (required): This skill relies on the native `unityMCP_*` tools (Unity MCP server) for live Unity inspection and changes. Without it, live-editor work is not possible. See `references/unity-mcp-playbook.md` for setup and usage.
- A Unity project with the Modular Avatar package installed.
- (Optional) VRChat SDK, NDMF, VRCFury, and related packages.

## Usage

Load the skill in opencode, then follow the workflow in `SKILL.md`. It acts as a workflow navigator — start every task with discovery, and report changes in the specified format.

## Structure

- `SKILL.md` — main workflow and policies
- `references/` — component guide, workflows, version notes, safety rules, official sources
- `scripts/` — local package inspection helpers (PowerShell / Python)
