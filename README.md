<div align="center">

# max-rust

**A modular Rust port of M.A.X. (1996) — the turn-based strategy classic by Interplay**

[![Rust](https://img.shields.io/badge/rust-1.75%2B-orange?logo=rust)](https://www.rust-lang.org)
[![License](https://img.shields.io/badge/license-GPL--3.0-blue)](LICENSE)
[![Status](https://img.shields.io/badge/status-architectural%20phase-yellow)]()
[![Based on](https://img.shields.io/badge/based%20on-M.A.X.%20Port-lightgrey)](https://github.com/klei1984/max)

```text
    ███╗   ███╗   █████╗   ██╗  ██╗    ██████╗  ██╗   ██╗ ███████╗ ████████╗
    ████╗ ████║  ██╔══██╗  ╚██╗██╔╝    ██╔══██╗ ██║   ██║ ██╔════╝ ╚══██╔══╝
    ██╔████╔██║  ███████║   ╚███╔╝     ██████╔╝ ██║   ██║ ███████╗    ██║
    ██║╚██╔╝██║  ██╔══██║   ██╔██╗     ██╔══██╗ ██║   ██║ ╚════██║    ██║
    ██║ ╚═╝ ██║  ██║  ██║  ██╔╝ ██╗    ██║  ██║ ╚██████╔╝ ███████║    ██║
    ╚═╝     ╚═╝  ╚═╝  ╚═╝  ╚═╝  ╚═╝    ╚═╝  ╚═╝  ╚═════╝  ╚══════╝    ╚═╝

              M E C H A N I Z E D   ·   A S S A U L T   ·   E X P L O R A T I O N
                                   rebuilt in Rust  ·  MMXXVI
```

*Native. Modular. Deterministic. Written in Rust.*

</div>

---

## About

**max-rust** is a from-scratch Rust reimplementation of *M.A.X.: Mechanized Assault & Exploration* (Interplay, 1996), a turn-based strategy game where you command a faction on an alien planet — building bases, managing resources, researching technology, and fighting for control.

This project is a port of the C++ project [**M.A.X. Port**](https://github.com/klei1984/max), which reimplemented the original game logic to run natively on modern systems. `max-rust` takes that foundation and rebuilds it around Rust's ownership model, a Cargo workspace, and strict module boundaries.

The goal is a game that:

- Runs natively on Linux, macOS, and Windows
- Has no 32-bit legacy constraints
- Separates game logic from rendering, input, and platform code
- Is testable without a window or the original game CD
- Does not reproduce the original's ~120-turn crash bug

## Screenshots

> Not yet playable. These placeholders will be replaced as subsystems
> land. The layout mirrors the planned in-game view.

<div align="center">

```text
┌───────────────────────────────────────────────────────────────────────────┐
│  TURN 01   │  METAL 0   FUEL 0   GOLD 0   │  PLAYER: RED   │  [ MENU ]    │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│        ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ ┌───────┐ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ │  BASE │ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ │  RED  │ ~ ~ ~ ~ ~ ~ ┌───────┐ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ └───────┘ ~ ~ ~ ~ ~ ~ │  UNIT │ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ │  BLUE │ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ └───────┘ ~ ~ ~ ~ ~ ~ ~ ~          │
│        ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~ ~          │
│                                                                           │
├───────────────────────────────────────────────────────────────────────────┤
│  [ MOVE ]  [ ATTACK ]  [ BUILD ]  [ RESEARCH ]  [ END TURN ]              │
└───────────────────────────────────────────────────────────────────────────┘
```

*Concept layout — not a rendered frame.*

</div>

## Philosophy

> **This is not a mechanical translation of C++.**

We rethink structure to fit Rust idioms. Where C++ uses custom smart pointers, global managers, and a monolithic platform layer, we use `Vec<T>`, explicit ownership via IDs, and a Cargo workspace with hard dependency boundaries.

If a C++ solution conflicts with Rust's ownership model or modularity, **we choose the Rust way**.

## Architecture

The project is a Cargo workspace. Each module is a separate crate with a minimal public API. Editing one subsystem does not rebuild the others.

```text
crates/
  max-core/     Pure game logic: units, map, turns, combat, resources.
                No platform, window, file, or audio dependencies.
  max-data/     Loading original resources (.DTA, maps, sprites).
                Populates max-core structs.
  max-ai/       Computer controllers: move, build, attack, explore.
                Connects to max-core via traits.
  max-render/   Drawing the map, units, and effects.
  max-ui/       Game interface: menus, panels, dialogs.
  max-audio/    Sound and music.
  max-input/    Mouse, keyboard, and touch -> commands for the core.
  max-bin/      Entry point. Wires everything together.
```

### Dependency rules

| Crate        | Depends on                          |
|--------------|-------------------------------------|
| `max-core`   | only `std` (+ `serde` if needed)    |
| `max-data`   | `max-core`                          |
| `max-ai`     | `max-core`                          |
| `max-render` | `max-core`                          |
| `max-ui`     | `max-core`                          |
| `max-audio`  | —                                   |
| `max-input`  | `max-core` (command types only)     |
| `max-bin`    | everything                          |

`max-core` never imports `winit`, `wgpu`, `egui`, `rodio`, or anything platform-specific. If the urge appears, the logic is in the wrong crate.

## What we replace from C++

| C++ (M.A.X. Port)                  | Rust (max-rust)                          |
|------------------------------------|------------------------------------------|
| `SmartPtr` / `SmartList` / `SmartArray` | `Vec<T>`, `HashMap<Id, T>`, ownership via IDs |
| Global managers (`unitsmgr`, `gamemgr`) | `World` struct passed as `&mut World`   |
| GNW platform layer                 | `winit` + `wgpu`/`softbuffer` + `egui`  |
| `smart` file managers              | `serde` or hand-written byte parsers     |
| AI as global functions             | `AiController` trait                     |

**GNW is not ported.** It is tied to 32-bit extensions and blocks clean 64-bit builds.

## Determinism

Turn logic is deterministic. There is no `rand()` without an explicit seed. The RNG is injected into `World` or into the system that needs it. This makes replays, tests, and AI self-play reproducible.

## Testing

- **`max-core`** is tested on **synthetic data** — no original resources, no window, no CD.
- Unit tests live in each crate. Integration tests live in `max-bin`.
- The data layer (`max-data`) is what feeds real game content into the core, but it is not required to validate logic.

## Compatibility with the original

- Game logic reproduces **M.A.X. Port** behavior, not a rebalanced variant.
- Save compatibility, if implemented, lives in `max-data` as a separate module — never in the core.
- The original's known defect (crash around turn 120 from a stack overflow) **does not reproduce** in Rust. That is a feature, not a bug.

## Roadmap

- [ ] **`max-core`** — `World`, `Unit`, map, turn, combat. Synthetic-data tests.
- [ ] **`max-ai`** — one simple controller playing against itself.
- [ ] **`max-data`** — loading maps and `unitvalues` from original files.
- [ ] **`max-render` + `max-input` + `max-ui`** — a minimal playable loop.
- [ ] **`max-audio`**, additional AI, saves, networking — later.

## Building

> Early development. The workspace is not yet in a runnable state.

```bash
git clone git@github.com:MonadTeq/max-rust.git
cd max-rust
cargo build --release
cargo test --workspace
```

You will need the original game data to run the full game. `max-core` tests do not require it.

## Requirements

- Rust 1.75 or newer
- Original M.A.X. (1996) game data for full play (not required for core tests)

## References

- **C++ source:** [klei1984/max](https://github.com/klei1984/max) — M.A.X. Port
- **Data structure reference:** [maxr-dot-org/maxr](https://github.com/maxr-dot-org/maxr)
- **Original game:** *M.A.X.: Mechanized Assault & Exploration* (1996), Interplay

## License

GPL-3.0 — matching the upstream M.A.X. Port project. See [LICENSE](LICENSE).

## Contributing

Contributions are welcome once the core is runnable. Until then, the project is in architectural phase. If you want to help, open an issue to discuss the subsystem you are interested in.

---

<div align="center">

*"Mechanized Assault & Exploration" — rebuilt in Rust, one crate at a time.*

</div>