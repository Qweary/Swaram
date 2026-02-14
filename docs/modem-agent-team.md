# Modem Hardening — Agent Team Configuration

## Team Overview

Team name: `modem-hardening`
Agent count: 4
Coordination pattern: Hybrid. The Test Engineer and Protocol Documentarian fan out in parallel (no file overlap, no dependencies). The Resilience Engineer depends on the Test Engineer's round-trip test harness being in place first (it extends the tests with noise/degradation scenarios). The Quality Gate runs last.

```
┌───────────────────┐   ┌──────────────────────┐
│  Test Engineer     │   │ Protocol Documentarian│
│  modem.rs tests   │   │ docs/modem-protocol.md│
│  tests/modem_*.rs │   │ src/bin/modem_demo.rs │
└────────┬──────────┘   └──────────┬────────────┘
         │                         │
         ▼                         │
┌───────────────────┐              │
│ Resilience Eng    │              │
│ modem.rs errors   │              │
│ bin/modem_*.rs    │              │
│ bin/channel_sim.rs│              │
└────────┬──────────┘              │
         │                         │
         └─────────┬───────────────┘
                   ▼
           ┌──────────────┐
           │ Quality Gate │
           │ read-only    │
           │ + review doc │
           └──────────────┘
```

CRITICAL BOUNDARY: No agent in this team may modify any music pipeline file (main.rs, chord_engine.rs, mapping_loader.rs, midi_output.rs, image_source.rs, image_analysis.rs, assets/mappings.json). The modem subsystem is fully independent.


## Agent Definitions

### Agent 1: Test Engineer

Role: Build comprehensive round-trip tests for the modem subsystem, covering clean encode/decode, simulated noise, burst loss, encryption, compression, and edge cases.

