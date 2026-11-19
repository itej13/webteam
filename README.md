# webteam

**A supervised team of Claude Code subagents that builds a website across all 11 layers of web development, in parallel where the work allows it, and in order where it doesn't.**

Every website, built properly, touches the same 11 layers:

| # | Layer | Covered by |
|---|---|---|
| 1 | Planning | `web-planner` |
| 2 | Design (UI/UX) | `web-designer` |
| 3 | Frontend | `web-frontend` |
| 4 | Backend | `web-backend` |
| 5 | Database | `web-backend` (whoever writes the API writes the schema) |
| 6 | Hosting & deployment | [web-starter](https://github.com/itej13/web-starter) template + `web-ship` |
| 7 | Security | `web-security` |
| 8 | Performance | `web-performance` |
| 9 | SEO (+ accessibility basics) | template + `web-seo` |
| 10 | Testing | template + `web-tester` |
| 11 | Monitoring & analytics | template + `web-ship` |

`webteam` gives each layer an owner and adds a **supervisor** (the `/webteam` skill, running in your main Claude Code session). The supervisor dispatches the agents in waves, checks each wave against objective gates, and stops to ask you before anything irreversible.

---

## Why waves, not "11 agents at once"

The obvious design, every layer agent running simultaneously, fails in practice:

- **Layers depend on each other.** The backend needs the data model, the frontend needs the API shape, and tests and audits need finished code. Agents started too early guess at work that doesn't exist yet, then conflict.
- **Parallel writers clobber shared files.** Two agents editing `layout.tsx` at once lose each other's work.
- **Every subagent starts cold.** Each one re-reads the project, so 11 agents cost far more tokens than one session, for no gain on small sites.
- **Supervisors that trust reports ship broken code.** "Done ✅" from an agent isn't evidence.

So `webteam`:

1. **Runs dependency-ordered waves.** Agents run in parallel only inside a wave, where their inputs already exist.
2. **Gives every file area exactly one owner** (the table lives in the project's `CLAUDE.md`), so parallel agents never collide.
3. **Gates every wave on facts.** The wave passes only if `npm run check` (lint + typecheck + unit tests + build) and, later, Playwright e2e pass, and the diff stays inside each agent's owned files. Then it commits.
4. **Refuses to run on small sites.** For ≤ 3 pages with no backend, it tells you to build solo, because the team isn't worth the tokens.

```
brief ──▶ 1 Plan ──▶ [you approve] ──▶ 2 Design ∥ Backend ──▶ 3 Frontend
                                                                   │
           6 Ship ◀── 5 Fix (owners, parallel) ◀── 4 Security ∥ Performance ∥ SEO ∥ Tests
                 │
          each arrow = gate: npm run check · ownership diff · STATUS.md · git commit
```

---

## The team

| Agent | Owns | Model · effort · max turns | Preloaded skill | Loads on demand (only when the trigger applies) |
|---|---|---|---|---|
| `web-planner` | `docs/PLAN.md` | inherit · high · 15 | none | `engineering:system-design`, `engineering:architecture`, `vercel:ai-sdk` |
| `web-designer` | `app/globals.css`, `components/ui/**` | inherit · medium · 40 | none: built-in **design brief** | `ui-ux-pro-max`, `landing-page`, `pick-ui-library`, `vercel:shadcn`, `impeccable`, `frontend-design:frontend-design`, `design-taste-frontend` (high-end only) |
| `web-frontend` | pages, layouts, feature components, `lib/content/**`; may only append to `site.routes` | sonnet · medium · 80 | `vercel:nextjs` + built-in **motion brief** | `vercel:next-cache-components`, `vercel:react-best-practices`, `vercel:shadcn`, `vercel:ai-sdk`, `emil-design-eng`, `apple-design` |
| `web-backend` | `app/api/**`, `lib/server/**`, `db/**`, `proxy.ts`, `next.config.ts`, server actions, and their unit tests | inherit · high · 80 | none | `vercel:marketplace`, `vercel:vercel-storage`, `vercel:auth`, `vercel:vercel-functions`, `vercel:ai-sdk`, `vercel:ai-gateway`, `vercel:workflow`, `vercel:queues`, `data:sql-queries` |
| `web-security` | review only (≤ 10-line fixes); also usable standalone on any repo | inherit · medium · 40 | none | `graphify` code-only graph for repos with more than ~80 files (attack-surface triage; leads, not findings), `security-review`, `vercel:vercel-firewall` |
| `web-performance` | review only (≤ 10-line fixes) | sonnet · low · 25 | none | `vercel:next-cache-components`, `vercel:cdn-caching`, `vercel:react-best-practices` |
| `web-seo` | `lib/site.ts`, sitemap, robots, OG images | haiku · low · 25 | none | `design:accessibility-review` |
| `web-tester` | `e2e/**`, plus unit tests where an owner left none | sonnet · medium · 50 | none | `anthropic-skills:webapp-testing`, `engineering:testing-strategy` |
| `web-ship` | `.github/**`, `instrumentation.ts`, `vercel.ts`, `.env.example` | haiku · low · 30 | none | `vercel:deploy`, `vercel:vercel-cli`, `vercel:env-vars`, `vercel:deployments-cicd`, `vercel:verification`, `engineering:deploy-checklist` |

Every skill is **optional**: agents skip any that aren't installed. Where they come from:

- `vercel:*`: the official Vercel plugin (`/plugin install vercel@claude-plugins-official`)
- `frontend-design:frontend-design`: the official frontend-design plugin
- `engineering:*`, `design:*`, `data:*`: Anthropic's knowledge-work plugins
- `anthropic-skills:webapp-testing`: [anthropics/skills](https://github.com/anthropics/skills)
- `security-review`: built into Claude Code
- `graphify`: [safishamsi/graphify](https://github.com/safishamsi/graphify) (`pip install graphifyy`); used code-only, with no AI cost
- `ui-ux-pro-max`, `impeccable`, `landing-page`, `pick-ui-library`, `emil-design-eng`, `apple-design`, `find-animation-opportunities`, `improve-animations`: community skills

---

## How it keeps token costs down

| Technique | Effect |
|---|---|
| **Size gate** | Small sites don't start the team at all. |
| **Model per agent** | Mechanical work (SEO, deploy) runs on Haiku; routine build and review work runs on Sonnet; only planning, design, backend and security use your main model. |
| **`effort` + `maxTurns` per agent** | Caps reasoning depth and stops runaway loops. |
| **One preloaded skill at most** | Preloaded skills are injected in full at startup. Everything else is a *skill map* in the agent's prompt: load X only when trigger Y applies. Large skills (e.g. a 17k-token design skill) are explicitly gated. |
| **Distilled briefs, not merged skills** | The designer and frontend agents carry a ~700-word design brief and a ~200-word motion brief, distilled from eight overlapping design and motion skills (~28k words). Routine sites need no skill loads at all; the originals stay available for depth. Merging the skills into one always-loaded file would have cost ~38k tokens per run and dropped their scripts and data. |
| **Explicit `tools:` lists** | Agents don't inherit your MCP connectors or the Agent tool, so their context carries no tool schemas they don't need and they can't fan out. |
| **Files as the shared memory** | The plan, contract and status live in `docs/`. Prompts say "read docs/PLAN.md" instead of pasting it. |
| **Diff-scoped reviews** | Review agents get only the files changed since the last review. |
| **Short report formats** | Every agent reports in ≤ 8 lines in a fixed format. |
| **Resume, don't respawn** | Fixes go back to the same agent via SendMessage where possible, so it keeps its context. |

---

## Install

**As a plugin** (agents appear as `webteam:web-planner`, …):

```
/plugin marketplace add itej13/webteam
/plugin install webteam@webteam
```

**Or by hand** (agents appear as `web-planner`, …):

```bash
git clone https://github.com/itej13/webteam.git
cp webteam/agents/*.md ~/.claude/agents/
mkdir -p ~/.claude/skills && cp -R webteam/skills/webteam ~/.claude/skills/
```

## Use

In an empty folder (or one created from [web-starter](https://github.com/itej13/web-starter)):

```
/webteam A booking site for a yoga studio: class schedule, sign-up, member dashboard, admin page to add classes.
```

The supervisor will:

1. Copy the template if the folder is empty (`npx degit itej13/web-starter`).
2. Run the planner and show you the plan, its assumptions, and which layers it decided to skip. **You approve before anything is built.**
3. Run the waves, committing after each passing gate and keeping `docs/STATUS.md` up to date with evidence per layer.
4. Collect everything that needs your consent (creating DB/auth accounts, production deploys, pushing to GitHub, real secrets) and ask once.
5. Finish with the STATUS table, the preview URL, and any open findings.

## Requirements

- Claude Code with subagent support.
- The [web-starter](https://github.com/itej13/web-starter) layout. Agents assume Next.js App Router, `npm run check`, `docs/PLAN.md` and the ownership table in `CLAUDE.md`. Other stacks need the ownership table and commands edited.
- Node 22+; a Vercel account for preview deploys (optional).

## First trial run

A 3-page bakery site with a contact API (validation, honeypot, logging), local only:

- **Five waves, about 6 minutes of agent time.** The result: 19 unit tests, 5 Playwright tests, 0 high-severity security findings, and two low-severity ones fixed in the fix wave.
- **The gates caught what reports alone would have missed.** One agent edited a field it didn't own, and the gate reverted it. A security fix (a required `Content-Type` header) could have broken the form; the e2e journeys proved it didn't.
- **Cost:** about 290k subagent tokens, roughly 35–47k per agent, mostly fixed startup context. That's why the size gate exists: below about 4 pages with light backend logic, a single session is cheaper.
- **The run exposed three gaps, now fixed:**
  - content files and `next.config.ts` had no owner;
  - test ownership was ambiguous (a file's owner now owns its unit tests);
  - the `site.routes` append rule was too loose.

## Limits

- Agents don't share live memory, only files. A vague API contract in `docs/PLAN.md` is the most common source of bugs, so the planner is told to be exact.
- The supervisor retries a failed wave once, then hands the problem to you instead of looping.
- It's tuned for Next.js on Vercel. The ideas (waves, ownership, factual gates) transfer; the commands don't.

## License

MIT
