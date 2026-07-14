---
name: jfrog-security-audit
version: 1.0.0
description: >
  Use when you need to audit a Python project for security issues.
  Verifies the dependency supply chain is proxied through JFrog Artifactory,
  then runs JFrog Advanced Security on the project directory — covering
  dependency CVEs, secrets detection, SAST, and IaC vulnerabilities
  in a single pass.
---

# JFrog Security Audit

## When to use

- Before merging a PR that changes dependencies or configuration
- After updating pyproject.toml or requirements
- As a pre-release check before deploying
- After onboarding a new skill or adding generated code

## Prerequisites

- JFrog CLI must be authenticated (`jf config show` to verify)
- The project must have a `pyproject.toml` (or `requirements.txt`) so the dependency resolver can build the tree

## Steps

1. Check the dependency source configuration:

   ```bash
   pip config list
   ```

   Inspect the output for `global.index-url` or `global.extra-index-url`.

   - If the index URL points to an Artifactory instance (e.g. contains `/artifactory/api/pypi/`), confirm that the supply chain is properly proxied through JFrog.
   - If the index URL points to `pypi.org` or `files.pythonhosted.org`, or if no custom index is configured, report this as a **High** severity supply-chain risk: dependencies are being resolved directly from the public registry, bypassing Artifactory's proxy, caching, and curation policies. This means packages are not scanned by Xray before installation and curation policies do not apply. Recommend configuring pip to resolve through a JFrog virtual repository.

2. Run the audit from the project root:

   ```bash
   jf audit .
   ```

   This performs four scans in one pass:
   - **Vulnerable Dependencies** — resolves the dependency tree and checks each package against JFrog Xray
   - **Secrets Detection** — scans files for leaked tokens, API keys, and credentials
   - **SAST** — static analysis for code-level security issues
   - **IaC Vulnerabilities** — checks infrastructure-as-code files for misconfigurations

3. Report the results. For each scan category that has findings:

   **Dependency Source (High if unproxied):**
   - Whether pip is configured to resolve through Artifactory or directly from PyPI
   - If direct, flag as a **High** severity supply-chain risk — packages are not scanned or curated before installation
   - Recommend setting the index URL to the project's JFrog virtual repository

   **Vulnerable Dependencies:**
   - CVE ID, severity, direct dependency, affected component and version, fixed version
   - Recommend the minimum version bump that resolves each issue

   **Secrets Detection:**
   - Severity, file path, line number, secret type
   - Recommend removing the secret from the file and using a secrets manager or environment variable instead

   **SAST:**
   - Severity, file path, line number, finding description
   - Recommend a code fix for each finding

   **IaC Vulnerabilities:**
   - Severity, file path, finding description
   - Recommend the configuration change needed

4. Summarize total counts by severity across all categories (critical, high, medium, low).

5. If no findings in any category, confirm the project is clean.

## Conventions

- Never skip or suppress findings from any category
- Report all severity levels — don't filter out LOW
- If the audit fails due to authentication, verify with `jf config show`
  and re-authenticate before retrying
- Always recommend the minimum version bump that resolves dependency issues,
  not just "upgrade to latest"
- For secrets findings, never print the full secret value — use the masked
  form from the scan output