Files OWNED (may create/modify):
- src/modem.rs (only the #[cfg(test)] mod tests block — add tests, do not change public API or non-test code)
- tests/modem_roundtrip.rs (new integration test file)

Files EXCLUDED (must NOT modify):
- src/main.rs, src/chord_engine.rs, src/mapping_loader.rs, src/midi_output.rs, src/image_source.rs, src/image_analysis.rs
- src/bin/* (the Resilience Engineer owns these)
- assets/*
- docs/*

Communication: Mark task `modem-tests` as complete. Message lead with a summary of test coverage (which round-trip paths are tested, which noise models, which edge cases).


### Agent 2: Protocol Documentarian

Role: Write a complete protocol specification for the AHX1 modem frame format and create an end-to-end demo CLI binary.

Files OWNED (may create/modify):
- docs/modem-protocol.md (new)
- src/bin/modem_demo.rs (new)

Files EXCLUDED (must NOT modify):
- src/modem.rs (read only — extract information from it but do not change it)
- src/bin/modem_encode.rs, src/bin/modem_decode.rs, src/bin/channel_sim.rs, src/bin/make_packetized.rs
- All music pipeline files
- assets/*

Communication: Mark task `modem-protocol-doc` as complete. Message lead confirming the demo binary compiles and listing any Cargo.toml changes needed (the new [[bin]] target).


### Agent 3: Resilience Engineer

Role: Improve error reporting, add structured diagnostics, implement graceful degradation when FEC can't fully recover, and enhance the simulate mode.

Files OWNED (may create/modify):
- src/modem.rs (public API additions: error types, diagnostic structs, enhanced decode functions)
- src/bin/modem_encode.rs
- src/bin/modem_decode.rs
- src/bin/channel_sim.rs

Files EXCLUDED (must NOT modify):
- tests/* (Test Engineer owns these)
- docs/* (Documentarian owns these)
- All music pipeline files
- assets/*

Communication: Mark task `modem-resilience` as complete. Message lead with new public types/functions added to modem.rs and any CLI flag changes.


### Agent 4: Quality Gate

Role: Validate compilation, tests, lint, module boundaries, and review the protocol documentation for accuracy against the actual code.

Files OWNED (may create/modify):
- docs/modem-hardening-review.md (review report)

Files EXCLUDED (must NOT modify):
- src/* (all source read-only)
- assets/*

Communication: Mark task `modem-quality-gate` as complete with PASS/FAIL verdict.


---

## Spawn Prompts

### Spawn: Test Engineer

```
You are the Test Engineer for the AudioHax MFSK data modem. Your job is to build comprehensive round-trip tests for the modem subsystem in src/modem.rs. You may add tests inside modem.rs's #[cfg(test)] block and create a new integration test file at tests/modem_roundtrip.rs. Do NOT modify any modem public API or non-test code. Do NOT modify any music pipeline files (main.rs, chord_engine.rs, mapping_loader.rs, midi_output.rs, image_source.rs, image_analysis.rs).

BUILD/TEST/LINT:
  cargo build --release
  cargo test
  cargo fmt
  cargo clippy -- -W clippy::all

MODEM SUBSYSTEM OVERVIEW:
The modem is in src/modem.rs, exposed via src/lib.rs as `pub mod modem`. Key public functions you'll use in tests:

Frame construction:
  build_frame(filename, data, compress, encrypt_key_hex) -> Result<Vec<u8>>
  parse_frame_header(frame) -> Result<(filename, compressed, encrypted, payload_start, payload_len, crc)>
  extract_frame(frame, decrypt_key_hex) -> Result<(filename, data)>

Packetization (repetition-based):
  packetize_stream(data, pkt_payload_size, repeats) -> Vec<u8>
  depacketize_stream(data, expected_repeats) -> Result<Vec<u8>>

Packetization (Reed-Solomon):
  packetize_stream_rs(data, shard_size, data_shards, parity_shards) -> Result<Vec<u8>>
  packetize_stream_rs_interleaved(data, data_shards, parity_shards, shard_size) -> Vec<u8>
  depacketize_stream_rs(data) -> Result<Vec<u8>>

Symbol encoding:
  bytes_to_symbols(bytes, m_tones) -> Vec<u8>
  symbols_to_bytes(symbols, m_tones) -> Vec<u8>
  bits_per_symbol(m_tones) -> usize

Audio rendering/detection:
  render_symbols_to_samples(channels_symbols, params) -> Vec<i16>
  build_tone_frequencies(params) -> Vec<Vec<f32>>
  goertzel_mag_squared(samples, target_freq, sample_rate) -> f32
  split_round_robin(symbols, channels) -> Vec<Vec<u8>>

Channel simulation:
  simulate_channel_bytes(data, random_flip_prob, burst_erase_prob, avg_burst_len) -> Vec<u8>

Modem parameters: ModemParams::default() gives 48kHz sample rate, 30ms symbols, 32 tones/channel, 4 channels, 400Hz base, 400Hz channel spacing, 30Hz tone spacing, 8 preamble repeats of a pilot tone.

YOUR DELIVERABLES:

1. Clean round-trip tests (in modem.rs #[cfg(test)]):
   - test_frame_roundtrip_plain: build_frame with no compression or encryption, then extract_frame, verify filename and data match exactly.
   - test_frame_roundtrip_compressed: build_frame with compress=true, extract, verify.
   - test_frame_roundtrip_encrypted: build_frame with a known 32-byte hex key, extract with same key, verify. Also test that extract with wrong key fails.
   - test_frame_roundtrip_compressed_encrypted: both flags on, verify.
   - test_symbol_encoding_roundtrip: bytes_to_symbols then symbols_to_bytes for various m_tones (8, 16, 32, 64, 256), verify byte-level identity.
   - test_bits_per_symbol: verify bits_per_symbol(32) == 5, bits_per_symbol(16) == 4, bits_per_symbol(256) == 8, bits_per_symbol(1) == 0.

2. Packetization round-trip tests (in modem.rs #[cfg(test)]):
   - test_packetize_repetition_roundtrip: packetize then depacketize with repeats=3, verify data identity.
   - test_packetize_rs_roundtrip: packetize_stream_rs then depacketize_stream_rs with data_shards=4, parity_shards=2, shard_size=128, verify.
   - test_packetize_rs_interleaved_roundtrip: same with interleaved variant.
   - test_packetize_small_payload: test with very small payloads (1 byte, 0 bytes, 127 bytes) to check edge cases.
   - test_packetize_large_payload: test with a payload larger than one RS block (e.g., 2048 bytes with shard_size=128, data_shards=4 → block_size=512, so 4 blocks).

3. Full audio round-trip test (in tests/modem_roundtrip.rs as integration test):
   - test_full_audio_roundtrip: This is the most important test. It exercises the COMPLETE pipeline in-memory without touching audio hardware or the filesystem:
     a. Create a known payload (e.g., b"AudioHax modem test payload 1234567890" repeated a few times to get ~200 bytes).
     b. build_frame(filename, payload, compress=true, encrypt=None).
     c. packetize_stream_rs(frame, shard_size=128, data_shards=4, parity_shards=2).
     d. bytes_to_symbols(packetized, params.m_tones).
     e. split_round_robin(symbols, params.channels).
     f. Prepend preamble symbols to each channel (same logic as modem_encode.rs).
     g. render_symbols_to_samples(channel_symbols, params) → get i16 samples.
     h. Now DECODE: iterate over samples in windows of samples_per_symbol, run goertzel_mag_squared for each channel and tone, pick max → detected symbols per channel.
     i. Preamble detection: find the preamble pattern in each channel's detected symbols and trim.
     j. Round-robin reinterleave the detected symbols.
     k. symbols_to_bytes → depacketize_stream_rs → extract_frame.
     l. Assert recovered filename and payload match the original EXACTLY.
   This test proves the encode→render→detect→decode pipeline works end-to-end in pure Rust with no external dependencies.

4. Noise tolerance tests (in tests/modem_roundtrip.rs):
   - test_audio_roundtrip_with_bitflips: Same as the full audio test, but after render_symbols_to_samples, inject random bit flips into the sample buffer (flip random bits in the i16 values with probability 0.001). Verify the round-trip still succeeds thanks to RS-FEC. This probability should be low enough that RS can correct it.
   - test_audio_roundtrip_with_burst_loss: After rendering, zero out a contiguous burst of 10-20 symbol windows (simulating a dropout). With interleaved RS and sufficient parity, recovery should still succeed. If RS parity=2 with data=4, you can lose up to 2 shards per block — size the burst accordingly so it's recoverable.
   - test_audio_roundtrip_noise_floor: After rendering, add Gaussian white noise to every sample (amplitude ±500 on i16 scale, roughly -30dB relative to signal). Verify Goertzel detection still picks the correct tones and the round-trip succeeds.
   - test_audio_roundtrip_unrecoverable: Inject enough damage that RS cannot recover (e.g., erase 50% of symbols). Verify that depacketize_stream_rs returns an Err rather than silently producing garbage. This tests graceful failure.

5. Edge case tests (in modem.rs #[cfg(test)]):
   - test_empty_payload: build_frame with empty data, verify it round-trips.
   - test_unicode_filename: build_frame with a Unicode filename ("テスト.jpg"), verify it round-trips.
   - test_max_payload: test with a large payload (64KB) to verify no overflow in length fields (payload_len is u32).
   - test_crc_mismatch_detection: manually corrupt one byte of the compressed payload after build_frame, then extract_frame — verify the CRC warning fires (currently it's an eprintln, so at minimum verify the data differs).

TESTING APPROACH:
Use #[cfg(test)] mod tests in modem.rs for unit tests that exercise individual functions. Use tests/modem_roundtrip.rs for integration tests that chain multiple functions together (especially the full audio pipeline). Import the crate as `use audiohax::modem;` in the integration test file.

For noise injection in the audio tests, operate directly on the Vec<i16> sample buffer — this is more realistic than simulate_channel_bytes because it simulates analog channel impairments rather than digital byte-level corruption.

For deterministic tests, seed any random number generators with a fixed seed so tests are reproducible. Use rand::SeedableRng with a ChaCha8Rng or similar. Non-deterministic tests are acceptable if they test statistical properties (e.g., "recovery succeeds with >95% probability at this noise level"), but prefer deterministic where possible.

CODING CONVENTIONS:
- Use Result types and proper assertions (assert_eq!, assert!(result.is_ok())).
- Write descriptive test names and add a comment at the top of each test explaining what it verifies.
- Run cargo fmt before finishing.
- Tests should not write to the filesystem (no temp files). Everything in memory.
- Tests should complete in under 10 seconds each (keep payload sizes reasonable).

When complete, mark task "modem-tests" as done and message the lead with: number of tests added, which round-trip paths are covered, and any issues discovered while writing the tests (e.g., API limitations, edge cases that revealed bugs).
```


### Spawn: Protocol Documentarian

```
You are the Protocol Documentarian for the AudioHax MFSK data modem. Your job is to write a complete protocol specification and create an end-to-end demo CLI. You own docs/modem-protocol.md and src/bin/modem_demo.rs exclusively. Do NOT modify src/modem.rs, any existing src/bin/ files, or any music pipeline files.

BUILD/TEST/LINT:
  cargo build --release
  cargo test
  cargo fmt
  cargo clippy -- -W clippy::all

IMPORTANT: Before writing any documentation, READ src/modem.rs carefully using the view tool. The protocol spec must be derived from the actual code, not from descriptions in this prompt. Any discrepancy between this prompt and the source code should be resolved in favor of the source code.

YOUR DELIVERABLES:

1. Protocol Specification (docs/modem-protocol.md):

Write a document thorough enough that an independent developer could implement a compatible decoder in another language (Python, C, etc.) using only this document and no access to the Rust source. The document should have these sections:

   a. Overview: What the modem does (encodes arbitrary files into audio using MFSK), what features it supports (compression, encryption, FEC), and the high-level pipeline (file → frame → packetize → symbolize → render audio, and the reverse).

   b. Modem Parameters: Document all ModemParams fields with their default values and valid ranges. Explain the frequency plan: how base_freq_hz, channel_spacing_hz, tone_spacing_hz, channels, and m_tones together define the frequency grid. Include the formula for computing the frequency of tone T on channel C:
      freq(C, T) = base_freq_hz + C * channel_spacing_hz + T * tone_spacing_hz
   Document the symbol timing: samples_per_symbol = sample_rate * symbol_ms / 1000.

   c. Frame Format (AHX1): Document the exact byte layout of the frame header. Read build_frame() and parse_frame_header() in modem.rs to get the precise layout. Include:
      - Magic bytes (4 bytes: "AHX1")
      - Flags byte (bit 0 = compressed, bit 1 = encrypted)
      - Filename length (2 bytes, big-endian u16)
      - Filename (variable, UTF-8)
      - Payload length (4 bytes, big-endian u32)
      - CRC32 (4 bytes, big-endian u32) — computed over the compressed payload BEFORE encryption
      - Payload (variable)
   Document the compression (gzip via flate2) and encryption (AES-256-GCM: 12-byte nonce prepended to ciphertext, key is 32 bytes provided as 64-char hex string) details precisely.

   d. Packetization — Repetition-Based (PKT1): Document the packet format for packetize_stream():
      - Magic (4 bytes: "PKT1")
      - Sequence number (4 bytes, big-endian u32)
      - Payload length (2 bytes, big-endian u16)
      - CRC32 (4 bytes, big-endian u32)
      - Payload (variable)
      Each packet is repeated N times contiguously. Decoder uses majority voting across repetitions.

   e. Packetization — Reed-Solomon (RS1): Document the RS packetization format. Read packetize_stream_rs() in modem.rs to get the exact shard packet header format. Include the RS parameters (data_shards, parity_shards, shard_size), the block structure, and the interleaving variant. Explain that the decoder must scan for shard headers, group by block_index, and reconstruct using reed-solomon-erasure (GF(2^8)).

   f. Symbol Encoding: Document bytes_to_symbols and symbols_to_bytes. Explain bits_per_symbol(m_tones) = floor(log2(m_tones)), MSB-first packing, and how trailing bits are padded.

   g. Audio Rendering: Document how symbols become audio samples. Explain the Hann window applied per symbol, the per-channel amplitude normalization, and the final i16 normalization (0.9 * i16::MAX / peak).

   h. Preamble: Document the preamble structure (params.preamble_symbols repeated params.preamble_repeats times, prepended to each channel independently). Explain that the decoder must detect the preamble pattern per channel and align/trim before reinterleaving.

   i. Detection: Document the Goertzel-based detector. For each symbol window, compute goertzel_mag_squared for every tone on every channel, pick the tone with maximum energy per channel. Document the round-robin interleave/deinterleave ordering.

   j. Reference Parameters Table: A single table with all default values suitable for copy-paste into an implementation.

Use precise byte offsets and field sizes throughout. Include hex examples where helpful (e.g., a sample frame header for a 10-byte uncompressed, unencrypted file named "test.bin"). Write clearly enough for a programmer with no Rust experience.

2. End-to-End Demo CLI (src/bin/modem_demo.rs):

Create a new binary that demonstrates the complete modem pipeline in a single command, without requiring audio hardware. The demo should:

   a. Accept CLI arguments:
      --input <file>: input file to encode (required)
      --output <file>: output file path for recovered data (default: "demo_recovered")
      --key <64-char-hex>: optional AES-256-GCM encryption key
      --compress: enable gzip compression
      --rs-data <N>: RS data shards (default: 4)
      --rs-parity <N>: RS parity shards (default: 2)
      --rs-shard-size <N>: RS shard size (default: 128)
      --noise <level>: inject noise (none, low, medium, high — maps to increasing bit-flip probability and burst loss)
      --verbose: print detailed diagnostics at each pipeline stage

   b. Execute the full pipeline in-memory:
      1. Read input file
      2. build_frame (with compression and encryption if requested)
      3. packetize_stream_rs (or interleaved)
      4. bytes_to_symbols
      5. split_round_robin + prepend preamble
      6. render_symbols_to_samples (→ i16 audio buffer)
      7. Optionally inject noise into the audio buffer based on --noise level
      8. Detect: iterate symbol windows, Goertzel per channel/tone, pick max
      9. Preamble detection and trimming per channel
      10. Round-robin reinterleave
      11. symbols_to_bytes → depacketize_stream_rs → extract_frame
      12. Write recovered file to --output path

   c. Print a summary showing: input size, frame size, packetized size, symbol count, audio duration, noise level applied, recovery success/failure, output size, byte-for-byte match (yes/no).

   d. If --verbose, print per-stage details: frame header fields, RS block count, symbols per channel, samples rendered, detection accuracy (compare detected symbols to original symbols if no noise), RS correction count if available.

   IMPORTANT: You need to add a [[bin]] entry to Cargo.toml for modem_demo. Add ONLY this entry — do not modify any existing entries or dependencies. The entry should be:
   ```toml
   [[bin]]
   name = "modem_demo"
   path = "src/bin/modem_demo.rs"
   ```

   If you are uncertain whether Cargo.toml already has [[bin]] entries, READ Cargo.toml first before modifying it. If there are no existing [[bin]] entries (the existing bins may be auto-discovered), add the section carefully without breaking auto-discovery of the other bins. The safest approach is to add explicit [[bin]] entries for ALL existing bins when you add the new one.

CODING CONVENTIONS:
- Use anyhow for error handling in the demo binary.
- Write /// doc comments on the main function and any helper functions.
- No unwrap() on user-facing operations — handle errors gracefully with context.
- The demo should work without OpenCV, MIDI, or FluidSynth (it only uses modem functionality).
- Run cargo fmt and cargo clippy before finishing.

When complete, mark task "modem-protocol-doc" as done and message the lead confirming: the doc is written, the demo binary compiles and runs, and any Cargo.toml changes you made.
```


### Spawn: Resilience Engineer

```
You are the Resilience Engineer for the AudioHax MFSK data modem. Your job is to improve error reporting, add structured diagnostics, implement graceful degradation, and enhance the simulation mode. You own src/modem.rs (public API and non-test code), src/bin/modem_encode.rs, src/bin/modem_decode.rs, and src/bin/channel_sim.rs. Do NOT modify test code in modem.rs (the Test Engineer owns #[cfg(test)]). Do NOT modify any music pipeline files or docs/.

BUILD/TEST/LINT:
  cargo build --release
  cargo test
  cargo fmt
  cargo clippy -- -W clippy::all

IMPORTANT: Before making changes, READ the current source files thoroughly using the view tool. Understand the existing error handling, the simulate_channel_bytes function, the depacketize_stream_rs flow, and the channel_sim.rs binary.

MODEM SUBSYSTEM OVERVIEW:
src/modem.rs contains the core modem: frame construction (build_frame, parse_frame_header, extract_frame), packetization (repetition-based PKT1 and Reed-Solomon RS1), symbol encoding, MFSK audio rendering, Goertzel detection, and a simple channel simulator. The bin utilities are CLI wrappers: modem_encode.rs builds frames and writes WAV, modem_decode.rs reads WAV and recovers files, channel_sim.rs simulates channel impairments on raw byte streams.

CURRENT ERROR HANDLING STATE:
- Most functions return Result<T, Box<dyn Error>> with string-based errors via std::io::Error::new().
- CRC mismatches in extract_frame produce only an eprintln warning — the function continues and may return corrupted data.
- RS depacketize failures are opaque: "RS decode failed" with no detail on which blocks failed or how many shards were missing.
- The decoder binary falls through multiple strategies (RS → repetition → raw) but doesn't report which strategy succeeded or what was lost.
- simulate_channel_bytes exists but operates at the byte level (pre-symbolization), not at the audio sample level.

YOUR DELIVERABLES:

1. Structured Error Types: Replace the ad-hoc Box<dyn Error> returns in modem.rs with a proper error enum using thiserror. Define a ModemError enum with variants including:
   - FrameError { reason: String } for frame construction/parsing failures
   - CrcMismatch { expected: u32, computed: u32 } for CRC check failures
   - DecryptionFailed { detail: String } for AES-GCM failures
   - DecompressionFailed { detail: String } for gzip failures
   - RsDecodeFailed { block_index: usize, shards_present: usize, shards_needed: usize } for RS block failures
   - PacketCorrupted { seq: u32, detail: String } for individual packet issues
   - InsufficientData { expected: usize, actual: usize } for truncated inputs

   BACKWARD COMPATIBILITY: The existing public functions that return Result<T, Box<dyn Error>> must continue to compile. The simplest approach is to implement From<ModemError> for Box<dyn Error> (which thiserror gives you automatically since ModemError will impl std::error::Error). Alternatively, change the return types to Result<T, ModemError> — but if you do this, verify that modem_encode.rs and modem_decode.rs still compile (they use ? with Box<dyn Error> in main). The safest path: keep Box<dyn Error> returns on existing functions but construct ModemError variants internally and convert via .into(). New functions can return Result<T, ModemError> directly.

2. Decode Diagnostics Struct: Add a DecodeReport struct that captures structured information about a decode attempt:

   ```rust
   pub struct DecodeReport {
       pub strategy_used: DecodeStrategy,       // which strategy succeeded
       pub rs_blocks_total: usize,              // total RS blocks in stream
       pub rs_blocks_recovered: usize,          // blocks where RS succeeded
       pub rs_blocks_failed: usize,             // blocks where RS couldn't recover
       pub rs_shards_corrected: usize,          // total erasure corrections across all blocks
       pub crc_match: bool,                     // frame-level CRC result
       pub frame_compressed: bool,
       pub frame_encrypted: bool,
       pub payload_size_bytes: usize,
       pub warnings: Vec<String>,               // non-fatal issues encountered
   }

   pub enum DecodeStrategy {
       ReedSolomon,
       ReedSolomonInterleaved,
       Repetition { repeats: usize },
       RawFrame,  // fell through to direct frame parse
   }
   ```

   Add a new function `depacketize_stream_rs_with_report(data: &[u8]) -> Result<(Vec<u8>, DecodeReport), ModemError>` that returns both the recovered data and the diagnostic report. The existing depacketize_stream_rs should continue to work unchanged (it can call the new function internally and discard the report).

3. Graceful Degradation: Currently, if RS can't reconstruct a block, the entire depacketize fails. Implement a partial recovery mode:
   - Add a `depacketize_stream_rs_partial(data: &[u8]) -> (Vec<u8>, DecodeReport)` that never returns Err. Instead, it reconstructs what it can, marks failed blocks in the DecodeReport, and fills unrecoverable blocks with zeros (or a configurable fill byte). The report's rs_blocks_failed count tells the caller how much data was lost.
   - This enables the decoder to output a partially recovered file (useful for images where partial data is still viewable) rather than failing entirely.
   - In modem_decode.rs, add a --partial flag that uses partial recovery and prints the decode report, so the user can see exactly what was recovered and what was lost.

4. Enhanced Simulation at the Audio Level: The existing simulate_channel_bytes operates on bytes before symbolization. Add a function that simulates impairments on the rendered audio samples directly, which is more realistic:

   ```rust
   pub fn simulate_channel_audio(
       samples: &mut Vec<i16>,
       params: &AudioNoiseParams,
   )
   ```

   Where AudioNoiseParams includes:
   - white_noise_amplitude: i16 (additive Gaussian noise per sample)
   - dropout_probability: f32 (probability that a symbol-window of samples gets zeroed)
   - frequency_offset_hz: f32 (shifts all tone frequencies by this amount to simulate clock drift — advanced, implement only if straightforward)
   - gain_variation: f32 (random per-symbol amplitude scaling ±this fraction, simulating fading)

   Wire this into modem_encode.rs's existing --simulate path as an alternative mode (e.g., --sim-audio-noise <amplitude> --sim-audio-dropout <prob>), and into channel_sim.rs as a new mode ("audio" mode that operates on WAV files instead of raw bytes).

5. Improved Decoder Output: Update modem_decode.rs to:
   - Print a structured summary after decode showing the DecodeStrategy used, RS correction statistics, CRC result, and any warnings.
   - When --verbose is present (add this flag if it doesn't exist), print per-block RS status (recovered/failed, shards present vs needed).
   - Exit with distinct exit codes: 0 = full recovery, 1 = partial recovery (with --partial), 2 = total failure.

6. CRC Enforcement: Currently extract_frame prints a CRC warning but returns the data anyway. Change this behavior:
   - Default: CRC mismatch returns Err(ModemError::CrcMismatch { ... }).
   - Add a --ignore-crc flag to modem_decode.rs that preserves the current permissive behavior.
   - This is a subtle breaking change — update extract_frame's signature or add a new extract_frame_strict alongside the existing one. Ensure modem_encode.rs and any other callers still compile.

IMPORTANT CONSTRAINTS:
- Do NOT modify the #[cfg(test)] mod tests block in modem.rs — the Test Engineer owns it. If your changes break existing tests, fix the public API to remain backward compatible, not the tests.
- Do NOT modify docs/ — the Documentarian owns it.
- Run cargo test frequently during development. The Test Engineer is working in parallel and may have added tests that depend on the current API.
- When adding CLI flags, follow the existing arg-parsing pattern in modem_encode.rs (manual args iteration, no clap dependency).

CODING CONVENTIONS:
- Use thiserror for ModemError.
- Write /// doc comments on all new public types and functions.
- Use anyhow in binary code (bins) for convenience.
- Run cargo fmt before finishing.

When complete, mark task "modem-resilience" as done and message the lead with: the ModemError variants added, new public functions, CLI flag changes, and any backward compatibility notes.
```


### Spawn: Quality Gate

```
You are the Quality Gate for AudioHax modem hardening. You validate that the Test Engineer, Protocol Documentarian, and Resilience Engineer have produced correct, compiling, well-tested code. You own docs/modem-hardening-review.md only. Do NOT modify any src/ or assets/ files.

PROJECT CONTEXT:
Three agents have been working on the MFSK data modem (src/modem.rs and src/bin/ utilities). One added comprehensive round-trip tests (unit and integration), one wrote a protocol specification and demo CLI, and one improved error handling with structured types and graceful degradation. The modem is completely independent from the music pipeline.

VALIDATION SEQUENCE:

1. Compilation:
   cargo build --release 2>&1
   If this fails, document errors and message lead immediately.

2. Format: cargo fmt -- --check

3. Lint: cargo clippy -- -W clippy::all 2>&1

4. Test suite:
   cargo test 2>&1
   Pay special attention to any modem_roundtrip tests or modem-related unit tests. All must pass.

5. Module boundary audit:
   - Verify NO modem agent modified any music pipeline file (main.rs, chord_engine.rs, mapping_loader.rs, midi_output.rs, image_source.rs, image_analysis.rs, assets/mappings.json).
   - Verify the new modem_demo binary compiles and is properly declared in Cargo.toml.
   - Verify the Test Engineer did not modify modem.rs outside the #[cfg(test)] block.

6. Protocol documentation review:
   Read docs/modem-protocol.md and cross-reference against src/modem.rs:
   - Verify the frame header byte layout matches build_frame() and parse_frame_header() exactly. Check byte offsets, endianness, and field sizes.
   - Verify the PKT1 packet format matches packetize_stream().
   - Verify the RS shard header format matches packetize_stream_rs().
   - Verify the frequency formula matches render_symbols_to_samples() and build_tone_frequencies().
   - Verify the preamble description matches the encode/decode logic.
   - Flag any discrepancy as a blocking issue — an inaccurate protocol spec is worse than no spec.

7. Resilience review:
   - Verify ModemError is defined with thiserror and has the key variants (CrcMismatch, RsDecodeFailed at minimum).
   - Verify depacketize_stream_rs_with_report (or similar) returns a DecodeReport with meaningful data, not all zeros or placeholders.
   - Verify the partial recovery function actually fills unrecoverable blocks rather than panicking.
   - Verify CRC enforcement: extract_frame (or its strict variant) returns an error on CRC mismatch by default.
   - Check backward compatibility: do existing tests still pass? Does the existing modem_encode → modem_decode workflow still function?

8. Test coverage review:
   - Verify there is at least one test for each round-trip path: plain, compressed, encrypted, compressed+encrypted.
   - Verify there is at least one full audio pipeline test (encode → render → detect → decode in memory).
   - Verify there is at least one noise tolerance test.
   - Verify there is a test for unrecoverable damage that checks for graceful failure (Err, not panic).
   - Verify tests do not write to the filesystem (no temp files left behind).

9. Demo binary verification:
   - Verify src/bin/modem_demo.rs compiles.
   - If possible, run it with a small test input: create a small text file, run the demo, verify it reports success.

DELIVERABLES:
Write docs/modem-hardening-review.md with sections: Compilation, Lint, Tests, Module Boundaries, Protocol Accuracy, Resilience Logic, Test Coverage, Demo Binary, Blocking Issues, Non-Blocking Issues, and Overall Verdict (PASS / PASS WITH ISSUES / FAIL).

Mark task "modem-quality-gate" as done and message lead with the verdict and summary.
```


---

## Task List with Dependency Chains

```
Task ID: modem-tests
Title: Comprehensive round-trip tests for MFSK modem
Assignee: Test Engineer
Status: pending
Dependencies: none
Description: Add unit tests in modem.rs for frame, symbol, and packetization round-trips. Add integration tests in tests/modem_roundtrip.rs for full audio pipeline (encode→render→detect→decode) with clean signal, noise injection, burst loss, and unrecoverable damage scenarios. All tests in-memory, no filesystem, deterministic where possible.

Task ID: modem-protocol-doc
Title: Protocol specification and end-to-end demo CLI
Assignee: Protocol Documentarian
Status: pending
Dependencies: none
Description: Write docs/modem-protocol.md with complete byte-level protocol spec derived from modem.rs source. Create src/bin/modem_demo.rs that runs the full encode→render→detect→decode pipeline in-memory with optional noise injection and encryption, printing a structured summary. Add Cargo.toml [[bin]] entry.

Task ID: modem-resilience
Title: Structured errors, diagnostics, graceful degradation, audio-level simulation
Assignee: Resilience Engineer
Status: pending
Dependencies: modem-tests
Description: Add ModemError enum with thiserror. Add DecodeReport struct and depacketize_stream_rs_with_report(). Add partial recovery mode. Add audio-level noise simulation. Update modem_decode.rs with --partial, --verbose, --ignore-crc flags and structured decode summary. Enforce CRC by default. Maintain backward compatibility with existing API.

Task ID: modem-quality-gate
Title: Validate modem hardening work
Assignee: Quality Gate
Status: pending
Dependencies: modem-tests, modem-protocol-doc, modem-resilience
Description: Run build/test/lint. Audit module boundaries (no music pipeline files touched). Cross-reference protocol doc against actual modem.rs code. Review error types and partial recovery logic. Verify test coverage. Test demo binary. Produce review report.
```


---

## Lead Command Sequence

```bash
# ── 0. Prerequisites ──────────────────────────────────────────────
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# Verify clean state
cargo build --release
cargo test

# Feature branch
git checkout -b feature/modem-hardening

# ── 1. Spawn the two independent agents in parallel ───────────────
# These have no dependencies and no file overlap.

# Spawn 1: Test Engineer
#   → paste Test Engineer spawn prompt

# Spawn 2: Protocol Documentarian
#   → paste Protocol Documentarian spawn prompt

# ── 2. Monitor and wait for Test Engineer ─────────────────────────
# The Resilience Engineer depends on modem-tests being complete,
# because it needs to modify modem.rs public API without breaking
# the tests that the Test Engineer is writing. Wait for modem-tests
# task to complete before spawning the Resilience Engineer.
#
# The Protocol Documentarian can continue in parallel — it only
# reads modem.rs and creates new files.

# ── 3. Spawn Resilience Engineer after Test Engineer completes ────
# Spawn 3: Resilience Engineer
#   → paste Resilience Engineer spawn prompt

# ── 4. Wait for all three to complete ────────────────────────────
# Monitor task status. The Documentarian may finish before the
# Resilience Engineer — that's fine, no dependency between them.

# ── 5. Spawn Quality Gate ────────────────────────────────────────
# Spawn 4: Quality Gate
#   → paste Quality Gate spawn prompt

# ── 6. Handle results ───────────────────────────────────────────
# If PASS: proceed to integration.
# If protocol doc has inaccuracies: fix them yourself or re-spawn
#   the Documentarian with specific corrections.
# If backward compat broken: re-spawn Resilience Engineer with
#   the specific test failure.

# ── 7. Integration test ─────────────────────────────────────────
# Run the demo binary end-to-end:
echo "Hello AudioHax modem test" > /tmp/test_input.txt

cargo run --release --bin modem_demo -- \
  --input /tmp/test_input.txt \
  --output /tmp/test_recovered.txt \
  --compress \
  --noise medium \
  --verbose

# Verify recovery:
diff /tmp/test_input.txt /tmp/test_recovered.txt

# Test with encryption:
cargo run --release --bin modem_demo -- \
  --input /tmp/test_input.txt \
  --output /tmp/test_recovered_enc.txt \
  --compress \
  --key 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef \
  --noise low \
  --verbose

diff /tmp/test_input.txt /tmp/test_recovered_enc.txt

# ── 8. Final validation and commit ──────────────────────────────
cargo build --release
cargo test
cargo fmt
cargo clippy -- -W clippy::all

git add -A
git commit -m "Modem hardening: comprehensive tests, protocol spec, structured errors, graceful degradation

- Add round-trip tests for frame, packetization, and full audio pipeline
- Add noise tolerance tests (bitflips, burst loss, noise floor)
- Add unrecoverable damage test verifying graceful failure
- Write complete byte-level protocol specification (docs/modem-protocol.md)
- Create modem_demo binary for hardware-free end-to-end pipeline testing
- Add ModemError enum with thiserror for structured error reporting
- Add DecodeReport with per-block RS statistics and decode strategy tracking
- Add partial recovery mode (--partial) for best-effort file recovery
- Add audio-level noise simulation (white noise, dropout, gain variation)
- Enforce CRC by default with --ignore-crc escape hatch
- Add --verbose decode diagnostics and distinct exit codes"
```


---

## Design Rationale

The team is 4 agents, which is one more than the 3-agent sweet spot suggested in the agent teams reference. The fourth agent (Protocol Documentarian) is justified because protocol documentation requires a fundamentally different skill — reading code for specification extraction — and its file ownership (docs/ and a new binary) has zero overlap with the other agents. Collapsing it into another role would force that agent to context-switch between code modification and spec writing, which produces worse output for both.

The dependency chain has one critical sequencing constraint: the Resilience Engineer waits for the Test Engineer to finish. This is deliberate. The Resilience Engineer modifies modem.rs public API (adding error types, new function signatures, changing CRC behavior). If it runs in parallel with the Test Engineer, there's a high risk of API churn — the Test Engineer writes tests against the current API, then the Resilience Engineer changes the API, and the tests break. By sequencing them, the Test Engineer locks in the expected behavior as tests first, and the Resilience Engineer's constraint becomes "don't break these tests," which is much easier to validate. The Protocol Documentarian runs fully in parallel with everyone since it only reads source and creates new files.

The modem-tests → modem-resilience dependency means this team takes slightly longer than a fully parallel fan-out (roughly 1.5x the time of a single agent, vs 1x for full parallelism). This is the right tradeoff: the alternative is test-API churn that the Quality Gate catches, forcing re-spawns that cost more in total.

The Test Engineer's spawn prompt deliberately separates the full audio round-trip test from the noise tolerance tests. The clean round-trip is the foundation — if it fails, the noise tests are meaningless. The noise tests are calibrated to be on the edge of what RS-FEC can handle: with data_shards=4 and parity_shards=2, up to 2 out of 6 shards per block can be lost. The "unrecoverable" test intentionally exceeds this limit to verify graceful failure rather than silent corruption, which is exactly what the Resilience Engineer's partial recovery mode needs to handle.
