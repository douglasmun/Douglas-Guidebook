# 🤖 GitHub Agentic Workflows: From Scripts to Autonomous Intelligence

## Companion Document to *Git & GitHub Automation: Complete Textbook*

**Version:** 1.0 (June 2026)
**Skill Level:** Intermediate to Advanced
**Prerequisites:** Git & GitHub Automation Textbook (v4), working CI/CD pipeline
**Outcome:** Understand and build secure, AI-driven agentic workflows using the `gh aw` toolchain

**Author:** Douglas Mun | **License:** Educational Use

> **Status note:** GitHub Agentic Workflows are in **public / technical preview** and subject to change. This document reflects the official documentation as of June 2026. Always verify command syntax and configuration against the live docs at <https://github.github.com/gh-aw/> before relying on it in production.

---

## 📚 Table of Contents

- [Chapter 1: The Paradigm Shift — From Scripts to Agents](#chapter-1-the-paradigm-shift)
- [Chapter 2: How the Toolchain Works](#chapter-2-how-the-toolchain-works)
- [Chapter 3: Natural Language Automation](#chapter-3-natural-language-automation)
- [Chapter 4: What Agents Do — The Use Cases](#chapter-4-what-agents-do)
- [Chapter 5: The Security Architecture](#chapter-5-the-security-architecture)
- [Chapter 6: Quickstart — Your First Agentic Workflow](#chapter-6-quickstart)
- [Chapter 7: Authoring Your Own Workflows](#chapter-7-authoring-your-own-workflows)
- [Chapter 8: Capstone — Fusing Both Books](#chapter-8-capstone--fusing-both-books)
- [Quick Reference & Glossary](#quick-reference--glossary)
- [Sources](#sources)

---

# Chapter 1: The Paradigm Shift

## From Static Scripts to Autonomous Decision-Making

The previous textbook taught you to build CI/CD pipelines: structured, deterministic sequences of commands that run the same way every time. Push code → test → lint → deploy. Every outcome was predictable because every instruction was explicit.

GitHub Agentic Workflows change that contract. They let you automate repository tasks using AI agents that run within GitHub Actions — you write workflows in plain Markdown instead of complex YAML, and let AI handle intelligent decision-making for issue triage, pull request reviews, CI failure analysis, and repository maintenance.

Instead of you specifying every command, you describe an *objective* — and the agent determines *how* to achieve it.

```
Traditional Pipeline (what you learned before):
  Trigger → Step 1 → Step 2 → Step 3 → Done
  (author specifies every step in YAML)

Agentic Workflow (what this document teaches):
  Trigger → Agent reads context → Agent decides actions
         → Agent executes via tools → Buffered outputs
         → Security checks → Controlled write actions → Done
  (author describes the goal in Markdown; agent figures out the path)
```

This is not a concept sketch — it ships as a concrete, open-source tool. You add Markdown files to `.github/workflows/` describing automation goals in natural language; the `gh aw` CLI converts these into standard GitHub Actions workflows that execute using GitHub Copilot CLI or another coding agent. Critically for safety, workflows run with read-only permissions by default and use preapproved "safe outputs" for any write. The feature is in technical preview, built by GitHub Next, Microsoft Research, and Azure Core Upstream, and is open source under the MIT license.

### Why This Matters

Consider a build failure. A traditional pipeline stops, reports failure, and waits for a human. An agentic workflow can read the error, investigate the codebase, and propose a fix as a reviewable pull request — within the security boundaries described in Chapter 5. The broader vision is repository automation that triages incoming issues, investigates root causes of CI failures, maintains documentation, improves test coverage, and monitors compliance — all defined in simple Markdown files.

The difference is not one of raw power — it is one of *autonomy under guardrails*.

---

# Chapter 2: How the Toolchain Works

## The `gh aw` Extension and the Two-File Model

The single most important thing to understand — and the thing most people get wrong — is that agentic workflows are **not** authored as standard Actions YAML. They use a dedicated toolchain built around a GitHub CLI extension called `gh aw`.

### The `gh aw` CLI Extension

Everything starts with installing the extension:

```bash
gh extension install github/gh-aw
```

Prerequisites are an AI account (GitHub Copilot, Anthropic Claude, OpenAI Codex, or Google Gemini), a GitHub repository where you have write access, GitHub Actions enabled, GitHub CLI v2.0.0 or later installed and authenticated, and an operating system of Linux, macOS, or Windows with WSL.

If you hit authentication trouble, an install script is available: `curl -sL https://raw.githubusercontent.com/github/gh-aw/main/install-gh-aw.sh | bash`.

### Where Do the AI API Keys Live?

A natural question once you have an AI account: where does the API key actually go? The answer is that **the key never lives in your workflow files**. It is stored as a **GitHub Actions repository secret**, and the runtime injects it into the right container at execution time. You configure exactly one secret per engine:

| Engine | Secret name | Where the key comes from |
|---|---|---|
| Copilot (default) | `COPILOT_GITHUB_TOKEN` | A fine-grained GitHub PAT — Account permissions → Copilot Requests: Read, owned by your **personal** account |
| Claude | `ANTHROPIC_API_KEY` | The Anthropic Console |
| Codex | `OPENAI_API_KEY` (or `CODEX_API_KEY`) | The OpenAI platform |
| Gemini | `GEMINI_API_KEY` | Google AI Studio |

Most workflows need nothing beyond this single engine secret. You can set it three ways — via the CLI, the bootstrap helper, or the GitHub web UI:

```bash
# Set a specific secret directly
gh aw secrets set ANTHROPIC_API_KEY --value "YOUR_ANTHROPIC_API_KEY"

# Or let the bootstrap command detect which secrets your workflows
# need and prompt only for the ones that are missing
gh aw secrets bootstrap
```

In the web UI, the same thing lives under **Settings → Secrets and variables → Actions → New repository secret**.

Three things worth knowing. First, the Copilot token must be a fine-grained PAT owned by a personal account — GitHub Apps and OAuth tokens are not accepted for `COPILOT_GITHUB_TOKEN`. Second, for Claude, `CLAUDE_CODE_OAUTH_TOKEN` is **not** supported; the only accepted credential is `ANTHROPIC_API_KEY`, and a stray OAuth token is ignored. Third — and this is the security payoff — because the key is a GitHub Actions secret rather than a value in the `.md` or `.lock.yml`, it is never committed to your repository, and the secret-redaction layer (Chapter 5) scrubs it from logs and artifacts. For non-Copilot engines you can go further and replace the PAT with a GitHub App for short-lived, automatically-revoked tokens.

### The Two-File Model: `.md` and `.lock.yml`

Every agentic workflow consists of two files that live together in `.github/workflows/`: the `.md` file you edit and a generated `.lock.yml` that GitHub Actions runs. `gh aw compile` regenerates the `.lock.yml` from your markdown whenever you change the configuration.

```
.github/workflows/
├── daily-repo-status.md        ← YOU edit this (Markdown + frontmatter)
└── daily-repo-status.lock.yml  ← GENERATED by `gh aw compile` (Actions runs this)
```

Think of it like compiling source code: you write the human-friendly `.md` source; the compiler produces the machine-executable `.lock.yml` artifact. You commit both.

### Anatomy of a Workflow Markdown File

The `.md` file has two parts. The frontmatter — the configuration block between the `---` markers at the top — controls when the workflow runs, which AI engine powers it, and what tools it can access; changes here require recompiling. The body below contains your natural-language task description; changes there take effect immediately on the next run.

```markdown
---
# ─── FRONTMATTER (configuration) ───
# Changes here require `gh aw compile`
on:
  issues:
    types: [opened]
engine: copilot
network:
  firewall: true
  allowed:
    - defaults
safe-outputs:
  add-labels:
  add-comment:
---

# ─── BODY (natural language task) ───
# Changes here take effect on next run, no recompile needed

A new issue has been opened. Read its title and body, then apply an
appropriate label and post a brief, friendly comment explaining what
the author can expect next.
```

### The Compile Step

Whenever you change the frontmatter, you must recompile:

```bash
gh aw compile
```

This generates the lock file from the workflow frontmatter, and includes schema validation, expression safety checks, action pinning, and security scanning. You can opt into additional scanners:

```bash
gh aw compile --actionlint --zizmor --poutine   # run scanners; findings reported as warnings
gh aw compile --strict --zizmor                 # strict mode: fails the build on findings
```

By default these scanners surface findings as warnings; combining them with `--strict` makes the build fail on findings. (The separate `gh aw validate` command runs all linters with `--no-emit`, so you can check a workflow without regenerating its lock file.)

### What's Inside the `.lock.yml`?

You author the `.md`, but it is worth seeing what the compiler hands to GitHub Actions, because the `.lock.yml` is where all the security hardening becomes concrete. It is a standard GitHub Actions workflow — but a large, machine-generated one. A real compiled file such as the project's own `ci-doctor.lock.yml` runs to well over a thousand lines, which is exactly why you never write or edit it by hand.

Every lock file opens with a header the compiler controls. It begins with a single machine-readable metadata line, followed by an ASCII banner, an explicit **DO NOT EDIT** notice, and a human-readable manifest of every external dependency the workflow pulls in:

```yaml
# gh-aw-metadata: {"schema_version":"v3","frontmatter_hash":"a1b2c3…","strict":true,"agent_id":"copilot"}
#
#      ___                    _   _
#     / _ \                  | | (_)
#    | |_| | __ _  ___ _ __  | |_ _  ___
#    |  _  |/ _` |/ _ \ '_ \ | __| |/ __|
#    | | | | (_| |  __/ | | || |_| | (__
#    \_| |_/\__, |\___|_| |_| \__|_|\___|
#            __/ |
#           |___/   Workflows
#
# This file was automatically generated by gh-aw. DO NOT EDIT.
# To update this file, edit the corresponding .md file and run:
#   gh aw compile
#
# Source: .github/workflows/issue-triage.md
#
# Secrets used:
# - COPILOT_GITHUB_TOKEN
# - GITHUB_TOKEN
#
# Custom actions used:
# - actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
# - actions/github-script@3a2844b7e9c422d3c10d287c895573f7108da1b3 # v9.0.0
```

The `gh-aw-metadata` line is always first so tooling can parse it reliably; the `Secrets used` and `Custom actions used` sections list every `secrets.*` reference and external `uses:` dependency found in the compiled workflow, sorted and deduplicated.

Below the header is ordinary Actions YAML — but notice the hardening the compiler injected that you never had to write. Here is a trimmed, representative skeleton:

```yaml
name: "Issue Triage"

on:
  issues:
    types: [opened, reopened]

# Tightly scoped at the top level; individual jobs narrow further.
permissions: {}

concurrency:
  group: "gh-aw-${{ github.workflow }}-${{ github.event.issue.number }}"

jobs:
  # 1) Pre-flight: checks roles, reactions, stop-after deadline, etc.
  pre_activation:
    runs-on: ubuntu-slim
    permissions:
      contents: read
    # ...

  # 2) The agent itself — READ-ONLY. This is the only job that calls the model.
  agent:
    needs: pre_activation
    runs-on: ubuntu-latest
    permissions:
      contents: read          # no write scopes anywhere here
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          persist-credentials: false
      # ... sets up the firewall, MCP gateway, runs the agent, and
      #     writes requested actions to agent_output.json (an artifact) ...

  # 3) Threat detection — scans the buffered output. No write scopes.
  detection:
    needs: agent
    runs-on: ubuntu-slim
    # ... downloads agent_output.json, scans for secret leaks / bad patches ...

  # 4) Safe-output jobs — the ONLY jobs with write scopes, and only the
  #    narrow scopes each operation needs. Run after detection passes.
  add_labels:
    needs: detection
    runs-on: ubuntu-slim
    permissions:
      issues: write
    # ...
  add_comment:
    needs: detection
    runs-on: ubuntu-slim
    permissions:
      issues: write
    # ...
```

Three things in that skeleton are the whole security story made literal: the `agent` job has only `contents: read`; write permissions appear exclusively in the safe-output jobs at the bottom; and every `uses:` is pinned to a full commit SHA rather than a mutable tag. You get all of this for free from compiling — which is also why both files get committed, so reviewers can read exactly what will run.

### Why a Compiler at All?

The compiler is not just a convenience — it is a **security boundary**. AW enforces security constraints at compilation time through schema validation, expression allowlisting, and action pinning; the trusted compiler validates declarative configuration artifacts before they are deployed, rejecting misconfigurations and overly permissive specifications. We return to this in Chapter 5.

---

# Chapter 3: Natural Language Automation

## Describing Objectives Instead of Commands

The visible payoff of the two-file model is that the part you write and maintain — the body of the `.md` file — is plain natural language.

You write automation in Markdown instead of YAML: describe what you want in natural language, and the AI agent figures out how to do it.

### Three Ways to Create a Workflow

The official docs describe several authoring paths. You can author new agentic workflows using a coding agent or other AI chat system under your guidance; for interactive coding agents, this can be a conversation about what you want the workflow to do, with the agent asking clarifying questions and generating the workflow for you.

**1. From the GitHub web interface (with Copilot).** If you have access to GitHub Copilot, you can create and edit agentic workflows directly from the web interface; this technique is slower and non-interactive but useful to turn an idea into reality in a couple of minutes. You paste a prompt referencing the `create.md` helper, for example:

```text
Create a workflow for GitHub Agentic Workflows using
https://raw.githubusercontent.com/github/gh-aw/main/create.md

The purpose of the workflow is to triage new issues: label them by type
and priority, identify duplicates, ask clarifying questions when the
description is unclear, and assign them to the right team members.
```

**2. From a coding agent or VS Code Agent Mode.** You start your preferred coding agent in the context of your repository and enter a prompt referencing the same `create.md` file, replacing the last line with your desired workflow purpose and as much additional detail, context, goals, and guardrails as you like; this creates a new workflow markdown file in `.github/workflows/`, and some agents will open a pull request to add it.

**3. Manual editing.** You create the workflow file at `.github/workflows/<workflow-name>.md`, install the GitHub CLI and the `gh-aw` extension, then compile the markdown into a YAML lock file with `gh aw compile`, which generates `.github/workflows/<workflow-name>.lock.yml` based on your markdown. Then commit and push both files.

### Initializing the Repository for Agentic Authoring

To enable in-context authoring directly from github.com or the mobile app, you run an init step. Running `gh aw init` is required to enable the authoring experience in the GitHub code agent; it configures your repository so you can create and modify agentic workflows directly from github.com or the GitHub mobile app using the Copilot coding agent.

This command creates a dispatcher agent at `.github/agents/agentic-workflows.agent.md` (which registers the `agentic-workflows` custom agent in GitHub Copilot), sets up MCP server integration so Copilot has access to gh-aw tools, updates `.gitattributes` to mark generated `.lock.yml` files correctly, and configures VS Code settings for local authoring.

Once initialized, you and your team can create and edit workflows by opening a Copilot Chat session on github.com or the GitHub app and running `/agent agentic-workflows Create a new workflow that...`.

### Writing Good Objectives

The body is where you describe the task. Good bodies share three properties:

**1. State what success looks like, not the mechanism.** Let the agent determine the steps. Your job is to define the destination and the boundaries.

**2. Include explicit constraints.** Tell the agent what *not* to do — for example, "do not close any issues, only add labels and comments." Constraints in the body shape behavior at runtime.

**3. Define style and process.** In the daily-status sample, you can edit the "What to include" section to list things you struggle with regularly in your repository — issue backlog, CI setup, testing, performance, roadmap — and you can also customize the style and process sections to guide the coding agent's behavior.

### Natural Language Body, but YAML Frontmatter

There is an important nuance to "you write automation in Markdown instead of YAML." That is true of the **body** — the task description is prose. But the **frontmatter** at the top of the file, between the `---` markers, is strict YAML, and that is where most authoring friction actually happens. The agent is forgiving about how you phrase a task; the YAML parser is not forgiving about a stray tab or an unquoted value.

So it pays to be deliberate about the YAML half. Here is a clean, well-formed frontmatter block to anchor on:

```yaml
---
on:
  issues:
    types: [opened, reopened]
permissions:
  contents: read
engine: copilot
timeout-minutes: 15
tools:
  github:
    toolsets: [issues, repos]
network:
  firewall: true
  allowed:
    - defaults
safe-outputs:
  add-labels:
    allowed: [bug, question, duplicate]
    max: 3
  add-comment:
    max: 1
---
```

Two structural rules cover most of it. First, indentation is **spaces only** — never tabs — and each level is two spaces; a nested key like `toolsets:` sits under `github:`, which sits under `tools:`. Second, every key is followed by a colon **and a space** before its value (`engine: copilot`, not `engine:copilot`).

### Troubleshooting YAML Frontmatter

When a workflow won't compile, or compiles but ignores a setting, the cause is almost always one of a handful of YAML issues. The table maps the symptom to the fix:

| Symptom | Likely cause | Fix |
|---|---|---|
| `gh aw compile` fails with a parse error | A tab character, or inconsistent indentation | Use spaces only, two per level; configure your editor to show whitespace |
| A setting seems silently ignored | Misspelled field name — the compiler discards unknown fields without warning | Check the field name against the reference; run `gh aw compile --verbose` to see what actually parsed |
| `engine:` has no effect | Wrong key name (`agent:` instead of `engine:`) | Use `engine:` |
| `timeout` not respected | Wrong key (`timeout:` instead of `timeout-minutes:`) | Use `timeout-minutes:` |
| MCP/tools config ignored | `mcp-servers:` or `tool-sets:` used at the wrong level | MCP servers go under `tools:`; the key is `toolsets:` under `tools.github:` |
| A value like `+30d`, a cron string, or `on`/`no` behaves oddly | YAML is interpreting an unquoted scalar as the wrong type | Quote values that look like dates, expressions, or booleans: `stop-after: "+30d"` |
| Compile error mentions a missing required field | No `on:` trigger declared | Every workflow needs an `on:` block |

The two traps worth seeing directly are the ones the table can only hint at. The most common is the unquoted-scalar trap — YAML will read a value as a number, date, or boolean when you meant a string — and the second is the silently-ignored field name, where a typo looks exactly like the feature not working because unknown keys are discarded rather than flagged:

```yaml
# ✗ Risky / silently wrong
on:
  stop-after: +30d        # parsed as an odd scalar, not a duration
labels: [v1.0]            # 1.0 can be read as a float
agent: copilot            # unknown key — discarded; should be `engine`
timeout: 15               # unknown key — discarded; should be `timeout-minutes`

# ✓ Safe: quote ambiguous scalars, use the correct keys
on:
  stop-after: "+30d"
labels: ["v1.0"]
engine: copilot
timeout-minutes: 15
```

Three commands handle nearly all YAML debugging. Use `gh aw compile --verbose` to confirm which settings were parsed (if your setting is missing from the output, the field name is wrong); pipe through `gh aw compile 2>&1 | grep -i error` to isolate just the errors; and run `gh aw validate` to check a workflow without regenerating its lock file. For deep inspection of how the frontmatter was read, `DEBUG=parser:frontmatter gh aw compile` prints the parser's internal view.

---

# Chapter 4: What Agents Do

## The Core Use Cases

The official material centers on a recurring set of high-value applications. GitHub Agentic Workflows handle intelligent decision-making for issue triage, pull request reviews, CI failure analysis, and repository maintenance. The creating-workflows guide ships four canonical starter prompts that map directly to these.

### 4.1 Issue Triage

The purpose of an issue-triage workflow is to triage new issues: label them by type and priority, identify duplicates, ask clarifying questions when the description is unclear, and assign them to the right team members.

This is one of the highest-value applications because it is repetitive, labor-intensive, and requires reading comprehension rather than code execution. The agent reads the issue, reasons about its category, checks for duplicates, and applies labels and comments — all expressed through safe outputs (Chapter 5).

Here is what a complete issue-triage workflow looks like as a `.md` file. Notice that the frontmatter grants only `contents: read` to the agent, and that every write capability is declared under `safe-outputs:` — the agent itself never gets `issues: write`. The body is the natural-language task.

```markdown
---
on:
  issues:
    types: [opened, reopened]

# The agent runs read-only. It cannot write to the repo.
permissions:
  contents: read

engine: copilot

# Write capabilities are declared here. Each runs in its own
# scoped, validated job AFTER the agent finishes — never inside it.
safe-outputs:
  add-labels:
    allowed: [bug, feature-request, question, duplicate, needs-more-info]
    max: 3
  add-comment:
    max: 1
---

# Issue Triage

A new issue has just been opened. Read its title and body carefully.

1. Apply exactly ONE category label from the allowed set:
   - `bug` — a confirmed or highly likely software defect
   - `feature-request` — a request for new capability
   - `question` — a support request or request for clarification
   - `duplicate` — search existing open issues first; if a clear match
     exists, apply this label and reference the original in your comment
   - `needs-more-info` — the report is too vague to act on

2. Post a brief, friendly comment (2–3 sentences) explaining the label
   you chose and what the author can expect to happen next.

Do NOT close the issue. Do NOT assign it to anyone. If you genuinely
cannot determine a category, apply `needs-more-info` and ask one
specific clarifying question in your comment.
```

After writing this to `.github/workflows/issue-triage.md`, you compile it with `gh aw compile`, commit both files, and every newly opened issue is triaged automatically. The same shape — read-only frontmatter, a `safe-outputs:` block, a natural-language body — underlies all four use cases below; only the trigger, the declared safe outputs, and the task description change.

### 4.2 Activity Reporting

An activity-report workflow produces a daily report on recent activity in the repository, delivered as an issue, summarizing new issues, pull requests merged, and any open blockers.

This is the workflow used in the official quickstart. Once a run completes, a new issue is created in your repository with a "Daily Repo Report" that analyzes recent repository activity (issues, PRs, discussions, releases, code changes), progress tracking, goal reminders and highlights, project status and recommendations, and actionable next steps for maintainers.

### 4.3 Documentation Maintenance

A documentation-updater workflow runs daily and keeps the repository documentation up to date: it identifies doc files that are out of sync with recent code changes and opens a pull request with the necessary updates.

Documentation rot — docs falling behind code — is chronic in software projects. Because the agent's write path is a pull request through safe outputs, a human always reviews the proposed changes before they land.

This use case introduces a different safe output: `create-pull-request`. Where triage emits labels and comments, here the agent proposes code changes — but still never pushes directly. The change is buffered, threat-scanned, and applied as a reviewable PR by a separate job. Only the `safe-outputs:` block differs from the triage shape above:

```yaml
on:
  schedule:
    - cron: "0 9 * * 1-5"   # weekdays at 09:00 UTC
  workflow_dispatch:
permissions:
  contents: read
engine: copilot
safe-outputs:
  create-pull-request:
    title-prefix: "[docs] "
    labels: [documentation, automated]
    draft: true               # open as draft for human review
```

The body would instruct the agent to compare recently merged code against `README.md` and `docs/`, make only factual updates that follow from the changes, and take no action if nothing is out of sync. Two policy details matter here: `draft` is enforced — whatever you set in frontmatter is always used, and the agent cannot override it at runtime — and by default, PRs opened this way do not trigger CI (see the official *Triggering CI* reference if you need the test suite to run).

### 4.4 AGENTS.md Maintenance

An AGENTS.md-maintainer workflow runs weekly: it reviews merged pull requests and updated source files since the last run, then opens a pull request that keeps AGENTS.md accurate and current.

### 4.5 Beyond the Starters

These four are starting points, not limits. The vision extends to automation that improves test coverage, monitors compliance, and even boosts team morale — and Peli's Agent Factory showcases over 50 specialized agentic workflows for different use cases.

The official documentation organizes these into named **design patterns** — including IssueOps, LabelOps, DailyOps, ChatOps, and many more — each describing a reusable shape of agentic automation.

---

# Chapter 5: The Security Architecture

## Defense in Depth for Autonomous Agents

Agent autonomy creates attack surfaces a static script never had — most notably **prompt injection**, where an attacker hides instructions inside content the agent reads (an issue body, a PR description, a comment). GitHub Agentic Workflows answer this with a defense-in-depth architecture that protects against untrusted Model Context Protocol (MCP) servers and compromised agents. It combines substrate-enforced isolation, declarative specification, and staged execution; each layer enforces distinct security properties under different assumptions and constrains the impact of failures above it.

### The Threat Model

The architecture is explicit about what it defends against. It considers an adversary that may compromise untrusted user-level components, e.g., containers, and cause them to behave arbitrarily within their granted privileges. The adversary may attempt to access or corrupt the memory or state of other components, communicate over unintended channels, abuse legitimate channels to perform unintended actions, or confuse higher-level control logic by deviating from expected workflows. It is assumed the adversary does not compromise the underlying hardware or cryptographic primitives, and side-channel and covert-channel attacks are out of scope.

### The Three Trust Layers

The model is organized into three layers of trust, each constraining failures in the layer above. The next three subsections walk through them, with a sample showing how each layer shows up in a workflow you actually write.

#### 5.1 Substrate-Level Trust

Agentic Workflows run on a GitHub Actions runner virtual machine and trust Actions' hardware and kernel-level enforcement, including the CPU, MMU, kernel, and container runtime. They also rely on three privileged containers: a network firewall (trusted to configure connectivity via iptables and launch the agent container), an API proxy (which routes model traffic and may hold endpoint credentials), and an MCP Gateway (trusted to configure and spawn isolated MCP-server containers). Collectively, the substrate level ensures memory isolation, CPU and resource isolation, mediation of privileged operations, and kernel-enforced communication boundaries — even if an untrusted component is fully compromised.

You don't configure the substrate directly — it's the foundation GitHub provides. But it's *why* the network allowlist below actually holds: the firewall container that enforces it runs at this trusted layer, outside the agent's reach. The one substrate control you do shape from a workflow is network egress:

```yaml
# This allowlist is enforced by a privileged firewall container
# the agent cannot tamper with — that is what makes it a real boundary.
network:
  firewall: true
  allowed:
    - defaults   # certificates, basic infrastructure
    - node       # npm registry only
    # Anything not listed here is dropped at the proxy.
```

If this layer fails, higher-level guarantees may not hold — which is why it depends only on hardware, the kernel, and the container runtime, not on anything the agent influences.

#### 5.2 Configuration-Level Trust

AW trusts declarative configuration artifacts — Action steps, network-firewall policies, MCP server configurations — and the toolchains that interpret them. This layer constrains which components are loaded, how they are connected, which channels are permitted, and what privileges are assigned. Externally minted tokens such as agent API keys and GitHub access tokens are treated as imported capabilities that bound components' external effects, with declarative configuration controlling which tokens load into which containers.

This layer is the frontmatter you write, hardened by the compiler. When you compile with `--strict`, the configuration is rejected unless it follows least-privilege rules — no broad write permissions, explicit network config, no wildcard domains, pinned actions:

```yaml
---
# Configuration-level trust: the agent gets read-only, full stop.
permissions:
  contents: read        # no issues: write, no contents: write here
engine: copilot
network:
  allowed: [defaults]   # explicit; strict mode forbids wildcards like "*"
safe-outputs:
  add-labels:           # write capability declared as data, granted to a
    allowed: [bug, question]   # separate job — not to the agent
---
```

Security violations here arise from misconfiguration or overly permissive specifications — which is exactly what `gh aw compile --strict` and the scanners (actionlint, zizmor, poutine) exist to catch before deployment.

#### 5.3 Plan-Level Trust

AW relies on plan-level trust to constrain component behavior over time: a trusted compiler decomposes a workflow into stages, and for each stage the plan specifies which components are active and their permissions, the data produced, and how that data may be consumed by subsequent stages — ensuring important external side effects are explicit and vetted. The primary instantiation of this layer is the SafeOutputs subsystem, and plan-level trust limits the blast radius of a compromised component to the stage in which it is active.

This is the layer that turns "the agent decided to do X" into "X was buffered, inspected, and only then applied by a separate job." The `safe-outputs:` block from 5.2 is the author-facing surface of this layer; the staged execution it produces is detailed in *Safe Outputs: The Key Idea* immediately below. Plan-level violations come from incorrect plan construction or overly permissive stage definitions — but even then, a compromised stage can only affect the artifacts it passes to the next stage, not reach back into the substrate.

### Safe Outputs: The Key Idea

The single most important mechanism in the model: **the agent never has direct write access.** The agent job runs with minimal read-only permissions, and write operations are deferred to separate jobs that execute only after the agent completes — so even a fully compromised agent cannot directly modify repository state.

The flow works like this:

```
Agent Job (READ-ONLY permissions)
   │  Agent can read repo, issues, PRs via read-only MCP servers
   │  Wants to "create a PR"? It does NOT call the write API.
   │  Instead it writes a buffered action to agent_output.json (an artifact)
   ▼
Threat Detection Job (NO write permissions)
   │  Downloads the buffered artifact
   │  Analyzes for secret leaks, malicious patches, policy violations
   │  Emits a pass/fail verdict
   ▼
Safe Output Jobs (WRITE permissions, narrowly scoped, run only if "safe")
   │  create_issue        → issues: write
   │  add_comment         → issues: write
   │  create_pull_request → contents: write, pull-requests: write
   │  add_labels          → issues: write
   ▼
GitHub API
```

An agent can interact with read-only MCP servers such as the GitHub MCP server, but externalized writes like creating pull requests are buffered as artifacts rather than applied immediately. When the agent finishes, these artifacts pass through a deterministic sequence of filters and analyses — structural limits (e.g., a cap on the number of pull requests), policy enforcement, and sanitization to ensure tokens are not exported — before a later stage externalizes them. The net effect: the agent never requires write permissions, because all writes are performed by separate, validated jobs with minimal scoped permissions.

> **A naming note for when you write your own workflows.** In architecture diagrams the safe-output *jobs* appear with underscores (`create_issue`, `add_labels`). In the frontmatter you actually author, the corresponding *configuration keys* use hyphens — `create-issue`, `add-comment`, `create-pull-request`, `add-labels` — as in the Chapter 2 example. Same concepts, two spellings depending on whether you are looking at a generated job or a config key.

### The Agent Workflow Firewall (AWF)

The Agent Workflow Firewall prevents a compromised agent from exfiltrating data. It containerizes the agent, binds it to a Docker network, and uses iptables to redirect HTTP/HTTPS traffic through a Squid proxy container; the proxy controls egress via a configurable domain allowlist, restricting a compromised agent to permitted domains.

You configure the allowlist in frontmatter using ecosystem bundles:

```yaml
engine: copilot
network:
  firewall: true
  allowed:
    - defaults          # Basic infrastructure, certificates
    - python            # PyPI ecosystem
    - node              # npm ecosystem
    - "api.example.com" # Custom domain
```

AWF separates two concerns: filesystem (controlled access to host binaries and runtimes via chroot) and network (all traffic routed through the proxy enforcing the domain allowlist).

### Content Sanitization

User-generated content is sanitized before being passed to the agent; the pipeline operates at the activation stage boundary, so untrusted input is processed before it ever reaches the agent. The transformations include:

| Mechanism | Example | Protection |
|---|---|---|
| @mention neutralization | `@user` → `` `@user` `` | Prevents unintended user notifications |
| Bot trigger protection | `fixes #123` → `` `fixes #123` `` | Prevents automatic issue linking |
| XML/HTML tag conversion | `<script>` → `(script)` | Prevents injection via XML tags |
| URI filtering | `http://evil.com` → `(redacted)` | Restricts to HTTPS from trusted domains |
| Special characters | Unicode homoglyphs → normalized | Prevents visual spoofing attacks |
| Content limits | Large payloads → truncated | Enforces 0.5MB max size, 65k lines max |
| Control characters | ANSI escapes → stripped | Removes terminal manipulation codes |

### Threat Detection

A dedicated, isolated job inspects the agent's buffered output before any write happens. The threat detection job is a sub-stage within SafeOutputs: after the agent job completes and its outputs are buffered as artifacts, a separate detection job downloads these artifacts and invokes a prompted AI agent to analyze them for suspicious content. This detection agent runs in isolation from the original agent, has no write permissions, and its sole responsibility is to emit a pass/fail verdict that gates the subsequent safe output jobs; if it identifies threats, the workflow terminates before any writes are externalized.

Detection checks include identification of secret leakage, malicious code patterns, and policy violations, and authors can customize behavior by providing additional detection prompts or integrating external security scanners. For example:

```yaml
threat-detection:
  prompt: |
    Additionally check for:
    - References to internal infrastructure URLs
    - Attempts to modify CI/CD configuration files
  steps:
    - name: Run TruffleHog
      run: trufflehog filesystem /tmp/gh-aw --only-verified
    - name: Run Semgrep
      run: semgrep scan /tmp/gh-aw/aw.patch --config=auto
```

### Compilation-Time Security

Before a workflow ever runs, the compiler hardens it. AW enforces security constraints at compilation time through schema validation, expression allowlisting, and action pinning, rejecting misconfigurations and overly permissive specifications. The compile step runs security scanners: actionlint for workflow linting, zizmor for security vulnerabilities and privilege escalation, and poutine for supply-chain risks from third-party actions. Strict-mode enforcement rejects write permissions, requires explicit network config, forbids wildcard domains, and forbids deprecated fields.

### Integrity Filtering and Secret Redaction

Two more plan-level controls round out the model. Integrity filtering controls which GitHub content an agent can access based on author trust and merge status rather than push access alone; for public repositories, `min-integrity: approved` is applied automatically, restricting content to owners, members, and collaborators.

And outputs are scrubbed of secrets unconditionally. Before workflow artifacts are uploaded, all files in the `/tmp/gh-aw` directory are scanned for secret values and redacted; this executes with `if: always()`, ensuring secrets are protected even if the workflow fails at an earlier stage.

### Security Layers Summary

| Layer | Mechanism | Protects Against |
|---|---|---|
| Substrate | GitHub Actions VM (kernel, hypervisor) | Memory corruption, privilege escalation, host escape |
| Substrate | AWF network controls (iptables) | Data exfiltration, unauthorized API calls |
| Substrate | MCP sandboxing (container isolation) | Container escape, unauthorized tool access |
| Configuration | Action SHA pinning | Supply chain attacks, tag hijacking |
| Configuration | Security scanners | Privilege escalation, misconfigurations, supply-chain risks |
| Plan | Integrity filtering | Untrusted user input, context poisoning, social engineering |
| Plan | Content sanitization | @mention abuse, bot triggers |
| Plan | Secret redaction | Credential leakage in logs/artifacts |
| Plan | Threat detection | Malicious patches, secret leaks |
| Plan | SafeOutputs permission separation | Direct write access abuse |

> **A candid caveat from the docs.** The MCP gateway key mounted into the agent container is not a hard boundary. An agent running arbitrary code may extract the key from process memory or runtime state; treat this key as leaked by design and rely on substrate isolation, network policy, and staged permission separation for security. This honesty is a feature — the model assumes the agent *can* be compromised and is built to contain the blast radius anyway.

---

# Chapter 6: Quickstart

## Adding a Daily Status Workflow in About 10 Minutes

This follows the official quickstart, which adds a pre-built **Daily Repo Status Report** workflow to a repository where you are a maintainer.

### Prerequisites

You need an AI account (Copilot, Claude, Codex, or Gemini), a repository with write access, GitHub Actions enabled, GitHub CLI v2.0.0+, an authenticated CLI session (verify with `gh auth status`, and run `gh auth login --scopes repo,workflow` if needed), and Linux, macOS, or Windows with WSL.

### Step 1 — Install the Extension

```bash
gh extension install github/gh-aw
```

### Step 2 — Add the Sample Workflow

From your repository root:

```bash
gh aw add-wizard githubnext/agentics/daily-repo-status
```

The `add-wizard` command accepts workflow references in `<owner>/<repo>/<workflow-name>` format; here `githubnext/agentics/daily-repo-status` points to the `daily-repo-status.md` workflow in the `githubnext/agentics` repository. The interactive process will check prerequisites, let you select an AI engine (Copilot, Claude, Codex, or Gemini), set up the required secret, add the workflow `.md` file and its generated `.lock.yml` file to `.github/workflows/`, and optionally trigger an initial run.

On engine secrets: depending on the engine you choose, the wizard may prompt you to configure `COPILOT_GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GEMINI_API_KEY`. Note that `COPILOT_GITHUB_TOKEN` is a separate GitHub token with Copilot access, distinct from the default `GITHUB_TOKEN`.

### Step 3 — Wait for the Run

An automated workflow run can take 2–3 minutes. Once complete, a new issue is created in your repository with a "Daily Repo Report" analyzing recent activity, progress tracking, project status, and actionable next steps.

> **Expect the first run to fail in a fresh repo.** On the first run in a new repository, the workflow will surely fail because the secrets are not configured; the agentic workflow should detect the missing tokens and create an issue with instructions on how to configure them.

### Step 4 — Customize (Optional)

Open the workflow markdown file at `.github/workflows/daily-repo-status.md`, edit the "What to include" section to list what you want covered, and — if you changed the frontmatter — regenerate the compiled workflow with `gh aw compile`, then commit and push. You can trigger another run manually:

```bash
gh aw run daily-repo-status
```

After merging a workflow PR, you can trigger runs manually from the Actions tab on github.com, or use the `gh aw run` command from your terminal.

---

# Chapter 7: Authoring Your Own Workflows

## The Manual Path, End to End

Once you understand the toolchain, authoring from scratch follows a predictable lifecycle. You write a Markdown source, compile it to a lock file, commit both, configure secrets, and run — then iterate on the prompt as you watch real runs. The diagram below shows that loop.

```
   ┌─────────────────────────────────────────────────────────────┐
   │                     AUTHORING LIFECYCLE                      │
   └─────────────────────────────────────────────────────────────┘

        1. AUTHOR                2. COMPILE              3. COMMIT
   ┌──────────────────┐     ┌──────────────────┐   ┌──────────────────┐
   │ write <name>.md  │     │   gh aw compile  │   │  git add .md +   │
   │ • frontmatter    │────▶│   → .lock.yml    │──▶│      .lock.yml   │
   │ • natural-lang   │     │  validate · scan │   │  git commit/push │
   │   body           │     │  · pin actions   │   │                  │
   └──────────────────┘     └────────┬─────────┘   └────────┬─────────┘
            ▲                        │                      │
            │              ✗ errors  │ (fix frontmatter)    │
            │                        ▼                      ▼
            │               ┌──────────────────┐   ┌──────────────────┐
            │               │   re-author /    │   │  4. CONFIGURE    │
            │               │   fix & retry    │   │  engine secret   │
            │               └──────────────────┘   │  (gh aw secrets  │
            │                                       │   bootstrap)     │
            │                                       └────────┬─────────┘
            │                                                │
            │                                                ▼
   ┌──────────────────┐     ┌──────────────────┐   ┌──────────────────┐
   │  edit BODY only  │     │   6. OBSERVE     │   │   5. RUN         │
   │  → no recompile  │◀────│  gh aw logs      │◀──│  trigger event   │
   │  edit FRONTMATTER│     │  gh aw audit     │   │  or gh aw run    │
   │  → recompile ────┼────▶│  gh aw health    │   │                  │
   └──────────────────┘     └──────────────────┘   └──────────────────┘
        (iterate)
```

A key efficiency: editing the **body** does not require recompilation — the markdown body is loaded at runtime and can be edited directly on GitHub.com without recompiling. Only **frontmatter changes** require running `gh aw compile` again. That is why steps 1–3 and step 6 form a tight loop: most iteration is prompt-tuning in the body, and only structural changes send you back through the compiler.

### Step 1 — Author the Markdown File

Create `.github/workflows/<name>.md`. Use kebab-case, descriptive names (`pr-reviewer.md`, `weekly-summary.md`), and avoid spaces or special characters. Here is a realistic frontmatter for an automated pull-request reviewer — read-only, with every write going through safe outputs:

```yaml
---
description: "Reviews opened PRs and posts structured feedback as a comment"
emoji: "🔍"
labels: [automation, code-review]

on:
  pull_request:
    types: [opened, reopened, ready_for_review]
  reaction: eyes            # maintainers can re-trigger by reacting
  stop-after: "+30d"        # safety valve: auto-disable after 30 days

permissions:               # agent is read-only
  contents: read
  pull-requests: read

engine: copilot
timeout-minutes: 15         # cost and runaway-loop protection

tools:
  github:
    toolsets: [pull-requests, repos]   # read toolsets only

network:
  firewall: true
  allowed: [defaults]

safe-outputs:              # the only write paths, each a separate scoped job
  add-comment:
    max: 1
  add-labels:
    allowed: [risk/low, risk/medium, risk/high, needs-tests]
    max: 2
---
```

The body below the frontmatter would give the agent a precise output contract — what to read (title, description, full diff), exactly how to structure its review comment, which single risk label to apply, and hard boundaries ("do NOT approve or request changes — only comment and label; a human decides"). That output contract is the single biggest lever on review quality. What makes this realistic rather than a toy is the discoverability metadata, the minimal `permissions`/`tools`, the `timeout-minutes` ceiling, the `stop-after` pilot valve, and the allowlisted label set.

### Step 2 — Compile

```bash
gh aw compile
```

This generates `.github/workflows/pr-reviewer.lock.yml` from your markdown, and in doing so validates the schema, checks expression safety, pins actions to SHAs, and runs security scanning. If the frontmatter has a problem, compilation fails here with file, line, and column — fix it and recompile. For production, compile in strict mode (`gh aw compile --strict --zizmor`) so security findings fail the build rather than warn. For a walkthrough of what the generated `.lock.yml` actually contains — its header, manifest, and the read-only-agent/scoped-write-jobs structure — see *What's Inside the `.lock.yml`?* in Chapter 2.

### Step 3 — Commit Both Files

Always commit the `.md` source *and* the generated `.lock.yml`, for transparency:

```bash
git add .github/workflows/pr-reviewer.md
git add .github/workflows/pr-reviewer.lock.yml
git commit -m "Add pr-reviewer agentic workflow"
git push
```

### Step 4 — Configure Engine Secrets

If you have not already, set the secret for your chosen engine. The fastest path is the bootstrap command, which analyzes your workflows and prompts only for what is missing:

```bash
gh aw secrets bootstrap
```

If you are not using Copilot, also set `engine:` in the frontmatter accordingly (`claude`, `codex`, or `gemini`) and recompile.

### Step 5 — Run It

The workflow now triggers automatically on its declared events (here, opened or reopened PRs). To trigger a run on demand from the terminal:

```bash
gh aw run pr-reviewer
```

### Step 6 — Observe and Iterate

Watch real runs and tune. The toolchain ships first-class observability commands:

```bash
gh aw logs                   # Download and analyze run logs (tools, network, errors)
gh aw audit <run-id>         # Detailed report for a single run
gh aw audit <id-a> <id-b>    # Diff two runs to spot behavioral drift
gh aw logs --format markdown # Cross-run security/performance report
gh aw status                 # List workflows with state and schedules
gh aw health                 # Success/failure rates and cost trends
```

All workflow outputs — prompts, patches, logs — are saved as downloadable artifacts; failed runs can be investigated with `gh aw audit` to examine prompts, errors, and network activity. When reviews are too generic, add detail to the body; when the agent is too verbose, tighten the output contract. Because body edits need no recompile, this loop is fast.

Two things to keep in mind as you write more of these. Triggers use the same broad set of events as regular Actions (issues, pull requests, schedules, manual dispatch, comment commands), plus agentic-specific `on:` controls — `reaction:`, `stop-after:`, and `roles:` (restrict who may trigger). And the agent's capabilities come entirely from the `tools:` block; grant only the toolsets a workflow needs, as the reviewer above takes `pull-requests` and `repos` and nothing else.

---

# Chapter 8: Capstone — Fusing Both Books

## Bringing It All Together

Each previous chapter introduced one piece in isolation: the two-file model (Chapter 2), the natural-language body and its YAML frontmatter (Chapter 3), the safe-output use cases (Chapter 4), the three-layer security model (Chapter 5), and the authoring lifecycle (Chapter 7). This final chapter combines all of them into a single, realistic workflow that is more ambitious than any of the per-chapter samples — and then walks through exactly how each design decision maps back to a concept you have already met. It closes by fusing those concepts with the Git foundations from the companion textbook, showing how an agentic layer plugs directly into the pull-request lifecycle and CI pipeline you built there.

The workflow is a **Repository Health Steward**: a scheduled agent that runs every Monday morning, surveys the repository's recent activity and backlog, and takes three different kinds of action in one run — it files a grouped weekly health report as an issue, flags stale issues with a label, and opens a draft pull request fixing any obvious documentation drift it finds. Emitting three distinct safe-output types in one run is the clearest demonstration of why the architecture is shaped the way it is: the agent reasons holistically about the whole repository but never holds write access to any of it. Each of its three effects — the report, the labels, the PR — is buffered, threat-scanned, and applied by its own narrowly-scoped job. One read-only brain; three independent, least-privilege hands.

### The Complete Workflow

Save this as `.github/workflows/repo-health-steward.md`:

```markdown
---
description: "Weekly repository health review: report, triage stale issues, fix doc drift"
emoji: "🩺"
labels: [automation, maintenance, weekly]

on:
  schedule:
    - cron: "0 8 * * 1"     # every Monday at 08:00 UTC
  workflow_dispatch:         # allow manual runs for testing
  # Restrict who can trigger manual runs to repo maintainers/admins.
  roles: [admin, maintainer]
  # Pilot safety valve: auto-disable after 90 days unless renewed.
  stop-after: "+90d"

# LAYER 2 (configuration-level trust): the agent is strictly read-only.
# Not one write scope appears here. Every write is a safe output below.
permissions:
  contents: read
  issues: read
  pull-requests: read

engine: copilot

# Bound cost and runtime — this agent reads a lot, so give it room but cap it.
timeout-minutes: 20

# LAYER 1 (substrate-level trust): egress is locked to what the agent needs,
# enforced by the privileged firewall container outside the agent's reach.
network:
  firewall: true
  allowed: [defaults]

# Read-only toolsets. The agent can survey the repo but cannot mutate it.
tools:
  github:
    toolsets: [issues, pull-requests, repos, actions]

# LAYER 3 (plan-level trust): three independent safe outputs. Each becomes
# its own scoped, post-agent job that runs only after threat detection passes.
safe-outputs:
  # 1) A single weekly report issue, grouped so reruns update one thread.
  create-issue:
    title-prefix: "[health] "
    labels: [health-report, automated]
    close-older-issues: true   # keep only the latest week's report open
    max: 1
  # 2) Triage labels for stale items — restricted to an explicit allowlist.
  add-labels:
    allowed: [stale, needs-triage, help-wanted]
    max: 10
  # 3) A draft PR for documentation fixes — never auto-merged, always reviewed.
  create-pull-request:
    title-prefix: "[docs] "
    labels: [documentation, automated]
    draft: true
---

# Weekly Repository Health Review

You are the repository's health steward. Survey activity from the **last 7
days** plus the current open backlog, then take the actions below. Work
methodically and do only what each section authorizes.

## 1. File a weekly health report

Create ONE issue summarizing repository health, structured as:

### Activity (last 7 days)
Issues opened/closed, PRs opened/merged, and notable releases.

### Backlog health
Total open issues and PRs, how many are unlabeled, and how many have had
no activity in over 30 days.

### CI signal
Any workflows that failed repeatedly this week, by name and rough frequency.

### Top 3 recommendations
The three highest-leverage things a maintainer could do this week.

## 2. Triage stale issues

For open issues with no activity in over 30 days, apply the `stale` label.
For open issues that have no labels at all, apply `needs-triage`. Apply at
most 10 labels total this run; prioritize the oldest items. Do not remove
any existing labels.

## 3. Fix obvious documentation drift

Compare recently merged code changes against `README.md` and files under
`docs/`. If — and only if — you find a clear, factual mismatch (a renamed
flag, a changed default, a moved file path), open a single draft pull
request with the minimal corrections. Make no speculative edits.

## Boundaries

- Do NOT close issues or PRs. Do NOT merge anything. A human decides.
- Do NOT open the documentation PR unless there is a concrete, factual
  drift to fix. An empty or speculative PR is worse than none.
- If a section has nothing to act on, say so in the report and move on.
- If you complete the run with no GitHub action of any kind, call `noop`
  with a one-line explanation so the run does not fail silently.
```

### How Each Piece Traces Back

Read the frontmatter against the earlier chapters and every choice has a reason you already understand:

The **trigger block** combines a `schedule` cron with `workflow_dispatch` for manual testing, then narrows manual runs with `roles:` and adds a `stop-after:` deadline — the agentic-specific `on:` controls from Chapter 7. The **`permissions:`** block grants only read scopes; this is Chapter 5's configuration-level trust (Layer 2) made literal, and it is what forces every write onto the safe-output path. The **`network:`** allowlist is Layer 1 — enforced by the privileged firewall container the agent cannot touch. The **`tools:`** block grants read-only toolsets, matching the least-privilege principle from Chapter 7.

The **three `safe-outputs:` entries** are the heart of the example and the clearest expression of Chapter 5's Layer 3. At compile time, each one becomes a separate job in the generated `.lock.yml` (Chapter 2): the agent writes its intended report, labels, and patch into a buffered artifact; the threat-detection job scans that artifact; and only then do three independent jobs — `create_issue` with `issues: write`, `add_labels` with `issues: write`, and `create_pull_request` with `contents: write` plus `pull-requests: write` — apply the results. The agent itself never held any of those scopes. Note also the small but important details: `close-older-issues: true` keeps a single rolling report thread, `add-labels.allowed` constrains the agent to a fixed label vocabulary so a prompt-injection attempt cannot invent labels, and `draft: true` on the PR is an enforced policy the agent cannot override.

The **body** follows the "good objectives" guidance from Chapter 3: it states what success looks like rather than prescribing mechanism, gives the agent a precise output contract for the report, scopes each action with explicit limits ("at most 10 labels," "only … a clear, factual mismatch"), and closes with hard boundaries. The final instruction to call `noop` when nothing needs doing is the one operational detail that prevents a silent failure when the repository is already healthy.

### Deploying It

The lifecycle is exactly the loop from Chapter 7. Author the file, then:

```bash
# Compile in strict mode so security findings fail the build, not just warn.
gh aw compile --strict --zizmor

# Make sure the engine secret exists (Chapter 2 — keys live in Actions secrets).
gh aw secrets bootstrap

# Commit BOTH the source and the generated lock file.
git add .github/workflows/repo-health-steward.md
git add .github/workflows/repo-health-steward.lock.yml
git commit -m "Add repository health steward agentic workflow"
git push

# Trigger a manual run now rather than waiting for Monday, then watch it.
gh aw run repo-health-steward
gh aw logs
```

From here you iterate the way Chapter 7 describes: read the first few real runs with `gh aw logs` and `gh aw audit`, and tune the body — tighten the report format, adjust the staleness threshold, sharpen the documentation-drift criteria. Because those are body edits, they take effect on the next run with no recompile. Only if you change the schedule, the permissions, or the safe outputs do you compile again.

### What You Have Built

This one file is a standing maintenance teammate: every Monday it produces a health report, keeps the backlog tidy, and proposes documentation fixes for human review — while holding no write access to the repository at any point. That combination, autonomous initiative bounded by a verifiable least-privilege envelope, is the whole promise of GitHub Agentic Workflows in a single artifact. Everything else in the official documentation is a variation on the pattern you now have in front of you.

## Fusing Both Books: Agentic Review on the Pull Request Lifecycle

The Health Steward shows what *this* document teaches. The real capstone, though, is connecting it back to the companion **Git & GitHub Automation Textbook** — because agentic workflows do not replace the Git foundations and CI pipeline you built there. They plug into the exact same lifecycle.

Recall the spine of the textbook. Git stores each commit as a complete snapshot, and a branch is just a cheap, movable pointer — which is what makes the feature-branch-and-pull-request workflow practical. That PR workflow has three automation trigger points: when you push to a feature branch, when you open or update a pull request, and when you merge to `main`. The textbook's CI pipeline (`ci.yml`) hooks the first two, running checkout → setup-node → `npm ci` → lint → test on every push and PR so that broken code is blocked before it merges. And the cautionary tale at the heart of that book — the "$10,000 bug" that shipped to `main` on a Friday night — happened for one reason: the failing code path had **no test**. Deterministic CI is powerful, but it only ever checks what someone remembered to write a test for.

That gap is exactly where an agentic workflow earns its place. It hooks the *same* `pull_request` trigger as the textbook's CI pipeline, but instead of running pre-written assertions, it reads the diff and reasons about what the tests do not cover. Two complementary layers, one trigger point: deterministic checks that must pass, plus agentic judgment that flags the risks static tests are blind to.

### Two Layers on the Same Trigger

Here is the textbook's deterministic pipeline, unchanged — the gate that must go green:

```yaml
# .github/workflows/ci.yml  — the deterministic layer (from the textbook)
name: CI Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

And here is the agentic layer beside it — a test-gap reviewer that fires on the same pull-request event and asks the one question the $10K bug needed someone to ask: *is there a code path here with no test?*

```markdown
---
# .github/workflows/pr-test-gap-reviewer.md  — the agentic layer
description: "Flags changed code paths that lack test coverage on a PR"
emoji: "🧪"
labels: [automation, code-review, testing]

on:
  pull_request:
    types: [opened, reopened, synchronize, ready_for_review]

# Read-only, exactly like the Health Steward. The agent never writes code.
permissions:
  contents: read
  pull-requests: read

engine: copilot
timeout-minutes: 15

tools:
  github:
    toolsets: [pull-requests, repos]

network:
  firewall: true
  allowed: [defaults]

# Complements CI; it never blocks the merge by itself. It comments and labels;
# a human (and the deterministic CI gate) still decide.
safe-outputs:
  add-comment:
    max: 1
  add-labels:
    allowed: [needs-tests, test-gap-reviewed]
    max: 2
---

# Pull Request Test-Gap Review

You are reviewing pull request #${{ github.event.pull_request.number }},
working alongside the deterministic CI pipeline (lint + tests). CI already
checks whether existing tests pass. Your distinct job is to find behavior
that has **no test at all** — the gap that lets a green build still ship a bug.

## What to do

1. Read the diff. For each changed or added function, ask:
   - Does the PR add or update a test that exercises this behavior?
   - Does it handle malformed or missing input (the classic crash:
     reading a field that may be absent)?

2. Post ONE review comment listing, as a short checklist, the specific
   functions or branches that changed without a corresponding test.
   For each, name the file and the missing case in one line. If coverage
   looks complete, say so plainly and praise it.

3. Apply `needs-tests` if you found at least one untested changed path.
   Always apply `test-gap-reviewed` so maintainers can see the agent ran.

## Boundaries

- Do NOT approve, request changes, or block the merge. CI is the gate;
  you are advisory. A human decides.
- Judge only the diff and its immediate context — do not demand tests for
  unrelated existing code.
- If nothing changed that could carry logic (docs, config only), call
  `noop` with a one-line explanation.
```

### Where the Agent Plugs In

The textbook's PR-lifecycle-with-automation diagram gains one parallel lane. At the push and PR triggers, the deterministic pipeline and the agentic reviewer run side by side; the merge trigger is unchanged and still gated by branch protection:

```
 feature branch          pull request                review & merge
      │                       │                            │
   git push ──┬──▶ TRIGGER ①──┴──▶ TRIGGER ② (PR opened/updated)
              │                            │
       ┌──────┴───────┐            ┌───────┴────────┐
       ▼              ▼            ▼                ▼
  ci.yml         pr-test-gap   ci.yml          pr-test-gap
  (lint+test)    reviewer      re-runs         re-reviews
       │              │            │                │
   must pass     comments +    must pass       updated
   (the gate)    needs-tests   (the gate)      comment
       └──────┬───────┘            └───────┬────────┘
              ▼                            ▼
       branch protection ◀────────── human review
              │  (merge allowed only if CI is green)
              ▼
         TRIGGER ③  merge to main ──▶ deploy with confidence
```

The deterministic layer answers "do the existing tests pass?" The agentic layer answers "what isn't being tested at all?" Neither alone would have caught the Friday-night bug with certainty; together they make it far harder to ship.

### Mapping to Both Books

Read across the two documents and every concept lines up:

| Textbook concept | How the capstone uses it |
|---|---|
| Branch = cheap pointer; commit = snapshot | The agent reviews a PR's diff — the snapshot delta between the feature branch and `main` |
| The three PR automation triggers | The agentic reviewer hooks triggers ① and ②, exactly where `ci.yml` does |
| `on:` / `runs-on:` / `steps:` (WHEN/WHERE/WHAT) | The agentic frontmatter is the same `on:` trigger model, with the body supplying the WHAT in natural language instead of `steps:` |
| The $10K bug: no test for the failing path | The reviewer's entire purpose is to surface untested changed paths |
| Branch protection gates the merge | Unchanged — CI remains the gate; the agent is advisory and never blocks |
| `npm ci` + lint + test in `ci.yml` | Left fully intact; the agent adds a layer rather than replacing the pipeline |

This is the synthesis the two books were building toward. The first textbook gives you the deterministic foundation — Git's branch-and-merge model and a CI pipeline that blocks code failing the tests you wrote. This document adds an agentic layer that reasons about the code the way a careful teammate would, bounded by the same least-privilege, human-in-the-loop guarantees — and wired into the very same pull-request lifecycle. Deterministic where you can specify the rule, agentic where you need judgment, and Git's workflow holding both together.

---

# Quick Reference & Glossary

## Common Commands

| Command | Purpose |
|---|---|
| `gh extension install github/gh-aw` | Install the agentic workflows extension |
| `gh aw init` | Set up the repo for agentic workflows (non-interactive); enables in-context authoring from github.com / mobile |
| `gh aw new <name>` | Create a new workflow template from scratch |
| `gh aw add-wizard <owner>/<repo>/<name>` | Add a pre-built workflow interactively |
| `gh aw compile` | Regenerate `.lock.yml` from the `.md` source |
| `gh aw compile --strict --zizmor` | Compile in strict mode, failing on security findings |
| `gh aw validate` | Run all linters without regenerating lock files |
| `gh aw run <name>` | Trigger a workflow run from the terminal |
| `gh aw logs` | Download and analyze run logs |
| `gh aw audit <run-id>` | Investigate a specific run with a detailed report |
| `gh aw status` | List workflows with state, enabled/disabled status, and schedules |
| `gh aw health` | Show success/failure rates and cost trends over time |
| `gh aw disable <name>` / `gh aw enable <name>` | Pause or resume a workflow (disable also cancels in-progress runs) |

## Glossary

| Term | Definition |
|---|---|
| **Agentic Workflow** | A GitHub Actions automation defined in Markdown where an AI agent determines and executes tasks within security guardrails |
| **`gh aw`** | The GitHub CLI extension that compiles and manages agentic workflows |
| **Two-file model** | The `.md` source you edit plus the generated `.lock.yml` that Actions runs |
| **Frontmatter** | The configuration block between `---` markers at the top of the `.md` file; changes require recompiling |
| **Body** | The natural-language task description below the frontmatter; changes take effect on the next run |
| **SafeOutputs** | The subsystem that buffers agent writes as artifacts and applies them only via separate, scoped, validated jobs |
| **AWF (Agent Workflow Firewall)** | Containerizes the agent and routes egress through a Squid proxy enforcing a domain allowlist |
| **Threat Detection** | An isolated job that inspects buffered agent output and emits a pass/fail verdict before any write |
| **Integrity Filtering** | Restricts which GitHub content the agent can read, based on author trust and merge status |
| **MCP Gateway** | A trusted component that spawns isolated containers for MCP servers |
| **Engine** | The AI coding agent powering the workflow (Copilot by default; also Claude, Codex, Gemini) |

---

# Sources

This document is grounded in the following official GitHub sources (accessed June 2026):

1. **Your first agentic workflow (Quickstart)** — GitHub Docs
   <https://docs.github.com/en/copilot/how-tos/github-agentic-workflows/quickstart>
2. **Quick Start** — GitHub Agentic Workflows documentation site
   <https://github.github.com/gh-aw/setup/quick-start/>
3. **Creating Agentic Workflows** — GitHub Agentic Workflows documentation site
   <https://github.github.com/gh-aw/setup/creating-workflows/>
4. **GitHub Agentic Workflows are now in technical preview** — GitHub Changelog, Feb 13, 2026
   <https://github.blog/changelog/2026-02-13-github-agentic-workflows-are-now-in-technical-preview/>
5. **Community Feedback discussion** — GitHub Community
   <https://github.com/orgs/community/discussions/186451>

Additional official references worth bookmarking:

- **Security Architecture** — <https://github.github.com/gh-aw/introduction/architecture/>
- **Safe Outputs Reference** — <https://github.github.com/gh-aw/reference/safe-outputs/>
- **Threat Detection** — <https://github.github.com/gh-aw/reference/threat-detection/>
- **`gh-aw` open-source repository** — <https://github.com/github/gh-aw>
- Original textbook: [Git & GitHub Automation Textbook v4](https://github.com/douglasmun/Douglas-Guidebook/blob/main/Git_GitHub_Automation_Textbook_v4.md)

---

*Version 1.0 — Douglas Mun — June 2026*
*Educational Use License — Companion to Git & GitHub Automation Textbook v4*
*GitHub Agentic Workflows are in technical preview; verify details against the live documentation before production use.*
