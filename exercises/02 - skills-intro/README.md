# Exercise 02 — Agent Skills

## What You'll Do

Write a skill that teaches the agent your README standards, install it into your project, watch the agent follow it automatically, and publish it to a governed registry so others can use it too.

**Estimated time:** 20-30 minutes

### At a Glance

1. [Create the skill folder](#1-create-the-skill-folder)
2. [Create the SKILL.md from the template](#2-create-the-skillmd-from-the-template)
3. [Let the agent complete the skill](#3-let-the-agent-complete-the-skill)
4. [Install as a project skill](#4-install-as-a-project-skill)
5. [Open the current README](#5-open-the-current-readme)
6. [Apply the skill](#6-apply-the-skill)
7. [Compare the result](#7-compare-the-result)
8. [Publish to a JFrog Skills Repository](#8-publish-to-a-jfrog-skills-repository)

## Prerequisites

- [Main prerequisites](../../README.md#prerequisites) completed


---


In Exercise 01 you gave the agent specs and it built an app. But the auto-generated `README.md` is probably generic — missing example requests, a project structure overview, or links to the specs. We could fix it by hand. Instead, we'll encode our standard as a **skill** so the agent follows it in every project, every time.

<details>
<summary><strong>What is a skill?</strong></summary>

&nbsp;

A skill is a `SKILL.md` file — a markdown document with YAML frontmatter that teaches the agent a specific procedure.

- **Reusable** across tasks and projects
- **Loaded dynamically** — the agent reads the skill only when it's relevant, keeping context clean
- **Just markdown** — no special tooling, no compilation, no magic

Skills can live in two places:

| Scope | Path | Who it affects |
|-------|------|----------------|
| **Project** | `.github/skills/<skill-name>/SKILL.md` | Anyone working in this repo |
| **User** | `~/.copilot/skills/<skill-name>/SKILL.md` | You, across all your projects |

</details>


---


## Steps

### 1. Create the skill folder

Make sure you're in the `agentic-development-lab/` directory (the workspace root). Create a folder for your skill:

```bash
mkdir -p readme-standard
```

&nbsp;

### 2. Create the SKILL.md from the template

Copy the provided template into your skill folder:

```bash
cp "exercises/02 - skills-intro/skill-template.md" readme-standard/SKILL.md
```

> The `name` field in the frontmatter is the skill's identifier — it must be lowercase letters, digits, and hyphens only (matching `^[a-z0-9][a-z0-9-]*$`). The `description` is what the agent uses to decide whether this skill is relevant — write it like a trigger phrase. The `version` is required for publishing.

&nbsp;

### 3. Let the agent complete the skill

With the SKILL.md file open, prompt:

```
Finish this SKILL.md. Add conventions for tone, formatting, curl example
style, response example style, and what should not be included in a README.
Keep the whole file under 80 lines. Be specific and opinionated — these
are team standards, not suggestions.
```

<details>
<summary>Good conventions to check for</summary>

&nbsp;

- Curl examples use `localhost:8000` with realistic payloads
- Response examples show structure, not every field
- Tone is direct, no marketing fluff
- Formatting uses fenced code blocks with language hints
- There's a clear "what not to include" section

Adjust anything that doesn't match your preferences — this is *your* team standard.

</details>

&nbsp;

### 4. Install as a project skill

Your skill exists at the workspace root, but the agent doesn't know about it yet. Copy it into the project's `.github/skills/` directory to make it visible:

```bash
mkdir -p .github/skills
cp -r readme-standard .github/skills/readme-standard
```

> 📌 By placing the skill in `.github/skills/`, you've made it a **project skill** — active only in this workspace. Open a different project and the agent won't see it. This is the right choice when iterating on a new skill or when the standard is repo-specific. To make a skill follow you across *all* projects, install it as a **user skill** into `~/.copilot/skills/` instead.

&nbsp;

### 5. Open the current README

Before applying the skill, open `weather-app/README.md` (or the root `README.md` if the agent generated one there) and **take a quick look at its current state** — note the structure, what sections exist, and whether it has curl examples or response samples. You'll compare this to the result after the skill is applied.

&nbsp;

### 6. Apply the skill

Regenerate the README. Don't mention the skill by name — the whole point of the `description` field is that the agent should recognize when a skill is relevant and apply it automatically:

```
Regenerate the README.md for this project. Read the current codebase and
docs/SPEC.md to get accurate endpoint details, request/response shapes,
and project structure. Replace the existing README entirely.
```

**Verify the skill was loaded:** Look at the agent's response in the chat panel — VS Code shows which skills were read during the request. You should see your `readme-standard` skill listed. If it's not there, the agent didn't pick it up (see troubleshooting below).

<details>
<summary>Troubleshooting: agent didn't use the skill</summary>

&nbsp;

The agent only activates skills it considers relevant to the current task. If the generated README doesn't follow your conventions:

1. **Is the file in the right place?** Verify it exists at `.github/skills/readme-standard/SKILL.md`.
2. **Is the `description` clear enough?** If it says something vague like "README helper," try "Use when generating or updating a README.md file."
3. **Force it explicitly:**
   ```
   Read .github/skills/readme-standard/SKILL.md and apply those
   conventions to regenerate README.md. Use docs/SPEC.md and the current
   codebase as the source of truth for content.
   ```
4. **Check the context window.** If the conversation is long, the agent may have dropped the skill. Starting a new chat often works.

</details>

&nbsp;

### 7. Compare the result

Open `README.md` again and compare it to what you saw in Step 5. The new version should follow the conventions you defined in your skill — look for:

- Curl examples with realistic payloads against `localhost:8000`
- Response samples showing structure
- A project structure tree with descriptions
- Links to the spec docs
- The tone and formatting rules you specified

If something doesn't match your conventions, refine the SKILL.md instructions and regenerate.

&nbsp;

### 8. Publish to a JFrog Skills Repository

Your skill works locally, but it only exists on your machine. In a real team, you'd share it through a governed registry where skills are versioned, scanned, and discoverable.

First, create a **Skills repository** in Artifactory:

1. Go to **Administration → Repositories → Create a Repository → Local**
2. Select the **Skills** package type
3. Name it `<username>-skills-local` (where `<username>` is your JFrog username — e.g., `johnd-skills-local`)
4. Click **Create Repository**

Then publish your skill using the JFrog CLI:

```bash
jf skills publish readme-standard \
  --repo <username>-skills-local
```

The CLI reads the `name` (used as the slug) and `version` from your SKILL.md frontmatter. If semantic scanning is enabled on the repository, the skill will also be checked for malicious patterns before it becomes available.

Your skill is now versioned, scanned, and installable by anyone with access to your Skills repository. In Exercise 03 you'll install a different skill from the registry using `jf skills install`.


---


## Checkpoint

- [ ] Your SKILL.md is under 80 lines with opinionated conventions for tone, formatting, and examples
- [ ] The skill is installed as a project skill in `.github/skills/readme-standard/`
- [ ] The agent loaded the skill automatically (visible in the chat panel)
- [ ] `README.md` has been regenerated following the skill's conventions
- [ ] The skill is published to your `<username>-skills-local` Skills repository in Artifactory


---


## Key Takeaways

- **Skills are just markdown.** A `SKILL.md` with clear frontmatter and opinionated instructions is all it takes to teach the agent a repeatable standard.

- **The description field is the trigger.** Write it so the agent can match the skill to the right task without you mentioning it by name.

- **Project skills scope to the repo.** Install a skill into `.github/skills/` and it applies only in this workspace — useful for project-specific standards or when iterating on a new skill.

- **A governed registry changes the game.** Publishing to a JFrog Skills Repository means your skill is versioned, scanned for malicious patterns via semantic scanning, and installable with `jf skills install` — the difference between copying a file from Slack and installing a vetted package.


---


## Next Up

You've seen how skills *should* work — created, tested, and published through a governed registry. But what happens when skills come from less trustworthy sources? In the next exercise you'll experience the dark side of skills: unvetted instructions that look helpful but contain hidden dangers.

[Continue to Exercise 03 — Skills Governance →](../03%20-%20skills-governance/README.md)
