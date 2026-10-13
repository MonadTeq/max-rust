<div align="center">

# M.A.X. : RUST

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

*Rust in Pieces.*

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

## FAQ

<details>
<summary><strong>Why not just run M.A.X. Port?</strong></summary>

M.A.X. Port is a C++ reimplementation that already runs on modern systems. If your goal is simply to play the game, use it — it works, and it is actively maintained.

`max-rust` exists for a different reason: to rebuild the game around Rust's ownership model and a modular workspace. The C++ port still carries legacy constraints — custom smart pointers, a monolithic GNW platform layer, and 32-bit assumptions that block clean 64-bit builds. Rust lets us remove those constraints at the architectural level rather than patching around them.

</details>

<details>
<summary><strong>Do I need the original M.A.X. CD to play?</strong></summary>

Yes, for the full game. `max-rust` does not ship game assets — maps, sprites, sounds, and unit data come from the original 1996 release, just as M.A.X. Port requires.

The `max-core` crate, however, is designed to be tested on synthetic data. You can run `cargo test --workspace` and validate game logic without the CD. Only the data layer (`max-data`) and the playable binary need original resources.

</details>

<details>
<summary><strong>Why GPL-3.0?</strong></summary>

To match the upstream M.A.X. Port project. Since `max-rust` is a port of that codebase and reads its data formats, staying on the same license keeps the lineage clean and avoids license friction for anyone comparing or contributing to both.

</details>

<details>
<summary><strong>Will this fix the original's bugs?</strong></summary>

Some of them, structurally. The most famous defect in the original game — a crash around turn 120 caused by a stack overflow — does not reproduce in Rust, because the bug was tied to the DOS-era memory model and the Watcom toolchain rather than to game logic.

Other fixes depend on the subsystem. Gameplay behavior is intentionally reproduced, not rebalanced, so "fixing" a balance issue would require an explicit decision rather than a silent change.

</details>

<details>
<summary><strong>Is there multiplayer?</strong></summary>

Not yet, and not in the near term. The roadmap is single-player first: get the core, AI, data loading, and a minimal playable loop working before adding networking. If multiplayer lands, it will be a separate crate (`max-net`) that depends on `max-core` but does not touch rendering or input.

</details>

<details>
<summary><strong>Why not use `shared_ptr`-style semantics for units?</strong></summary>

Because Rust gives us something better: explicit ownership via IDs. In the C++ port, units are managed through custom smart pointers and global managers, which makes ownership implicit and cyclic references possible. In `max-rust`, `World` owns all units, and systems refer to them by `UnitId`. This removes the need for reference counting, eliminates cycles by construction, and makes serialization and testing straightforward.

</details>

<details>
<summary><strong>Can I contribute before the core is runnable?</strong></summary>

Yes, but the project is in architectural phase. The most useful contributions right now are design discussions, not code: review the crate boundaries, propose the shape of the `AiController` trait, or map out original data formats in `max-data/src/formats/`.

Open an issue describing the subsystem you are interested in before submitting a PR. Once `max-core` is stable and the first playable loop exists, the contribution surface will widen considerably.

</details>

<details>
<summary><strong>Why the name "max-rust"?</strong></summary>

It follows the pattern established by the C++ port: keep the original game's name, append the language. `M.A.X.` stays recognizable, and `-rust` makes it clear which implementation you are looking at. The tagline — *Rust in Pieces* — is just the obvious joke.

</details>

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

*Rust in Pieces — one crate at a time.*

</div>