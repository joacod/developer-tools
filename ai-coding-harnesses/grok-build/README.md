# Grok Build

[Grok Build](https://x.ai/cli) is SpaceXAI's terminal-based AI coding agent. It runs as an interactive TUI, headlessly for scripts, or via the Agent Client Protocol (ACP).

## Install

On macOS or Linux:

```sh
curl -fsSL https://x.ai/cli/install.sh | bash
```

On Windows PowerShell:

```powershell
irm https://x.ai/cli/install.ps1 | iex
```

Verify:

```sh
grok --version
```

## Use

Start from a project directory:

```sh
cd /path/to/project
grok
```

On first launch, Grok opens a browser for authentication. In non-browser environments, use an API key:

```sh
export XAI_API_KEY="xai-..."
grok
```

Headless one-shot:

```sh
grok -p "Explain this codebase"
```

## References

- [Grok Build](https://x.ai/cli)
- [Getting started docs](https://docs.x.ai/build/overview)
- [Source repository](https://github.com/xai-org/grok-build)
