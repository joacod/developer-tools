# Playwright MCP with Pi

Pi 0.99 and later include MCP. This guide still uses [`pi-mcp-adapter`](https://github.com/nicobailon/pi-mcp-adapter) because it adds the approval gates below and starts Playwright when a browser tool is called. Installing the adapter turns Pi's built-in MCP off. The official [`@playwright/mcp`](https://github.com/microsoft/playwright-mcp) server provides the browser.

This guide covers both global and project-local setups. The current setup in this repository is global for Pi.

## Prerequisites

- Node.js 22.19 or newer, and npm, are available. The repository's [Node.js guide](../../developer-environment/nvm-node) installs the current LTS, which meets that floor.
- Pi is installed and starts with `pi`.
- You run Pi from the project directory that should use Playwright.

## Install the Pi adapter

Check whether the adapter is already installed:

```bash
pi list
```

For a global installation, install adapter 5.2.0 if `pi-mcp-adapter` is not listed:

```bash
pi install npm:pi-mcp-adapter@5.2.0
```

If `pi list` shows a release older than 5.2.0, remove it and install 5.2.0:

```bash
pi remove npm:pi-mcp-adapter
pi install npm:pi-mcp-adapter@5.2.0
```

Release 5.2.0 includes the MCP SDK fix for [GHSA-6qxp-vccf-f47h](https://github.com/advisories/GHSA-6qxp-vccf-f47h). Release 2.25.0 does not. With that older SDK, a malicious or compromised MCP server could send saved sign-in tokens to an authorization server it chose.

For a project-only installation, run the install command with `-l` from that project's root:

```bash
pi install npm:pi-mcp-adapter@5.2.0 -l
```

The adapter package scope and the MCP server configuration scope are independent. A global adapter with a project-local `.mcp.json` is also a valid setup and is usually the simplest way to keep the package available without exposing browser tools to every project.

Pi packages run with the permissions of the user who starts Pi. Review the adapter source before installing it, and use only the official Playwright package below—not similarly named community packages.

## Choose a configuration scope

Use the same configuration content at either scope:

```json
{
  "settings": {
    "approveTools": [
      "*delete*",
      "*write*",
      "*update*",
      "*remove*",
      "*create*",
      "*auth*",
      "*token*",
      "*secret*",
      "*credential*"
    ]
  },
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

### Global for Pi — current setup

Write the configuration to Pi's global MCP file:

```text
~/.pi/agent/mcp.json
```

If `PI_CODING_AGENT_DIR` is set, use `$PI_CODING_AGENT_DIR/mcp.json` instead. This file applies to every Pi project for that user and is not committed to a repository.

If you want the configuration shared with other MCP-compatible hosts too, use the standard global file `~/.config/mcp/mcp.json` instead. Use the Pi-owned path when this integration should be available to Pi only.

### Project-local

Write the configuration to the project root:

```text
.mcp.json
```

Use this scope when only one project needs browser automation or when different projects need different Playwright versions or settings. Project-local configuration takes precedence over the global MCP layers.

If the target file already exists, merge the `playwright` entry and add these approval patterns to its existing `settings.approveTools` array. Keep every unrelated server and setting; do not replace the whole file. Do not create duplicate global and project entries unless you intentionally want the project to override the global server.

The `approveTools` patterns are glob matches. A matching MCP call remains visible but Pi asks whether to allow it once, allow it for the session, or deny it. They are approval gates, not a sandbox, and they do not cover a destructive tool whose name does not match a pattern. `"approveTools": "destructive"` gates tools the server marks as changing or deleting data. The name patterns above still cover credential-related tools that hint would miss.

The `@latest` tag follows the current official Playwright MCP release and `npx` downloads it when the server is first used. For reproducible or tightly controlled environments, replace `@latest` with a verified version of the same official `@playwright/mcp` package.

Keep credentials and other secrets out of MCP configuration files. Use environment variables or Pi's supported authentication flows instead, and review the file before committing a project-local configuration.

## Restart and verify

Restart Pi after installing the adapter or changing MCP configuration:

```text
/quit
```

Then start it again with `pi`. A global configuration should work from any project; a project-local configuration requires starting Pi from that project.

Inside Pi:

1. Open `/mcp` and confirm that `playwright` is listed.
2. Ask Pi to run `mcp({})` to show MCP status, or `mcp({ search: "screenshot" })` to discover a browser tool.
3. Use a browser tool to make the first connection. MCP servers are lazy, so Playwright may not start until this step.

If the server is missing, confirm the adapter is installed, check the selected configuration path, and restart Pi again. For browser or Playwright-specific issues, use the [official Playwright MCP documentation](https://github.com/microsoft/playwright-mcp).

## Working safely

- Global configuration exposes browser tools to every Pi project. Prefer project-local configuration for repositories that do not need them.
- Treat browser automation as an external side effect: use test accounts and review navigation, form submissions, and other state-changing actions before allowing them.
- Project-local MCP configuration is shared with anyone who can read the repository. Do not put API keys, passwords, cookies, or tokens in it.
- The adapter's approval patterns add confirmation prompts; they do not restrict filesystem, network, or browser access. Use a container, VM, or separate user for untrusted work.
- Remove the `playwright` entry from the selected configuration when the project no longer needs it. If no other project uses the adapter, remove the Pi package with `pi remove npm:pi-mcp-adapter`.
