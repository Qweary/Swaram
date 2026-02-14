# src/ — AudioHax Source Modules

## Music Pipeline Modules

image_source.rs: Loads images from file, camera, or preselected paths. Returns opencv::Mat.
Key type: `ImageSource` enum (Preselected, UserPath, CameraIndex, AIGenerated).

image_analysis.rs: Three-level visual feature extraction using OpenCV.
Key types: `GlobalFeatures` (avg_hue/saturation/brightness, edge_density, texture, shape_complexity), `ScanBarFeatures` (per-instrument: hue, saturation, brightness, edge_density, texture, hue_hist), `LocalFeatures` (fine-grained: edge_sharpness, edge_orientation_bias, texture_complexity, contour_circularity).
Key functions: `analyze_global()`, `analyze_scan_bar()`, `scan_image()`, `analyze_local_basic()`.

chord_engine.rs: All music theory logic. Scales (Ionian, Dorian, Phrygian, Lydian, Mixolydian, Aeolian), chord progressions with modal interchange and secondary dominants, harmonic complexity tiers.
Key types: `Chord` { name: String, notes: Vec<u8> }, `ChordEngine` { mappings: MappingTable }.
Key functions: `pick_progression()`, `generate_chords()`.

mapping_loader.rs: Parses assets/mappings.json into a `MappingTable` struct. Provides `lookup_range_map()` for visual→musical parameter translation.

midi_output.rs: MIDI port connection via midir. Sends note_on, note_off, program_change.
Key type: `MidiOut`. Requires a virtual MIDI port at runtime.

main.rs: CLI parsing, pipeline orchestration. Contains `worker_decide_action()` (the primary note decision point, ~line 139) and `play_scanned_steps_concurrent()` (barrier-synchronized worker-coordinator pattern). InstrumentAction and ScheduledEvent are defined here.

lib.rs: Crate root. Exposes `pub mod modem` for bin targets.

## Modem Modules (Independent Subsystem)

modem.rs: MFSK encode/decode, frame construction (AHX1 format), packetization (repetition PKT1 and Reed-Solomon RS1), Goertzel detection, channel simulation. No shared state with the music pipeline.
Key types: `ModemParams`, `DurationEstimate`. Key functions: `build_frame()`, `extract_frame()`, `packetize_stream_rs()`, `depacketize_stream_rs()`, `render_symbols_to_samples()`, `goertzel_mag_squared()`.

bin/modem_encode.rs, bin/modem_decode.rs: CLI wrappers for modem encode/decode.
bin/channel_sim.rs: Channel impairment simulator. bin/make_packetized.rs: Packetization utility.

## Concurrency Pattern

The worker-coordinator uses `std::sync::Barrier(num_instruments + 1)`. Workers compute InstrumentActions in parallel, barrier-sync, coordinator schedules MIDI events chronologically via `thread::sleep`, second barrier releases workers for next step.

## Conventions

Library code (all files except main.rs and bin/*): Result types, no unwrap, `///` doc comments on public items, thiserror for 3+ error variants. main.rs and bins: anyhow acceptable. Musical decisions must be documented with theory reasoning in comments. Run `cargo fmt` and `cargo clippy -- -W clippy::all` before committing. Tests in `#[cfg(test)] mod tests` within each file.
