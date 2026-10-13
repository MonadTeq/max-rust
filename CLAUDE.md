# CLAUDE.md — max-rust

A Rust port of M.A.X. (1996, Interplay), based on the C++ project
M.A.X. Port (klei1984/max). Goal: a natively running turn-based
strategy game on modern platforms, with the original's defects fixed
and a modular architecture.

## Core principle

This is NOT a mechanical translation of C++ code. We rethink the
structure to fit Rust idioms. If a C++ solution conflicts with Rust's
ownership model or modularity, we choose the Rust way, not the C++
copy.

## Workspace layout

Cargo workspace, each module is a separate crate. Split compilation
is mandatory: editing AI must not rebuild the renderer.

    crates/
      max-core/     # pure world logic: units, map, turns, combat,
                    # resources. NO platform, window, file, or audio
                    # dependencies.
      max-data/     # loading original resources (.DTA, maps, sprites).
                    # Populates max-core structs.
      max-ai/       # computer controllers (move, build, attack,
                    # explore, player). Connects to max-core via
                    # traits, not the other way around.
      max-render/   # drawing the map, units, effects.
      max-ui/       # game interface: menus, panels, dialogs.
      max-audio/    # sound and music.
      max-input/    # input (mouse, keyboard, touch) -> commands for
                    # the core.
      max-bin/      # entry point, wires everything together.

## Dependency boundaries (must not be violated)

    max-core   depends only on std (+ serde if needed)
    max-data   -> max-core
    max-ai     -> max-core
    max-render -> max-core
    max-ui     -> max-core
    max-audio  -> (nothing from the project)
    max-input  -> max-core (command types only)
    max-bin    -> everything

max-core NEVER imports winit, wgpu, egui, rodio, or anything else
platform-specific. If the urge appears, the logic is in the wrong
crate.

## What we replace from C++

- Custom SmartPtr/SmartList/SmartArray -> Vec<T>, HashMap<Id, T>,
  ownership via IDs. We do not carry over shared_ptr semantics.
- Global managers (unitsmgr, gamemgr) -> a World struct in max-core,
  passed as &mut World into systems.
- GNW (platform layer) -> winit + wgpu/softbuffer + egui. GNW is not
  ported.
- smart file managers -> serde or hand-written byte parsers in max-data.
- AI as a set of global functions -> an AiController trait.

## Code conventions

- Entity identifiers: newtype wrappers.
    struct UnitId(u32); struct PlayerId(u8); struct TilePos { x: i16, y: i16 }
- Turn logic is deterministic. No rand() without an explicit seed.
  RNG is injected into World or into the system.
- Errors: thiserror in library crates, anyhow only in max-bin.
- No panics in max-core on game data. Only Result.
- Public crate API is minimal. Everything else is pub(crate).
- Tests: unit tests in each crate, integration tests in max-bin.
  max-core is tested on synthetic data without original resources
  and without a window.

## Original data

Loading .DTA and sprites lives only in max-data. The core works with
ready-made structs. This lets us test logic without the game CD.
Format loaders are documented in max-data/src/formats/.

## Compatibility with the original

- Game logic must reproduce M.A.X. Port behavior, not "improve
  balance" without an explicit decision.
- Saves: if we implement compatibility with original save files, that
  is a separate module in max-data, not in the core.
- The original's known defect (~120-turn crash from stack overflow)
  does not reproduce in Rust. That is a feature, not a bug.

## What NOT to do

- Do not add a max-core dependency on game crates "just quickly."
- Do not use unsafe without a justifying comment.
- Do not copy C++ structures 1:1 if they do not fit Rust.
- Do not start by parsing original resources. Core on synthetics
  first, data later.
- Do not bring in GNW.

## Development order

1. max-core: World, Unit, map, turn, combat. Tests on synthetics.
2. max-ai: one simple controller playing against itself.
3. max-data: loading maps and unitvalues from original files.
4. max-render + max-input + max-ui: a minimal playable loop.
5. max-audio, remaining AI, saves, networking — later.

## References

- C++ source: klei1984/max (M.A.X. Port)
- Data for comparison: maxr-dot-org/maxr (resource structure)
- Original game: M.A.X. (1996), Interplay