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
`CONTEXT.md` has been rewritten twelve times without the map clearing, because a
vocabulary is not code. The same goes for `docs/spec/` (each ticket writes its
own section on closing), `docs/research/`, `tools/toolkit.json`, `.gitignore`,
and the `README.md` files that mark out `evidence/` and `observations/`.

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
- Documents people read are in Chinese; agent instruction files and code
  comments are in English. Identifiers, paths, commands, gate and slice names
  stay in English inside Chinese prose.
- Issue titles and bodies, PR titles and bodies, commit message bodies: Chinese,
  with a conventional-commit prefix in English.
