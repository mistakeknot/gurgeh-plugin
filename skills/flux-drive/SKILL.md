---
name: flux-drive
description: "Intelligent plan deepening — triages relevant agents, creates codebase-aware specialists, launches only what matters. Unlike /deepen-plan which fires 40-100+ generic agents, flux-drive identifies 5-20 relevant reviewers and creates project-specific specialists."
---

# Flux Drive — Intelligent Plan Deepening

You are executing the flux-drive skill. This skill deepens a plan file by launching **only relevant** agents, including custom codebase-aware specialists. Follow each phase in order. Do NOT skip phases.

## Input

The user provides a plan file path as an argument. If no path is provided, ask for one using AskUserQuestion.

Store the plan file path for use throughout all phases:
```
PLAN_FILE = <the path the user provided>
```

---

## Phase 0: Tier 2 Agent Fleet Management

Tier 1 agents ship with this plugin in `agents/` — they're always available and need no creation.
Tier 2 agents are project-specific specialists stored in `.claude/agents/fd-*.md`. They contain baked-in codebase knowledge.

### Step 0.1: Check Tier 2 Agent Status

Check if `.claude/agents/fd-*.md` files exist in the project root.

**If they don't exist** → go to Step 0.2 (create them).

**If they exist**, decide whether to regenerate:

1. Read all existing `fd-*.md` agent files (full content, not just frontmatter)
2. Read the project's `CLAUDE.md` and `AGENTS.md` for current state
3. Skim key source directories to understand the current tech stack
4. **Judge** whether the existing Tier 2 agents are still accurate and useful:
   - Do the agents cover the right domains for the project's current tech stack?
   - Is any baked-in knowledge now wrong? (e.g., agent says "BT v2" but project uses BT v1)
   - Are there new technologies/subsystems the agents don't cover?
   - Are there agents for technologies the project no longer uses?
5. If the agents are still good → skip to Phase 1
6. If any agents are stale, wrong, or missing → go to Step 0.2 to regenerate

**Trust your judgment here.** You can read the agents and the codebase — you know better than a commit-count heuristic whether the specialists are still relevant.

### Step 0.2: Identify Specialist Domains

Read the project's documentation to understand its technology stack:
1. Read `CLAUDE.md` and `AGENTS.md` in the project root
2. Use Glob to scan key directories: `cmd/`, `internal/`, `pkg/`, `src/`, `lib/`, `app/`
3. Identify technologies, frameworks, and subsystems NOT covered by Tier 1 generalists

Tier 1 already covers: architecture, UX, security, performance, code quality.
Tier 2 should cover **specific technologies/frameworks** used by this project.

Examples of what to look for:
- TUI framework specifics (Bubble Tea, Ink, blessed, etc.)
- Domain-specific schemas or data models
- Integration frameworks or protocols
- Database-specific patterns beyond generic SQL
- Language-specific framework conventions

Typically identify 2-5 specialist domains. Don't create Tier 2 agents for generic things Tier 1 handles.

### Step 0.3: Research and Create Tier 2 Agents

For each identified domain, launch a parallel `Task` with `subagent_type: Explore` to research:
- Current best practices and known pitfalls
- Framework-specific patterns and version-specific limitations
- Read relevant source files in the project to understand how it uses the technology

**Important**: Use `model: haiku` for these research agents to save cost — they're doing reconnaissance, not deep analysis.

After research completes, create each Tier 2 agent file in `.claude/agents/`:

Each agent file must have:
```markdown
---
name: fd-{domain}
description: "One-line description of what this specialist reviews"
model: inherit
---

[System prompt with baked-in knowledge from research]
[Project-specific context from codebase analysis]
[Review approach specific to this domain]
[Output format matching Tier 1 agents]
```

Create the `.claude/agents/` directory if it doesn't exist.

---

## Phase 1: Analyze Plan

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

Do this analysis yourself (no subagents needed). The profile drives triage in Phase 3.

---

## Phase 2: Discover Available Resources

Scan for ALL available agents, skills, and learnings. Read only frontmatter or first 10 lines of each — don't read full files.

### Sources to scan:

1. **Project custom agents** (Tier 1 + Tier 2):
   - Glob: `agents/fd-*.md` (plugin Tier 1 — these are in the gurgeh-plugin directory)
   - Glob: `.claude/agents/fd-*.md` (project Tier 2)

2. **User custom agents**:
   - Glob: `~/.claude/agents/*.md`

3. **Compound-engineering agents** (check the latest installed version):
   - Glob: `~/.claude/plugins/cache/every-marketplace/compound-engineering/*/agents/**/*.md`
   - Take only the latest version directory

