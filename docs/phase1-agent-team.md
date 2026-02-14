# Phase 1: Music Quality — Agent Team Configuration

## Team Overview

Team name: `music-quality`
Agent count: 4
Coordination pattern: Hybrid fan-out with pipeline quality gate. The Voice Leading Engineer, Phrase Architect, and Mapping Enricher fan out in parallel on their respective files. The Quality Gate runs after all three complete, validating compilation, tests, lint, module boundaries, and musical correctness.

```
┌─────────────┐   ┌──────────────┐   ┌─────────────────┐
│ Voice Lead   │   │ Phrase Arch  │   │ Mapping Enrich  │
│ Engineer     │   │              │   │                 │
│ chord_engine │   │ main.rs orch │   │ mapping_loader  │
│              │   │              │   │ mappings.json   │
└──────┬───────┘   └──────┬───────┘   └────────┬────────┘
       │                  │                     │
       └──────────────────┼─────────────────────┘
                          │
                    ┌─────▼──────┐
                    │  Quality   │
                    │   Gate     │
                    │ read-only  │
                    │ + tests/   │
                    └────────────┘
```


## Agent Definitions

### Agent 1: Voice Leading Engineer

Role: Implement proper voice leading, tendency tone resolution, and non-chord tone support in chord_engine.rs.

Files OWNED (may create/modify):
- src/chord_engine.rs

