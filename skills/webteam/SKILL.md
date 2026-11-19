---
name: webteam
description: Orchestrate a team of web-* subagents that build a website across all 11 layers (planning, design, frontend, backend, database, deploy, security, performance, SEO, testing, monitoring) in dependency-ordered parallel waves, with this session as the supervisor that verifies each wave. Use when the user runs /webteam, or asks to build a site "with the agent team" or with multiple agents.
---

# /webteam: supervisor playbook

You (the main session) are the supervisor. Subagents could start their own subagents, but you keep dispatching in this session: approvals reach the user here, and every gate is visible to them. You never trust a report on its own: a wave is done only when `npm run check` passes and you've looked at the diff.

Agent names: `web-planner` etc. when installed in `~/.claude/agents`, or `webteam:web-planner` etc. when installed as a plugin. Use whichever the Agent tool lists.

## 0. Size gate: is a team even worth it?
Read the user's brief. If the site has **≤ 3 pages and no backend** (landing page, portfolio, event page), say so and build it solo in this session from the template. The team costs several times the tokens and only pays off when there's real backend, data, or several user journeys. Continue only if the user still wants the team.

## 1. Setup
- If the folder has no `package.json`: `npx degit itej13/web-starter . && git init -q && npm install`.
- Commit the starting point (`git add -A && git commit -qm "chore: start from web-starter"`) so every wave is a reviewable diff.

## 2. Waves
Start every agent in a wave **in one message** (several Agent calls) so they run concurrently. Agents only write inside their owned files (table in the project's CLAUDE.md), so parallel agents never collide.

| Wave | Agents (parallel within a wave) | Needs |
|---|---|---|
| 1 Plan | web-planner | brief |
| **Human gate** | Show the user the plan summary, the `ASSUMPTION:` lines and the layers marked "no". Get approval before any build. | |
| 2 Foundations | web-designer ∥ web-backend (skip if no backend) | PLAN.md |
| 3 Build | web-frontend | design system + API contract |
| 4 Review | web-security ∥ web-performance ∥ web-seo ∥ web-tester | finished code |
| 5 Fix | each owner named in an `OWNER:` finding, in parallel, given only its findings | review reports |
| 6 Ship | web-ship | green check |

Skip any agent whose layer the plan marks "no".

## 3. Token discipline (each agent starts cold, so keep its prompt small)
- Each agent prompt is: the brief in ≤ 5 lines, "Read docs/PLAN.md and CLAUDE.md first", and the task. Don't paste the plan or earlier reports; they're in files.
- Review and fix agents get the changed-file list (`git diff --name-only <last-review-commit>`), not "review the project".
- Fix agents get only their own `OWNER:` lines.
- Agents already know which skills to load (each has a skill map); don't tell them.
- Per-agent models are pinned in frontmatter (haiku for SEO/ship, sonnet for frontend/perf/tests, your model for planning/design/backend/security). Don't override them unless a wave failed for lack of reasoning.

## 4. Gate after every wave (you do this, not an agent)
1. `npm run check`, plus `npm run e2e` from wave 4 on. Paste the failing lines if it fails.
2. `git diff --stat`: did any agent write outside its owned files? If so, revert those hunks and re-dispatch.
3. Update `docs/STATUS.md`: status plus *evidence* per layer, and cross-owner findings under "Open findings".
4. `git add -A && git commit -qm "<wave>: <summary>"`.
5. On failure: send the error back to the owning agent once (resume it with SendMessage if it's still addressable, rather than starting a new one). If it fails twice, stop and show the user the error and what you tried; don't loop.

## 5. Stop and ask the user before
Anything that creates accounts or costs money (Marketplace DB/auth provisioning), production deploys, pushing to GitHub or creating a repo, adding real secrets. Agents are told to list these, not do them. You collect them and ask once.

## 6. Finish
Report in under 10 lines: the STATUS.md table (layer → status → evidence), the preview URL, pending approvals, and open findings you chose not to fix (with the reason).