4. **Project learnings**:
   - Glob: `docs/solutions/*.md` (if exists)

Build a resource catalog as a table:

```
| # | Name | Type | Source | Custom? | Description (from frontmatter) |
```

This catalog is the input for Phase 3 triage.

---

## Phase 3: Triage

Score each resource against the plan profile from Phase 1:

- **2 (relevant)**: Domain directly overlaps with plan content. Custom agents get +1 bonus (they know this codebase).
- **1 (maybe)**: Adjacent domain. Include only for plan sections that are thin.
- **0 (irrelevant)**: Wrong language, wrong domain, no relationship to this plan.

### Selection Rules:

1. All resources scoring 2+ are included
2. Resources scoring 1 are included only if their domain covers a thin section
3. Cap at 20 total agents
4. **Deduplication**: If a custom agent and a generic agent cover the same domain, keep the custom one, drop the generic one. Custom agents are strictly better because they know the codebase.

### User Confirmation

Present the triage table using AskUserQuestion:

```
Flux Drive Triage — {plan_file_name}

| Agent | Score | Reason | Custom? | Action |
|-------|-------|--------|---------|--------|
| fd-architecture | 2+1 | Plan restructures module boundaries | Yes (T1) | Launch |
| fd-bubble-tea | 2+1 | Plan adds new TUI views | Yes (T2) | Launch |
| security-sentinel | 1 | Plan adds API endpoint (thin section) | No | Launch |
| dhh-rails-reviewer | 0 | Go project, not Rails | No | Skip |
| ... | ... | ... | ... | ... |

Launching N agents. Approve?
```

Options: `Approve` / `Edit selection` / `Add more` / `Cancel`

If user selects "Edit selection" or "Add more", adjust and re-present.
If user selects "Cancel", stop here.

---

## Phase 4: Targeted Launch

Launch all selected agents as parallel Task calls in a **single message**.

### How to launch each agent type:

**Custom agents (Tier 1 and Tier 2)**:
- `subagent_type: general-purpose`
- Include the agent's full system prompt in the task prompt
- Append: the plan content + specific focus area

**Compound-engineering agents**:
- Use the native `subagent_type` from the Task tool's agent list (e.g., `compound-engineering:review:architecture-strategist`)
- Append: the plan content + specific focus area

**Learnings/solutions docs**:
- Don't launch as agents. Instead, note relevant learnings to include in Phase 5 synthesis.

### Prompt template for each agent:

```
You are reviewing a plan for potential improvements and issues.

[Agent system prompt — full content for custom agents, omit for native agents]

## Plan to Review

[Full plan content from PLAN_FILE]

## Your Focus Area

You were selected because: [reason from triage table]
Focus on: [specific plan sections relevant to this agent's domain]
Depth needed: [thin sections need more depth, deep sections need only validation]

## Output Requirements

Write your FULL analysis to a file, then return only a 3-5 line summary.
Structure your analysis with:
1. Section-by-section review (only sections in your domain)
2. Specific issues found (numbered, with severity)
3. Specific improvements suggested (numbered, with rationale)
4. Overall assessment

Be concrete. Reference specific plan sections by name. Don't give generic advice.
```

---

## Phase 5: Synthesize

After all agents complete:

1. **Collect summaries** from agent responses
2. **Read full analysis files** from `docs/research/` for agents that produced high-value findings
3. **Group findings by plan section** — organize all agent findings under the plan section they apply to
4. **Deduplicate**: If multiple agents flagged the same issue, keep the most specific one (prefer custom agent versions)
5. **Flag conflicts**: If agents disagree, note both positions
6. **Priority from custom agents**: When a custom agent and generic agent give different advice on the same topic, prefer the custom agent's recommendation (it knows the codebase)

### Update the Plan

Read the current plan file, then add enhancements:

1. Add a header section at the top of the plan:
   ```markdown
   ## Flux Drive Enhancement Summary

   Reviewed by N agents (M custom, K generic) on YYYY-MM-DD.

   ### Key Findings
   - [Top 3-5 findings across all agents]

   ### Issues to Address
   - [ ] [Issue 1 — from agent X]
   - [ ] [Issue 2 — from agent Y]
   - ...
   ```

2. For each plan section that received feedback, add an enhancement block:
   ```markdown
   > **Flux Drive Notes** ({agent-name}):
   > [Concise finding or suggestion]
   ```

3. Write the updated plan back to `PLAN_FILE`.

### Final Report

Tell the user:
- How many agents ran and how many were custom vs generic
- Top findings (3-5 most important)
- Which plan sections got the most feedback
- Where full analysis files are saved (for deep dives)
