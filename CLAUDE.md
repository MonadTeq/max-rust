# CLAUDE.md — M.A.X.: Rust

Rust port of M.A.X. (1996, Interplay), built on top of the C++ project
M.A.X. Port (klei1984/max). The project runs in parallel with an
ongoing decompilation effort: we analyze the original binary and the
C++ port to reproduce game logic faithfully, then reimplement it in
Rust.

## Core principle

This is NOT a mechanical translation of C++. We rethink structure to
fit Rust idioms. When C++ and Rust conflict, the Rust way wins — except
where game logic behavior must be preserved exactly.

## Repository layout

    max-rust/
      Cargo.toml              # workspace virtual manifest
      CLAUDE.md               # this file
      crates/
        max-core/             # pure game logic, no platform deps
        max-data/             # original resource loaders (.DTA, maps, sprites)
        max-ai/               # AI controllers (move, build, attack, explore)
        max-render/           # map and unit rendering
        max-ui/               # menus, panels, dialogs
        max-audio/            # sound and music
        max-input/            # input -> commands for max-core
        max-net/              # network transport and lobby
        max-bin/              # entry point binary
      docs/
        decompilation/        # notes on original binary analysis
        formats/              # reverse-engineered file format specs
        network/              # protocol notes, packet captures, ENET details
      reference/
        original-asm/         # disassembly listings (read-only)
        cpp-port/             # snapshot of M.A.X. Port sources (read-only)

## Dependency rules (never violate)

    max-core   -> std only (+ serde if needed)
    max-data   -> max-core
    max-ai     -> max-core
    max-render -> max-core
    max-ui     -> max-core
    max-audio  -> (nothing from this project)
    max-input  -> max-core (command types only)
    max-net    -> max-core (command/event types only)
    max-bin    -> everything

max-core NEVER imports winit, wgpu, egui, rodio, tokio, enet, or any
platform or network crate. If the urge appears, the logic belongs in
another crate.

## Build and test

    cargo build --workspace
    cargo test --workspace
    cargo test -p max-core        # core tests use synthetic data only
    cargo clippy --workspace -- -D warnings
    cargo fmt --all

max-core tests must run without the original game CD, without a
window, and without a network. If a test needs real game data or a
socket, it belongs in max-data, max-net, or max-bin — not in max-core.

## Network

M.A.X. Port implements network play, and max-rust ports that subsystem
rather than inventing a new one. Key facts:

- The C++ port uses the ENET library for reliable UDP transport.
- The host multicasts messages to clients; there is a network lobby
  with scenario and mission selection.
- Network behavior is part of the decompilation target, not a
  greenfield feature. Reproduce it where practical.

### max-net boundaries

- max-net depends on max-core for Command and Event types only.
  It never touches World directly.
- Commands flow over the network and are applied through the same API
  used by max-input and max-ai. There is no side channel into World.
- Simulation is deterministic and lockstep-oriented. If two Worlds
  diverge under the same command sequence, that is a max-core bug, not
  a max-net bug.
- Transport choice (ENET bindings, laminar, or a custom UDP layer) is
  an implementation detail inside max-net. max-core must not know which
  one is used.

### Network decompilation notes

- Document protocol details in docs/network/ — packet layout, handshake,
  lobby state machine, reconnect behavior.
- When reproducing a network behavior, add a doc comment with the
  original symbol or the C++ function name, same convention as
  game-logic functions.
- If we deliberately deviate from the C++ port (for example, replacing
  IPX-era semantics with modern equivalents), record the decision in
  docs/decompilation/decisions.md.

## Decompilation workflow

We maintain byte-level fidelity with the original where game logic is
concerned. The workflow is:

1. Analyze the original binary (or the C++ port, when it already
   reimplements the function faithfully).
2. Record findings in docs/decompilation/ — function purpose, inputs,
   outputs, side effects, edge cases, offsets.
3. Map the function to a Rust module and signature.
4. Implement, then verify against the original behavior using recorded
   test cases or replay data.

For network code, the same workflow applies, but findings go to
docs/network/ instead of docs/decompilation/.

