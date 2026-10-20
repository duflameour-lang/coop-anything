# games/fallout-4/NOTES.md

## Identity
- Game: Fallout 4, Steam, engine: Creation Engine
- Exe: <game dir>\Fallout4.exe, version 1.10.163.0,
  SHA-256 DE0D932E3D66D61619C36A2F47678964E1AB7EE4840E7BE57B37C0F15C0D5BD1
- OS: Windows 11 (build 26200)
- F4SE: 0.6.23 (f4se_1_10_163.dll in game dir), runtime 1.10.163.0

## Triage (phase 0) - complete
- No games/fallout-4 write-up, no open fallout issues/PRs in coop-anything before ours.
- Existing attempts:
  - Fallout Together (TiltedEvolution issue #216): prototype, unreleased.
    VERIFIED 2026-10-20: no commits for ref `fallout` in repo - branch does not
    exist. Issue closed "completed" 2025-10 with no release. Revive path dead.
  - F4MP: abandoned, collective archived April 2021 (https://opencollective.com/f4mp).
  - VaultMP: Fallout 3 only - not applicable.
- Decision: build from zero, playbook stages. Hook point: F4SE 0.6.23 plugin
  (official F4SE source: https://github.com/ianpatt/f4se).

## Phase 2 (workspace) - in progress
- Claim issue: cranebeaker/coop-anything#77 (owner: duflameour-lang, 2026-10-20)
- Permissions granted by owner 2026-10-20: GitHub, tool downloads.
