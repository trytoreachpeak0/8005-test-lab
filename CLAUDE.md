# 8005-test-lab

An agent-first multi-machine tool: a system-agnostic core plus one adapter per
system under test. Two modes over the same core — Run (execute a scenario,
judge it, produce Evidence) and Watch (observe a running production system,
check invariants, repair, produce Observation). The first adapter is the
WIRE_TO_GATE MVP, which lives in `8005-agv-control-server`,
`8005-agv-onboard-hmi` and `slots-simulator`.

## State of the repository

No product code yet. The design is being settled with `/wayfinder`; the map is
the issue labelled `wayfinder:map`, its tickets are that issue's sub-issues.
**Do not write product code here until the map has cleared and a spec exists**
— that means `src/`, `tests/`, `adapters/` and `lab.ps1`.

Everything that is not product code does land as it is decided, and always has:
`CONTEXT.md` has been rewritten over and over without the map clearing
(`git log --oneline -- CONTEXT.md` counts the times), because a vocabulary is
not code. The same goes for `docs/spec/` (each ticket writes its own section on
closing), `docs/agent-notes.md`, `docs/research/`, `tools/toolkit.json`,
`.gitignore`, and the `README.md` files that mark out `evidence/` and
`observations/`.

## What to read, and when it reaches you

**Whether this file reaches a session at its start depends on where the session
was launched.** Claude Code loads `CLAUDE.md` from the working directory and
every directory above it; files in subdirectories are pulled in only when it
reads a file there. This repository is a subdirectory of the workspace root,
where sessions on this map are launched — so until 2026-09-09 this file always
arrived *after* the session had already read something under
`repos/8005-test-lab/`.

Since 2026-09-09 the workspace root `CLAUDE.md` imports this file
(`@repos/8005-test-lab/CLAUDE.md`), so a session launched **there** now gets it
at start. That fixes the common case and nothing else: a session launched inside
this repository, given it with `--add-dir`, or working from a standalone clone
still does not get the import. **So the rule stands — never write a rule here
whose failure mode is "the agent never saw it".** See
`docs/spec/91-agent-briefing.md` section 1.

Reading order for a new session:

1. Workspace root `CLAUDE.md` — write authority table, PowerShell 7.
2. This file — the rules below, and where everything else lives.
3. `docs/agent-notes.md` — what this map has already got wrong. Read it before
   starting, not when stuck.
4. `pwsh ./lab.ps1 help --json` — every command, parameter, outcome value and
   exit code, derived live from the `param` blocks. **No document repeats that
   table.** (Not available until `lab.ps1` lands; until then `docs/spec/80-cli.md`
   is the reference, and it is the only place that is.)
5. `docs/spec/` — only when changing the core or an adapter. The index is
   `docs/spec/README.md`; adding a spec file means adding its row there.

**Tickets are not on that chain.** They are a decision archive, not a manual,
and `gh issue list --state all` shows how many there are by now. Read one only
when a named pointer sends you there — a spec file saying "decided in #17", or
a stub file naming its own ticket. Never go hunting through the issue list for
"the relevant ticket"; that is a search every session would have to redo.

## Rules that apply from day one

- Vocabulary is `CONTEXT.md`. "Agent" means the AI only; there is no resident
  process called an agent in this design. The core says `machine` and `role`;
  words like vehicle, slot or barcode belong to an adapter, never the core.
- Write authority for the systems under test is defined in the workspace root
  `CLAUDE.md` at `C:\Users\szy\Desktop\8005-workspace\CLAUDE.md`. Reaching a
  machine through the lab never grants write access to the repository whose
  code runs there.
- PowerShell 7 only, `#Requires -Version 7` on every `.ps1`. **There is no .NET
  project here and none is planned** — the day pwsh cannot carry it, that is a
  ticket, not a decision to make while editing. ADR 0056 gets referenced when
  such a project exists, not before.
- **Module dependency direction, easy to break by accident.** `Lab.Judge`
  depends on nothing else and does no I/O; `Lab.Remote` and `Lab.Probe` never
  depend on `Lab.Core`. Those two are synced or deployed to the machines under
  test, so depending on the control host means shipping the control host to a
  production vehicle.
- **Naming.** Directories and Markdown are lowercase kebab-case, governed by
  `file-naming-convention/` in the workspace root. PowerShell sources follow the
  PowerShell community convention instead — `Lab.Core.psd1`, `Verb-Noun.ps1` —
  which that document explicitly leaves to each language's own habits.
- **Third-party binaries never enter git.** The single source of truth is
  `tools/toolkit.json`: version, per-file SHA-256, upstream URL, licence. Same
  "recipe, not the build" habit as `remote-ops/onboard-hmi/payload/` and
  `releases/`.
- **Never write a bare `Import-Module Pester`.** All three machines carry
  Windows' own Pester 3.4.0 on `PSModulePath`, and it is incompatible with the
  Pester 5 syntax the tests are written in. Import the Toolkit copy by path.
- **Whether a piece of code needs a test is decided by one question: when this
  breaks, does it make a noise?** A predicate evaluator that scores `unknown` as
  `pass` leaves Watch permanently green; a broken NDJSON framer fails the very
  next command. The first must be covered, the second need not be. Full table in
  `docs/spec/95-self-test.md`. There is no coverage percentage here, on purpose.
- **Replay never lives under `tests/`.** `lab invariant replay` produces
  Evidence — it is a deliverable, one of the destination's own completion
  criteria. Filed under `tests/` it would share a green with "the engine is
  tested", and those are different claims.
- Documents people read are in Chinese; agent instruction files and code
  comments are in English. Identifiers, paths, commands, gate and slice names
  stay in English inside Chinese prose.
- Issue titles and bodies, PR titles and bodies, commit message bodies: Chinese,
  with a conventional-commit prefix in English.
