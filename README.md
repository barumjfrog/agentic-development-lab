# Agentic Development Lab

Build software with AI agents — and learn how to govern what they do.

This hands-on lab walks you through agentic development using [VS Code](https://code.visualstudio.com/) with [GitHub Copilot](https://github.com/features/copilot/) as your IDE. You'll start by letting an agent build an entire app from specs, then progressively add skills, MCP integrations, and security governance — all backed by the [JFrog Platform](https://jfrog.com/platform/).

The exercises work in a live workshop with an instructor or self-paced on your own.

&nbsp;

## What You'll Learn

- How to scope work with specifications so an agent can build from them
- How to encode team standards as portable, versioned **skills**
- How the **JFrog Skills Registry** makes skills shareable, scannable, and safe
- Why unvetted MCP servers are a supply-chain risk — and how the **JFrog MCP Registry** governs them

&nbsp;

## Workshop Structure

```
Specs ──▸ Skills ──▸ Governance ──▸ MCP Servers ──▸ MCP Registry
 01         02          03              04              05
```

| Exercise | Title | Time | What you'll do |
|:---:|---|---|---|
| 01 | From Specs to Running App | 20-30 min | Hand the agent two spec files and watch it build a Python API |
| 02 | Agent Skills | 20-30 min | Write a skill, install it, and publish it to a governed registry |
| 03 | Skills Governance | 15-20 min | Audit a project with a curated skill from the JFrog registry |
| 04 | MCP Servers | 25-30 min | Let the agent install MCP servers, then inspect what it configured |
| 05 | MCP Governance | 20-25 min | Replace raw MCPs with governed ones from the JFrog MCP Registry |

**Total estimated time: ~2 hours**

&nbsp;

## Prerequisites

- [VS Code 1.116+](https://code.visualstudio.com/) (`Code > About` to check version)
- [GitHub Copilot](https://github.com/features/copilot/) extension — **latest version**, with **Copilot Chat** signed in (a Business/Enterprise seat from your organization is recommended; org-managed accounts also need an admin to enable **Settings → Copilot → Policies → Editor preview features** for the chat plugin in Exercise 05)
- [Python 3.11+](https://www.python.org/downloads/)
- [Node.js](https://nodejs.org/) (for running npm/npx packages in Exercises 04-05)
- [JFrog CLI](https://docs.jfrog.com/jfrog-applications/jfrog-cli/) v2.102.0 or newer (`jf --version` to verify) with the Skills module (`jf skills --help` to verify)
- Authenticate to the [`jfroggcpworkshopnyc.jfrog.io`](https://jfroggcpworkshopnyc.jfrog.io/) JFrog Platform instance via the JFrog CLI

### Clone and Open the Lab

Clone the repo **from your terminal** — it's public, so the HTTPS clone needs no credentials:

```bash
git clone https://github.com/barumjfrog/agentic-development-lab.git
cd agentic-development-lab
code .
```

Then sign into Copilot with your **enterprise** account:

1. In VS Code, click the **Accounts** icon (bottom-left person icon)
2. Select **Sign in with GitHub to use GitHub Copilot**
Caution: Must have an account with model access 
3. Once signed in, open Copilot Chat (`Cmd+Shift+I` on macOS / `Ctrl+Shift+I` on Windows) — you should see:
   - A **model selector** at the top of the chat panel
   - A **mode selector** (Ask / Edit / Agent) at the bottom

> **Why clone from the terminal?** The clone needs no GitHub account at all, and your Copilot license (for AI features) is a separate sign-in. Cloning in the terminal avoids VS Code asking which GitHub account to use for git operations.

&nbsp;

## How to Use This Repo

- Each exercise has its own folder under `exercises/` — do them in order, each one builds on the last
- The `weather-app/` folder contains the project you'll build and work with throughout
- Prompts in the exercises are suggestions — feel free to rephrase them in your own words

&nbsp;

## Exercises

### [01 — From Specs to Running App](exercises/01%20-%20setup/README.md) · 20-30 min

Hand the agent two spec files and watch it build a working Python API from scratch. This is where you learn that agentic development starts with scoping — and that asking the agent to plan before building is your friend for big tasks.

---

### [02 — Agent Skills](exercises/02%20-%20skills-intro/README.md) · 20-30 min

Write a skill that teaches the agent your README standards, publish it to the JFrog Skills Registry, install it from the registry, and watch the agent follow it automatically. By the end you'll understand what skills are and how a governed registry makes them shareable and safe.

---

### [03 — Skills Governance](exercises/03%20-%20skills-governance/README.md) · 15-20 min

Install a public skill that looks helpful but introduces security issues, discover the damage, then audit the project with a curated skill from the JFrog registry. This is where you learn why skills need the same governance as packages.

---

### [04 — MCP Servers](exercises/04%20-%20mcp-intro/README.md) · 25-30 min

Ask the agent to install filesystem and GitHub MCP servers, then inspect what it configured. Explore what that access actually means, what can go wrong, and why organizations need a layer of control over what the agent can reach.

---

### [05 — MCP Governance](exercises/05%20-%20mcp-governance/README.md) · 20-25 min

Replace the raw MCP servers from Exercise 04 with governed ones from the JFrog MCP Registry. Install the [JFrog Plugin for VS Code](https://github.com/jfrog/vscode-plugin) and let its **Agent Guard** feature discover and install MCP servers through the registry, then compare the experience — vetted versions, no plaintext credentials, tool-level policies, and central visibility.
