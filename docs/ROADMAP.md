# AudioHax Development Roadmap

## Current State Assessment

The image-to-music pipeline is functional end-to-end: images load, visual features extract at three levels, hue-based mode selection works well, chord progressions generate with modal interchange and secondary dominants, and multi-instrument MIDI plays through FluidSynth via virtual ports. The concurrency model (barrier-synchronized worker-coordinator) is solid. However, the musical output sounds algorithmic rather than composed. The modem subsystem is complete and working with RS-FEC, AES-GCM, and multi-channel MFSK, but lacks hardening (no adaptive sync, limited error reporting, no streaming mode). The interactive/game track exists only as a design concept.

The single highest-impact area is music quality. The current system makes reasonable harmonic choices (mode selection from hue, complexity tiers from saturation) but executes them without any of the elements that make music sound musical: voice leading, phrase structure, dynamic contour, rhythmic variety, or articulation nuance.

---

## Phase 1: Musical Expression (Highest Priority)

This phase transforms the output from data sonification into something that sounds composed. Every task here addresses a specific musical deficiency with a concrete, theory-grounded approach.

### 1.1 Voice Leading Engine

Complexity: HIGH. Owner: chord_engine.rs. Dependencies: none.

The current generate_chords() function produces chord tones without considering what the previous chord was or where each voice sat. The fix is a proper voice leading algorithm in chord_engine.rs.

The implementation should maintain a VoiceState struct that tracks each voice's current MIDI pitch across chord changes. When a new chord arrives, each voice finds its nearest available chord tone by minimal semitone distance (closest-pitch-class algorithm). Ties break by preferring common tones (notes shared between consecutive chords stay put), then contrary motion to the bass, then stepwise motion over leaps. The bass voice is exempt — it takes the chord root in the appropriate octave, since root-position bass movement is idiomatic for this kind of generative texture.

Beyond basic smoothness, the engine should enforce classical prohibitions: no parallel perfect fifths or octaves between any voice pair (check interval between voices at time T and T+1; if both are P5 or P8, adjust the upper voice by step). Resolve tendency tones — if a voice has the leading tone (scale degree 7 in major), it should resolve up to tonic; if it has the chordal seventh, it should resolve down by step.

Tests should verify: maximum interval between consecutive pitches in any voice is a perfect fifth (7 semitones) except bass, parallel fifths/octaves are absent in output, common tones are retained when available.

### 1.2 Phrase Structure and Formal Grouping

Complexity: HIGH. Owner: main.rs (orchestration), chord_engine.rs (progression logic). Dependencies: none (can parallel with 1.1).

Currently every scan step is treated identically — same progression, same behavior, no sense of beginning, middle, or end. The fix introduces a PhraseManager that groups steps into musical phrases.

The scan's total steps should be divided into phrases of 4 or 8 steps (configurable, defaulting to 8). Each phrase has an internal arc: tension builds through the first 60-70% of the phrase and releases through the remainder. This arc controls three parameters simultaneously. Harmonic rhythm (how often chords change) should start slower and accelerate mid-phrase or vice versa, not change at every step uniformly. The chord progression should not repeat identically across phrases — the PhraseManager selects or varies progressions per phrase, with later phrases introducing more complex harmony (secondary dominants, modal interchange) as the scan progresses through the image, creating a sense of development. Phrase boundaries should feature cadential patterns: authentic cadences (V-I) at strong boundaries, half cadences (ending on V) at weaker ones, and occasional deceptive cadences (V-vi) for variety.

At a higher level, the full scan should have macro form. If the image has 40 steps grouped into 5 phrases of 8, those 5 phrases should follow something like A-B-A'-C-A'' (arch form) or a gradual intensification-to-release arc, driven by the global image features. A bright, saturated image gets a more energetic formal trajectory; a dark, desaturated image gets a more contemplative one.

Tests should verify: phrase boundaries produce cadential chord pairs, no two consecutive phrases use identical progressions, harmonic rhythm varies within phrases.

### 1.3 Dynamic Contour and Velocity Shaping

Complexity: MEDIUM. Owner: main.rs (worker_decide_action), midi_output.rs (velocity output). Dependencies: 1.2 (needs phrase boundaries).

Velocity currently maps directly to saturation — a static value per scan-bar section. Musical phrasing demands dynamic shape over time.

Introduce a DynamicContour system that overlays a velocity envelope on each phrase. The contour follows a basic messa di voce shape (crescendo to phrase climax around 60-70% through, then diminuendo to phrase end) with these modifiers: strong beats within the phrase (beats 1 and 3 in a 4-step sub-grouping) get a slight velocity accent (+5-10), phrase-initial notes get a gentle accent for articulative clarity, phrase-final notes taper in velocity to create a sense of breathing. The overall velocity range should map to the image's saturation (low saturation = pp to mp range, high saturation = mf to ff range), but the contour shape applies within that range.

Additionally, introduce subito dynamics: if the image features change dramatically between consecutive scan steps (large hue or brightness delta), insert a sudden dynamic shift — subito piano or subito forte — to create surprise and maintain listener interest.

