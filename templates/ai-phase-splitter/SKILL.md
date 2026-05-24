---
name: ai-phase-splitter
description: >
  Use this skill whenever the user wants to break down a software requirement, feature request, project spec, or any development task into sequential, atomic coding phases ready to be copy-pasted into AI coding agents (Claude, Cursor, Copilot, Windsurf, etc.).

  Trigger this skill when the user:
  - Pastes a software requirement document, PRD, or feature spec
  - Says things like "chia task", "chia phase", "break this into phases", "split this into steps for my agent", "tôi cần chia requirement này", "giúp tôi chia task"
  - Wants to feed a large requirement into an AI coding agent but it's too big for one prompt
  - Asks how to structure work for AI agents or autonomous coding workflows
  - Mentions "coding agent", "AI agent", "Cursor agent", "Claude Code", "autonomous coding"

  Always use this skill for ANY software breakdown task, even if the user phrases it casually.
---

# AI Phase Splitter

Breaks software requirements into atomic, sequential coding phases — each formatted as a ready-to-copy prompt for AI coding agents.

## Core Principles

1. **Zero Dropped Requirements**: Every detail in the input must appear in some phase. Nothing omitted.
2. **Sequential Dependency**: Each phase only depends on code from prior phases. Order: Environment → Database/Models → Core Logic/Services/APIs → UI/Components → Integration/Testing.
3. **Right Granularity**: Each phase fits an AI agent's context window without overwhelming it, but produces a testable, standalone outcome.
4. **Prompt-Ready Output**: Each phase must be copy-pasteable directly into a coding agent — no extra explanation needed.

---

## Step 1: Clarify Before Splitting

Before generating phases, ask the user the following clarifying questions **in a single message**. Do not skip this step.

```
Before I split this into phases, I need a few quick details:

1. **Tech stack**: What languages, frameworks, and databases are you using? (e.g., Next.js + PostgreSQL + Prisma, or Django + React + MySQL)
2. **Starting point**: Are we building from scratch, or is there an existing codebase? If existing, what's already built?
3. **Target agent**: Which AI coding agent will you be using? (e.g., Claude Code, Cursor, Copilot Workspace, Windsurf) — this helps me tune phase size.
4. **Deployment target**: Where will this run? (e.g., Vercel, AWS, local, Docker)
5. **Anything to avoid**: Any patterns, libraries, or approaches I should NOT use?
```

Wait for the user's answers before proceeding.

---

## Step 2: Architectural Summary

After receiving clarification, output a brief architectural summary **before** the phases:

```
## 🏗️ Architectural Plan

**Project:** [Name from the requirement]
**Stack:** [Confirmed tech stack]
**Total Phases:** [N]
**Splitting Strategy:** [2-3 sentences explaining how you're dividing the work and why — e.g., "I'm separating auth from core CRUD because they have different middleware dependencies. UI phases come after APIs are stable to avoid rework."]

**Phase Overview:**
- Phase 1: [Name] — [1-line description]
- Phase 2: [Name] — [1-line description]
- ...
```

---

## Step 3: Generate the Phases

Output every phase using this exact template. Do not skip or abbreviate any phase.

---

### Phase [X] of [Total]: [Name of the Phase]

**[COPY THE TEXT BELOW TO YOUR AI AGENT]**

---

**Context:** "We are building [Brief feature/project name] using [tech stack]. Previously, we have completed: [Summary of what was built in all prior phases, or 'This is the project setup — nothing has been built yet.' for Phase 1]. Do not modify existing core architecture unless explicitly instructed."

**Current Objective:**
[1-2 sentences clearly stating the goal of this specific phase and what it delivers.]

**Specific Tasks & Technical Requirements:**
- Task 1: [Detailed task with specific logic, field names, API contracts, or constraints drawn directly from the user's requirement]
- Task 2: [...]
- Task 3: [...]
- *(Add as many tasks as needed — do not group unrelated work into one bullet)*

**Definition of Done (DoD — Verification):**
- [ ] [Measurable outcome, e.g., "Running `npm run dev` shows no errors"]
- [ ] [Measurable outcome, e.g., "POST /api/users returns 201 with `{id, email, createdAt}`"]
- [ ] [Measurable outcome, e.g., "The login form renders and submits without console errors"]

---

*(Repeat for every phase)*

---

## Rules for Phase Design

| Rule | Detail |
|------|--------|
| **Never combine DB schema + UI in one phase** | Models must be stable before UI is built |
| **Auth is always its own phase** | Auth middleware affects everything downstream |
| **One API resource per phase** (for CRUD-heavy apps) | Prevents context overflow in agents |
| **Testing/integration is always last** | Can only test what's fully built |
| **Max ~6 tasks per phase** | If more needed, split into Phase Xa and Phase Xb |
| **Always reference the tech stack** | Every phase context must name the stack |

---

## Handling Ambiguity

If the requirement is vague or contradictory in places:
- Flag it explicitly: `⚠️ Ambiguity: The requirement mentions X but doesn't specify Y. I've assumed [assumption] — please correct me if wrong.`
- Make a reasonable assumption and proceed rather than blocking output.
- Group the assumption into Phase 1's DoD so it gets validated early.

---

## Output Language

Match the user's language. If the user writes in Vietnamese, output phases in Vietnamese (but keep technical terms like "API", "POST /route", "migration" in English as-is). If in English, output in English.