Files EXCLUDED (must NOT modify):
- src/main.rs
- src/mapping_loader.rs
- src/midi_output.rs
- src/image_source.rs
- src/image_analysis.rs
- src/modem.rs
- src/lib.rs
- src/bin/*
- assets/*

Communication: Mark task `voice-leading-core` as complete when done. Message lead with a summary of public API additions/changes so the Phrase Architect knows what to call.


### Agent 2: Phrase Architect

Role: Implement phrase grouping, dynamic contour, rhythmic variety, and melodic independence in main.rs orchestration and worker_decide_action().

Files OWNED (may create/modify):
- src/main.rs

Files EXCLUDED (must NOT modify):
- src/chord_engine.rs
- src/mapping_loader.rs
- src/midi_output.rs
- src/image_source.rs
- src/image_analysis.rs
- src/modem.rs
- src/lib.rs
- src/bin/*
- assets/*

Communication: Mark task `phrase-structure` as complete when done. If you need new public methods from chord_engine.rs, message the Voice Leading Engineer describing the signature you need — do not modify chord_engine.rs yourself. In the interim, code against the expected signature and leave a `// TODO: requires voice-leading API` comment.


### Agent 3: Mapping Enricher

Role: Extend mapping_loader.rs parsing and assets/mappings.json with new visual-to-musical parameter mappings for articulation, rhythmic density, dynamic range, and temporal change detection.

Files OWNED (may create/modify):
- src/mapping_loader.rs
- assets/mappings.json

Files EXCLUDED (must NOT modify):
- src/main.rs
- src/chord_engine.rs
- src/midi_output.rs
- src/image_source.rs
- src/image_analysis.rs
- src/modem.rs
- src/lib.rs
- src/bin/*

Communication: Mark task `mapping-enrichment` as complete when done. Message lead with a summary of new MappingTable fields and lookup functions so other agents know what's available.


### Agent 4: Quality Gate

Role: After the three implementation agents complete, run the full validation suite, review musical logic for correctness, and produce a review report.

Files OWNED (may create/modify):
- tests/* (integration tests only)
- docs/phase1-review.md (review report)

Files EXCLUDED (must NOT modify):
- src/* (all source files are read-only for this agent)
- assets/*

Communication: Mark task `quality-gate` as complete when done. Message lead with pass/fail status and any issues found.


---

## Spawn Prompts

### Spawn: Voice Leading Engineer

```
You are the Voice Leading Engineer for AudioHax, a Rust project that converts images into music via MIDI. You own src/chord_engine.rs exclusively. Do NOT modify any other source files.

PROJECT CONTEXT:
AudioHax scans images left-to-right, extracts visual features (hue, saturation, brightness, edge density, texture) at each scan position, maps them to musical parameters, and outputs multi-channel MIDI. The project owner has a music performance degree (trombone) and professional experience — your musical decisions must meet a professional theory standard.

BUILD/TEST/LINT:
  cargo build --release
  cargo test
  cargo fmt
  cargo clippy -- -W clippy::all
Run all four before marking any task complete.

CURRENT STATE OF chord_engine.rs:
ChordEngine has pick_progression() which selects Roman numeral progressions by mode family (warm/cool/neutral), and generate_chords() which converts progressions to Vec<Chord> where each Chord has a name and a Vec<u8> of MIDI note numbers. The current generate_chords() does basic Roman-to-MIDI conversion with modal interchange and secondary dominant insertion, but performs NO voice leading — each chord is voiced independently with no memory of the previous chord's voicing.

YOUR DELIVERABLES:

1. VoiceState tracking: Add a struct (e.g., VoiceState or VoiceLeadingContext) that maintains each voice's current MIDI pitch across chord changes. This state persists across the full scan so voices move smoothly from chord to chord.

2. Closest-pitch-class voice leading: When a new chord arrives, each non-bass voice finds the nearest available chord tone by minimal semitone distance. Ties break by: (a) prefer common tones (notes present in both old and new chord stay put), (b) prefer contrary motion to bass, (c) prefer stepwise motion over leaps. The bass voice takes the chord root in the appropriate octave (typically octave 2-3, MIDI 36-59).

3. Parallel motion prohibition: After computing the initial voice leading, check all voice pairs. If any two voices move in parallel perfect fifths (interval of 7 semitones at both time T and T+1) or parallel perfect octaves/unisons (0 or 12 semitones), adjust the upper voice by one scale step to break the parallel. Document which voice was adjusted and why in a debug-level comment or log.

4. Tendency tone resolution: If a voice holds the leading tone (scale degree 7 in major modes, or raised 7 in minor when it appears as part of a V chord), it must resolve upward by half step to the tonic. If a voice holds a chordal seventh, it must resolve downward by step. These constraints override the closest-pitch-class algorithm when they apply.

5. Non-chord tone support: Add a method (e.g., embellish_voice() or generate_melodic_line()) that can insert passing tones (diatonic stepwise motion between two chord tones), neighbor tones (step away from and back to a chord tone), and suspensions (hold a note from the previous chord into the new chord, then resolve down by step on a subsequent beat). This method should accept a "dissonance_level" parameter (0.0 to 1.0) that controls how frequently non-chord tones appear — 0.0 means pure chord tones only, 1.0 means maximum embellishment. The image's texture complexity will drive this parameter (but that mapping happens in main.rs, not here).

6. Public API: Expose a method like voice_led_chords() or resolve_voice_leading() that takes the raw chord sequence from generate_chords(), a VoiceState, the current scale/mode, and returns a Vec of voiced chords where each chord is a Vec<u8> of MIDI pitches with proper voice leading applied. The Phrase Architect in main.rs will call this instead of using raw generate_chords() output directly.

MUSIC THEORY CONSTRAINTS TO ENFORCE:
- Maximum melodic interval in any non-bass voice: perfect fifth (7 semitones). Larger leaps are forbidden except in bass.
- No parallel perfect fifths or octaves between any voice pair.
- No voice crossing: voice N should not go below voice N-1 or above voice N+1.
- Common tones retained when possible (this is the strongest voice leading principle).
- Leading tone resolves up to tonic. Chordal seventh resolves down by step.
- When two constraints conflict, prioritize: tendency tone resolution > parallel avoidance > common tone retention > minimal motion.

TESTING REQUIREMENTS:
Write unit tests in a #[cfg(test)] mod tests block within chord_engine.rs. Tests must verify:
- No voice moves more than 7 semitones between consecutive chords (except bass).
- No parallel perfect fifths or octaves exist in output.
- Common tones are retained when the algorithm has a choice.
- Leading tone resolution occurs when the leading tone is present.
- Voice crossing does not occur.
- The embellish method produces passing/neighbor tones that are diatonic to the current scale.
Use concrete chord sequences for testing: e.g., I-V-vi-IV in C Ionian, i-iv-V-i in A Aeolian.

CODING CONVENTIONS:
- Use Result types with explicit error handling, not unwrap().
- Write /// doc comments on all public functions and types.
- Use thiserror if you add more than 2 error variants.
- Document music-theory reasoning in comments (e.g., "// Resolve leading tone B -> C per tendency tone rule").
- Run cargo fmt before finishing.

When complete, mark task "voice-leading-core" as done and message the lead with a summary of the new public API (method signatures, new types).
```


### Spawn: Phrase Architect

```
You are the Phrase Architect for AudioHax, a Rust project that converts images into music via MIDI. You own src/main.rs exclusively. Do NOT modify any other source files.

PROJECT CONTEXT:
AudioHax scans images left-to-right across N steps (default 40), extracting visual features at each position and playing multi-instrument MIDI via a worker-coordinator concurrency pattern with barrier synchronization. The project owner has a music performance degree (trombone, professional experience). Musical output must sound composed, not algorithmic.

BUILD/TEST/LINT:
  cargo build --release
  cargo test
  cargo fmt
  cargo clippy -- -W clippy::all
Run all four before marking any task complete.

CURRENT STATE OF main.rs:
worker_decide_action() is the note decision function (approx lines 139-177). It currently has only two behaviors: if edge_density > threshold, play an ascending arpeggio of chord tones; otherwise, play a single sustained chord tone selected by instrument index. Velocity maps directly to saturation (static). Timing humanization is random jitter (±percent on hold duration). There is no phrase structure, no dynamic contour, no melodic independence, no articulation variety.

The coordinator loop in play_scanned_steps_concurrent() collects InstrumentAction structs from workers, builds ScheduledEvents sorted by time, and executes them. InstrumentAction contains events as Vec<(note: u8, velocity: u8, hold_ms: u64, offset_ms: u64)>.

YOUR DELIVERABLES:

1. PhraseManager: Create a struct (defined within main.rs or a small inline module) that groups the total scan steps into phrases. Default phrase length is 8 steps (configurable). Each phrase tracks its index within the scan, its position in the phrase (0.0 to 1.0 normalized), and its role in the macro form. The macro form should follow an arch shape: opening → development → climax → resolution, with the climax phrase positioned around 60-70% through the scan. The PhraseManager should be constructible from (total_steps, phrase_length) and queryable: given a step index, return the phrase index, position within phrase (0.0-1.0), position within macro form (0.0-1.0), and whether this step is a phrase boundary.

2. Dynamic contour: Replace the static velocity_from_saturation() with a function that takes saturation AND phrase position AND macro position. Within each phrase, velocity follows a messa di voce contour: crescendo from phrase start to ~60-70% through, then diminuendo to phrase end. The saturation sets the overall dynamic range (low saturation = pp-mp velocity range 40-75, high saturation = mf-ff velocity range 75-110). Strong beats within the phrase (positions 0 and 4 in an 8-step phrase, or 0 and 2 in a 4-step phrase) get a +8 velocity accent. The macro form multiplies an additional envelope: opening phrases at ~85% intensity, climax phrase at 100%, resolution at ~70%.

3. Rhythmic variety: Replace the binary arpeggio/sustained decision with a rhythm pattern system. Define at least 6 rhythm patterns as sequences of (relative_onset, relative_duration) tuples normalized to one step: whole note (single attack, full duration), half notes (two attacks), quarter-note pulse (four attacks), dotted-quarter + eighth, syncopated (off-beat attacks), and rest-inclusive (attacks with deliberate silence gaps). Edge density selects the base activity level (low density → whole/half, high → quarters/eighths/syncopated), but add controlled variation: the same edge density should NOT always produce the same pattern. Use phrase position to influence this — phrase beginnings favor simpler patterns, mid-phrase allows complexity, phrase endings return to simpler patterns with ritardando (gradually increasing hold times in the last 2 steps of each phrase).

4. Voice role assignment: Assign musical roles based on instrument index and instrument count. Voice 0 is always bass (plays chord roots, lower register MIDI 36-59). The highest-index voice is melody (gets wider pitch range, more rhythmic freedom, non-chord tones when the voice leading API supports them). Middle voices are harmonic fill (block chord tones with voice leading). Each role should have different rhythm pattern selection biases: bass favors longer notes with walking patterns between roots; melody favors more active, varied patterns; harmonic voices favor sustained support. Implement this as a VoiceRole enum (Bass, Melody, Harmonic) with methods that influence pattern selection and register.

5. Phrase-boundary cadences: At phrase boundaries (last 1-2 steps of a phrase), force the chord selection to create cadential motion. If the chord engine provides a V chord at the penultimate step and I/i at the final step, great. If not, modify the chord index selection to prefer V-I (authentic), V-vi (deceptive, use sparingly for variety — perhaps every 3rd or 4th phrase), or IV-I (plagal, at the final phrase). This may require selecting different chords from the existing progression at boundary steps rather than just cycling through chords[step % len].

6. Musical timing: Replace random jitter with phrase-aware timing modification. Instead of uniform random noise, apply: slight rubato at phrase boundaries (stretch the last 2 steps by 10-20%), subtle swing feel (odd-numbered sub-beats slightly delayed, configurable 0-30% of sub-beat duration), and accelerando approaching the climax phrase in the macro form (compress step timing by up to 10%). Keep the jitter_percent CLI parameter but reinterpret it as the maximum envelope of musical timing variation rather than random noise.

IMPORTANT API NOTES:
The Voice Leading Engineer is simultaneously building a voice_led_chords() or similar method in chord_engine.rs. If that API is not yet available when you implement, code against the expected interface: a method that takes &[Chord], a mutable VoiceState, and returns voiced chords with smooth voice leading. Leave // TODO comments at call sites. For now, continue using the existing chords[step % chords.len()] but structure your code so swapping in voice-led chords is a one-line change.

The Mapping Enricher is simultaneously adding new fields to MappingTable for articulation, rhythmic density, and dynamic range. If those fields aren't available yet, use the existing ScanBarFeatures directly (edge_density, avg_saturation, avg_brightness, texture_complexity) and leave // TODO comments noting which mapping lookups should replace hardcoded thresholds.

TESTING REQUIREMENTS:
Write unit tests in a #[cfg(test)] mod tests block within main.rs. Tests must verify:
- PhraseManager correctly groups steps into phrases of the configured length.
- Velocity varies within a phrase (not constant across all steps).
- Phrase climax velocity exceeds phrase start and end velocities.
- Strong-beat accents produce higher velocity than adjacent weak beats.
- At least 3 distinct rhythm patterns are used across a full scan.
- Phrase-end steps have longer hold times than mid-phrase steps (ritardando).
- Bass voice notes fall within MIDI 36-59 range.

CODING CONVENTIONS:
- anyhow is acceptable for error handling in main.rs.
- Write /// doc comments on all public functions and new types.
- Document music-theory reasoning in comments.
- Run cargo fmt before finishing.

When complete, mark task "phrase-structure" as done and message the lead summarizing: new types added, changes to worker_decide_action signature, changes to the coordinator loop, and any TODO items that depend on other agents.
```


### Spawn: Mapping Enricher

```
You are the Mapping Enricher for AudioHax, a Rust project that converts images into music via MIDI. You own src/mapping_loader.rs and assets/mappings.json exclusively. Do NOT modify any other source files.

PROJECT CONTEXT:
AudioHax uses a JSON configuration system (assets/mappings.json) to define how visual features map to musical parameters. mapping_loader.rs parses this JSON into a MappingTable struct that other modules query. The project owner has a music performance degree — mappings should produce musically meaningful results, not arbitrary data sonification.

BUILD/TEST/LINT:
  cargo build --release
  cargo test
  cargo fmt
  cargo clippy -- -W clippy::all
Run all four before marking any task complete.

CURRENT STATE:
mappings.json has three sections: "global" (whole-image features → macro musical decisions), "instrument_section" (per-instrument scan bar features → note-level decisions), and "fine_detail" (local region features → expression). MappingTable in mapping_loader.rs mirrors this structure. lookup_range_map() converts a float value to a string by checking which range it falls in. The existing mappings cover hue→mode, saturation→harmonic complexity, brightness→tempo, edge density→rhythm, contrast→articulation, texture→modal color, and some fine detail mappings.

YOUR DELIVERABLES:

1. Articulation mappings: Expand the "instrument_section" in mappings.json with a proper articulation system. The existing contrast_to_articulation (Legato/Portato/Staccato) is a good start. Add articulation intensity ranges that map to note duration as a fraction of step duration: Legato = 0.95-1.05 (slight overlap), Portato = 0.75-0.85, Normal = 0.60-0.70, Staccato = 0.35-0.45, Marcato = 0.50-0.60 (with a velocity boost flag). Add the corresponding struct fields and parsing in mapping_loader.rs. Create a lookup function like lookup_articulation() that returns both the duration fraction and whether a velocity accent applies.

2. Rhythmic density mappings: The existing edge_density_to_rhythm maps to note value names (HalfNotes, QuarterNotes, EighthNotes). Extend this to return a numeric density value (1-8, representing attacks per step) so main.rs can select from rhythm patterns by activity level rather than hardcoded names. Also add edge_orientation_to_rhythmic_character: horizontal edges → sustained/legato patterns, vertical edges → punchy/staccato patterns, diagonal → mixed. Add the corresponding struct fields and lookup functions.

3. Dynamic range mappings: Add a saturation_to_dynamic_range mapping that returns a (min_velocity, max_velocity) pair. Low saturation (0-30%) → (40, 75) representing pp to mp. Medium saturation (31-70%) → (60, 100) representing mp to f. High saturation (71-100%) → (80, 115) representing mf to ff. This gives the Phrase Architect concrete velocity bounds to shape dynamics within. Add the struct field and a lookup_dynamic_range() function.

4. Temporal change sensitivity: Add a new top-level section "temporal" in mappings.json with thresholds for detecting musically significant changes between consecutive scan steps. Define: hue_shift_threshold (degrees of hue change that triggers a key/mode modulation), brightness_delta_threshold (brightness change magnitude that triggers subito dynamics), edge_density_delta_threshold (texture change that triggers rhythmic shift). These are configuration values the Phrase Architect and image analysis will consume. Add them to MappingTable and provide a lookup interface.

5. Instrument role mappings: Add an "instrument_roles" section that maps instrument index patterns to General MIDI program numbers with musical logic. Rather than the current arbitrary (inst_idx * 7) % 128 program assignment in main.rs, define role-based presets: bass role → programs like 33 (acoustic bass), 34 (electric bass), 43 (cello); melody role → programs like 74 (flute), 69 (oboe), 57 (trumpet), 1 (acoustic piano); harmonic role → programs like 49 (strings), 5 (electric piano), 90 (pad). Group these by "color temperature" so warm hues select from warm-timbre instruments and cool hues from cool-timbre instruments. Add struct fields and a lookup_instrument_program(role, hue) function.

6. Backward compatibility: All new fields in MappingTable must be Optional (wrapped in Option<>) or use #[serde(default)] so that the existing mappings.json (without new sections) still loads without error. Write a test that verifies the current mappings.json parses successfully, then a test with the extended mappings.json.

TESTING REQUIREMENTS:
Write unit tests in mapping_loader.rs. Tests must verify:
- Current mappings.json parses without error (backward compatibility).
- Extended mappings.json with all new sections parses correctly.
- lookup_articulation() returns correct duration fractions for known contrast values.
- lookup_dynamic_range() returns correct velocity bounds for known saturation values.
- Instrument role lookups return valid GM program numbers (0-127).
- Temporal thresholds load and are accessible.

CODING CONVENTIONS:
- Use Result types with explicit error handling, not unwrap().
- Write /// doc comments on all public functions and types.
- Use #[serde(default)] for backward compatibility on new fields.
- Run cargo fmt before finishing.

When complete, mark task "mapping-enrichment" as done and message the lead with: new struct fields added to MappingTable, new public lookup functions, and the updated mappings.json schema.
```


### Spawn: Quality Gate

```
You are the Quality Gate for AudioHax Phase 1 (Music Quality). You validate that the Voice Leading Engineer, Phrase Architect, and Mapping Enricher have produced correct, compiling, well-tested code that actually implements the musical logic it claims to. You own tests/ and docs/phase1-review.md. Do NOT modify any src/ or assets/ files.

PROJECT CONTEXT:
AudioHax converts images to music via MIDI. The project owner has a music performance degree (trombone, professional experience). Three agents have been working in parallel: one on voice leading in chord_engine.rs, one on phrase structure in main.rs, one on mapping extensions in mapping_loader.rs and mappings.json. Your job is to verify their work compiles, passes tests, meets lint standards, respects module boundaries, and — most critically — implements musically correct logic.

VALIDATION SEQUENCE (execute in this order):

1. Compilation check:
   cargo build --release 2>&1
   If this fails, document the errors and message the lead immediately. Do not proceed.

2. Format check:
   cargo fmt -- --check
   If formatting issues exist, document them. These are non-blocking but should be noted.

3. Lint check:
   cargo clippy -- -W clippy::all 2>&1
   Document any warnings. Warnings about unused variables or imports from in-progress integration are acceptable. Warnings about unsafe code, logic errors, or correctness issues are blocking.

4. Test suite:
   cargo test 2>&1
   All tests must pass. Document any failures with the test name and error message.

5. Module boundary audit: Read each modified file and verify:
   - chord_engine.rs contains NO references to main.rs types, no MIDI output logic, no image analysis logic.
   - main.rs does not contain music theory logic that belongs in chord_engine.rs (scale construction, interval calculation, chord voicing rules should be in chord_engine.rs; main.rs should call chord_engine methods).
   - mapping_loader.rs contains no hardcoded musical values — all values come from mappings.json.
   - No file was modified by an agent that doesn't own it.

6. MUSICAL LOGIC REVIEW (this is the most important step):

   For chord_engine.rs voice leading:
   - Read the voice leading algorithm. Verify it actually tracks previous voice positions (not just computing each chord independently).
   - Check that the parallel fifth/octave detection compares intervals between voice pairs at consecutive time steps, not just within a single chord.
   - Verify tendency tone resolution: does the code actually identify the leading tone (7th scale degree) and chordal sevenths, and force their resolution direction? Or does it just have a comment saying it will?
   - Check that voice crossing prevention compares actual MIDI pitches between adjacent voices, not just indices.
   - Verify the non-chord tone generation produces notes that are diatonic to the current scale (not chromatic random steps).
   - Run the voice leading tests and verify they test ACTUAL musical constraints, not just "function doesn't panic."

   For main.rs phrase structure:
   - Verify PhraseManager actually groups steps and that phrase position (0.0-1.0) is computed correctly.
   - Check that dynamic contour produces velocity values that vary within a phrase — specifically that mid-phrase velocity > start/end velocity (messa di voce shape). If the velocity calculation is just saturation * some_constant with no phrase-position modulation, the task is NOT complete.
   - Verify rhythm patterns are actually distinct (different onset/duration sequences), not just the same pattern with different names.
   - Check that phrase-end ritardando actually increases hold_ms in the last 2 steps, not just adds random noise.
   - Verify bass voice notes are in the correct register (MIDI 36-59), not the same octave as melody.
   - Check that voice roles (Bass, Melody, Harmonic) actually produce different behavior, not just different labels.

   For mapping_loader.rs:
   - Verify backward compatibility: the EXISTING mappings.json (before enrichment) must still parse. If the Enricher added required fields without defaults, this is a blocking issue.
   - Check that new lookup functions return sensible values (velocity ranges within 0-127, duration fractions between 0.0 and 1.1, GM programs 0-127).

7. Integration assessment: Do the three agents' outputs fit together? Can main.rs call the new chord_engine voice leading API? Can main.rs use the new mapping lookups? Are there type mismatches or import issues between the modules? Note any integration gaps that the lead will need to resolve.

DELIVERABLES:
Write docs/phase1-review.md with sections: Compilation Status, Lint Status, Test Results, Module Boundary Audit, Musical Logic Review (per-agent), Integration Assessment, Blocking Issues, Non-Blocking Issues, and Overall Verdict (PASS / PASS WITH ISSUES / FAIL).

If the verdict is FAIL, message the lead immediately with the blocking issues. If PASS or PASS WITH ISSUES, mark task "quality-gate" as done and message the lead with the verdict and a one-paragraph summary.

CODING CONVENTIONS:
- If you write integration tests in tests/, they should test cross-module interactions (e.g., generate chords via chord_engine, verify voice leading, then verify they fit the phrase structure's expected input format).
- Do not modify src/ files even if you find issues — document them for the lead to assign back to the owning agent.
```


---

## Task List with Dependency Chains

These tasks go into the Claude Code task system at `~/.claude/tasks/music-quality/`.

```
Task ID: voice-leading-core
Title: Implement voice leading engine in chord_engine.rs
Assignee: Voice Leading Engineer
Status: pending
Dependencies: none
Description: Add VoiceState tracking, closest-pitch-class resolution, parallel fifth/octave prohibition, tendency tone resolution, non-chord tone support (passing tones, neighbor tones, suspensions), and a public voice_led_chords() API. Include unit tests for all voice leading constraints.

Task ID: phrase-structure
Title: Implement phrase grouping, dynamic contour, and rhythmic variety in main.rs
Assignee: Phrase Architect
Status: pending
Dependencies: none
Description: Add PhraseManager for step grouping, messa di voce dynamic contour with macro-form envelope, 6+ rhythm patterns selected by edge density and phrase position, voice role assignment (Bass/Melody/Harmonic), phrase-boundary cadential chord selection, and musical timing (rubato, swing, ritardando). Include unit tests for phrase structure and dynamic variation.

Task ID: mapping-enrichment
Title: Extend mapping system with articulation, dynamics, rhythm, and instrument role mappings
Assignee: Mapping Enricher
Status: pending
Dependencies: none
Description: Add articulation duration fractions, rhythmic density values, dynamic range velocity bounds, temporal change thresholds, and instrument role GM program lookups to mapping_loader.rs and mappings.json. Maintain backward compatibility with existing mappings.json. Include unit tests for all new lookups.

Task ID: quality-gate
Title: Validate compilation, tests, lint, module boundaries, and musical correctness
Assignee: Quality Gate
Status: pending
Dependencies: voice-leading-core, phrase-structure, mapping-enrichment
Description: Run cargo build/test/fmt/clippy. Audit module boundaries. Review voice leading algorithm for actual musical correctness (not just code correctness). Review phrase dynamics for genuine contour (not static values). Review mapping backward compatibility. Produce docs/phase1-review.md with detailed findings and an overall PASS/FAIL verdict.
```


---

## Lead Command Sequence

Run these commands from the repo root as team lead. The comments explain what each step does.

```bash
# ── 0. Prerequisites ──────────────────────────────────────────────
# Ensure agent teams are enabled
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# Verify the project compiles cleanly before spawning agents
cargo build --release
cargo test
cargo fmt -- --check
cargo clippy -- -W clippy::all

# Create a feature branch for Phase 1 work
git checkout -b feature/phase1-music-quality

# ── 1. Plan review (optional, in plan mode) ──────────────────────
# Open Claude Code in plan mode and paste:
#   "Review the Phase 1 music quality roadmap in docs/ROADMAP.md.
#    Confirm the task breakdown and file ownership make sense
#    given the current state of chord_engine.rs and main.rs.
#    Flag any risks or missing dependencies."
# This is cheap (plan mode) and catches issues before expensive spawns.

# ── 2. Spawn the three implementation agents in parallel ──────────
# In your Claude Code lead session, run these three spawn commands.
# Paste the full spawn prompt for each agent from the sections above.
# The agents will fan out and work concurrently.

# Spawn 1: Voice Leading Engineer
#   → paste the Voice Leading Engineer spawn prompt above

# Spawn 2: Phrase Architect
#   → paste the Phrase Architect spawn prompt above

# Spawn 3: Mapping Enricher
#   → paste the Mapping Enricher spawn prompt above

# ── 3. Monitor progress ──────────────────────────────────────────
# Check task status periodically:
#   "Show me the status of all tasks in the music-quality team."
# Agents will message you when they complete or if they need input.
# The Voice Leading Engineer may message about API surface changes.
# The Phrase Architect may message about chord_engine API it needs.
# Relay these messages between agents if they need coordination.

# ── 4. Spawn Quality Gate after all three complete ────────────────
# Once voice-leading-core, phrase-structure, and mapping-enrichment
# are all marked complete:

# Spawn 4: Quality Gate
#   → paste the Quality Gate spawn prompt above

# ── 5. Handle Quality Gate results ───────────────────────────────
# If PASS: proceed to integration.
# If PASS WITH ISSUES: review docs/phase1-review.md, fix non-blocking
#   issues yourself or re-spawn the owning agent with fix instructions.
# If FAIL: read the blocking issues, re-spawn the responsible agent
#   with a targeted fix prompt including the specific failure.

# ── 6. Integration pass (lead does this personally) ──────────────
# After Quality Gate passes, do a manual integration pass:
#   - Wire the new voice_led_chords() API into main.rs if TODO markers remain
#   - Wire the new mapping lookups into worker_decide_action() if TODO markers remain
#   - Run a full playback test with a real image:
#     cargo run --release -- play --instruments 4 --steps 32
#   - Listen to the output. Does it sound musical? Are phrases audible?
#     Do dynamics shape? Does voice leading produce smooth motion?

# ── 7. Final validation and commit ───────────────────────────────
cargo build --release
cargo test
cargo fmt
cargo clippy -- -W clippy::all
git add -A
git commit -m "Phase 1: music quality — voice leading, phrase structure, dynamic contour, mapping enrichment

- Add VoiceState tracking with closest-pitch-class resolution
- Prohibit parallel fifths/octaves, enforce tendency tone resolution
- Add PhraseManager with messa di voce dynamic contour
- Implement 6+ rhythm patterns with phrase-aware selection
- Add voice role assignment (Bass/Melody/Harmonic)
- Extend mappings with articulation, dynamic range, rhythm density
- Add temporal change thresholds for visual discontinuity detection
- Add instrument role GM program mappings by timbre temperature
- Comprehensive tests for voice leading and phrase constraints"
```


---

## Risk Mitigation Notes

The biggest risk is merge friction between the three parallel agents. The file ownership boundaries are strict (each agent owns different files), so there should be no direct git conflicts. However, there will be API boundary mismatches: the Phrase Architect codes against chord_engine methods that may not exist yet, and vice versa. The TODO-marker convention handles this — after the Quality Gate passes, you do a manual integration pass (step 6) to wire the pieces together. This is intentional: having the lead do integration ensures a human musician evaluates how the pieces fit musically, not just syntactically.

The second risk is musical correctness that looks right in code but sounds wrong in playback. The Quality Gate catches obvious logic errors (static velocity, missing voice state tracking), but only your ears can judge whether the messa di voce contour is too aggressive, whether the rhythm patterns feel natural, or whether the voice leading produces interesting harmonic motion versus just "correct" motion. Step 6 (the listening test) is the real quality gate. Budget 30-60 minutes for iterative tuning after the agent work completes.

The third risk is scope creep within agents. Each spawn prompt is deliberately specific about deliverables and explicitly excludes certain files. If an agent messages saying "I also need to modify midi_output.rs to support CC messages," the right answer is "document what you need as a TODO and I'll handle it in integration." Keep the agents focused on their owned files.