Tests should verify: velocity is never static across an entire phrase (variance > 0), phrase climax velocity exceeds phrase start/end velocity, strong-beat accents are present.

### 1.4 Melodic Independence and Basic Counterpoint

Complexity: HIGH. Owner: chord_engine.rs, main.rs. Dependencies: 1.1 (voice leading must work first).

Currently all instruments play block chords. The system needs role differentiation: a melody voice with independent contour, inner voices providing harmonic support, and a bass voice with its own idiomatic movement.

Assign roles based on instrument index. Voice 0 is bass (takes chord roots with occasional passing tones between roots — scale-wise walkups/walkdowns). Voice N-1 (highest) is melody. Remaining voices are inner/harmonic.

The melody voice should have a MelodicContourGenerator that produces stepwise motion with occasional leaps (leap followed by step in opposite direction, per Fux's counterpoint principles). The melody selects from chord tones on strong beats and may use passing tones (diatonic steps between chord tones), neighbor tones (step away and back), suspensions (holding a note from the previous chord and resolving down by step on the next beat), and appoggiaturas (leap to a non-chord tone resolving by step) on weak beats. The proportion of non-chord tones should map to image texture complexity — smoother regions produce more consonant melodies, rougher regions introduce more dissonance.

Inner voices follow voice leading rules from 1.1 but can incorporate rhythmic offset: instead of all voices attacking simultaneously (block chord style), inner voices can enter slightly after the beat or sustain across beat boundaries to create rhythmic independence. Edge orientation from local analysis could drive this — horizontal edges map to longer held notes, vertical edges to more rhythmic activity.

Tests should verify: melody voice moves by step at least 70% of the time, leaps are followed by contrary stepwise motion, parallel fifths/octaves between melody and bass are absent, non-chord tones resolve by step.

### 1.5 Articulation and Rhythmic Variety

Complexity: MEDIUM. Owner: midi_output.rs (duration/CC output), main.rs (decision logic), mapping_loader.rs (new mappings). Dependencies: 1.2 (phrase context needed).

Currently notes are either sustained or arpeggiated. MIDI gives us much more expressive range through note duration relative to step duration and control change messages.

Implement an ArticulationEngine that selects from: legato (note duration = 100% of step, slight overlap with next note via early note-on), portato (duration ~80%), normal (duration ~70%), staccato (duration ~40%), marcato (high velocity + ~60% duration). The articulation selection should respond to image contrast (high local contrast = staccato, low = legato, as already sketched in mappings.json under contrast_to_articulation) and vary within phrases (phrase beginnings and endings tend toward legato, active mid-phrase sections can be more detached).

For rhythmic variety, replace the binary arpeggio/sustain decision with a rhythm pattern library. Patterns include: sustained whole-note values, quarter-note pulse, dotted-quarter + eighth patterns, syncopated patterns (eighth-quarter-eighth), eighth-note runs, and rests as musical gesture (a voice dropping out for a beat creates space). Edge density selects the base pattern, but the pattern should vary across phrase repetitions — the same edge density shouldn't always produce the same rhythm.

Add MIDI CC support in midi_output.rs for expression (CC 11) mapped to phrase-level dynamic contour, modulation (CC 1) for vibrato on sustained notes (driven by texture complexity), and sustain pedal (CC 64) at phrase boundaries.

Tests should verify: note durations vary (not all identical), CC messages appear in output, rest events occur in at least some configurations, rhythmic patterns differ between phrases with similar edge density.

---

## Phase 2: Visual-Musical Intelligence

This phase improves how image features translate into musical decisions, making the connection between what you see and what you hear more meaningful and less arbitrary.

### 2.1 Temporal Change Analysis

Complexity: MEDIUM. Owner: image_analysis.rs. Dependencies: Phase 1 complete.

Currently each scan step is analyzed independently. Add delta analysis between consecutive steps: compute the magnitude and direction of change in hue, saturation, brightness, and edge density. Large changes map to musical tension (dissonant intervals, dynamic accents, harmonic surprise); gradual changes map to smooth transitions. This gives the music a sense of responding to the image's visual narrative rather than just its local properties.

### 2.2 Visual Phrase Detection

Complexity: HIGH. Owner: image_analysis.rs. Dependencies: 2.1.

Instead of fixed-length musical phrases, detect natural section boundaries in the image — edges in the scan direction where features change significantly. These visual boundaries become phrase boundaries, so the music's formal structure mirrors the image's spatial structure. A highly varied image produces many short phrases; a gradual gradient produces long, sustained phrases.

### 2.3 Mapping System Overhaul

Complexity: MEDIUM. Owner: mapping_loader.rs, assets/mappings.json. Dependencies: 2.1.

Extend mappings.json to support temporal rules (e.g., "if hue has been stable for 4+ steps and suddenly shifts, trigger a modulation to a new key center"), compound conditions (multiple features combine to select a behavior), and weighted randomization (several possible musical responses to a visual feature, chosen with weighted probability to avoid deterministic output).

### 2.4 Register and Orchestration Intelligence

Complexity: MEDIUM. Owner: chord_engine.rs. Dependencies: Phase 1 complete.

Assign instrument registers and timbres more intelligently. Brightness already maps to register, but the system should ensure voices don't crowd the same octave. Implement orchestral spacing principles: wider intervals in the bass, closer voicing in the upper register, and avoid crossing voices (Voice 2 should not go below Voice 1). Map MIDI program changes to image color temperature — warm hues get warmer timbres (strings, brass), cool hues get cooler ones (woodwinds, vibraphone).

---

## Phase 3: Modem Hardening

The MFSK modem works but needs production-grade robustness.

### 3.1 Adaptive Synchronization

Complexity: HIGH. Owner: modem.rs. Dependencies: none (independent track).

Add a preamble/sync-word system that allows the decoder to lock onto the signal in the presence of noise or partial signal loss. Currently the decoder assumes perfect alignment. Implement chirp-based sync detection and a PLL-style symbol clock recovery.

### 3.2 Channel Quality Estimation

Complexity: MEDIUM. Owner: modem.rs. Dependencies: 3.1.

During decode, estimate per-channel SNR from the preamble or pilot symbols. Use this to weight soft-decision inputs to the Reed-Solomon decoder, improving error correction performance under asymmetric channel conditions.

### 3.3 Streaming Mode

Complexity: MEDIUM. Owner: modem.rs, src/bin/modem_encode.rs. Dependencies: 3.1.

Support streaming encode/decode where data arrives incrementally rather than all at once. This requires chunked framing with independent RS blocks per chunk and a continuous sync maintenance strategy.

### 3.4 Error Reporting and Diagnostics

Complexity: LOW. Owner: modem.rs, src/bin/modem_decode.rs. Dependencies: none.

Surface detailed decode diagnostics: per-channel BER estimates, RS correction counts, frame integrity results, and a confidence score for the overall decode. Output as structured JSON for tooling integration.

---

## Phase 4: Interactive Vision

This phase adds real-time image manipulation and begins the interactive art installation path.

### 4.1 Mouse/Touch Image Smearing

Complexity: MEDIUM. Owner: new module interaction.rs, image_source.rs. Dependencies: Phase 1 complete.

Allow real-time modification of the source image through mouse dragging (smear pixels in drag direction). The modified image feeds back into the scan pipeline, so the user directly sculpts the music by painting over the image. Requires an event loop around the scan pipeline instead of the current one-shot scan.

### 4.2 Camera Feed Integration

Complexity: MEDIUM. Owner: image_source.rs. Dependencies: 4.1 (needs event loop).

Use OpenCV's camera capture to continuously feed frames into the pipeline, creating a live audio-visual instrument. Frame rate and scan rate need decoupling — the scan should complete its current pass before switching to a new frame, or blend between frames smoothly.

### 4.3 Visual Feedback Overlay

Complexity: LOW. Owner: new module display.rs. Dependencies: 4.1.

Render the scan bar position, detected features, and musical decisions as an overlay on the displayed image, so the user can see the connection between visual features and musical output in real time.

---

## Phase 5: Game Integration and Installation

### 5.1 Game Event Protocol

Complexity: HIGH. Owner: new module game_bridge.rs. Dependencies: Phase 4 complete.

Define a protocol (likely a local WebSocket or UDP channel) through which a game engine can send events (player position, health, environment state) that modify the source image or directly influence musical parameters. This makes AudioHax a dynamic game soundtrack engine.

### 5.2 Real-Time Audio Effects

Complexity: HIGH. Owner: new module effects.rs or integration with an audio processing crate. Dependencies: 4.2.

Add real-time audio effects (reverb, delay, filtering) controlled by image features, applied to the FluidSynth output or as MIDI CC automation. This requires either piping FluidSynth audio through a processing chain or switching to a direct audio synthesis approach for tighter control.

### 5.3 Installation Deployment Package

Complexity: MEDIUM. Owner: build tooling, documentation. Dependencies: 4.1, 4.2, 4.3.

Package AudioHax for unattended installation deployment: auto-start configuration, crash recovery, hardware setup documentation (camera, display, touch surface), and a simplified configuration UI for gallery staff.

---

## Recommended Starting Point

Begin with Phase 1 tasks 1.1 (Voice Leading Engine) and 1.2 (Phrase Structure) in parallel, since they touch different primary files (chord_engine.rs vs main.rs orchestration) and have no dependency on each other. Follow immediately with 1.3 (Dynamic Contour), which depends on 1.2's phrase boundaries. These three tasks together will produce the most dramatic improvement in output quality — the music will have smooth voice motion, formal structure with cadences, and dynamic shape instead of flat velocity.

For a Claude Code agent team tackling Phase 1, the recommended split is: a music theory agent owning chord_engine.rs for tasks 1.1 and 1.4, an orchestration agent owning main.rs and the phrase/dynamic systems for tasks 1.2 and 1.3, an output agent owning midi_output.rs and articulation logic for task 1.5, and a quality gate agent that runs the full test/lint/fmt suite and validates that voice leading constraints, phrase structure, and dynamic contour actually hold in the test output. The modem subsystem (Phase 3) can run as an independent track whenever bandwidth allows, since it shares no state with the music pipeline.
