# Exercise 05 — MCP Governance

## What You'll Do

In Exercise 04 you installed MCP servers the way most developers do today — the agent picked an npm package, used `@latest`, stored credentials in a local JSON file, and exposed every tool the server offers. It worked, but it was ungoverned.

In this exercise you'll replace that setup with the **JFrog MCP Registry**. You'll install the same two MCP servers — filesystem and GitHub — but this time through a governed Gateway that controls versions, credentials, and tool access. By the end you'll have a concrete side-by-side comparison.

**Estimated time:** 20-25 minutes

### At a Glance

1. [Clean up Exercise 04](#step-1--clean-up-exercise-04)

**Install the Gateway (Steps 2-4)**

2. [Open the MCP Registry](#2-open-the-mcp-registry)
3. [Install the JFrog VS Code plugin](#3-install-the-jfrog-vs-code-plugin)
4. [Understand what the plugin does](#4-understand-what-the-plugin-does)

**Goal A — Filesystem MCP (Steps 5-8)**

5. [Install the filesystem MCP](#5-install-the-filesystem-mcp)
6. [Check the available tools](#6-check-the-available-tools)
7. [Use it](#7-use-it)
8. [Notice the version](#8-notice-the-version)

**Goal B — GitHub MCP (Steps 9-11)**

9. [Install the GitHub MCP](#9-install-the-github-mcp)
10. [Use it](#10-use-it)
11. [Notice the tool restrictions](#11-notice-the-tool-restrictions)

## Prerequisites

- [Main prerequisites](../../README.md#prerequisites) completed


---


## Step 1 — Clean up Exercise 04

Before setting up the Registry, remove the MCP servers from Exercise 04.

Prompt the agent:

```
Remove all MCP servers from this project's configuration.
Clear the .vscode/mcp.json file.
```

Verify:

- `.vscode/mcp.json` is empty or contains only `{}`
- **MCP: List Servers** from the Command Palette no longer shows the filesystem or GitHub servers

> The MCP packages installed via `npx` in Exercise 04 are cached globally in `~/.npm/_npx/`, not in the project directory. Without the config in `mcp.json`, they're inert — VS Code won't launch them. If you want to reclaim disk space, run `npm cache clean --force`.

> In Exercise 04, credentials were stored in plaintext in that file. Treat `.vscode/mcp.json` like `.env` and keep sensitive values out of version control.


---


## Steps 2-4 — Install and Verify the Gateway

&nbsp;

### 2. Open the MCP Registry

In the JFrog Platform, first select the **Agentic Development Lab** project from the project picker in the top navigation bar. Then navigate to **AI/ML > Registry > MCP Servers**.

<img src="../../resources/mcp-registry.png" alt="MCP Registry in JFrog Platform" width="800" />

&nbsp;

### 3. Install the JFrog VS Code plugin

Click **Install MCP** on any available MCP server, then select **VS Code** as your IDE. Click the installation link shown in the UI, or open this link directly:

```
vscode://chat-plugin/install?source=jfrog/vscode-plugin
```

This installs a **Copilot chat plugin** — not a traditional VS Code extension. It has no UI, no sidebar panel, and no settings page. Instead, it works by injecting governance instructions into your workspace every time a Copilot session starts.

<img src="../../resources/jfrog-agent-plugin.png" alt="JFrog agent plugin installation" width="800" />

After installing, **restart VS Code entirely** (not just the chat panel — the plugin's session hooks need a full restart to activate).

> **Note:** The plugin uses your existing JFrog CLI authentication. If you haven't configured it yet, run `jf config add` in your terminal and follow the prompts. Alternatively, set the environment variables `JFROG_PLATFORM_URL`, `JF_PROJECT`, and `JFROG_ACCESS_TOKEN` in your shell profile. Restart VS Code after configuring.

&nbsp;

### 4. Understand what the plugin does

The JFrog plugin is a **prompt-engineering layer**. On every Copilot session start, a hook script injects governance instructions and a scripts folder into your workspace: `.github/copilot-instructions.md` (Gateway workflow instructions) and `.github/scripts/` (helper scripts used by the workflow).

You can see the source of these injected instructions in the [plugin's template](https://github.com/jfrog/vscode-plugin/blob/main/plugin/templates/copilot-instructions.md).

Copilot reads `.github/copilot-instructions.md` automatically and follows the rules inside. When you ask the agent to install an MCP, it:

1. Runs the bundled lookup script to query the JFrog MCP Registry for approved servers
2. Adds an entry to `.vscode/mcp.json` that launches the MCP through `npx @jfrog/mcp-gateway`
3. The Gateway — a local proxy (`@jfrog/mcp-gateway` npm package) — wraps the real MCP server, enforcing tool policies on every call

Each MCP from the MCP list gets its own attached JFrog Gateway process. That process proxies communication between VS Code and the MCP server while enforcing governance controls for tool access and runtime behavior.

Each MCP gets its own entry in `mcp.json` and appears individually in VS Code's MCP server list. Run **MCP: List Servers** from the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`) — you should see each governed MCP listed by name (e.g., `filesystem-mcp`), not a single "Gateway" entry.

Think about what this means: in Exercise 04, the agent wrote its own MCP configuration pointing directly to npm packages. Here, the injected instructions ensure the agent always routes through the Gateway. The governance comes from the instructions themselves — and from the Gateway enforcing tool policies at runtime.

Before continuing, ask the agent:

```
Which MCP servers are available for installation from the JFrog plugin?
```

The agent will read your JFrog CLI configuration (`~/.jfrog/jfrog-cli.conf.v6`), present a picker with the available server IDs, and ask for your JFrog project name. When it asks, provide the **project key** (not the display name): `agentic-development-lab`.

> **Important:** The agent's prompt says "project name" but the catalog API requires the **project key**. The project key is the hyphenated identifier (e.g., `agentic-development-lab`), not the human-readable name shown in the UI (e.g., "Agentic Development Lab"). Using the display name will fail.

Once you provide both, it runs the catalog lookup and lists the approved MCP servers — for this lab: `filesystem-mcp` and `io.github.github/github-mcp-server`.


---


## Goal A — Filesystem MCP (local, governed)

In Exercise 04 you installed the Filesystem MCP directly from npm. Now you'll install it through the Gateway and see what's different: which version gets installed, which tools are exposed, and why.

&nbsp;

### 5. Install the filesystem MCP

Prompt the agent:

```
Install the filesystem MCP server for this project.
```

The agent installs the approved filesystem MCP through the Gateway. If you're in a new chat session, it will ask for the JFrog server and project again — that's expected. Compare this to Exercise 04, where the agent chose whatever npm package it wanted.

Verify the install: run **MCP: List Servers** from the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`) or open `.vscode/mcp.json` — you should see a `filesystem-mcp` entry that launches through `npx @jfrog/mcp-gateway`.

&nbsp;

### 6. Check the available tools

Prompt the agent:

```
List all the tools available from the filesystem MCP server.
```

You should see the governed tool list — potentially a subset of what was available in Exercise 04. The Gateway applies **tool policies** set by your project admin, so tools like `write_file` or `create_directory` may be restricted depending on the policy.

<img src="../../resources/mcp-tool-permissions.png" alt="MCP tool permissions in JFrog Platform" width="800" />

These permissions are configured centrally in the JFrog Platform — not in a local JSON file on each developer's machine.

&nbsp;

### 7. Use it

Prompt the agent:

```
List all Python files in this project using filesystem tools.
```

It works the same as Exercise 04 — but the setup behind it is completely different.

&nbsp;

### 8. Notice the version

The MCP server installed through the Gateway is **not** `@latest`. It's a specific version that was resolved through Artifactory and passed your organization's curation policies.

One such policy is the **zero-day filter** — npm packages less than 30 days old are automatically blocked. This prevents the exact supply chain attack you saw in Exercise 04, where `@latest` could pull a freshly published (and potentially compromised) package.

<img src="../../resources/curation-npm-policy-zero-day-filter.png" alt="Curation policy: npm zero-day filter" width="800" />

In Exercise 04, every VS Code restart pulled whatever version was currently on npm. Here, the version is pinned, scanned, and governed.


---


## Goal B — GitHub MCP (remote, governed)

In Exercise 04 you pasted a GitHub token directly into the agent's chat, and it stored it in plaintext in `mcp.json`. Now you'll install the same GitHub MCP through the Gateway — the agent will ask for the token as a required configuration value, and it gets stored as an environment variable in `mcp.json` under the Gateway entry.

&nbsp;

### 9. Install the GitHub MCP

Prompt the agent:

```
Install the GitHub MCP server.
```

The agent queries the catalog, finds `io.github.github/github-mcp-server`, and sees it requires an `Authorization` header. It will ask you to provide a GitHub token — enter it in the format `token ghp_...` (with the `token ` prefix).

The agent writes the entry to `.vscode/mcp.json` with the `Authorization` value stored as an environment variable that the Gateway reads at startup and applies as an HTTP header to the upstream MCP server.

&nbsp;

### 10. Use it

Verify the agent is authenticated:

```
Use the GitHub MCP to tell me who I'm authenticated as.
```

The agent calls `get_me` and returns your GitHub username. Then try reading a public repo:

```
List the root contents of the jfrog/jfrog-cli repository using the GitHub MCP.
```

Then list some recent issues:

```
List the 3 most recent open issues on jfrog/jfrog-cli.
```

&nbsp;

### 11. Notice the tool restrictions

In Exercise 04, the raw GitHub MCP exposed every tool — including `create_issue`, `add_issue_comment`, `delete_branch`, `fork_repository`, and more. Through the Gateway, the tool policy restricts the available tools to a governed subset.

Prompt the agent:

```
List all the tools available from the GitHub MCP server.
```

Compare the list to what you had in Exercise 04. Tools like `create_issue`, `add_issue_comment`, and `fork_repository` are not available — the admin has restricted write operations through the JFrog Platform's tool-level policies. The agent can read repositories, list issues, and browse contents, but cannot modify anything it shouldn't.


---


## Compare

### Before vs After

| | Exercise 04 (raw MCP) | Exercise 05 (Registry) |
|---|---|---|
| **Package source** | `npx @latest` from npm — unvetted | Scanned and version-pinned from Artifactory |
| **Curation** | None — any version, any time | Zero-day filter blocks packages less than 30 days old |
| **Credentials** | Plaintext token in `.vscode/mcp.json` | Managed through Gateway env vars — agent prompts for values at install time |
| **Configuration** | Agent picks args, scope, env vars | Gateway configures automatically from platform settings |
| **Tool control** | All tools exposed — `delete_branch`, `create_repository`, everything | Filtered to allowed tools per project policy |
| **Visibility** | Invisible JSON file on one developer's laptop | Centrally managed in JFrog Platform — admin sees all |




---


## Key Takeaways

- **The Registry is the governed alternative to npm.** MCP servers are scanned, versioned, and approved before developers can install them — the same model as package management through Artifactory.

- **Curation policies apply to MCP servers too.** The same zero-day filter that protects your npm dependencies also governs which MCP server versions can be installed. No more pulling unvetted `@latest` on every restart.

- **Credentials are managed through the Gateway.** Required credentials are declared in the MCP catalog metadata. The agent prompts for them at install time and stores them as environment variables in the Gateway entry — not as raw config the agent invented on its own.

- **Tool-level policies give admins control.** The Gateway filters which tools each MCP server exposes, based on project policies. The agent only sees what it's allowed to use.

- **The agent experience doesn't change.** The agent still calls tools by name. The difference is that the Gateway controls which tools exist, which versions run, and what credentials are used.

- **This completes the governance stack.** Skills are governed through the Skills Registry (Exercises 02-03). MCP servers are governed through the MCP Registry (Exercises 04-05). The agent operates freely within the boundaries your organization defines.
