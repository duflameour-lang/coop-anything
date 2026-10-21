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
## Stage 2 (playbook) - step 1 done, 2026-10-21
- UDP transport verified: plugin sends PosPacket{magic "F4CP", seq, x,y,z}
  to 127.0.0.1:31337 at 10 Hz; standalone recv.exe tool decodes it live.
- Caveat: first ticks after world load still emit the 2048/2048/0 placeholder
  (player placed into world slightly later); filter client-side for now,
  proper fix later (send only after first real position read).
- Caveat: send-only, no receive path in-game yet; no world id in packet yet.
## Stage 2 step 1 verified - 2026-10-21
- Live coordinates over UDP confirmed: seq 660-687 stationary (packets
  flow, coords frozen - expected), seq 688+ movement, X/Y changing,
  Z stable on flat ground. No sequence gaps. ~10 pps.
- Tooling: recv.exe (tools/recv.cpp) is our first test bot - the test
  runs without a second human (playbook rule: test with bots).
## Stage 3A result - 2026-10-21
- Full chain proven: rx packets arrive (circle path in log), memcpy writes
  to TESObjectREFR::pos - own pos reads confirm the write.
- BUT: renderer/physics ignore the logical pos field; player appears
  stationary despite coords changing. Two-position reality: logical pos
  vs physics/animation position.
- Next (3B): use the engine's MoveRefrToPosition (v0.6.23 GameReferences.h,
  addr 0x013FE7E0, signature verified) instead of raw memcpy; also move
  writes onto the game thread (currently background thread - unsafe for
  real mutation, acceptable for this throwaway test).
## Stage 3B implemented - 2026-10-21 (awaiting in-game test)
- v6 architecture: net thread only records the latest ghost position
  (mutex-guarded, version counter); the actual move runs as an
  ITaskDelegate posted through F4SE's task interface (AddTask). F4SE pumps
  those tasks from a hook inside the engine's message-queue processing
  (Hooks_Threads.cpp -> MessageQueueProcessTask), i.e. on the game thread.
- Move call mirrors the only in-tree caller, f4se/PapyrusObjectReference.cpp
  ConnectObjects: MoveRefrToPosition(refr, &nullHandle, refr->parentCell,
  GetWorldspace(refr), pos, rot). Offsets from tag v0.6.23 sources:
  MoveRefrToPosition 0x013FE7E0, TESObjectREFR::GetWorldspace 0x0040F290,
  parentCell +0xB8, g_invalidRefHandle 0x038CCE04 (value read at runtime).
  Rotation = euler Z (radians) derived from move direction.
- Backpressure: at most one apply task in flight, posted only when the
  ghost version changed - if the bot stops, the player is released within
  ~100 ms instead of being pinned to the last ghost position.
- recv.exe fix: the anchor was sometimes landing on the 2048/2048/0
  pre-load placeholder; the bot now skips it (anchor = first real pos).
- Correction vs earlier session notes: PlaceAtMe_Native in tag v0.6.23 is
  0x0140B0E0 (GameObjects.cpp), not 0x01159E80. Verified by reading the
  tag source; matters for the Stage 4 puppet spawn.