### Rules for decompilation work

- reference/original-asm/ and reference/cpp-port/ are READ-ONLY. Never
  edit them. They are reference snapshots.
- Every non-trivial reimplementation must have a corresponding note in
  docs/decompilation/ (or docs/network/ for network code) explaining
  what was reproduced and what was intentionally changed.
- When behavior is ambiguous, reproduce the original, even if it looks
  like a bug. Mark it with a comment: `// ORIGINAL BEHAVIOR: <explain>`.
- Do not "improve" balance, AI heuristics, combat math, or network
  timing without an explicit decision recorded in
  docs/decompilation/decisions.md.
- The original's ~120-turn crash from stack overflow is NOT reproduced.
  Rust's memory model makes it impossible by construction. This is a
  feature; note it where relevant.

### Naming convention for decompiled functions

Rust modules that reimplement a specific original function should carry
a doc comment with the original symbol:

    /// Reimplements: UNIT_MoveUnit (original offset 0x000A1234)
    /// Source: docs/decompilation/unit/move_unit.md
    pub fn move_unit(...) -> Result<(), MoveError> { ... }

For network functions, reference the C++ port symbol or the packet
name:

    /// Reimplements: NET_SendLobbyUpdate (M.A.X. Port)
    /// Source: docs/network/lobby.md
    pub fn send_lobby_update(...) -> Result<(), NetError> { ... }

This makes it trivial to trace any Rust function back to its
decompilation note and to the original binary offset.

## Code conventions

- Entity IDs are newtype wrappers:
    struct UnitId(u32); struct PlayerId(u8);
    struct TilePos { x: i16, y: i16 }
- Turn logic is deterministic. No rand() without an explicit seed.
  The RNG is injected into World or into the system that needs it.
- Errors: thiserror in library crates, anyhow only in max-bin.
- No panics in max-core on game data. Only Result.
- Public crate API is minimal. Everything else is pub(crate).
- unsafe requires a comment explaining why it is necessary and why the
  invariants hold.

## What we replace from C++

| C++ (M.A.X. Port)                  | Rust (max-rust)                          |
|------------------------------------|------------------------------------------|
| SmartPtr / SmartList / SmartArray  | Vec<T>, HashMap<Id, T>, ownership via IDs |
| Global managers (unitsmgr, gamemgr)| World struct passed as &mut World        |
| GNW platform layer                 | winit + wgpu/softbuffer + egui           |
| smart file managers                | serde or hand-written byte parsers       |
| AI as global functions             | AiController trait                       |
| ENET via C++ wrappers              | max-net crate (transport is internal)    |

GNW is not ported. It is tied to 32-bit extensions and blocks clean
64-bit builds.

## Development order

1. max-core: World, Unit, map, turn, combat. Synthetic-data tests.
2. max-ai: one simple controller playing against itself.
3. max-data: loading maps and unitvalues from original files.
4. max-render + max-input + max-ui: minimal playable loop.
5. max-audio, remaining AI, saves.
6. max-net: transport, lobby, lockstep verification.

Network comes last because it requires a stable, deterministic
max-core. Building it earlier means building on sand.

## What NOT to do

- Do not add a max-core dependency on game, platform, or network
  crates "just quickly."
- Do not edit reference/original-asm/ or reference/cpp-port/.
- Do not copy C++ structures 1:1 if they do not fit Rust.
- Do not start by parsing original resources. Core on synthetics first.
- Do not bring in GNW.
- Do not silently change game behavior. Document it or reproduce it.
- Do not put network types (sockets, packets, peers) into max-core.
  Command and Event are the only network-facing types in the core.
- Do not commit large binary files (original game data, packet
  captures) to the repo.

## References

- C++ source: klei1984/max (M.A.X. Port)
- Network docs: https://klei1984.github.io/max/network/
- Data structure reference: maxr-dot-org/maxr
- Original game: M.A.X. (1996), Interplay
- Decompilation notes: docs/decompilation/
- Network notes: docs/network/
- Format specs: docs/formats/
