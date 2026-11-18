# Agent skills

Skills I wrote for AI coding agents. Each folder under `skills/` is one skill: a `SKILL.md` plus
any reference files it reads. They use the plain `SKILL.md` format, so they work in any agent
that loads skills from a folder.

Most of them started inside [Baton](https://www.npmjs.com/package/batonhq), my tool for running
several coding agents on one repo. The ones marked "Baton" in the table call Baton commands or its
MCP server and need Baton installed. The rest work on their own.

## Skills

Agents: CC = Claude Code, Cx = Codex, Cu = Cursor, AG = Antigravity.

| Skill | What it does | Works with |
|---|---|---|
| [`assign-task`](skills/assign-task/SKILL.md) | Split a request across teammates by who owns which area and propose each task with a full brief. | CC, Cx, Cu, AG (Baton) |
| [`basic-setup`](skills/basic-setup/SKILL.md) | Set up a new project right from the first commit, or audit one that drifted: structure, secret-leak hooks, scripts. | CC, Cx, Cu, AG |
| [`better-auth`](skills/better-auth/SKILL.md) | Reference for the Better Auth library: real option names, schema, plugins and the edge cases that break integrations. | CC, Cx, Cu, AG |
| [`bug-fix`](skills/bug-fix/SKILL.md) | Fix a bug without regressions: reproduce first, find the root cause, get an approved plan, verify, then commit. | CC, Cx, Cu, AG |
| [`create-baton-skill`](skills/create-baton-skill/SKILL.md) | Write a new skill or upgrade an existing one: pick its shape, draft it, score it, test it against a baseline. | CC, Cx, Cu, AG |
| [`delegate`](skills/delegate/SKILL.md) | Keep the planning in one agent and hand self-contained halves of a task to other agents. | CC (Baton) |
| [`e2e-verify`](skills/e2e-verify/SKILL.md) | Prove a UI change works by running Playwright in a throwaway browser against the local app. | CC, Cx, Cu, AG (Baton) |
| [`handoff`](skills/handoff/SKILL.md) | Hand unfinished work to another agent before a session runs out, and pick up a handoff on the other side. | CC, Cx, Cu, AG (Baton) |
| [`map-codebase`](skills/map-codebase/SKILL.md) | Build a knowledge graph and a short `CODEBASE.md` map so agents stop reading the whole repo. | CC, Cx, Cu, AG (Baton) |
| [`memory-light`](skills/memory-light/SKILL.md) | Keep context lean on long work: recall before exploring, write facts to files, compact or hand off early. | CC, Cx, Cu, AG |
| [`monolith-split`](skills/monolith-split/SKILL.md) | Split one codebase into a frontend and separately deployed backend services without losing auth checks or data. | CC, Cx, Cu, AG |
| [`offline-conversion`](skills/offline-conversion/SKILL.md) | Make an online-only feature work offline without losing sales or other people's writes. | CC, Cx, Cu, AG |
| [`privacy-compliance`](skills/privacy-compliance/SKILL.md) | Take an app from "we have some legal pages" to an evidenced privacy gap report and working controls (India, EU/UK, US). | CC, Cx, Cu, AG |
| [`safe-refactor`](skills/safe-refactor/SKILL.md) | Restructure code without changing behaviour, using a worktree, a green test baseline and the code graph. | CC, Cx, Cu, AG |
| [`site-search`](skills/site-search/SKILL.md) | Build or fix search inside a site or app: right-sized engine, tested relevance, measured latency. | CC, Cx, Cu, AG |
| [`stack-migration`](skills/stack-migration/SKILL.md) | Move a codebase to another stack feature by feature, in phases, with a parity check before each next phase. | CC, Cx, Cu, AG |
| [`token-efficient-coding`](skills/token-efficient-coding/SKILL.md) | Spend fewer tokens: navigate a map, read small slices, make small diffs, never re-read. | CC, Cx, Cu, AG |
| [`traceable-changes`](skills/traceable-changes/SKILL.md) | One logical change per commit, clear messages and worktrees, so any agent's bug is easy to find and revert. | CC, Cx, Cu, AG |
| [`verify-before-done`](skills/verify-before-done/SKILL.md) | Check a change really works before calling it done: re-read, confirm APIs exist, run build and tests, get a skeptic pass. | CC, Cx, Cu, AG |

## Install

Copy a skill folder into the skills folder your agent reads.

```bash
git clone https://github.com/Rakshan001/agent-skills.git
cp -R agent-skills/skills/bug-fix ~/.claude/skills/     # Claude Code, every project
cp -R agent-skills/skills/bug-fix .claude/skills/       # Claude Code, this project only
cp -R agent-skills/skills/bug-fix .agents/skills/       # this project, for agents that read .agents/skills
```

Where each agent looks:

| Agent | All projects | One project |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex | `~/.agents/skills/` or `~/.codex/skills/` | `.agents/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` or `.claude/skills/` |
| Antigravity | `~/.gemini/antigravity/skills/` | `.agents/skills/` |

These paths change between agent versions, so check your agent's docs if a skill does not show
up. Keep the whole folder: several skills read files from their `references/` folder.

## License

MIT. See [LICENSE](LICENSE).
