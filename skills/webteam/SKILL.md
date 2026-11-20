---
name: webteam
description: Orchestrate a team of web-* subagents that build a website on any web stack (Next.js, Astro, SvelteKit, Django, Rails, Laravel, …) across all 11 layers (planning, design, frontend, backend, database, deploy, security, performance, SEO, testing, monitoring) in dependency-ordered parallel waves, with this session as the supervisor that verifies each wave. Works in an empty folder or an existing repo. Use when the user runs /webteam, or asks to build a site "with the agent team" or with multiple agents.
---

# /webteam: supervisor playbook

You (the main session) are the supervisor. Subagents could start their own subagents, but you keep dispatching in this session: approvals reach the user here, and every gate is visible to them. You never trust a report on its own: a wave is done only when the plan's `check` command passes and you've looked at the diff.

`docs/PLAN.md` § **Stack** is the contract every agent works from: framework, commands (`check`, `e2e`, `dev`, `build`), the file-ownership table and the deploy target. Below, "`check`" means the command written there.

Agent names: `web-planner` etc. when installed in `~/.claude/agents`, or `webteam:web-planner` etc. when installed as a plugin. Use whichever the Agent tool lists.

## 0. Size gate: is a team even worth it?
Read the user's brief. If the site has **≤ 3 pages and no backend** (landing page, portfolio, event page), say so and build it solo in this session. The team costs several times the tokens and only pays off when there's real backend, data, or several user journeys. Continue only if the user still wants the team.

## 1. Setup
- `git init -q` if the folder isn't a repo yet.
- **Existing code:** if the tree has uncommitted changes, ask the user before committing them as the baseline. Every wave must be a reviewable diff.
- **Empty folder:** don't scaffold yet. The stack is chosen in the plan, and you scaffold after the human gate.

## 2. Waves
Start every agent in a wave **in one message** (several Agent calls) so they run concurrently. Agents only write inside their owned files (the Ownership table in PLAN.md § Stack), so parallel agents never collide.

| Wave | Agents (parallel within a wave) | Needs |
|---|---|---|
| 1 Plan | web-planner | brief |
| **Human gate** | Show the user the plan summary, the **Stack section** (framework, commands, ownership, deploy target), the `ASSUMPTION:` lines and the layers marked "no". Get approval before any build. | |
| Scaffold + baseline | you (below) | approved stack |
| 2 Foundations | web-designer ∥ web-backend (skip if no backend) | PLAN.md |
| 3 Build | web-frontend | design system + API contract |
| 4 Review | web-security ∥ web-performance ∥ web-seo ∥ web-tester | finished code |
| 5 Fix | each owner named in an `OWNER:` finding, in parallel, given only its findings | review reports |
| 6 Ship | web-ship | green check |

Skip any agent whose layer the plan marks "no".

**Scaffold + baseline** (you, not an agent):
1. *Empty folder only:* move `docs/PLAN.md` to the scratchpad (scaffolders refuse or overwrite non-empty folders), run the plan's Source (`npx degit itej13/web-starter .`, or the framework's official command with non-interactive flags into `.`), then put PLAN.md back over the template's copy. For web-starter, first carry the template's § Stack (with its Ownership table) into the planner's PLAN.md, replacing the "web-starter default" placeholder. Install dependencies plus any dev tools the `check` command needs.
2. Run `check` once. The bare baseline must be green before any agent starts. If the command itself is wrong, fix it in PLAN.md (you). If an existing repo is already red, show the user and ask; don't let agents inherit a failing baseline.
3. Create `docs/STATUS.md` if missing: a table of layer · owner · status · evidence, plus an "Open findings" section.
4. Commit (`chore: scaffold <stack>` or `chore: baseline`).

## 3. Token discipline (each agent starts cold, so keep its prompt small)
- Each agent prompt is: the brief in ≤ 5 lines, "Read docs/PLAN.md (§ Stack has your paths and commands) and CLAUDE.md if present, first", and the task. Don't paste the plan or earlier reports; they're in files.
- Review and fix agents get the changed-file list (`git diff --name-only <last-review-commit>`), not "review the project".
- Fix agents get only their own `OWNER:` lines.
- Agents already know which skills to load (each has a skill map); don't tell them.
- Per-agent models are pinned in frontmatter (haiku for SEO/ship, sonnet for frontend/perf/tests, your model for planning/design/backend/security). Don't override them unless a wave failed for lack of reasoning.

## 4. Gate after every wave (you do this, not an agent)
1. `check`, plus `e2e` from wave 4 on. Paste the failing lines if it fails.
2. `git diff --stat`: did any agent write outside its rows of the Ownership table? If so, revert those hunks and re-dispatch.
3. Update `docs/STATUS.md`: status plus *evidence* per layer, and cross-owner findings under "Open findings".
4. `git add -A && git commit -qm "<wave>: <summary>"`.
5. On failure: send the error back to the owning agent once (resume it with SendMessage if it's still addressable, rather than starting a new one). If it fails twice, stop and show the user the error and what you tried; don't loop.

## 5. Stop and ask the user before
Anything that creates accounts or costs money (hosted DB/auth provisioning, paid plans), production deploys, pushing to GitHub or creating a repo, adding real secrets. Agents are told to list these, not do them. You collect them and ask once.

## 6. Finish
Report in under 10 lines: the STATUS.md table (layer → status → evidence), the preview URL (or the manual deploy steps), pending approvals, and open findings you chose not to fix (with the reason).
