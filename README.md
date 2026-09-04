# My Claude Code Setup

The configuration I actually work with — three models split by what each is good at, with rules
written down so the split does not have to be re-explained every session.

Claude Code conducts. [Antigravity](https://github.com/yuting0624/antigravity-for-claude-code)
(Gemini) does bulk mechanical work. [Codex](https://github.com/openai/codex-plugin-cc) handles
root-cause hunts and long unattended runs. Claude reviews every diff and runs the test gate
itself.

Nothing here is theoretical. The model choices in `CLAUDE.md` come from tests I ran on this
machine, and the numbers are in the file.

## What is in here

| File | What it does |
|---|---|
| `CLAUDE.md` | The global instruction file. Routing table, model benchmarks, scope control, delegation rules |
| `settings.json` | Hook wiring, enabled plugins, marketplaces, theme |
| `no-over-engineering.md` | A hard contract to paste on top of any "design this / plan this" request |
| `hooks/` | Three shell hooks that push code discovery through a knowledge graph instead of grep |
| `agents/` | Subagent definition used by the `impeccable` skill |
| `.mcp.json` | MCP server wiring |
| `codex/AGENTS.md` | Codex's standing instructions — role, scope contract, Spark mode |
| `codex/config.toml` | Codex model, effort, plugins, marketplaces |

Skills live in a separate repository: **[claude-skills-public](https://github.com/rehantariq121/claude-skills-public)**.

## Install

```bash
git clone https://github.com/rehantariq121/claude-code-setup.git
cd claude-code-setup
cp CLAUDE.md settings.json no-over-engineering.md .mcp.json ~/.claude/
cp -r hooks agents ~/.claude/
chmod +x ~/.claude/hooks/*
cp codex/AGENTS.md codex/config.toml ~/.codex/
```

Read `settings.json` before copying it over your own — it enables four plugins and wires five
hooks, and those are choices, not defaults.

---

## The part worth reading: the routing table

The single most useful thing in this setup. Before any non-trivial task, size it, then pick the
executor, the model and the reasoning effort to match. Decide silently in one line.

| Tier | Looks like | Who | Model | Effort |
|---|---|---|---|---|
| T1 trivial | typo, rename, one-line fix, config value | Claude inline | — | — |
| T2 bulk mechanical | scaffolding from a pattern, mass tests, migrations, long reads | Gemini | `gemini-3.7-flash-low` | — |
| T2 agentic | make it run: broken build, red suite, framework upgrade, screenshot-driven UI fix | Gemini | `gemini-3.8-flash-low` | — |
| T3 ordinary feature | normal feature or bug fix inside a known pattern | Claude, or Codex if long | `gpt-5.6-terra` | medium |
| T3 fast | small scoped fix, file already known, verifiable by test | Codex Spark | `gpt-5.3-codex-spark` | xhigh |
| T4 hard | root cause after 2 failures, race conditions, algorithms | Codex | `gpt-5.6-sol` | high |
| T4 alt | huge-context reasoning, cross-repo analysis | Gemini | `gemini-3.1-pro-high` | — |
| T5 critical | auth, payments, security, schema, architecture, irreversible | Claude only | — | — |

Escalate a tier when a silently wrong answer would be expensive. Drop a tier when copying a
pattern that already exists.

### Why those specific Gemini models

Hand-tested on this machine, not vendor benchmarks. `CLAUDE.md` has the full write-up; the short
version:

- **Default to `gemini-3.8-flash-low`.** At LOW effort it passed everything: a hidden 10-case code
  suite 10/10, a 4,006-case money-rounding suite 4006/4006, a 695 KB (~174k token) needle hunt 3/3
  with search tools forbidden, chart-image reading 4/4. Higher effort settings spent more thinking
  tokens for the same answer.
- **Keep bulk mechanical work on `gemini-3.7-flash-low`.** On an identical trivial prompt 3.8 took
  34.5s against 3.7's 16.5s, at roughly 40% higher cost per task. Mechanical work does not need the
  extra reasoning, so 3.8 there is pure loss.
- **Never Gemini for money maths.** 3.8's commission fix used float division and lost 64 cents
  above 2^53, where 3.7's integer version stayed exact. Run the verifier yourself either way.

Escalate on failure, not on suspicion: `-medium` after one failed attempt, `-high` after two. Past
that it is a Codex job, not a bigger Gemini.

### The rule that matters most

> Never trust a delegated agent's self-reported pass. Run the gate yourself in a clean tree.

This is in `CLAUDE.md` because it was earned. Gemini has patched installed packages and stubbed
mocks to force a green build. A model that grades its own homework is not a test gate.

Never delegated at all: auth, payments, data-dropping migrations, secrets, CI config, schema,
anything irreversible.

---

## Keeping Codex in scope

Codex over-engineers by default. Instructions alone do not fix it — this setup uses three levers
together:

1. **Effort.** Override per call with `-c model_reasoning_effort="medium"` for ordinary work. Keep
   `high` for root-cause hunts. Spark is the exception: always `xhigh`.
2. **Standing constraints.** `codex/AGENTS.md` carries a `scope-control` block — no new files, no
   new dependencies, no new abstractions, a ~150-line / 3-file diff budget — plus hard limits, a
   five-question self-audit, and a definition of "done". It loads on every run.
3. **Prompt shape.** Always name the files. Unbounded file scope is what lets it wander.

```
Task: <one sentence, the requirement only>
Touch only these files: <explicit list>
Do not: create files, add dependencies, refactor anything unrelated, rename symbols.
Done when: <observable acceptance criteria>
First reply with the root cause and the minimal fix in 3 lines, then make the change.
```

Required back from every delegation: root cause, files changed, the verification commands and
their **actual output**, and any remaining uncertainty. No skipped tests, no weakened assertions,
no claimed success without evidence.

`no-over-engineering.md` is the counterpart for *new* design work, where "no new files" and a diff
budget do not apply and therefore protect nothing.

---

## Code discovery through a graph, not grep

Three hooks in `hooks/` plus the `codebase-memory-mcp` server in `.mcp.json` redirect code
exploration — find a function, trace callers, understand architecture — to graph queries instead of
grep. Grep stays for string literals, error messages, configs and non-code files.

- `cbm-session-reminder` — fires on startup, resume, clear and compact
- `cbm-subagent-reminder` — injects the same rule into every spawned subagent, via JSON `additionalContext`
- `cbm-code-discovery-gate` — a `PreToolUse` hook on Grep and Glob. Despite the name it never
  blocks a call; it adds graph context and fails silently

The hooks and the MCP server are installed by
[codebase-memory-mcp](https://github.com/search?q=codebase-memory-mcp), not written by me. They are
here so the wiring in `settings.json` makes sense.

---

## Plugins

Enabled via `settings.json`, installed from their own marketplaces (their source is not vendored
here):

| Plugin | Repo | What it does |
|---|---|---|
| `codex` | [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) | Delegate to Codex for root-cause work and long runs |
| `antigravity` | [yuting0624/antigravity-for-claude-code](https://github.com/yuting0624/antigravity-for-claude-code) | Delegate bulk mechanical work to Gemini |
| `caveman` | [JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman) | Strips filler from responses; substance stays |
| `ponytail` | [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) | Enforces the simplest solution that works |

`caveman` and `ponytail` together are the reason this setup produces short answers and small
diffs. One governs the prose, the other governs the code.

---

## What is deliberately not here

Ordinary configuration repositories leak more than people expect. Excluded on purpose:

- `.credentials.json`, `~/.codex/auth.json` — OAuth tokens
- `projects/` — 550 MB of full conversation transcripts, containing client code and everything ever
  pasted into a session
- `telemetry/`, `history.jsonl`, `sessions/`, `shell-snapshots/`, `tasks/`, `paste-cache/`,
  `file-history/`, `session-env/`
- `plugins/cache/` — 21 MB of four plugins' source, none of it mine. Install them from the
  marketplaces above instead
- `~/.codex/*.sqlite` — memories, logs, queue state
- The per-project trust list from `config.toml` — 47 entries naming local client project folders
- A personal spelling lookup table from `CLAUDE.md`, and machine-specific absolute paths
  throughout, replaced with `~` and `%USERPROFILE%`

If you fork a setup repository from anyone — including this one — check these before your first
push.

## Licence

MIT for my own configuration files. `hooks/`, `agents/impeccable-manual-edit-applier.md` and the
generated blocks in `codex/AGENTS.md` come from third-party tools and keep their own terms.
