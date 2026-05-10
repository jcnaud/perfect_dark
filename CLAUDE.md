# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build

Requires GCC/G++ (10+). **Visual Studio is not supported.** SDL2, zlib, and OpenGL must be available.

```sh
# Configure (once)
cmake -G"Unix Makefiles" -Bbuild .

# Build
cmake --build build -j4

# Build for alternate ROM regions
cmake -G"Unix Makefiles" -Bbuild . -DROMID=pal-final
cmake -G"Unix Makefiles" -Bbuild . -DROMID=jpn-final
```

Default build type is `RelWithDebInfo` (`-Og`). The binary is `build/pd.x86_64` (Linux/macOS) or `build/pd.exe` (Windows).

## Running

Place `pd.ntsc-final.z64` in a `data/` directory next to the binary, then run the executable. Runtime settings are in `pd.ini` (created on first run).

For the `port-net` branch (netplay), forward port 27100. Network settings are in `pd.ini` under `[Net]`, `[Net.Client]`, `[Net.Server]`.

## Architecture

This is an N64 game decompilation with a PC port layer on top.

```
src/          — Decompiled N64 game code (do not restructure)
port/src/     — PC port layer (platform abstraction)
port/fast3d/  — OpenGL rendering (libultraship Fast3D renderer)
port/external/— Embedded third-party libs (ENet, minimp3)
include/      — N64 API compatibility headers (ultra64.h etc.)
```

### Game code (`src/`)

- `src/game/` — ~230 C files: gameplay, AI, props, weapons, menus, multiplayer
- `src/lib/` — N64 library reimplementations: `model.c` (rendering), `collision.c`, `anim.c`, matrix/audio
- `src/include/` — Game headers; `types.h` defines all major structs; `constants.h` defines all enums/flags; `data.h` / `bss.h` declare all globals

Key subsystems:
- **Props** (`src/game/prop.c`): everything in the world is a `struct prop`. Identified by `syncid` (u32) on the network.
- **Chrs** (`src/game/chr.c`, `chraction.c`): AI characters. Iterated via `g_Chrnums[]` / `g_ChrSlots[]` / `g_ChrIndexes[]`. Each chr has a `prop`, and optionally an `aibot` (for multiplayer bots) or `model`.
- **Players** (`src/game/player.c`, `playermgr.c`): `g_Vars.players[i]`, active player is `g_Vars.currentplayer` (set by `setCurrentPlayerNum()`). Player props are `PROPTYPE_PLAYER`.
- **AI** (`src/game/chrai.c`, `chraction.c`): `chraiExecute()` is the per-frame AI entry; `chraTick()` calls it from `chrTick()` → `botTick()` chain.
- **Stage init** (`src/game/lv.c` `lvReset()`): calls `setupCreateProps()` first (world props), then loops over players calling `playerReset()` / `playerSpawn()`, then `netSyncIdsAllocate()`.

### Port layer (`port/src/`)

- `input.c` — SDL2 keyboard/mouse/gamepad; `mouseLocked` flag; `inputMouseGetScaledDelta()` requires `mouseLocked == true` to return deltas
- `pdsched.c` — Frame scheduler: `schedStartFrame()` → network receive → game tick; `schedEndFrame()` → game render → `inputUpdate()` → `netEndFrame()` (send)
- `config.c` — Reads/writes `pd.ini`; `CONTROLMODE_PC = 8` enables KB+mouse for a player slot

### Network layer (`port/src/net/`) — `port-net` branch

Architecture: server-authoritative UDP via ENet. The server runs full game simulation; clients send inputs and receive state updates.

**Key globals:**
- `g_NetMode`: `NETMODE_NONE=0`, `NETMODE_SERVER=1`, `NETMODE_CLIENT=2`
- `g_NetLocalClient`: the local player's `struct netclient`
- `g_NetTick`: monotonically increasing frame counter
- `g_NetNextSyncId`: next available prop syncid for runtime-spawned props

**Message flow:**
- `netStartFrame()` → receives packets → `netClientEvReceive()` / `netServerEvReceive()` → dispatch by message ID
- `netEndFrame()` → records player moves → writes `SVC_PLAYER_MOVE` / `SVC_CHR_POSITIONS` / other updates → `netFlushSendBuffers()` → ENet flush
- Unreliable channel (`g_NetMsg`): position updates, inputs. Reliable channel (`g_NetMsgRel`): game events, stage start/end.

**Message types** (`port/include/net/netmsg.h`):
- `SVC_STAGE_START (0x10)`: starts the game; includes prop syncid table (server → client) so both sides have identical `prop->syncid` mappings
- `SVC_PLAYER_MOVE (0x20)`: player position/input state
- `SVC_CHR_POSITIONS (0x44)`: bot/AI position+angle+damage, sent every frame by server
- `SVC_CHR_DAMAGE (0x42)` / `SVC_CHR_DISARM (0x43)`: chr damage events
- `SVC_PROP_MOVE (0x30)` / `SVC_PROP_SPAWN (0x31)` / related: prop state updates

**Prop syncid system:**
Syncids are assigned in `netSyncIdsAllocate()` (called at end of `lvReset`) using array position: `prop->syncid = prop - g_Vars.props + 1`. After client calls `mpStartMatch()`, it receives the server's full syncid table from `SVC_STAGE_START` and overwrites its local assignments to ensure both sides match.

**Co-op specifics:**
- `g_Vars.coopplayernum`: the co-op partner's player slot (≥0 in co-op, -1 in combat sim)
- Bot AI is disabled on clients: `botTickUnpaused`, `botApplyMovement`, `chraiExecute` are all gated with `g_NetMode != NETMODE_CLIENT`
- Bot positions are server-authoritative via `SVC_CHR_POSITIONS`
- `netbufReadPropPtr()` finds props by syncid via linear scan of `g_Vars.props[]`

**Identifying props in messages:** Use `netbufWritePropPtr(buf, prop)` / `netbufReadPropPtr(buf)` which serialize/deserialize by syncid. For chrs, use `chrnum` (via `chrFindByLiteralId()`) instead of prop syncid.

## Key types and conventions

- `f32` = float, `s32` = int32, `u32` = uint32, `s16` = int16 (N64 typedef convention)
- `struct coord { f32 x, y, z; }` — 3D position
- `TICKS(n)` — converts seconds to game ticks (60 Hz base)
- `g_Vars.lvupdate60` / `lvupdate240` — per-frame delta time scalars
- `g_Vars.currentplayer` / `g_Vars.currentplayernum` — active player context; many functions use this implicitly; always save/restore with `setCurrentPlayerNum()`
- `PLAYERCOUNT()` — returns `g_Vars.numplayers`
- `PROPTYPE_CHR=3`, `PROPTYPE_PLAYER=6` — prop type constants in `constants.h`
- `CONTROLMODE_PC=8`, `CONTROLMODE_NA=9` (remote/dummy player)
- Platform-specific net code is guarded with `#ifndef PLATFORM_N64`
