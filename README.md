# Codemap Skill

Task-driven generation of **Call-Path Slice codemaps** for coding projects — a single structured Markdown file written to `<project>/.codemaps/`. The output doubles as `@`-injection context for Claude and as a visual reading source for the local viewer.

Skill definition: [`SKILL.md`](./SKILL.md)
中文版: [`README.zh.md`](./README.zh.md)

## Demo

<video src="https://github.com/user-attachments/assets/d102ff5e-347d-478a-9549-d619a16f7ef2"
       controls muted playsinline width="800"
       poster="https://raw.githubusercontent.com/tttinkl/codemaps-skill/main/image.png">
  Your viewer doesn't support inline video.
</video>

## Install

Install into your local AI agents (Claude Code / Cursor / Codex / Gemini CLI, …) in one shot via the [skills CLI](https://github.com/obra/skills):

```bash
pnpm dlx skills add tttinkl/codemaps-skill
```

Pick the target agent when prompted.

## Usage

Once activated, prompt your AI agent with things like:

- "build a codemap" / "/codemap make a code map for the X flow" / "help me understand this call chain"

The agent runs the 6-phase workflow from `SKILL.md` and produces `.codemaps/<slug>.md`.

## Companion VS Code Extension

`.codemaps/**/*.md` files can be rendered as an interactive call graph by the **Codemap Viewer** extension (works in VS Code / Windsurf / Cursor).

**Download & install**:

1. Grab the latest `.vsix` from [GitHub Releases](https://github.com/tttinkl/codemaps-skill/releases)
2. Install with the actual path to the `.vsix` (cd to the download directory, or use an absolute path):

   ```bash
   code --install-extension ~/Downloads/codemaps-vscode-<version>.vsix
   ```

   For Windsurf / Cursor, use the corresponding `windsurf` / `cursor` CLI.

## Preconditions

The skill self-checks before activating:

- [x] The working directory is a coding project (real source, not a docs repo)
- [x] You have stated a task / entry point (e.g. "the login flow", "`/api/orders` POST handler")

If either is missing, the skill asks back rather than guessing.

## Screenshot

![Codemap Viewer screenshot](image.png)

## License

MIT
