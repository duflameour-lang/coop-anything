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
## Stage 1 (playbook) - step 1 done, 2026-10-20
- f4-coop plugin (classic F4SE 0.6.23 Query/Load API) builds with
  cl /LD /std:c++17 against ianpatt/f4se tag v0.6.23 headers only.
- First successful run in a copied install (robocopy, 653 files,
  0 failures): f4coop.log shows "loaded, handle=1 runtime=0x010A0A30",
  GameDataReady, GameLoaded.
- For the next agent: F4SE headers are not self-contained; typedef
  UInt32/UInt64 before including PluginAPI.h; /I must point at the
  f4se repo root. The owner's Git Bash mangles multi-line pastes
  (eats #, ->, braces): code files go through Notepad, terminal gets
  single-line commands only.
- Still open for Stage 1 exit test: per-frame log + player position
  read (g_player RelocPtr 0x05AA4388, pos at TESObjectREFR+0xD0,
  per F4SE v0.6.23 sources) - not implemented yet.
## Stage 1 (playbook) - COMPLETE, 2026-10-20
- Position thread verified in game: hundreds of samples while walking,
  world-scale coords (~-80186, 91135, 7852), stable Z on flat ground.
- Artifact: first tick after world load reads 2048/2048/0 (player object
  exists before being placed); real coords from next tick.
- Honest caveats: 10 Hz background poll, not a per-frame hook; world id
  not read yet (needed later to hide puppets in other cells).