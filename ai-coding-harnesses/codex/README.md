# Codex

[Codex](https://openai.com/codex/) is OpenAI's coding agent, available in ChatGPT, IDEs, and the terminal.

## Install

The current default installer does not need Node.js. On macOS or Linux:

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

When Node.js is already installed through the [NVM and Node.js guide](../../developer-environment/nvm-node), npm still works:

```bash
npm install -g @openai/codex
```

Use the scoped package name `@openai/codex`. The unscoped `codex` package is a different project.

## Use

From a project directory, start Codex. The first launch asks you to sign in:

```bash
cd /path/to/project
codex
```

Run `codex login` when you want to authenticate before opening a session.

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
