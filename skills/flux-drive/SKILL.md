---
name: flux-drive
description: "Intelligent plan deepening — triages relevant agents from a static roster, launches only what matters in background mode. Lean on context, rich on feedback."
---

# Flux Drive — Intelligent Plan Deepening

You are executing the flux-drive skill. This skill deepens a plan file by launching **only relevant** agents selected from a static roster. Follow each phase in order. Do NOT skip phases.

## Input

The user provides a plan file path as an argument. If no path is provided, ask for one using AskUserQuestion.

Store the plan file path and derive the output directory for use throughout all phases:
```
PLAN_FILE = <the path the user provided>
PLAN_DIR  = <directory containing PLAN_FILE>
OUTPUT_DIR = {PLAN_DIR}/../research/flux-drive
```

**Note:** The output directory is derived from the plan file's location, not the current working directory. This ensures cross-project reviews write research files alongside the plan, not in the invoking project's directory.

---

## Phase 1: Analyze Plan + Static Triage

### Step 1.1: Analyze the Plan

Read the plan file at `PLAN_FILE`. Extract a structured profile:

```
Plan Profile:
- Languages: [list]
- Frameworks: [list]
- Domains touched: [architecture, security, performance, UX, data, API, etc.]
- Technologies: [specific tech mentioned]
- Section analysis:
  - [Section name]: [thin/adequate/deep] — [1-line summary]
  - ...
- Estimated complexity: [small/medium/large]
```

Do this analysis yourself (no subagents needed). The profile drives triage in Step 1.2.

### Step 1.2: Select Agents from Roster

Consult the **Agent Roster** below and score each agent against the plan profile:

- **2 (relevant)**: Domain directly overlaps with plan content.
- **1 (maybe)**: Adjacent domain. Include only for plan sections that are thin.
- **0 (irrelevant)**: Wrong language, wrong domain, no relationship to this plan.

**Tier bonuses**: Tier 1 agents get +1 (they know this codebase). Tier 2 agents get +1 (project-specific).

**Selection rules**:
1. All agents scoring 2+ are included
2. Agents scoring 1 are included only if their domain covers a thin section
3. **Cap at 8 agents total** (hard maximum)
4. **Deduplication**: If a Tier 1 or Tier 2 agent covers the same domain as a Tier 3 agent, drop the Tier 3 one. Codebase-aware agents are strictly better.
5. Prefer fewer, more relevant agents over many marginal ones

### Step 1.3: User Confirmation

Present the triage using AskUserQuestion:

```
Flux Drive Triage — {plan_file_name}

| Agent | Tier | Score | Reason | Action |
|-------|------|-------|--------|--------|
| fd-architecture | T1 | 2+1 | Plan restructures module boundaries | Launch |
| fd-user-experience | T1 | 2+1 | Plan adds new TUI views | Launch |
| security-sentinel | T3 | 1 | Plan adds API endpoint (thin section) | Launch |
| ... | ... | ... | ... | ... |

Launching N agents (M codebase-aware, K generic). Approve?
```

Options: `Approve` / `Edit selection` / `Cancel`

If user selects "Edit selection", adjust and re-present.
If user selects "Cancel", stop here.

---

## Agent Roster

### Tier 1 — Codebase-Aware (gurgeh-plugin)

These agents ship with the plugin and have baked-in knowledge of the project's architecture, conventions, and patterns.

| Agent | subagent_type | Domain |
|-------|--------------|--------|
| fd-architecture | gurgeh-plugin:fd-architecture | Module boundaries, component structure, cross-tool integration |
| fd-user-experience | gurgeh-plugin:fd-user-experience | CLI/TUI interaction, keyboard ergonomics, terminal constraints |
| fd-code-quality | gurgeh-plugin:fd-code-quality | Naming, test strategy, project conventions, idioms |
| fd-performance | gurgeh-plugin:fd-performance | Rendering, data processing, resource usage |
| fd-security | gurgeh-plugin:fd-security | Threat model, credential handling, access patterns |

### Tier 2 — Project-Specific (.claude/agents/fd-*.md)

Check if `.claude/agents/fd-*.md` files exist in the project root. If so, include them in triage. Use `subagent_type: general-purpose` and include the agent file's full content as the system prompt in the task prompt.

**Note:** `general-purpose` agents have full tool access (Read, Grep, Glob, Write, Bash, etc.) — the same as Tier 1 agents. The difference is that Tier 1 agents get their system prompt from the plugin automatically, while Tier 2 agents need it pasted into the task prompt.

If no Tier 2 agents exist, skip this tier entirely. Do NOT create them — that's a separate workflow.

### Tier 3 — Generic Specialists (compound-engineering)

These are general-purpose reviewers without codebase-specific knowledge. Only use when no Tier 1/2 agent covers the domain.

