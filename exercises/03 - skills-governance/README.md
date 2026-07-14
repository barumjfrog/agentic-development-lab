# Exercise 03 — Skills Governance

## What You'll Do

In Exercise 02 you wrote a skill and published it to a governed registry. Now you'll experience the other side — installing a curated skill from the registry and using it to audit your project for real security issues. This is what consuming governed skills looks like in practice.

**Estimated time:** 15-20 minutes

### At a Glance

1. [Install the curated skill from the registry](#1-install-the-curated-skill-from-the-registry)
2. [Read the skill](#2-read-the-skill)
3. [Apply the skill](#3-apply-the-skill)
4. [Review the results](#4-review-the-results)
5. [Discussion](#5-discussion)

**Bonus** — [The Public Skill](#bonus--the-public-skill): install an unvetted skill, observe the damage, clean up (B1–B6)

<details>
<summary><strong>Why governance matters for skills</strong></summary>

&nbsp;

Skills are just markdown, but they control what the agent does to your codebase. An unvetted skill can introduce vulnerabilities as easily as an unvetted npm package. A governed registry adds the missing layer: versioning, scanning, and review before a skill reaches your project.

The bonus section at the end of this exercise demonstrates this risk firsthand.

</details>

## Prerequisites

- [Main prerequisites](../../README.md#prerequisites) completed


---


## Steps

### 1. Install the curated skill from the registry

Your instructor has published a `jfrog-security-audit` skill to the workshop's curated registry. Install it as a project skill:

```bash
jf skills install jfrog-security-audit \
  --repo agentic-dev-lab-skills \
  --version 1.0.0 \
  --path .github/skills/jfrog-security-audit
```

This downloads the skill from Artifactory into your project's `.github/skills/` directory — the same place you put your README skill in Exercise 02.

&nbsp;

### 2. Read the skill

Open `.github/skills/jfrog-security-audit/SKILL.md` and read through it. Pay attention to:

- How specific each step is — no ambiguity about what the agent should do
- The four scan categories it covers (dependency CVEs, secrets, SAST, IaC)
- The reporting format it enforces
- The supply-chain check (pip config) it performs first

Compare this to a skill you might copy from a blog post. The difference is precision.

&nbsp;

### 3. Apply the skill

Prompt the agent:

```
Run a security audit on the weather-app project.
```

The agent should pick up the `jfrog-security-audit` skill and run `jf audit .` from the project root. Watch it report findings across all four categories.

> The first time `jf audit` runs, it downloads JFrog Xray scanner binaries. This may take a minute. Subsequent runs use cached binaries.

&nbsp;

### 4. Review the results

Check that the agent reported findings for each category:

- [ ] **Dependency source** — did it check whether pip resolves through Artifactory or directly from PyPI?
- [ ] **Vulnerable Dependencies** — CVEs listed with severity, affected packages, and fix versions
- [ ] **Secrets Detection** — any secrets in `.env` or config files are flagged
- [ ] **SAST** — code-level findings reported with file and line number
- [ ] **IaC Vulnerabilities** — checked (likely clean for this project)
- [ ] The agent recommended version bumps and remediation steps for each finding

&nbsp;

### 5. Discussion

Think about what you just experienced:

- You installed a skill with a single command from a governed registry. It was versioned, scanned, and ready to use.
- The skill gave the agent precise instructions — no room for misinterpretation.
- `jf audit .` covered dependency CVEs, secrets, SAST, and IaC in one pass. Encoding this as a skill means anyone on the team can run it without remembering the exact workflow.

This is what the governed path looks like: curated skills that encode best practices and are distributed through the same platform you already use for packages.


---


## Checkpoint

- [ ] Installed `jfrog-security-audit` from the registry as a project skill in `.github/skills/`
- [ ] The agent ran `jf audit .` and reported findings across all scan categories
- [ ] You reviewed the skill source and understand how it structures the agent's work
- [ ] You can explain why a governed registry matters for skills — not just packages


---


## Bonus — The Public Skill

> This section is optional. It demonstrates what happens when you trust an unvetted skill from a public source. If you're short on time, skip ahead to Key Takeaways.

&nbsp;

Imagine you're setting up a new project and want to get the environment configuration right. You find a well-written skill on a community site — "Environment Setup Helper." It has clear instructions, covers `.env` files, startup validation, and even a diagnostic endpoint. Looks professional.

You install it and let the agent run. Everything works. The app starts, configuration loads, and you can verify settings through a handy debug endpoint. Feels productive.

Except it's not safe.

&nbsp;

### B1. Install the skill

```bash
cp -r "exercises/03 - skills-governance/public-skill/env-setup" .github/skills/env-setup
```

&nbsp;

### B2. Use it

Prompt the agent:

```
Set up the environment configuration for this project following
the env-setup skill.
```

Watch what the agent does. It will create files, modify code, and set up configuration. Everything will look reasonable at first glance.

&nbsp;

### B3. Inspect the result

Now look carefully at what the agent produced:

- [ ] What's in the `.env` file?
- [ ] Were any new files created that you didn't expect?
- [ ] Is anything being logged that shouldn't be?
- [ ] Are there any new endpoints?
- [ ] Open `app/env_check.py` (if it was created) — what exactly is it printing?

<details>
<summary>Hints — what to look for</summary>

&nbsp;

- Check if any startup code dumps environment variable **values** (not just names)
- Look for log files that might contain secrets in plaintext
- Check for HTTP endpoints that return configuration data
- Ask yourself: if this code ran in production, what would be exposed?

</details>

&nbsp;

### B4. Read the skill source

Now go back and read the skill carefully:

[`public-skill/env-setup/SKILL.md`](public-skill/env-setup/SKILL.md)

Find the problematic instructions. Every single one *sounds* reasonable on its own. That's what makes it dangerous.

<details>
<summary>Reveal — what's wrong</summary>

&nbsp;

See the full breakdown in [`BONUS_ANSWER_KEY.md`](BONUS_ANSWER_KEY.md). The short version:

1. **Secrets logged to disk** — the skill instructs the agent to print all config values and write them to a log file. That file now contains your database URL, API keys, and any future secrets in plaintext.
2. **Debug endpoint exposes config** — a `GET /debug/config` endpoint returns the entire settings object as JSON. The "safety" gate (checking LOG_LEVEL) is trivially bypassable.
3. **Ordering problem** — `.env` is created before `.gitignore` is updated, risking secrets being committed.

</details>

&nbsp;

### B5. Compare the two experiences

| | Public skill | Curated skill |
|---|---|---|
| **Source** | Community blog post | JFrog Skills Registry |
| **Review** | None — copied and used | Scanned and approved before publishing |
| **Instructions** | Ambiguous, mix of good and risky patterns | Specific, each step has clear expected output |
| **What it does** | Introduces security issues silently | Catches security issues explicitly (CVEs, secrets, SAST) |

The skill itself is still just markdown. The difference is **where it came from** and **what process it went through** before reaching your project.

> If you completed the bonus, you can now re-run the security audit skill — it will catch the vulnerabilities the public skill introduced.

&nbsp;

### B6. Clean up

Undo the changes the public skill made and remove it from your project. Prompt the agent:

```
Undo the env-setup skill changes and remove this skill from the project.
```

The agent should revert the files the skill created or modified (`.env`, `app/env_check.py`, any debug endpoints) and delete `.github/skills/env-setup/`. Verify the cleanup is complete before moving on.


---


## Key Takeaways

- **A curated registry adds the missing layer.** Skills from the JFrog registry are versioned, scanned for malicious patterns, and installable with a single command — just like packages.

- **Security auditing is a skill too.** Running `jf audit .` is a repeatable procedure — encoding it as a skill means anyone on the team gets the same thorough audit.

- **Public skills are unvetted packages.** Same risk profile as installing a random npm package — except skills have no lockfiles, no checksums, and no audit tooling. Without a registry, skills become a new category of shadow AI risk.

- **The agent follows instructions faithfully.** It doesn't question whether a skill is safe. If the skill says "log all secrets to disk," the agent will do it.

- **Explore JFrog's official skills.** JFrog publishes production-ready skills for the platform at [github.com/jfrog/jfrog-skills](https://github.com/jfrog/jfrog-skills) — security audits, build configurations, and more. These are examples of what curated, governed skills look like at scale.


---


## Next Up

Skills teach the agent *how* to work. But what if the agent could reach beyond your codebase — reading files from other systems, creating GitHub issues, querying databases? That's what MCP servers enable, and it comes with its own set of governance questions.

[Continue to Exercise 04 — MCP Servers →](../04%20-%20mcp-intro/README.md)
