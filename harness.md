## 1. Harness in the world

Github Copilot
Claude Code
OpenAI Codex
Cursor

Tongyi Lingma
Qoder
Trae
Comate
CodeGeeX

## The .md file convention in harness
| File | Used by | Purpose |
------------------------------
| README.md | everyone (human) | Not agent-specific, but agents read it for project context. |
| AGENTS.md | cross-tool standard | Project-level instructions for **agents**: how to build, test, conventions, do/don'ts. The vendor-neutral readme for agents. Placed at repo root (can be nested per folder) |
| CLAUDE.md | Claude Code | Claude's memory file: conventions, commands, architecture notes. Auto-loaded. Supports nested ones per directory and a personal ~/.claude/CLAUDE.md. |
| SKILL.md | Anthropic agent skills | Defines a skill: a reusable, model-invoked capability/workflow. Has YAML (name, description) formatter + Markdown body of instructions. Lazy-loaded |
| .github/copilot-instructions.md | Github Copilot | Repo-wide custom instructions auto-applied to every Copilot request in that repo |
