# CLAUDE.md — AudioHax

## WHY: Project Purpose

AudioHax is a Rust creative technology project that converts images into expressive,
musically coherent output. The owner is a trained musician and security professional. Music quality is the primary objective — generated output should sound composed,
not algorithmic. Use proper music theory terminology; do not oversimplify.

Three capability tracks share this repo:
1. IMAGE-TO-MUSIC PIPELINE — scans images, extracts visual features via OpenCV, maps them
   to musical parameters, applies music theory logic, outputs multi-channel MIDI to FluidSynth.
2. MFSK DATA MODEM — encodes arbitrary data into audio using multi-channel frequency shift
   keying with Reed-Solomon FEC, AES-GCM encryption, and gzip compression (src/modem.rs,
   src/bin/modem_encode.rs, src/bin/modem_decode.rs).
3. INTERACTIVE/GAME VISION (future) — real-time image manipulation, game integration,
   interactive art installation deployment.

## WHAT: Repository Layout

```
src/
  main.rs            — CLI, worker-coordinator orchestration, note decision logic
  image_source.rs    — image loading (file, camera, preselected)
  image_analysis.rs  — three-level feature extraction (global, scan-bar, local)
  chord_engine.rs    — music theory: modes, progressions, voicing, harmonic complexity
  mapping_loader.rs  — JSON config parsing, visual→musical parameter lookup
  midi_output.rs     — MIDI port connection, note/program/CC messages
  modem.rs           — MFSK encode/decode, RS-FEC, AES-GCM, frame construction
  lib.rs             — crate root (exposes modem module for bin targets)
  bin/
    modem_encode.rs  — CLI encoder
    modem_decode.rs  — CLI decoder
    make_packetized.rs — packetization utility
assets/
  mappings.json      — visual-to-musical parameter rules
  images/            — input images
docs/                — detailed conventions, music theory context, roadmap
```

## HOW: Build, Test, Lint

```bash
cargo build --release
cargo test
cargo fmt
cargo clippy -- -W clippy::all
```

Run the music pipeline (requires virtual MIDI port + FluidSynth running):
```bash
cargo run --release -- play
cargo run --release -- path/to/image.jpg play --instruments 6 --steps 60
```

Run the modem:
```bash
cargo run --release --bin modem_encode -- output.wav input.jpg --compress --rs-data 4 --rs-parity 5 --rs-shard-size 128
cargo run --release --bin modem_decode -- input.wav output --rs-data 4 --rs-parity 5
```

CLI defaults: --instruments 4, --steps 40, --ms-per-step 250, --thickness 0.10, --jitter-percent 15, root MIDI 60 (C4).

## HOW: Platform Setup

Windows: loopMIDI port named "AudioHaxOut", FluidSynth with `-a dsound -m winmidi`.
macOS: IAC Driver enabled in Audio MIDI Setup, FluidSynth with `-a coreaudio -o midi.driver=coremidi`.
Linux: `sudo modprobe snd_virmidi`, FluidSynth with `-a alsa -o midi.driver=alsa_seq`.
Linux also needs: `sudo apt install libopencv-dev llvm-dev libclang-dev clang libasound2-dev`.

## HOW: Module Ownership & Boundaries

Each module has a single responsibility. Do not bleed logic across boundaries:

image_source.rs / image_analysis.rs — image I/O and feature computation only. No music logic.
chord_engine.rs — all music theory: scales, modes, progressions, voicing, voice leading, counterpoint. This is where musicality lives.
mapping_loader.rs + assets/mappings.json — configuration-driven visual→musical translation. No hardcoded mappings elsewhere.
midi_output.rs — MIDI transport only. No note selection logic.
modem.rs + src/bin/* — fully independent subsystem. Shares no state with the music pipeline. Uses lib.rs for crate-level access.
main.rs — orchestration and glue. Contains worker_decide_action() (the primary note decision point, lines ~139-177) and the barrier-synchronized worker-coordinator pattern. Changes here touch all subsystems; route through quality gate.

## HOW: Coding Conventions

Use Result types with explicit error handling in library code; unwrap/anyhow acceptable in main.rs and bins. Write `///` doc comments on all public items. Use thiserror when a module has 3+ error variants. Every new function needs tests — musical tests should validate note ranges, chord membership, and voice leading constraints, not just "doesn't panic." Run `cargo fmt` and `cargo clippy` before committing. Feature branches only; never commit to master directly.

## HOW: Music Quality Standards

Voice leading: minimize intervallic leaps, resolve tendency tones, avoid parallel fifths/octaves.
Phrase structure: group steps into 4-8 step phrases with antecedent-consequent shaping.
Dynamics: velocity must contour over time — crescendo to climax, taper at phrase end, accent strong beats.
Rhythm: musical timing variation (rubato, phrase-end ritardando), not random jitter.
Articulation: staccato, legato, portato, accents — not just on/off.
Document the music-theory reasoning in code comments when making musical decisions.

## Progressive Disclosure

For detailed information, read these docs/ files:
- docs/ROADMAP.md — phased development plan with task breakdowns
- docs/architecture.md — data flow, concurrency model, analysis pipeline (see also project knowledge: audiohax-architecture.md)
- docs/music-theory.md — scales, voice leading rules, phrase structure algorithms (see also project knowledge: music-theory-context.md)
- docs/conventions.md — full Rust style, git workflow, testing policy (see also project knowledge: development-conventions.md)
- docs/agent-teams.md — spawn prompts, file ownership, coordination patterns (see also project knowledge: agent-teams-reference.md)

## Agent Team Quick Reference

3-5 agents max per swarm. Natural ownership splits: image agent (image_source.rs, image_analysis.rs), music agent (chord_engine.rs, mapping_loader.rs, mappings.json), output agent (midi_output.rs, main.rs playback), modem agent (modem.rs, bin/*), quality gate (reads all, writes tests/reports only). Spawn prompts must be fully self-contained — teammates do not inherit lead context. Always run the quality gate last: `cargo fmt && cargo clippy -- -W clippy::all && cargo test`.
