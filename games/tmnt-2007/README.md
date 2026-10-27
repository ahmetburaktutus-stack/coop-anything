# TMNT (2007, PC)

> **Work in progress, not stable.** This is an experimental prototype under active development. It is not
> released and not ready to play; expect missing moves, visual glitches, and untested levels.

| | |
|---|---|
| Engine | Ubisoft Jade ("Engine v428-42" in the game's log; gameplay scripts compiled into the exe) |
| Versions tested | Retail PC, `TMNTGame.exe` 1.0.0.188 (UPX-packed, SecuROM traces), SHA-256 `701d9f00f8c1edf790466f2ecfeb72a64fe7ccdc491f7c370ae11c1942740589`; store/edition unknown |
| Projects | Local couch co-op by DLL injection (private, in progress; no public repo yet) |
| QA level | automated self-test in Level 1 only (isolation + leak-mutation twin + scripted team moves); campaign untested |
| Last updated | 2026-10-27 |

## Where things stand

A second player can play a second turtle on screen at the same time as player 1, in one game process on one PC with
two controllers (XInput, DualSense over raw HID, or Nintendo Switch 2 Pro Controller over WinUSB). In levels where only
one turtle is loaded in memory (such as Level 1, *Mystical Jungle*), the mod duplicates player 1's turtle object and
detaches its shared skeleton so both turtles have visible bodies (both appear as Leonardo).

Input is separated by swapping the game's 544-byte Gear input buffer around the per-frame input pipeline and player 2's
AI update. Air leapfrog (`state 11`) and ground charged team attacks (`state 25` -> `state 14`) can be triggered with
player 2 as partner, even in Level 1 where team moves are normally locked.

Still an early prototype with rough edges: after a team move finishes, the partner plays its single-player exit leap
before snapping back beside player 1; cloned turtles in solo levels look identical; and testing beyond the opening area
of Level 1 is minimal. Not released.

## Existing work

- No co-op mod for the PC game was found (GitHub, Nucleus Co-op handler hub, web search; 2026-10-12). Only the PSP and
  DS versions had multiplayer.
- [xan1242/UbiTMNTHax](https://github.com/xan1242/UbiTMNTHax): ASI plugin for the same exe (windowed, skip launcher,
  multiple instances). Not co-op, but a reference for hooking this build.
- [hogsy/jaded](https://github.com/hogsy/jaded): community fork of the 2006 Jade engine source; useful for structures,
  does not load TMNT data.

## How the game works (for co-op)

Details and addresses (for the build above) are in [NOTES.md](NOTES.md).

- Normally only one turtle is on screen; a brother appears briefly for team moves (Brother Throw, Super Family
  Attack). Turtle Swap only changes which turtle object is "main actor 0"; the other turtle objects stay alive.
- The engine has two main-actor slots (the second is empty in solo play) and per-turtle "IO ports", but a port is only
  an on/off gate: every present turtle reads the same shared input object.
- Input goes through Ubisoft's Gear input library (DirectInput 8 and XInput 1.3). The game merges all pads into one
  input state, so a second physical pad simply also controls player 1.
- The launcher's mutex blocks a second instance, so multi-instance split screen is not an easy route.

## What we changed

All in an injected DLL (a host DLL injected once, plus a hot-reloadable mod DLL swapped while the game runs). No game
files are modified.

- **A second turtle:** in multi-turtle levels, the game's own `Turtle_Spawn` places an idle brother next to player 1
  with its port open. In solo-turtle levels (Level 1), the mod clones `main actor 0` via the engine's object duplicator
  (`0x00491C40`), unshares its skeleton (`0x006AC230` / `0x006ABF40`), clears `ForceInvisible` (`0x00200000`), and passes
  it through `Turtle_Spawn` using a temporary manager slot.
- **Its own input context:** the game's 544 bytes of raw/processed input state (`0x015021C0..0x015023DF`) are swapped
  around the input pipeline (`0x0063EC60`) and around player 2's AI tick, giving player 2 its own input history buffer.
  DualSense (HID) and Switch 2 Pro Controller (WinUSB) are polled directly and hidden from DirectInput enumeration so
  the game does not merge them into player 1.
- **Enemies target player 2:** in the enemy AI tick, main actor 0 is swapped to player 2 when that enemy is closer to
  it, and restored after.
- **Health bar** for player 2 (D3D9 EndScene hook on the game's real device), bound to the turtle's 0-100 health (`AI+0x174`).
- **Camera:** shifts player 1's translation toward the horizontal midpoint only while the camera object's AI runs;
  player 2 is leashed to player 1 and brought back if it falls into a pit.
- **Team moves with player 2:** hooks `GetCoopPartner` (`0x005759D0`), `CheckCoopLock` (`0x005C1DF0`), and
  `Turtle_LockCoop` (`0x005C17C0`) so air leapfrog (`state 11`) and ground charged team attacks (`state 25` -> `state 14`)
  use player 2 as partner (while blocking quick-tap tag switching). At the end of the move, `SetTurtleState`
  (`0x00585C50`) redirects `state 0` (absent) to `state 2` (playable) and teleports player 2 back beside player 1.

## QA

An isolated windowed test copy (`testlane.ps1` + `lanectl.py`) boots into Level 1 with Bink videos skipped, focus-loss
messages blocked, and D3D9 back-buffer capture to MP4/contact sheets:

- `selftest both`: drives player 1 only, then player 2 only, verifying isolation (`otherMoved == 0`), movement, jump,
  and attack states (`isolation A: PASS`, `isolation B: PASS`), followed by a `leak` mutation twin that feeds the same
  input to both players and verifies the isolation assertions fail (`isolation A: FAIL (LEAK)`).
- `coopmove`: records unfixed (`team_raw.mp4`) vs. fixed (`team_fixed.mp4`) air leapfrog (`state 11`) and ground
  charged team attack (`state 25` -> `state 14`) in Level 1.

Only the start of Level 1 is covered by these scripts; the rest of the campaign is untested.

## How to play

Not released.

## Known issues

- **Visual pop after team moves:** during the second half of a team move animation (especially air leapfrog `0x74`), the
  partner turtle still plays its single-player exit leap off-stage for ~0.5-1.0 s before the state ends and teleports it
  back beside player 1.
- **Identical clone in solo levels:** in levels where only one turtle is loaded (like Level 1), player 2 is a cloned
  Leonardo with the same model and bandana color.
- **Attack action ID difference on clone:** in the Level 1 self-test, the unarmed/armed attack buttons trigger slightly
  different combo action indices on the clone (`7`/`11` vs `8`/`6`), so combat parity is still incomplete.
- **Limited test coverage:** Mikey/Don/Raph team attacks (`states 15..17`), downed revive (`state 24`), boss fights,
  cutscenes, checkpoint reloads, and level-to-level transitions have not been thoroughly tested.
- **Hot-plug limitation:** controllers connected after the game starts when no joystick was present at launch are hidden
  to avoid a null-list crash in the game's pad manager (`0x0063DD40`); they require a game restart.

## Next steps

- Cut the single-player exit leap short during team moves so player 2 lands naturally instead of teleport-snapping back
  when the animation ends.
- Test multi-brother levels (states `15..17`, downed revive `state 24`, level transitions, and boss scripts).
- Investigate why the cloned turtle selects a different combo action index (`7`/`11` vs `8`) on attack input.
