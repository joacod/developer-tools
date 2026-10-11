# Codex

[Codex](https://openai.com/codex/) is OpenAI's coding agent, available in ChatGPT, IDEs, and the terminal.

## Install

This guide uses the npm installation path from the [Codex CLI documentation](https://developers.openai.com/codex/cli/):

```bash
npm install -g @openai/codex
```

If you need Node.js and npm first, see the [NVM and Node.js guide](../../developer-environment/nvm-node).

## Use

From a project directory, sign in and start Codex:

```bash
cd /path/to/project
codex login
codex
```

Complete the browser sign-in flow, then describe the task. For example:

```text
Tell me about this project
```

Codex can inspect the repository, edit files, and run local commands. Review its changes before committing.

## Instructions

Codex reads `~/.codex/AGENTS.md` for instructions that apply to every project.

Copy the global template there:

```bash
cp instructions/global/AGENTS.md ~/.codex/AGENTS.md
```

The template keeps two personal rules:

- Branch names stay `<type>/<short-description>` unless a `codex/` prefix is explicitly requested. `ticket-to-pr` follows the same rule.
- Add a compatibility layer only when persisted data, an external consumer, or an explicit requirement needs it.

| Scope | Path | Description |
| --- | --- | --- |
| Global | [`instructions/global/AGENTS.md`](./instructions/global/AGENTS.md) | Personal Codex rules that apply in every project. |

## References

- [Codex](https://openai.com/codex/)
- [Codex CLI documentation](https://developers.openai.com/codex/cli/)
