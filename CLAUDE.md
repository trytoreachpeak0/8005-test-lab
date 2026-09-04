# 8005-test-lab

An agent-first multi-machine test lab: a system-agnostic core plus one adapter
per system under test. The first adapter is the WIRE_TO_GATE MVP, which lives
in `8005-agv-control-server`, `8005-agv-onboard-hmi` and `slots-simulator`.

## State of the repository

Decisions only, no code yet. The design is being settled with `/wayfinder`;
the map is the issue labelled `wayfinder:map`, its tickets are that issue's
sub-issues. Do not write product code here until the map has cleared and a
spec exists.

## Rules that apply from day one

- Vocabulary is `CONTEXT.md`. "Agent" means the AI only; there is no resident
  process called an agent in this design. The core says `machine` and `role`;
  words like vehicle, slot or barcode belong to an adapter, never the core.
- Write authority for the systems under test is defined in the workspace root
  `CLAUDE.md` at `C:\Users\szy\Desktop\8005-workspace\CLAUDE.md`. Reaching a
  machine through the lab never grants write access to the repository whose
  code runs there.
- PowerShell 7 only, `#Requires -Version 7` on every `.ps1`. Any .NET project
  follows the toolchain baseline in
  `8005-agv-program/docs/adr/cross/0056-dotnet-toolchain-baseline.md`.
- Documents people read are in Chinese; agent instruction files and code
  comments are in English. Identifiers, paths, commands, gate and slice names
  stay in English inside Chinese prose.
- Issue titles and bodies, PR titles and bodies, commit message bodies: Chinese,
  with a conventional-commit prefix in English.