- Reproducible build: build.bat (vswhere -> vcvars64 -> cl /LD /std:c++17,
  ws2_32); .gitignore fixed to actually exclude *.dll/*.exe/*.obj.
- Unknown until tested: does MoveRefrToPosition on the *player* at 10 Hz
  behave (camera, physics, cell transitions)? Interior case relies on
  GetWorldspace()==NULL being accepted (same as Papyrus MoveTo).
- Test plan: fresh recv.exe, launch test copy via F4SE, load an outdoor
  save; expect visible dragging in a ~4.3 m circle + "moveRefr #N" lines
  with cell/ws pointers and post-move readback in f4coop.log.
## Stage 3B first test result - 2026-10-21 (owner playtest, v6)
- CORE RESULT: MoveRefrToPosition on the player WORKS. The engine accepted
  network-driven moves - the circle executed (moveRefr lines in the log).
  Authoritative network movement of the local player is proven; this is the
  core mechanic of co-op movement. (Whether the world was ever visible is
  unconfirmed - see the bug below.)
- BUG: applying moves during save load hangs the loading screen forever
  (infinite load; owner observed the save loading endlessly while the log
  showed the circle running). v6 armed applying off kMessage_GameLoaded,
  which fires before the player is placed (same root as the 2048/2048/0
  artifact); the engine places the player, we teleport it 10x/s, and the
  loading screen never dismisses.
- v7 fix, three gates:
  1. Lifecycle: apply armed only by kMessage_PostLoadGame(success);
     disarmed on PreLoadGame/NewGame (F4SEMessagingInterface).
  2. UI: task re-checks UI::numPauseGame ("isInMenuMode", UI singleton
     base+0x058D0898, field +0x1E0, per GameMenus.h tag v0.6.23) - covers
     loading screens, pause menu, Pip-Boy. Blocks log transitions.
  3. Distance: skip any move >5000 units from current pos (stale-anchor /
     corrupt-packet protection).
  Log lines now have timestamps (HH:MM:SS.mmm) for load-window forensics.
- Unknowns for next test: does PostLoadGame fire before or after the fade
  finishes (log timestamps will show), is numPauseGame reliably 0 in
  gameplay (VATS/Pip-Boy should block - watch "blocked:" lines), and does
  the load now complete normally.
- Retest plan: fresh recv.exe, load save, expect normal load, then
  circling in-game; if the load still hangs, the fallback is a time-delay
  grace after PostLoadGame before the first apply.
## Stage 3B second test result - 2026-10-21 (owner playtest, v7)
- v7 crashed the game on every save load. Root cause found in F4SE source,
  Hooks_SaveLoad.cpp:51: kMessage_PostLoadGame is dispatched with
  data = (void*)result - the bool is VALUE-ENCODED AS THE POINTER (0x1/0x0),
  not a pointer to a bool. v7's handler did *(bool*)msg->data -> read at
  address 0x1 -> access violation on the game thread at the exact moment
  the load finishes. Log forensics: PreLoadGame logged, PostLoadGame line
  absent (deref was before the logline), rx echo kept flowing until the
  process died. LESSON for all F4SE plugin work: several F4SE messages
  (PostLoadGame, GameDataReady per Hooks_GameData.cpp:29) value-encode
  their payload as the data pointer; check the Hooks_*.cpp dispatch site
  before dereferencing msg->data.
- Also learned from the v6 section of the same log: the v6 moves ran during
  the load screen and the ENGINE CONSUMED THEM - post-move readback showed
  pos advancing to the previous target one step late (engine applies the
  move asynchronously, next frame), cell/ws pointers sane (exterior).
  MoveRefrToPosition itself is not the crasher and not the load-hang bug:
  the hang comes from applying it during the load, the crash came from
  the message handler. No moves have yet run in a fully-loaded game -
  that is what v8's retest verifies.
- v8: PostLoadGame handler reads ok = (msg->data != nullptr); comment
  added on GameDataReady too (same encoding, handler already safe).
- Bonus diagnostics now in the log: timestamps (v7) made this whole
  forensics possible in one reading; keep them.
## Stage 3B third test result - 2026-10-21 (owner playtest, v8)
- Crash FIX CONFIRMED: "PostLoadGame ok=1 - apply armed" logged, no crash.
- Infinite load is BACK and now fully understood:
  - Zero "blocked: menu/pause mode" lines -> UI::numPauseGame (if the
    offset is even right; the UI class layout in GameMenus.h has no
    STATIC_ASSERTs) does not flag the loading screen. The pause gate
    never fired during the whole load.
  - moveRefr #1 ran 7 s after arming, WHILE the load screen was still up;
    subsequent moves ticked only once per ~12 s - F4SE's task queue is
    pumped from the normal frame loop, so the game never reached normal
    frames. Each move restarts cell streaming around the player, so the
    loading screen can never dismiss. Hang mechanism confirmed.
- v9 fix: 10 s grace after PostLoadGame before any move applies
  (kArmGraceMs; logged as "grace elapsed, applying moves"). During the
  grace the load finishes undisturbed, then the circle starts in a fully
  live game. numPauseGame gate stays as a second, observable layer - if
  opening the Pip-Boy next test produces no "blocked:" line, the offset
  is wrong and the gate gets fixed or dropped.
- Stage 3B exit criteria for the next test: load completes normally,
  "grace elapsed" ~10 s after PostLoadGame, then moveRefr lines at a
  steady ~10/s cadence (task pump healthy) and visible circling in-game.
## Stage 3B fourth test + Stage 4 pivot - 2026-10-21 (owner playtest, v9)
- GRACE WORKS: load completed, owner saw the world (pos settles confirm),
  "grace elapsed, applying moves" at the right time.
- NEW FINDING: the FIRST player MoveRefrToPosition in a live game
  triggered a loading screen (move #1 was 535 units, same cell, same ws -
  console moveto never loads for that) and the same hang loop followed
  (move cadence collapsed to one per 5-20 s = task queue pumped rarely,
  the game stuck in the load path again). Conclusion: player moves take a
  special engine path (prime suspect: pTargetHandle semantics or the
  player-only position pipeline).
- DECISION: stop moving the player entirely. The co-op architecture never
  needs it - the local player is driven by local input; ONLY puppets move
  by network. Stage 3B's actual goal (engine-level network-driven
  movement proven: positions apply, physics settles) is ACHIEVED.
- v10 = Stage 4: one-time puppet spawn via PlaceAtMe_Native (vm from
  g_gameVM base+0x058D3308, +0xB0 m_virtualMachine; stackId 0; target =
  player; form = player->baseForm +0xE0 = a copy of the player's own NPC,
  no FormID guesswork), then the same task pipeline drives the PUPPET in
  the circle. PreLoadGame/NewGame clear the puppet (stale after world
  change). Distance guard now measures from the puppet.
- This test splits the hypothesis space cleanly: if the puppet circles
  without loading screens, the player-path issue is noted and skipped
  (revisit with disassembly when player teleport sync ever matters); if
  the puppet ALSO triggers load screens, our MoveRefrToPosition call
  semantics (nullHandle?) are wrong and the next step is disassembling
  MoveRefrToPosition and its Papyrus native call site in Fallout4.exe.