| Agent | subagent_type | Domain |
|-------|--------------|--------|
| architecture-strategist | compound-engineering:review:architecture-strategist | System design, component boundaries |
| code-simplicity-reviewer | compound-engineering:review:code-simplicity-reviewer | YAGNI, minimalism, over-engineering |
| performance-oracle | compound-engineering:review:performance-oracle | Algorithms, scaling, bottlenecks |
| security-sentinel | compound-engineering:review:security-sentinel | OWASP, vulnerabilities, auth |
| pattern-recognition-specialist | compound-engineering:review:pattern-recognition-specialist | Anti-patterns, duplication, consistency |
| data-integrity-guardian | compound-engineering:review:data-integrity-guardian | Migrations, data safety, transactions |

---

## Phase 2: Launch

### Step 2.0: Prepare output directory

Create the research output directory before launching agents, using the path derived from the plan file location:
```bash
mkdir -p {OUTPUT_DIR}
```

### Step 2.1: Launch agents

Launch all selected agents as parallel Task calls in a **single message**.

**Critical**: Every agent MUST use `run_in_background: true`. This prevents agent output from flooding the main conversation context.

### How to launch each agent type:

**Tier 1 agents (gurgeh-plugin)**:
- Use the native `subagent_type` from the roster (e.g., `gurgeh-plugin:fd-architecture`)
- Set `run_in_background: true`

**Tier 2 agents (.claude/agents/)**:
- `subagent_type: general-purpose`
- Include the agent file's full content as the system prompt
- Set `run_in_background: true`

**Tier 3 agents (compound-engineering)**:
- Use the native `subagent_type` from the roster (e.g., `compound-engineering:review:architecture-strategist`)
- Set `run_in_background: true`

### Prompt template for each agent:

```
You are reviewing a plan for potential improvements and issues.

## Plan to Review

[Full plan content from PLAN_FILE]

## Your Focus Area

You were selected because: [reason from triage table]
Focus on: [specific plan sections relevant to this agent's domain]
Depth needed: [thin sections need more depth, deep sections need only validation]

## Output Requirements

Write your findings to: {OUTPUT_DIR}/{agent-name}.md

The file MUST start with a YAML frontmatter block for machine-parseable synthesis:

---
agent: {agent-name}
tier: {1|2|3}
issues:
  - id: P0-1
    severity: P0
    section: "Phase 2"
    title: "Short description of the issue"
  - id: P1-1
    severity: P1
    section: "Signals Overlay"
    title: "Short description"
improvements:
  - id: IMP-1
    title: "Short description"
    section: "Phase 2"
verdict: safe|needs-changes|risky
---

After the frontmatter, structure the prose analysis as:

### Summary (3-5 lines)
[Your top findings — this is the most important section]

### Section-by-Section Review
[Only sections in your domain]

### Issues Found
[Numbered, with severity: P0/P1/P2. Must match frontmatter.]

### Improvements Suggested
[Numbered, with rationale]

### Overall Assessment
[1-2 sentences]

Be concrete. Reference specific plan sections by name. Don't give generic advice.
```

After launching all agents, tell the user:
- How many agents were launched
- That they are running in background
- Estimated wait time (~2-3 minutes)

---

## Phase 3: Synthesize

### Step 3.0: Wait for all agents

**Do NOT start synthesis until all agents have completed.** Starting early leads to missed findings and plan re-edits.

Wait by polling the output directory:
```bash
ls {OUTPUT_DIR}/
```

You expect N files (one per launched agent). Poll every 30 seconds. If all N files exist, proceed. If after 5 minutes some are missing, proceed with what you have and note missing agents as "no findings."

### Step 3.1: Collect Results

For each agent's output file, read the **YAML frontmatter** first. This gives you a structured list of all issues and improvements without reading full prose. Only read the prose body if:
- An issue needs more context to understand
- You need to resolve a conflict between agents
- The frontmatter is missing or malformed (fallback to reading Summary + Issues sections)

If an agent's output file doesn't exist or is empty, note it as "no findings" and move on.

### Step 3.2: Deduplicate and Organize

1. **Group findings by plan section** — organize all agent findings under the plan section they apply to
2. **Deduplicate**: If multiple agents flagged the same issue, keep the most specific one (prefer Tier 1/2 over Tier 3)
3. **Flag conflicts**: If agents disagree, note both positions
4. **Priority from codebase-aware agents**: When a Tier 1/2 and Tier 3 agent give different advice on the same topic, prefer the codebase-aware recommendation

### Step 3.3: Update the Plan

Read the current plan file, then add a summary section at the top:

```markdown
## Flux Drive Enhancement Summary

Reviewed by N agents (M codebase-aware, K generic) on YYYY-MM-DD.

### Key Findings
- [Top 3-5 findings across all agents]

### Issues to Address
- [ ] [Issue 1 — from agent X] (severity)
- [ ] [Issue 2 — from agent Y] (severity)
- ...
```

For each plan section that received feedback, add an inline note:

```markdown
> **Flux Drive** ({agent-name}): [Concise finding or suggestion]
```

Write the updated plan back to `PLAN_FILE`.

### Step 3.4: Report to User

Tell the user:
- How many agents ran and how many were codebase-aware vs generic
- Top findings (3-5 most important)
- Which plan sections got the most feedback
- Where full analysis files are saved (`{OUTPUT_DIR}/`)
