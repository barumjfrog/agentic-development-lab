# Exercise 04 — MCP Servers

## What You'll Do

Give the agent access to two external systems through MCP servers — one that operates **locally** on your filesystem, and one that connects to a **remote service** (GitHub) over the network. Instead of configuring them manually, you'll ask the agent to set them up. Then you'll inspect what it created, test what it can do, and explore the risks that come with each connection.

**Estimated time:** 25-30 minutes

### At a Glance

1. [Ask the agent to set up a Filesystem MCP](#1-ask-the-agent-to-set-up-a-filesystem-mcp)
2. [Inspect what the agent created](#2-inspect-what-the-agent-created)
3. [Verify the connection](#3-verify-the-connection)
4. [Use it](#4-use-it)
5. [Create a GitHub token](#5-create-a-github-token)
6. [Ask the agent to set up a GitHub MCP](#6-ask-the-agent-to-set-up-a-github-mcp)
7. [Inspect the updated config](#7-inspect-the-updated-config)
8. [Use it](#8-use-it)

<details>
<summary><strong>What is an MCP server?</strong></summary>

&nbsp;

An MCP (Model Context Protocol) server is a program that gives the AI agent access to external tools and data sources. When you add an MCP server, you're granting the agent new capabilities: reading files, querying databases, creating GitHub issues, sending messages.

MCP servers expose **tools** — functions the agent can call:

| Server | Example tools |
|---|---|
| Filesystem | `read_file`, `write_file`, `list_directory` |
| GitHub | `create_issue`, `create_pull_request`, `search_repositories` |

The agent discovers available tools automatically and uses them when relevant to your prompt.

> **The key thing to understand:** whatever permissions you give an MCP server, the agent will use. If the GitHub MCP has a token with write access to all repos, the agent can write to all repos. There is no permission layer between the agent and the MCP server's capabilities.

</details>

## Prerequisites

- [Main prerequisites](../../README.md#prerequisites) completed


---


## Steps

### 1. Ask the agent to set up a Filesystem MCP

The Filesystem MCP runs entirely on your machine — it talks to your local disk through Node.js `fs` operations. No network calls, no external APIs.

Prompt the agent:

```
I need you to have access to this project's filesystem through MCP.
Set up a filesystem MCP server scoped to the current project directory.
```

Watch what the agent does. It should create a configuration file and set up the server.

&nbsp;

### 2. Inspect what the agent created

Open `.vscode/mcp.json` — the file the agent just created. Examine it carefully:

- Which npm package did it choose?
- What version? Is it pinned or does it use `@latest`?
- What scope did it set — `.`, an absolute path, something broader?

You should see something like:

```json
{
  "servers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem@latest",
        "."
      ]
    }
  }
}
```

This is how MCP connections are configured in VS Code — a JSON file in `.vscode/` that defines server names, how to launch them, and what arguments to pass.

The agent created this for you. It decided which package, which version, and which scope. You didn't review any of those choices until now.

&nbsp;

### 3. Verify the connection

Run **MCP: List Servers** from the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`). The filesystem server should appear as connected.

If it shows an error, reload the window (`Cmd+Shift+P` → **Developer: Reload Window**).

&nbsp;

### 4. Use it

Prompt the agent:

```
Using the filesystem tools, list all Python files in this project
and give me a summary of what each one does.
```

Watch the agent use `list_directory` and `read_file` tools to explore the project.

> Note the `@latest` tag in the MCP configuration. This means you're pulling whatever version is currently published to npm every time you start VS Code. If that package is compromised, you're running compromised code with access to your filesystem. This is the same attack vector as the [`postmark-mcp` incident](https://thehackernews.com/2025/09/first-malicious-mcp-server-found.html), where a counterfeit npm package silently exfiltrated email content by BCC'ing every message to the attacker's server.

<details>
<summary>Optional: experiment with the scope</summary>

&nbsp;

The server is scoped to `.` (current directory). Consider what happens if you widen it:

- What if you change `.` to `~` (home directory)?
- What if you change it to `/`?
- What files exist in your home directory? `.ssh/`, `.env`, `.aws/credentials`, `.gitconfig`?

Try it (then change it back):

```json
"args": ["-y", "@modelcontextprotocol/server-filesystem@latest", "/Users"]
```

Prompt:

```
List the contents of my home directory.
```

The agent will happily comply. It has no concept of "this file is sensitive." It reads whatever you've given it access to.

> **Revert your scope back to `.` after this demonstration.**

</details>

&nbsp;

### 5. Create a GitHub token

Now you'll add a second MCP that connects to a **remote service** — GitHub's API over HTTPS. Every tool invocation sends an authenticated HTTP request using your personal access token.

Create a personal access token at https://github.com/settings/tokens with `repo` scope. Keep it handy.

&nbsp;

### 6. Ask the agent to set up a GitHub MCP

Prompt the agent:

```
I also need GitHub access through MCP so you can read and create issues.
Set up a GitHub MCP server. My personal access token is: <paste-token-here>
```

Watch the agent update `.vscode/mcp.json` to add the GitHub server configuration.

&nbsp;

### 7. Inspect the updated config

Open `.vscode/mcp.json` again. Notice:

- Your GitHub token is now stored in **plaintext** in a project config file
- Is `.vscode/` in `.gitignore`? If not, your token could be committed to version control
- The agent stored credentials in a file without warning you about the risk

This is exactly the kind of thing that happens when you let the agent configure infrastructure without reviewing it — it optimizes for "make it work," not "make it secure."

&nbsp;

### 8. Use it

Check **MCP: List Servers** from the Command Palette. The GitHub server should appear with its available tools listed.

First, have the agent **read** a public repo:

```
Using the GitHub MCP, list the root contents of the jfrog/jfrog-cli repository.
```

Then have the agent **explore** further:

```
Using the GitHub MCP, show me the README file from jfrog/jfrog-cli.
```

Notice what just happened: the agent read from a GitHub repo using your token without any confirmation step beyond your initial prompt.

<details>
<summary>Tools the GitHub MCP exposes</summary>

&nbsp;

Depending on the server version, the GitHub MCP typically exposes:

| Tool | What it means |
|---|---|
| `create_repository` | The agent can create new repos in your account |
| `push_files` | The agent can push code to any branch |
| `create_or_update_file` | The agent can modify any file in any repo |
| `delete_branch` | The agent can delete branches |
| `search_code` | The agent can search across all repos you have access to |

Your personal access token with `repo` scope grants all of this. The agent doesn't have a subset of these permissions — it has all of them.

</details>


---


## Discussion

You asked the agent to configure two external integrations. It did — quickly and correctly. But it also chose npm packages you didn't vet, used `@latest` with no version pinning, stored a credential in a project file, and gave itself access to tools you may not have intended.

Notice something else: the agent picked whichever npm packages it knew about. A different agent (or the same agent on a different day) might choose entirely different packages for the same purpose. There's no guarantee it picked the official or most secure option — just whatever was in its training data.

This is what an **MCP Registry and Gateway** solves:

- **Central registry** of approved MCP servers — developers connect only to vetted servers, not random npm packages
- **Tool-level permissions** — an admin can allow the GitHub MCP but disable `delete_branch` and `create_repository` for specific projects or teams
- **Gateway proxy** — a local proxy sits between VS Code and the MCP servers, handling authentication, enforcing RBAC, and logging every tool call
- **Automated policy gates** — every MCP server is scanned against security policies before it can be used

<details>
<summary>The governance gap — why organizations need this</summary>

&nbsp;

**Visibility** — How does your organization know which MCP servers each developer has installed? Right now, the answer is: it doesn't. MCP configs live on individual laptops in JSON files.

**Credential sprawl** — Every MCP server that accesses an external service needs a token or API key. Ten developers using five MCP servers each means fifty credentials scattered across machines with no rotation, no audit trail, and no central management.

**Tool control** — The GitHub MCP exposes `delete_branch` and `create_repository`. For a junior developer working on a feature, those tools are unnecessary and dangerous. But the MCP server is all-or-nothing — you can't disable individual tools from the VS Code config.

**Supply chain risk** — Both servers were installed via `npx` with `@latest`. There's no verification, no checksum, no pinned version. A compromised package runs with your filesystem and GitHub permissions.

</details>


---


## Checkpoint

- [ ] Asked the agent to install two MCP servers and watched it configure them
- [ ] Inspected `.vscode/mcp.json` and understood its structure
- [ ] Used both filesystem and GitHub tools through the agent
- [ ] Identified the risks: unpinned versions, plaintext credentials, broad scope, no tool-level control


---


## Key Takeaways

- **The agent configures what you ask for.** It won't question whether the scope is too broad, the version is pinned, or the credential storage is safe. You have to review what it produces — just like you review code.

- **MCP servers are trust decisions.** Every server you add gets the full permissions of the credentials you give it. There is no intermediary permission layer.

- **Scope aggressively.** The Filesystem MCP scoped to `.` is a very different risk profile than scoped to `~`. Always use the narrowest scope possible.

- **`@latest` is a supply chain risk.** Pin your MCP server versions the same way you'd pin npm dependencies.

- **Config files are an attack surface.** `.vscode/mcp.json` contains credentials and connection definitions. Treat it like `.env`.

- **Tool-level control requires a gateway.** You can't disable specific MCP tools from the client config — you need an intermediary that enforces policies before requests reach the server.


---


## Next Up

You've seen how MCP servers give the agent access to external systems — and how that access is currently ungoverned. In the next exercise, the JFrog MCP Registry brings all of this under control.

[Continue to Exercise 05 — MCP Governance →](../05%20-%20mcp-governance/README.md)
